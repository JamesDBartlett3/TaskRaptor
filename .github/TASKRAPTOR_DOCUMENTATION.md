# TaskRaptor - Developer Documentation

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Features](#features)
4. [Data Flow](#data-flow)
5. [Authentication & Security](#authentication--security)
6. [Bug History & Fixes](#bug-history--fixes)
7. [Code Structure](#code-structure)
8. [Testing & Debugging](#testing--debugging)
9. [Development Notes](#development-notes)

---

## Overview

### Purpose
A single-file HTML application that provides a user-centric view of Asana tasks. Unlike Asana's native interface which is project-centric, this application shows all tasks assigned to the authenticated user in a hierarchical view with advanced filtering, caching, and editing capabilities.

### Key Differentiators
- **User-centric** instead of project-centric
- **Hierarchical view** showing task/subtask relationships
- **Filtered subtasks** (only shows subtasks assigned to user or unassigned)
- **24-hour caching** with instant load + background refresh
- **Advanced filtering** by completion status, date ranges, and parent projects
- **WYSIWYG editing** for task notes and comments
- **Works offline** from file:// URLs (no web server required)
- **Single-file deployment** (HTML + CSS + JavaScript)

### Technical Stack
- **Pure HTML/CSS/JavaScript** (no build process)
- **Quill.js 1.3.6** (rich text editor)
- **localStorage** for caching and persistence
- **Asana REST API** (direct browser-to-API communication)
- **No backend required**

---

## Architecture

### File Structure
```
taskraptor.html (single file containing)
├── HTML structure
├── <style> CSS definitions
└── <script> JavaScript application logic
```

### Core Data Structures

```javascript
// Global variables
let personalAccessToken = '';      // Asana PAT for authentication
let currentUserId = '';            // Authenticated user's Asana GID
let workspaceId = null; // Target workspace (fetched from API and stored in localStorage)
let allTasksMap = new Map();       // Map of GID -> Task object (all tasks + subtasks)
let rootItems = [];                // Array of root-level tasks to display
let allParents = new Set();        // Set of JSON strings for parent filter dropdown
let cachedData = null;             // Cache loaded from localStorage

// Filter state (persisted to localStorage)
let filterState = {
    completion: 'uncompleted',     // 'uncompleted' | 'completed' | 'both'
    dateRange: 'last30',           // 'last7' | 'last30' | 'last90' | 'custom' | 'all'
    customStart: null,             // ISO date string
    customEnd: null,               // ISO date string
    parentFilter: ''               // Parent project GID or empty string
};
```

### Task Object Structure
```javascript
{
    gid: '1234567890',              // Asana task GID (unique identifier)
    name: 'Task name',              // Task title
    completed: true/false,          // Completion status
    assignee: { gid: '...', name: '...' }, // Or null if unassigned
    due_on: '2025-12-31',          // ISO date string or null
    modified_at: '2025-12-01T10:30:00Z', // ISO datetime
    parent: { gid: '...', name: '...' }, // Or null if no parent
    notes: 'Task description',      // Rich text (HTML)
    subtasks: [...],                // Array of subtask objects
    subtasksLoaded: true/false,     // Flag to prevent redundant API calls
    comments: [...] | null          // Array of comment objects or null if not loaded
}
```

### Cache Structure
```javascript
// Stored in localStorage as 'taskraptor_cache'
{
    timestamp: 1733364000000,       // Date.now() when cached
    data: {
        allTasksMap: [[gid, task], ...], // Array of entries (for Map reconstruction)
        rootItems: [...],           // Array of root task objects
        allParents: [...]           // Array of parent JSON strings
    }
}
```

---

## Features

### 1. Authentication
- **Personal Access Token (PAT)** authentication
- PAT stored in localStorage (key: 'taskraptor_pat')
- Auto-login if PAT exists
- Logout clears PAT and cache

### 2. Task Loading with Pagination
- Fetches ALL tasks assigned to user across all pages
- Uses `offset` parameter for pagination
- API default limit is 100 tasks per page
- Continues until `next_page.offset` is null
- Progress bar shows fetch progress (0-30%)

### 3. 24-Hour Caching System
- **Instant load**: Displays cached data immediately on page load
- **Background refresh**: Triggers fresh fetch 1 second after displaying cache
- **Cache validation**: Invalidates cache after 24 hours
- **Cache key**: 'taskraptor_cache' in localStorage
- **Manual refresh**: Button to force reload from API

### 4. Hierarchical Task Display
- **Root Detection**: Task is shown at root level ONLY if:
  - It has no parent, OR
  - Its parent is NOT in allTasksMap (parent not in user's task set)
- **Subtask Loading**: Recursively loads subtasks for all tasks
- **Filtering**: Only shows subtasks that are:
  - Assigned to current user, OR
  - Unassigned (and parent is assigned to user)
- **Visual Hierarchy**: Indentation (40px per level) shows depth

### 5. Expand/Collapse Functionality
- **Chevrons**: ▶ (collapsed) / ▼ (expanded)
- **Chevron display**: Only shown if task has subtasks
- **Instant expansion**: Subtasks render from memory (pre-loaded)
- **Click on name**: Expands to show notes/comments (different from subtasks)
- **Click on chevron**: Toggles subtask visibility

### 6. Advanced Filtering

#### Completion Status Filter
- **Uncompleted only** (default)
- **Completed only**
- **Both**
- Applies recursively (if parent matches filter, all children shown)

#### Date Range Filter
- **Last 7 days**
- **Last 30 days** (default)
- **Last 90 days**
- **Custom date range** (start/end pickers)
- **All time**
- **Important**: Date filter ONLY applies to completed tasks
- Uncompleted tasks always show regardless of date

#### API-Level Date Filtering
- Uses `completed_since` parameter when fetching completed tasks
- Reduces payload size by filtering at API level
- Only fetches completed tasks within date range

#### Parent Project Filter
- Dropdown populated from breadcrumb parent projects
- Shows only tasks under selected parent
- Clickable breadcrumb links also filter by parent
- "All Projects" option clears filter

### 7. Sorting Options
- **Sort by Name** (alphabetical)
- **Sort by Due Date** (chronological, with nulls last)
- **Sort by Completion Status** (incomplete first)
- **Sort by Last Modified** (most recent first)

### 8. Filter Badges
- Visual indicators showing active filters
- Always shows completion status
- Shows date range if not "all time"
- Shows parent filter if selected
- Example: `Uncompleted | Last 90 days | Engineering Team`

### 9. Breadcrumb Trails
- Shows full parent chain for root tasks
- Format: "Parent1 → Parent2 → Parent3"
- Clickable links filter by that parent
- Built using `getParentChain()` API calls
- Only fetches parents outside current task set

### 10. Task Editing (WYSIWYG)
- **Quill.js** rich text editor
- Edit task notes (description)
- Add new comments
- URL linkification (converts URLs to clickable links)
- Filtered comments (only shows user's own comments + system stories)
- Save updates back to Asana via API

### 11. Task Completion Toggle
- Checkbox click marks task complete/incomplete
- Updates Asana via API
- Visual strikethrough for completed tasks
- Updates cache immediately

### 12. Progress Tracking
- Multi-stage progress bar with messages
- Stages:
  - 0-30%: Fetching tasks (pagination)
  - 30-40%: Processing tasks
  - 40-70%: Loading subtasks
  - 70-90%: Building hierarchy
  - 90-100%: Finalizing
- Progress bar hidden after completion

### 13. Toast Notifications
- Success, error, info, warning types
- Auto-dismiss after 5 seconds (default)
- Manual close button
- Shows for all operations (save, delete, refresh, etc.)

### 14. Task Statistics
- Shows in summary line:
  - Number of root items displayed
  - Total tasks (including subtasks)
  - Completed count
  - Remaining count
  - "(from cache)" indicator if loaded from cache
- Debug info also shown

---

## Data Flow

### Initial Page Load
```
1. Load PAT from localStorage
2. If PAT exists:
   a. Authenticate with Asana API
   b. Get current user info
   c. Load cached data if available
      - Display cached data immediately
      - Wait 1 second
      - Trigger background refresh
   d. If no cache, show progress and load fresh
3. If no PAT:
   - Show login form
```

### Fresh Data Load (Full Refresh)
```
1. Clear allTasksMap, rootItems, allParents
2. Fetch assigned tasks (with pagination):
   - Loop until offset is null
   - Progress: 0-30%
3. Filter by date (if needed)
4. Add all filtered tasks to allTasksMap
5. Load subtasks recursively for each task:
   - Check subtasksLoaded flag
   - If not loaded, fetch from API
   - Add subtasks to allTasksMap
   - Recurse for each subtask
   - Progress: 40-70%
6. Build hierarchy:
   - Iterate through ALL tasks in allTasksMap
   - Identify root tasks (no parent OR parent not in map)
   - Build parent chains for breadcrumbs
   - Populate parent filter options
   - Progress: 70-90%
7. Remove duplicates
8. Save to cache
9. Display tasks
10. Hide progress bar
```

### Background Refresh
```
1. Same as Fresh Data Load, but:
   - Don't clear cache initially
   - Don't show progress bar
   - Display happens silently
   - Toast notification on completion
```

### Task Expansion (Chevron Click)
```
1. User clicks chevron (▶)
2. Check if subtasks already rendered
3. If not rendered:
   - Get task from allTasksMap
   - Render each subtask (already loaded in memory)
   - Append to details div
4. Toggle chevron (▶ ↔ ▼)
5. Show/hide subtask elements
```

### Task Detail Expansion (Name Click)
```
1. User clicks task name
2. Check if comments already loaded
3. If not loaded:
   - Fetch comments from API
   - Filter comments (user's + system stories)
4. Render task details:
   - Task notes (WYSIWYG editor)
   - Comments section
   - Task metadata
5. Toggle details visibility
```

### Task Completion Toggle
```
1. User clicks checkbox
2. Update task.completed in allTasksMap
3. Send PUT request to Asana API
4. Update checkbox state
5. Toggle strikethrough styling
6. Show toast notification
7. Update task statistics
```

### Filter Application
```
1. User changes filter (completion/date/parent)
2. Save filter state to localStorage
3. Rebuild display:
   - Get all tasks from allTasksMap
   - Apply completion filter (recursive)
   - Apply date filter (completed tasks only)
   - Apply parent filter (if set)
   - Sort tasks
   - Render to DOM
4. Update filter badges
5. Update task statistics
```

---

## Authentication & Security

### Personal Access Token (PAT)
- **Storage**: localStorage (key: 'taskraptor_pat')
- **Not encrypted**: User should understand this is stored in plaintext
- **Auto-login**: Token persists across page reloads
- **Logout**: Clears token from storage

### API Calls
- **Direct browser-to-API**: No proxy server
- **CORS**: Asana API allows CORS from file:// and https:// origins
- **Rate Limits**: Standard Asana API limits apply (150 requests/minute)
- **No server-side secrets**: PAT is only secret, managed by user

### CORS & File URLs
- **Works from file:// URLs**: localStorage works, Asana API allows CORS
- **Cannot preview in Claude's browser**: Authentication doesn't work in iframe
- **User workflow**:
  1. Download HTML file to Windows machine
  2. Open in browser (Chrome, Edge, Firefox)
  3. Enter PAT to authenticate
  4. When updates made: Re-download, overwrite, refresh browser

---

## Bug History & Fixes

### Session 1: Initial Development
Created basic milestone-based editor with task viewing.

### Session 2: Pagination Issue
**Problem**: API returned exactly 100 tasks (suspiciously round number)
**Root Cause**: Asana API default page limit is 100, wasn't paginating
**Solution**: Implemented pagination loop using `offset` parameter
```javascript
do {
    const url = offset ? `${apiUrl}&offset=${offset}` : apiUrl;
    const data = await fetch(url);
    offset = data.next_page ? data.next_page.offset : null;
} while (offset);
```

### Session 3: Performance & Caching
**Problem**: Slow load times on every page refresh
**Solution**: Implemented 24-hour cache with instant load + background refresh
- Cache stored in localStorage with timestamp
- `loadCache()` checks age, invalidates after 24 hours
- `displayCachedData()` loads instantly, triggers background refresh after 1 second

### Session 4: Advanced Filtering
**Problem**: No way to filter tasks by date or completion status
**Solution**: Added modal dialog with three filter types:
- Completion status (uncompleted/completed/both)
- Date ranges (last 7/30/90 days, custom, all)
- Parent project filter
- API-level filtering using `completed_since` parameter

### Session 5: Critical Bug - No Tasks Displayed
**Problem**: "Root items before filtering: 0" despite "Fetched 199 tasks"
**Root Cause**: `getParentChain()` → `getTaskDetails()` added tasks to `allTasksMap` as side effect. By the time code checked `if (!allTasksMap.has(rootTask.gid))`, task was already there.
**Solution**: Separate `rootGids` Set to track which tasks added as roots
```javascript
const rootGids = new Set();
if (!rootGids.has(rootTask.gid)) {
    rootItems.push(rootTask);
    rootGids.add(rootTask.gid);
}
```

### Session 6: Date Filter Bug
**Problem**: Uncompleted tasks not modified in 30 days were filtered out
**Root Cause**: `filterTasksByDate()` checked `modified_at` for all tasks
**Solution**: Date filtering only applies to completed tasks
```javascript
function filterTasksByDate(tasks) {
    if (filterState.completion === 'uncompleted') {
        return tasks; // No date filter for uncompleted
    }
    // Date filter only for completed or 'both' mode
}
```

### Session 7: Completion Filter Not Applied
**Problem**: Filter set to "uncompleted" but showed nothing
**Root Cause**: `displayTasks()` never called `filterTasksByCompletion()`
**Solution**: Added completion filtering step with recursive filtering that includes parent tasks if they have matching subtasks

### Session 8: Root-Level Tasks Ignored
**Problem**: Tasks with no parent (already at root level) were skipped
**Root Cause**: `if (parentChain.length > 0) { /* only add if has parents */ }`
**Solution**: Handle both cases:
```javascript
let rootTask;
if (parentChain.length > 0) {
    rootTask = parentChain[0]; // Use top-most parent
} else {
    rootTask = task; // Task IS the root
}
```

### Session 9: Duplicate Tasks
**Problem**: Same task appeared multiple times in list
**Solution**: Three-layer duplicate prevention:
1. During building: `rootGids` Set prevents duplicate additions
2. After building: Deduplication pass removes any that slipped through
3. Cache loading: Deduplicates cached data before display

### Session 10: Progress Bar Jumping
**Problem**: Progress percentage jumped between different values
**Solution**: Single `currentProgress` variable, only moves forward, staged updates

### Session 11: Redundant API Calls
**Problem**: Same task fetched multiple times
**Solution**: `getTaskDetails()` returns immediately from `allTasksMap` if already fetched

### Session 12: Missing Breadcrumbs
**Problem**: Breadcrumb trails not displaying
**Solution**: Fixed `renderTask()` to build breadcrumb from parent chain with infinite loop prevention using visited Set

### Session 13: CURRENT SESSION - Hierarchy & Chevron Issues

#### Bug 1: Broken Hierarchy
**Problem**: Subtasks showing at root level instead of nested under parents
**Symptoms**: Tasks outlined in red were supposed to be sub-items, not root items
**Root Cause**: Code was building hierarchy BEFORE loading subtasks, and checking wrong set of tasks (`assignedGids` only contained directly-assigned tasks, not their subtasks)
**Solution**: Reordered flow - load ALL subtasks FIRST, then build hierarchy
```javascript
// OLD FLOW (BROKEN):
1. Fetch assigned tasks
2. Filter tasks  
3. Build hierarchy (allTasksMap is EMPTY!)
4. Load subtasks

// NEW FLOW (FIXED):
1. Fetch assigned tasks
2. Filter tasks
3. Add all to allTasksMap
4. Load ALL subtasks recursively (populates allTasksMap)
5. Build hierarchy from allTasksMap (now contains everything!)
```
Changed detection logic:
```javascript
// OLD: Checked if parent was directly assigned to user
const isRootItem = !task.parent || !assignedGids.has(task.parent.gid);

// NEW: Checks if parent is in allTasksMap (includes all subtasks)
const isRootItem = !task.parent || !allTasksMap.has(task.parent.gid);
```

#### Bug 2: Missing Chevrons
**Problem**: Tasks with subtasks didn't show expand/collapse chevrons
**Root Cause**: Chevron detection (`hasSubtasks = task.subtasks && task.subtasks.length > 0`) ran before subtasks were loaded
**Solution**: Now loads ALL subtasks during initial load, THEN detects which tasks have subtasks

#### Bug 3: Chevron Doesn't Rotate
**Problem**: Chevron stayed as ▶ even when expanded
**Root Cause**: Logic was checking `detailsDiv.style.display !== 'none'` instead of current icon text
**Solution**: Fixed `toggleSubtasks` to properly toggle based on icon text
```javascript
const isExpanded = icon.textContent === '▼';
if (isExpanded) {
    icon.textContent = '▶';
    // hide subtasks
} else {
    icon.textContent = '▼';
    // show subtasks
}
```

#### Bug 4: Slow Subtask Loading
**Problem**: Subtasks took long time to display, no loading indicator
**Root Cause**: Making redundant API calls on every expand
**Solution**: Added `subtasksLoaded` flag to prevent re-fetching
```javascript
async function loadSubtasksFiltered(taskGid, parentAssignedToUser) {
    const task = allTasksMap.get(taskGid);
    if (!task) return;
    
    // If subtasks already loaded, skip
    if (task.subtasksLoaded) {
        return;
    }
    // ... fetch and load
    task.subtasksLoaded = true;
}
```
Now subtasks render instantly from memory (already loaded during initial load)

---

## Code Structure

### HTML Structure
```html
<div class="header">
    <!-- Title and subtitle -->
</div>

<div class="controls">
    <button id="btnFilters">🔍 Filters</button>
    <select id="sortBy"><!-- Sort options --></select>
    <div class="filter-badges"><!-- Active filter indicators --></div>
    <button id="btnRefresh">🔄 Refresh</button>
    <button id="btnLogout">🚪 Logout</button>
</div>

<div id="progressSection">
    <!-- Progress bar and message -->
</div>

<div id="content">
    <div id="summary"><!-- Task statistics --></div>
    <div id="taskList"><!-- Rendered tasks go here --></div>
</div>

<!-- Modals -->
<div id="filterModal"><!-- Filter dialog --></div>
<div id="authModal"><!-- Login form --></div>
<div id="editModal"><!-- Task edit form with Quill editor --></div>
```

### Key Functions

#### Authentication
- `authenticate()` - Login with PAT
- `authenticateAndLoad()` - Verify PAT and load user info
- `logout()` - Clear PAT and cache

#### Data Loading
- `loadUserTasks(background)` - Main data loading function
- `loadSubtasksFiltered(taskGid, parentAssignedToUser)` - Recursive subtask loading
- `getParentChain(taskGid)` - Get breadcrumb trail
- `getTaskDetails(taskGid)` - Fetch single task from API

#### Caching
- `saveCache(data)` - Save to localStorage with timestamp
- `loadCache()` - Load from localStorage, check age
- `displayCachedData()` - Display cached data immediately
- `clearCache()` - Remove cache entry

#### Filtering
- `filterTasksByCompletion(tasks)` - Filter by completion status (recursive)
- `filterTasksByDate(tasks)` - Filter by date range (completed tasks only)
- `getDateFilter()` - Convert date range setting to ISO string
- `applyParentFilter(tasks)` - Filter by parent project

#### Hierarchy Building
- `buildHierarchy()` - Determine root vs nested tasks
- `detectRootTasks()` - Identify tasks to show at root level

#### Display
- `displayTasks()` - Main display function (filter, sort, render)
- `renderTask(task, level, isRoot)` - Render single task HTML
- `renderTaskDetails(taskGid)` - Render task notes and comments
- `toggleSubtasks(event, taskGid)` - Expand/collapse subtasks
- `toggleTaskExpand(event, taskGid)` - Expand/collapse task details

#### Editing
- `editTaskNotes(event, taskGid)` - Open edit modal
- `saveTaskEdit(taskGid)` - Save changes to Asana
- `addComment(taskGid)` - Add new comment
- `loadComments(taskGid)` - Fetch task comments

#### Task Operations
- `toggleTaskCompletion(event, taskGid)` - Mark complete/incomplete
- `sortTasks(tasks)` - Sort tasks based on current sort setting

#### UI Helpers
- `showToast(message, type, duration)` - Show notification
- `showProgress()` / `hideProgress()` - Toggle progress bar
- `updateProgress(current, total, message)` - Update progress bar
- `displayFilterBadges()` - Show active filter indicators
- `populateParentFilter()` - Populate parent dropdown

#### Utilities
- `escapeHtml(text)` - Prevent XSS
- `linkifyText(text)` - Convert URLs to links
- `savePAT(token)` / `loadPAT()` - PAT storage helpers
- `saveFilterState()` / `loadFilterState()` - Filter persistence

### Critical Global State
```javascript
let allTasksMap = new Map();  // MUST contain ALL tasks + subtasks before building hierarchy
let rootItems = [];           // Only populated after hierarchy building
let filterState = {...};      // Persisted to localStorage
let cachedData = null;        // Loaded once on page load
```

---

## Testing & Debugging

### Console Logging
The app has extensive console logging for debugging:

```javascript
// Sample debug output during load:
console.log(`Fetched ${allTasks.length} tasks`);
console.log(`After date filter: ${filteredTasks.length} tasks`);
console.log(`Building hierarchy from ${allTasksArray.length} total tasks`);
console.log(`Task 0: "Write Intro Message" - Has parent: yes (123456)`);
console.log(`  Parent in allTasksMap: true`);
console.log(`  Is root item: false`);
console.log(`✓ Added root #1: "Project Opening" (789012) with 5 subtasks`);
console.log(`Total root items found: 9`);
console.log(`rootGids size: 9`);
console.log(`allTasksMap size: 124`);
```

### Debug Info in UI
Summary line includes debug section:
```
Debug: Completion filter = "uncompleted" | Date filter = "last90" | 
Root items = 201 | After filters = 18
```

### Common Issues to Check

1. **No tasks displaying**
   - Check console: "Root items before filtering: X"
   - If X = 0: Hierarchy building is broken
   - Check allTasksMap size vs filteredTasks length

2. **Duplicate tasks**
   - Check console: "Found X duplicate root items"
   - Verify rootGids Set is being used properly
   - Check for duplicate GIDs in rootItems array

3. **Wrong hierarchy**
   - Log first 5 tasks with parent info
   - Verify allTasksMap is populated BEFORE hierarchy building
   - Check isRootItem logic

4. **Slow loading**
   - Check for redundant API calls
   - Verify subtasksLoaded flag is working
   - Check cache hit/miss rate

5. **Filter not working**
   - Check filterState in console
   - Verify filter is being applied in displayTasks()
   - Check recursive filter logic for parent/child relationships

### Testing Checklist

- [ ] Fresh load (no cache): All tasks appear correctly
- [ ] Cached load: Instant display, background refresh works
- [ ] Filter by completion: Shows correct tasks
- [ ] Filter by date: Only affects completed tasks
- [ ] Filter by parent: Shows correct subset
- [ ] Sort: All sort options work correctly
- [ ] Expand/collapse: Chevrons rotate, subtasks show/hide
- [ ] Task completion: Checkbox updates Asana
- [ ] Task editing: Notes save correctly
- [ ] Comments: Can add and view comments
- [ ] Breadcrumbs: Show correct parent chain
- [ ] Pagination: Fetches all tasks (>100)
- [ ] Logout: Clears cache and PAT

---

## Development Notes

### Why Single File?
- **No build process**: Edit and refresh
- **Easy deployment**: Just download and open
- **No dependencies**: Everything self-contained except Quill CDN
- **Works offline**: After initial Quill load

### localStorage vs Cookies
- **localStorage chosen** because it works with file:// URLs
- **Cookies don't work** from file:// URLs
- localStorage limit: 5-10MB (plenty for task cache)

### Why No Backend?
- **Asana API allows CORS**: Direct browser-to-API works
- **PAT auth**: No server-side secrets needed
- **Simplifies deployment**: Just an HTML file
- **User owns their data**: PAT stays in their browser

### Performance Considerations
- **Caching critical**: Reduces API calls by 99%
- **Pagination essential**: Some users have 200+ tasks
- **Recursive loading**: Can be slow but necessary for complete hierarchy
- **subtasksLoaded flag**: Prevents exponential API calls

### Future Enhancement Ideas
- [ ] Bulk task operations (mark multiple complete)
- [ ] Task creation from UI
- [ ] Project/tag assignment
- [ ] Custom fields display/editing
- [ ] Export to CSV/JSON
- [ ] Dark mode toggle
- [ ] Keyboard shortcuts
- [ ] Multi-workspace support
- [ ] Search within tasks
- [ ] Task templates

### Known Limitations
- **PAT required**: Users must have Asana PAT
- **Single or multiple workspaces**: Workspace ID fetched from API and stored in localStorage
- **No real-time sync**: Must refresh to see changes from Asana
- **No offline editing**: Requires internet for API calls
- **Limited custom fields**: Only shows basic task properties

### API Endpoints Used
```
GET  /users/me                        - Get current user
GET  /workspaces/{id}/tasks/search    - Fetch assigned tasks
GET  /tasks/{id}/subtasks             - Fetch subtasks
GET  /tasks/{id}                      - Get task details
GET  /tasks/{id}/stories              - Get comments/stories
PUT  /tasks/{id}                      - Update task
POST /tasks/{id}/stories              - Add comment
```

### Important API Fields
```
opt_fields=name,completed,assignee,due_on,modified_at,parent,notes
```
These are the minimal fields needed for the app to function.

---

## Workspace Configuration

### Current Workspace
- **Workspace ID**: Dynamically fetched from Asana API
- **Workspace Name**: (varies by user)
- **Location**: Hardcoded in `loadUserTasks()` function

### Milestone Context
The app was originally created to manage tasks under the milestone:
- **Milestone Name**: "James Bartlett's AI Vanguard IDP"
- **Milestone GID**: `1212274499183804`
- However, the current version is user-centric, not milestone-centric

---

## Version History

### v1.0 - Basic Milestone Editor
- Hardcoded milestone ID
- Simple task list
- Basic editing

### v2.0 - WYSIWYG Editing
- Added Quill.js
- Recursive subtask loading
- URL linkification

### v3.0 - Milestone Selection
- Removed hardcoded IDs
- Added milestone picker
- Filtered comments

### v4.0 - User-Centric View (Major Restructure)
- Changed from milestone-centric to user-centric
- localStorage instead of cookies
- Toast notifications
- Advanced filtering

### v4.1 - Pagination & Performance
- API pagination support
- 24-hour caching
- Background refresh
- Progress tracking

### v4.2 - Hierarchy Fixes (Current)
- Fixed root detection logic
- Fixed chevron display/rotation
- Fixed slow subtask loading
- Added subtasksLoaded flag
- Reorganized data flow (load subtasks before building hierarchy)

---

## File Location & Access

### In Claude's Environment
- **Path**: `/mnt/user-data/outputs/taskraptor.html`
- **Not accessible** to user directly
- **Must download** to use

### User Workflow
1. **Download**: File download link from Claude
2. **Save**: To local Windows machine
3. **Open**: In browser (Chrome, Edge, Firefox)
4. **Authenticate**: Enter Asana PAT
5. **Update cycle**: Download → Overwrite → Refresh browser

### Development Workflow (Moving to VS Code)
1. **Copy file** to VS Code workspace
2. **Edit** in VS Code
3. **Test** by opening in browser
4. **Iterate**: Edit → Save → Refresh browser

---

## Critical Code Patterns

### Preventing Infinite Loops
```javascript
// Always use visited Set when traversing parent chains
const visited = new Set();
while (currentGid && !visited.has(currentGid)) {
    visited.add(currentGid);
    // traverse...
}
```

### Safe Task Lookup
```javascript
// Always check if task exists before using
const task = allTasksMap.get(taskGid);
if (!task) return; // Early exit if not found
```

### Preventing Duplicates
```javascript
// Use Set to track already-processed items
const processed = new Set();
items.forEach(item => {
    if (processed.has(item.gid)) return;
    processed.add(item.gid);
    // process item...
});
```

### API Error Handling
```javascript
try {
    const response = await fetch(url, { headers: {...} });
    if (!response.ok) throw new Error('API error');
    const data = await response.json();
    // process data...
} catch (error) {
    showToast('Error: ' + error.message, 'error');
    console.error(error);
}
```

---

## Questions to Ask When Debugging

1. **Is allTasksMap populated before building hierarchy?**
   - Check console: `allTasksMap size: X`
   - Should be > 0 before "Building hierarchy" message

2. **Are subtasks loaded before checking hasSubtasks?**
   - Check: Does task.subtasksLoaded = true?
   - Are subtasks in task.subtasks array?

3. **Is the root detection logic correct?**
   - For each task, is parent.gid in allTasksMap?
   - If yes, task should NOT be root
   - If no, task SHOULD be root

4. **Are filters being applied in the right order?**
   - Completion filter first
   - Then date filter (only for completed)
   - Then parent filter
   - Then sort

5. **Is the cache valid?**
   - Check timestamp: Is it < 24 hours old?
   - Does cached data have all required fields?
   - Are duplicates removed from cache?

---

## Contact & Support

### Original Context
This app was developed through an extended conversation with Claude (Anthropic).
Full transcript available at: `/mnt/transcripts/2025-12-04-21-31-02-asana-task-viewer-pagination-caching-filtering.txt`

### User Information
- **User**: Experienced Python, SQL, PowerShell developer
- **Skill Level**: Advanced (skip basic troubleshooting)
- **Location**: Des Moines, Iowa, US
- **Time Zone**: (Infer from location)

### Development Environment
- **Primary**: VS Code (transitioning to this)
- **Previous**: Claude.ai web interface
- **Testing**: Local browser (Chrome/Edge/Firefox)

---

## Quick Reference

### Essential Variables
```javascript
WORKSPACE_ID = null // Fetched during authentication
filterState.completion = 'uncompleted' | 'completed' | 'both'
filterState.dateRange = 'last7' | 'last30' | 'last90' | 'custom' | 'all'
```

### Essential Functions to Know
- `loadUserTasks()` - Main data loading
- `displayTasks()` - Main display
- `renderTask()` - Single task render
- `toggleSubtasks()` - Expand/collapse subtasks
- `loadSubtasksFiltered()` - Recursive subtask loading

### Key localStorage Keys
- `taskraptor_pat` - Personal Access Token
- `taskraptor_cache` - Task cache (24-hour TTL)
- `taskraptor_filters` - Filter preferences
- `taskraptor_expand_states` - Task expansion state

### Important Flags
- `task.subtasksLoaded` - Prevents redundant API calls
- `task.completed` - Completion status
- `background` parameter - Silent refresh vs user-initiated

---

## End of Documentation

This document provides a comprehensive overview of the TaskRaptor application. It should enable any developer (including future Claude instances) to understand the app's architecture, features, bug history, and development context.

**Last Updated**: 2025-12-04  
**Version**: 4.2  
**Status**: Production-ready with recent hierarchy fixes

For questions or issues, refer to the code comments and console logging within the HTML file itself.
