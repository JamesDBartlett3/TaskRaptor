# Copilot Instructions - TaskRaptor Technical Documentation

## Document Purpose

This document is designed for:
- **AI Assistants** (like Claude) to understand the codebase structure and make informed modifications
- **Developers** who need to maintain or extend this application
- **Code Reviewers** who need to understand architectural decisions and implementation details

## Table of Contents

1. [Application Overview](#application-overview)
1. [Target Audience & Use Cases](#target-audience--use-cases)
1. [Architecture & Design Philosophy](#architecture--design-philosophy)
1. [Core Data Structures](#core-data-structures)
1. [Application Lifecycle](#application-lifecycle)
1. [Key Systems Explained](#key-systems-explained)
1. [Function Reference](#function-reference)
1. [State Management](#state-management)
1. [API Integration Patterns](#api-integration-patterns)
1. [Performance Optimizations](#performance-optimizations)
1. [Known Limitations & Edge Cases](#known-limitations--edge-cases)
1. [Development Guidelines](#development-guidelines)
1. [Anti-Patterns to Avoid](#anti-patterns-to-avoid)

---

## Application Overview

### What This Is

A single-file HTML application that provides a **user-centric** view of Asana tasks, contrasting with Asana's native **project-centric** interface. The application runs entirely in the browser with no backend server required.

### Key Differentiators from Asana Native UI

| Feature | This App's Unique Advantage |
|---------|----------------------------|
| **Unified Hierarchical View** | Display multiple task hierarchies simultaneously with full parent/child relationships visible across different projects. See all your assigned tasks and their subtasks in one expandable tree view, regardless of which projects they belong to. Asana's My Tasks shows flat lists - you must navigate into each task individually to see its subtasks. |
| **Filtered Subtask View** | Only shows tasks assigned to you or unassigned descendants, creating a cleaner personal view without subtasks assigned to teammates |
| **Instant Load with Background Refresh** | 24-hour cache provides instant page loads (<100ms), then updates in the background. |
| **Single-File Deployment** | Zero installation - download one HTML file and open in browser. Works from `file://` URLs without a web server. No account setup, no deployment pipeline. |

### Technical Stack

```
Single File Architecture:
├── HTML (structure)
├── CSS (styling - embedded <style> tag)
└── JavaScript (logic - embedded <script> tag)

External Dependencies:
└── Quill.js 1.3.6 (rich text editor via CDN)

Browser APIs Used:
├── localStorage (caching, authentication, preferences)
├── Fetch API (Asana REST API communication)
└── DOM Parser (HTML content processing)
```

---

## Target Audience & Use Cases

### Primary Users

**Individual Contributors** who:
- Find Asana's interface cluttered or overwhelming
- Work on tasks across multiple projects
- Need a clean, simple view of only *their* assigned tasks
- Want faster load times with instant page loads
- Prefer hierarchical task views over project-based views

### Common Use Cases

1. **Daily Task Review**: Quickly see all incomplete tasks assigned to you with instant load from cache
2. **Project Navigation**: Use breadcrumb trails to understand task context
   > 📝 TODO: Breadcrumbs will move below title text, and clickable area will extend to include breadcrumb trail
3. **Focused Work**: Filter by completion status and date ranges
4. **Quick Updates**: Inline editing of notes, comments, and task completion
   > 📝 TODO: Marking task complete will prompt modal if it has subtasks, asking to also complete all subtasks
5. **Cross-Project Hierarchy**: View subtasks from multiple projects in a unified tree view
6. **Parent Project Filtering**: Focus on tasks within a specific project

### NOT Designed For

- ❌ Team management or multi-user views
- ❌ Project planning or timeline views
- ❌ Creating new projects or bulk operations
- ❌ Complex task dependencies or Gantt charts

---

## Architecture & Design Philosophy

### Single-File Philosophy

**Why Single File?**
- **Zero deployment complexity**: Save to disk, open in browser
- **No build process**: Direct edit and refresh workflow
- **Version control friendly**: Entire app in one file to track
- **User simplicity**: Non-technical users can update by replacing one file

**Trade-offs Accepted:**
- Harder to navigate large file (3200+ lines)
- Cannot use modern JS modules
- Limited code organization compared to multi-file projects

### Client-Side Only Architecture

```
┌─────────────┐
│   Browser   │
│             │
│  ┌───────┐  │      HTTPS        ┌──────────────┐
│  │ HTML  │  │  ◄──────────────► │  Asana API   │
│  │ CSS   │  │                   │ (app.asana.  │
│  │ JS    │  │                   │  com/api)    │
│  └───┬───┘  │                   └──────────────┘
│      │      │
│  ┌───▼────┐ │
│  │localStorage│
│  │  Cache  │ │
│  │  PAT    │ │
│  │Filters  │ │
│  └────────┘ │
└─────────────┘
```

**No Backend Means:**
- ✅ No server costs
- ✅ No deployment pipeline
- ✅ Works from `file://` URLs
- ❌ PAT stored in plaintext in localStorage (user must understand security)
- ❌ Subject to browser localStorage limits (~10MB)
- ❌ CORS requirements (Asana API supports this)

### State Management Pattern

**Global State Pattern** (not React/Vue style):
```javascript
// All state lives in global variables
let allTasksMap = new Map();    // Single source of truth for all tasks
let rootItems = [];              // Array of root-level display items
let filterState = {...};         // User preferences
let expandStates = {...};        // UI state persistence
```

**State Persistence:**
- `localStorage` used as persistence layer
- State saved immediately on user actions
- No "dirty" flags or batch updates

---

## Core Data Structures

### 1. Global Variables (State Containers)

```javascript
// Authentication & User Context
let personalAccessToken = '';    // Asana PAT - stored in localStorage
let currentUserId = '';          // Authenticated user's Asana GID
let WORKSPACE_ID = null;         // Workspace GID - fetched from API and stored in localStorage

// Core Data Store
let allTasksMap = new Map();     // Map<string, Task> - GID → Task object
                                 // Contains ALL tasks including subtasks
                                 // This is the SINGLE SOURCE OF TRUTH

let rootItems = [];              // Array<Task> - Root-level tasks to display
                                 // Only populated AFTER hierarchy building

let allParents = new Set();      // Set<string> - JSON strings of parent objects
                                 // Used to populate parent filter dropdown

// Editor State
let activeEditor = null;         // Quill editor instance for notes
let commentEditors = new Map();  // Map of task GIDs to comment editors

// Cache
let cachedData = null;           // Loaded once from localStorage on startup

// UI State
let expandStates = {};           // Object<string, boolean> - taskGid → isExpanded
const tasksBeingLoaded = new Set(); // Set<string> - GIDs currently loading

// Filter State (persisted to localStorage)
let filterState = {
    completion: 'uncompleted',   // 'uncompleted' | 'completed' | 'both'
    dateRange: 'last30',         // 'last7' | 'last30' | 'last90' | 'custom' | 'all'
    customStart: null,           // ISO date string or null
    customEnd: null,             // ISO date string or null
    parentFilter: ''             // Parent GID or empty string
};
```

### 2. Task Object Structure

```javascript
// Complete Task Object Shape
{
    // Asana API Fields
    gid: '1234567890',              // Unique identifier (string)
    name: 'Task name',              // Display name
    completed: false,               // Boolean completion status
    assignee: {                     // Assignee object or null
        gid: '0987654321',
        name: 'John Doe'
    },
    due_on: '2025-12-31',          // ISO date string or null
    modified_at: '2025-12-05T10:30:00.000Z', // ISO timestamp
    parent: {                       // Parent reference or null
        gid: '1111111111',
        name: 'Parent Task'
    },
    notes: 'Plain text notes',     // Plain text version
    html_notes: '<p>HTML notes</p>', // HTML formatted version
    memberships: [{                 // Project/section memberships
        project: {
            gid: '2222222222',
            name: 'Project Name'
        },
        section: {
            gid: '3333333333',
            name: 'Section Name'
        }
    }],
    
    // Application-Added Fields
    subtasks: [],                   // Array<Task> - Child tasks
    comments: null | [],            // null = not loaded, [] = loaded
    subtasksLoaded: false,          // Boolean flag to prevent re-fetching
    commentsShown: 3,               // Number of comments to display
    
    // Special Item Flags (for placeholder items)
    isProject: false,               // True if this is a project placeholder
    isSection: false,               // True if this is a section placeholder
    isPlaceholder: false,           // True if created as missing parent
    
    // UI Enhancement Fields
    collapsedParentChain: [{        // Array of parent objects (for breadcrumb)
        gid: '4444444444',
        name: 'Collapsed Parent'
    }]
}
```

### 3. Cache Object Structure

```javascript
// Stored in localStorage as 'asana_cache'
{
    timestamp: 1733364000000,       // Date.now() when cached
    data: {
        allTasksMap: [              // Array of [key, value] pairs
            ['gid1', taskObject1],
            ['gid2', taskObject2],
            // ... all tasks
        ],
        rootItems: [                // Array of Task objects
            taskObject1,
            taskObject3,
            // ... root-level tasks only
        ],
        allParents: [               // Array of JSON strings
            '{"gid":"xxx","name":"Parent 1"}',
            '{"gid":"yyy","name":"Parent 2"}'
        ]
    }
}
```

### 4. Filter State Object

```javascript
// Stored in localStorage as 'taskraptor_filters'
{
    completion: 'uncompleted',      // Which tasks to show
    dateRange: 'last30',            // Time window for completed tasks
    customStart: '2025-01-01',      // Custom date range start (if dateRange='custom')
    customEnd: '2025-12-31',        // Custom date range end (if dateRange='custom')
    parentFilter: '1234567890'      // Parent GID to filter by (empty = no filter)
}
```

---

## Application Lifecycle

### Page Load Sequence

```
1. window.onload
   │
   ├─→ loadExpandStates()          // Restore UI state
   │
   ├─→ loadPAT()                   // Check for saved token
   │   │
   │   ├─→ IF token exists:
   │   │   │
   │   │   ├─→ authenticateAndLoad()
   │   │   │   │
   │   │   │   ├─→ Verify PAT with Asana API (/users/me)
   │   │   │   │
   │   │   │   ├─→ loadFilterState()  // Restore filter preferences
   │   │   │   │
   │   │   │   ├─→ loadCache()        // Check for cached data
   │   │   │   │   │
   │   │   │   │   ├─→ IF cache exists AND < 24 hours old:
   │   │   │   │   │   │
   │   │   │   │   │   ├─→ displayCachedData()
   │   │   │   │   │   │   ├─→ Restore allTasksMap from cache
   │   │   │   │   │   │   ├─→ Restore rootItems from cache
   │   │   │   │   │   │   ├─→ populateParentFilter()
   │   │   │   │   │   │   ├─→ displayTasks()
   │   │   │   │   │   │   └─→ showBackgroundLoadingBanner()
   │   │   │   │   │   │
   │   │   │   │   │   └─→ setTimeout(refreshData(background=true), 1000)
   │   │   │   │   │       └─→ loadUserTasks(background=true)
   │   │   │   │   │
   │   │   │   │   └─→ IF cache is stale or missing:
   │   │   │   │       └─→ loadUserTasks(background=false)
   │   │   │
   │   │   └─→ Show main UI
   │   │
   │   └─→ IF no token:
   │       └─→ Show login form
```

### Data Loading Sequence (loadUserTasks)

This is the most complex flow in the application:

```
loadUserTasks(background)
│
├─→ 1. FETCH ASSIGNED TASKS (with pagination)
│   │
│   ├─→ Build API URL with filters:
│   │   ├─→ Base: /tasks?assignee={userId}&workspace={workspaceId}
│   │   ├─→ Add opt_fields (name, completed, assignee, etc.)
│   │   ├─→ IF completion='completed': add &completed=true
│   │   ├─→ IF dateRange set: add &completed_since={date}
│   │   └─→ IF dateRange set: add &modified_since={date}
│   │
│   ├─→ Paginate through all results:
│   │   ├─→ Fetch page with current offset
│   │   ├─→ Add results to allTasks array
│   │   ├─→ Update progress (0-30%)
│   │   └─→ Continue while next_page.offset exists
│   │
│   └─→ Result: allTasks[] with all directly assigned tasks
│
├─→ 2. FILTER BY DATE (client-side)
│   │
│   ├─→ filterTasksByDate(allTasks)
│   │   ├─→ IF dateRange='all': return all
│   │   ├─→ IF completion='uncompleted': return all (ignore date)
│   │   ├─→ IF completion='completed': filter by completed_at
│   │   └─→ IF completion='both': keep all uncompleted, filter completed
│   │
│   └─→ Result: filteredTasks[] with date-filtered tasks
│
├─→ 3. INITIALIZE TASK STRUCTURE
│   │
│   ├─→ For each task in filteredTasks:
│   │   ├─→ Ensure task.subtasks = []
│   │   ├─→ Set task.comments = null (not loaded yet)
│   │   └─→ Set task.subtasksLoaded = false
│   │
│   ├─→ Clear allTasksMap
│   │
│   └─→ Add all filteredTasks to allTasksMap
│
├─→ 4. CREATE MISSING PARENT ITEMS
│   │
│   ├─→ createMissingParentItems()
│   │   ├─→ Collect all referenced parent GIDs
│   │   ├─→ Find parents not in allTasksMap
│   │   ├─→ Create placeholder items for missing parents
│   │   │   ├─→ Extract from task.memberships (project/section)
│   │   │   ├─→ Set isProject/isSection/isPlaceholder flags
│   │   │   └─→ Add to allTasksMap
│   │   │
│   │   └─→ Link tasks to their sections/projects
│   │
│   └─→ Result: allTasksMap now includes placeholder parents
│
├─→ 5. LOAD ALL SUBTASKS RECURSIVELY (CRITICAL STEP)
│   │
│   ├─→ For each task in filteredTasks:
│   │   │
│   │   ├─→ loadSubtasksFiltered(taskGid, parentAssignedToUser=true)
│   │   │   │
│   │   │   ├─→ Check if already loaded (subtasksLoaded flag)
│   │   │   ├─→ Fetch subtasks from API: /tasks/{gid}/subtasks
│   │   │   │
│   │   │   ├─→ For each subtask:
│   │   │   │   ├─→ Check if assigned to user OR (unassigned AND parent assigned to user)
│   │   │   │   ├─→ IF matches: Add to parent.subtasks[]
│   │   │   │   ├─→ Add to allTasksMap
│   │   │   │   └─→ RECURSE: loadSubtasksFiltered(subtaskGid, ...)
│   │   │   │
│   │   │   └─→ Set subtasksLoaded = true
│   │   │
│   │   └─→ Update progress (40-70%)
│   │
│   └─→ Result: allTasksMap contains ALL tasks + ALL subtasks
│       ⚠️ CRITICAL: Hierarchy building REQUIRES this to be complete!
│
├─→ 6. BUILD HIERARCHY (Determine Root Items)
│   │
│   ├─→ Clear rootItems array
│   ├─→ Clear allParents set
│   ├─→ Initialize rootGids set (prevent duplicates)
│   │
│   ├─→ For each task in allTasksMap:
│   │   │
│   │   ├─→ getParentChain(taskGid)  // Walk up to find true root
│   │   │   │
│   │   │   ├─→ Follow parent references up the chain
│   │   │   ├─→ Stop when no parent OR parent not in allTasksMap
│   │   │   └─→ Return chain from root to this task
│   │   │
│   │   ├─→ Determine trueRoot:
│   │   │   ├─→ IF parentChain.length > 0: trueRoot = parentChain[0]
│   │   │   └─→ ELSE: trueRoot = task (task IS the root)
│   │   │
│   │   ├─→ IF trueRoot.gid NOT in rootGids:
│   │   │   ├─→ Add trueRoot to rootItems[]
│   │   │   ├─→ Add trueRoot.gid to rootGids
│   │   │   └─→ Add to allParents (for filter dropdown)
│   │   │
│   │   └─→ Update progress (70-90%)
│   │
│   ├─→ Deduplication pass (safety check)
│   │   ├─→ Check for duplicate GIDs in rootItems
│   │   └─→ Remove duplicates if found
│   │
│   └─→ Result: rootItems[] contains unique root-level tasks
│
├─→ 7. COLLAPSE SINGLE-CHILD CHAINS
│   │
│   ├─→ collapseSingleChildChains()
│   │   │
│   │   ├─→ For each root in rootItems:
│   │   │   ├─→ Walk down while only one child exists
│   │   │   ├─→ IF chain length > 1:
│   │   │   │   ├─→ Use deepest child as new root
│   │   │   │   └─→ Store parent chain in collapsedParentChain
│   │   │   └─→ ELSE: Keep original root
│   │   │
│   │   └─→ Replace rootItems with collapsed version
│   │
│   └─→ Result: Cleaner hierarchy with collapsed single-child chains
│
├─→ 8. SAVE TO CACHE
│   │
│   ├─→ saveCache({
│   │       allTasksMap: Array.from(allTasksMap.entries()),
│   │       rootItems: rootItems,
│   │       allParents: Array.from(allParents)
│   │   })
│   │
│   └─→ Store in localStorage with timestamp
│
├─→ 9. DISPLAY & FINALIZE
│   │
│   ├─→ populateParentFilter()  // Fill dropdown with parent options
│   ├─→ displayTasks()          // Render to DOM
│   ├─→ displayFilterBadges()   // Show active filters
│   ├─→ Update progress (100%)
│   ├─→ showToast('Success')
│   └─→ hideProgress() OR hideBackgroundLoadingBanner()
```

### Display Sequence (displayTasks)

```
displayTasks()
│
├─→ 1. APPLY PARENT FILTER
│   ├─→ IF parentFilter is set:
│   │   └─→ Filter rootItems to match parent OR descendants of parent
│   └─→ Result: tasksToDisplay[]
│
├─→ 2. APPLY COMPLETION FILTER
│   ├─→ filterTasksByCompletion(tasksToDisplay)
│   │   │
│   │   ├─→ IF completion='uncompleted':
│   │   │   └─→ Keep only incomplete tasks (+ ancestors with incomplete descendants)
│   │   │
│   │   ├─→ IF completion='completed':
│   │   │   └─→ Keep only complete tasks (+ ancestors with complete descendants)
│   │   │
│   │   └─→ IF completion='both':
│   │       └─→ Keep all tasks
│   │
│   └─→ Result: Filtered tasks with recursive logic
│
├─→ 3. APPLY SORTING
│   ├─→ sortTasks(tasksToDisplay)
│   │   ├─→ Read current sort setting (name/due_date/completed/modified)
│   │   └─→ Sort array accordingly
│   │
│   └─→ Result: Sorted task array
│   │
│   └─→ 📝 TODO: Sort selection not currently persisted; will be added to localStorage
│
├─→ 4. RENDER EACH TASK
│   ├─→ For each task in tasksToDisplay:
│   │   │
│   │   ├─→ renderTask(task, level=0, isRoot=true)
│   │   │   │
│   │   │   ├─→ Build breadcrumb (if isRoot)
│   │   │   ├─→ Build task header HTML
│   │   │   ├─→ Add expand icon (if has subtasks/notes)
│   │   │   ├─→ Add checkbox (if real task)
│   │   │   ├─→ Add action buttons (due date, subtask, rename, notes)
│   │   │   ├─→ Create task-details div
│   │   │   │
│   │   │   ├─→ IF expanded (from saved state):
│   │   │   │   └─→ Trigger renderTaskDetails(taskGid) asynchronously
│   │   │   │
│   │   │   └─→ Return DOM element
│   │   │
│   │   └─→ Append to DOM
│   │
│   └─→ Update summary stats
```

---

## Key Systems Explained

### 1. Caching System (24-Hour Cache)

**Purpose**: Instant page loads with stale-while-revalidate pattern

**How It Works:**

```javascript
// Cache Save (happens after every successful data load)
function saveCache(data) {
    const cacheData = {
        timestamp: Date.now(),  // Store when cached
        data: data               // Store all data
    };
    localStorage.setItem('asana_cache', JSON.stringify(cacheData));
}

// Cache Load (happens on page load)
function loadCache() {
    const cached = localStorage.getItem('asana_cache');
    if (!cached) return null;
    
    const cacheData = JSON.parse(cached);
    const ageHours = (Date.now() - cacheData.timestamp) / (1000 * 60 * 60);
    
    // Invalidate after 24 hours
    if (ageHours > 24) {
        localStorage.removeItem('asana_cache');
        return null;
    }
    
    return cacheData.data;
}

// Usage Pattern:
// 1. Load cache immediately → Display instantly
// 2. After 1 second → Refresh in background
// 3. Update display when fresh data arrives
```

**Cache Invalidation Rules:**
- ✅ Automatic after 24 hours
- ✅ Manual via "Refresh" button
- ✅ When filter changes require API reload (completion or date filters)
- ❌ NOT invalidated when parent filter changes (client-side only)

**Cache Contents:**
- `allTasksMap`: ALL tasks including subtasks (as array of entries)
- `rootItems`: Root-level display tasks
- `allParents`: Parent options for dropdown

**Limitations:**
- localStorage ~10MB limit (typically ~1000-2000 tasks)
- No partial updates (always full replace)
- No versioning (cache format change = manual clear needed)

### 2. Hierarchy Building System

**The Problem:**
- Asana tasks can be nested arbitrarily deep
- User might be assigned a subtask but not its parent
- App needs to show hierarchical view with breadcrumbs

**The Solution - Multi-Phase Process:**

#### Phase 1: Fetch Directly Assigned Tasks
```javascript
// Get tasks where assignee = currentUser
GET /tasks?assignee={userId}&workspace={workspaceId}

// Returns: Tasks directly assigned to user
// Does NOT include: Parent tasks or subtasks (yet)
```

#### Phase 2: Load ALL Subtasks Recursively
```javascript
// For EACH directly assigned task, fetch its subtasks
// Then for EACH subtask, fetch ITS subtasks
// Continue until no more subtasks exist

async function loadSubtasksFiltered(taskGid, parentAssignedToUser) {
    // Fetch subtasks
    const subtasks = await fetchSubtasks(taskGid);
    
    // Filter to only user's tasks or unassigned when parent is assigned to user
    for (const subtask of subtasks) {
        if (isAssignedToUser || (isUnassigned && parentAssignedToUser)) {
            // Add to allTasksMap
            allTasksMap.set(subtask.gid, subtask);
            
            // RECURSE - load this subtask's subtasks
            await loadSubtasksFiltered(subtask.gid, isAssignedToUser);
        }
    }
}
```

**Critical**: This phase MUST complete before hierarchy building!

#### Phase 3: Build Hierarchy (Determine Roots)
```javascript
// For EACH task in allTasksMap (includes subtasks now):
for (const task of allTasksMap.values()) {
    // Walk UP the parent chain to find the true root
    const parentChain = await getParentChain(task.gid);
    
    // True root is either:
    // 1. The top-most parent in the chain, OR
    // 2. The task itself if no parents
    const trueRoot = parentChain.length > 0 ? parentChain[0] : task;
    
    // Add to rootItems if not already there
    if (!rootGids.has(trueRoot.gid)) {
        rootItems.push(trueRoot);
        rootGids.add(trueRoot.gid);
    }
}
```

**Why This Works:**
- A task is a "root" if its parent is NOT in `allTasksMap`
- Since `allTasksMap` contains all user-assigned tasks + subtasks,
- Any parent NOT in the map means it's not assigned to user
- Therefore, that task becomes a root (displayed at top level)

#### Phase 4: Create Missing Parent Items
```javascript
// Some tasks reference parents that aren't in allTasksMap
// (e.g., project or section not assigned to user)
// Create placeholder items for these

function createMissingParentItems() {
    // Find parent GIDs referenced but not in allTasksMap
    const missingParents = findMissingParentGids();
    
    // For each missing parent:
    for (const parentGid of missingParents) {
        // Extract info from task.memberships
        const projectInfo = extractProjectInfo(parentGid);
        
        // Create placeholder
        const placeholder = {
            gid: parentGid,
            name: projectInfo.name,
            isProject: true,  // or isSection
            isPlaceholder: true,
            subtasks: [],
            completed: false
        };
        
        allTasksMap.set(parentGid, placeholder);
    }
}
```

**Example Scenario:**

```
User is assigned to:
- "Fix bug #123" (subtask of "Sprint Tasks", which is in "Engineering Project")

Asana Hierarchy:
Engineering Project (NOT assigned to user)
└── Sprint Tasks (NOT assigned to user)
    └── Fix bug #123 (ASSIGNED to user) ← Only this returned by API

After Loading:
1. Fetch: Returns "Fix bug #123"
2. Load subtasks: "Fix bug #123" has no subtasks
3. Build hierarchy:
   - "Fix bug #123" references parent "Sprint Tasks"
   - "Sprint Tasks" NOT in allTasksMap
   - Therefore, "Fix bug #123" becomes a ROOT
4. Display:
   - Breadcrumb shows: "Engineering Project → Sprint Tasks → Fix bug #123"
   - But only "Fix bug #123" is in task list (as root item)

> **📝 TODO**: Root tasks without due dates will display the nearest subtask's due date with visual indicator. Clicking the inherited due date will navigate to that subtask.
```

### 3. Filtering System

**Three Types of Filters with Different Behaviors:**

#### A. Completion Status Filter
> **📝 TODO**: Labels will change from "Uncompleted"/"Completed" to "Incomplete Only"/"Completed Only" for clarity

- **Values**: 'uncompleted', 'completed', 'both'
- **Applied**: Both API-level AND client-side
- **Behavior**: Recursive (includes ancestors with matching descendants)
- **Cache Impact**: ✅ Requires cache clear + API reload

```javascript
function filterTasksByCompletion(tasks) {
    if (filterState.completion === 'both') return tasks;
    
    return tasks.filter(task => {
        // Direct match
        if (filterState.completion === 'completed' && task.completed) return true;
        if (filterState.completion === 'uncompleted' && !task.completed) return true;
        
        // Recursive check: Keep task if ANY descendant matches
        return hasMatchingDescendant(task, filterState.completion);
    });
}
```

**Why Recursive?**
- Without it, parent tasks would disappear if they don't match
- User couldn't see hierarchy of incomplete subtasks under complete parent

#### B. Date Range Filter
- **Values**: 'last7', 'last30', 'last90', 'custom', 'all'
- **Applied**: API-level (reduces payload)
- **Special Rule**: Only applies to COMPLETED tasks!
- **Cache Impact**: ✅ Requires cache clear + API reload

```javascript
// API URL construction
if (filterState.completion === 'completed' && dateFilter) {
    apiUrl += `&completed_since=${dateFilter}`;
}

// Client-side (for 'both' mode)
function filterTasksByDate(tasks) {
    if (filterState.dateRange === 'all') return tasks;
    
    // ALWAYS include uncompleted tasks (ignore date)
    if (filterState.completion === 'uncompleted') return tasks;
    
    return tasks.filter(task => {
        if (!task.completed) return true;  // Always keep incomplete
        
        // Check if completion date is within range
        const completedAt = new Date(task.completed_at);
        return completedAt >= filterDate;
    });
}
```

#### C. Parent Project Filter
- **Values**: Parent GID or empty string
- **Applied**: Client-side only
- **Behavior**: Shows parent + all descendants
- **Cache Impact**: ❌ No cache clear (uses existing data)

```javascript
function applyParentFilter(tasks) {
    if (!filterState.parentFilter) return tasks;
    
    return tasks.filter(task => {
        // Is this the selected parent?
        if (task.gid === filterState.parentFilter) return true;
        
        // Is this a descendant of the selected parent?
        return isDescendantOf(task.gid, filterState.parentFilter);
    });
}
```

**Filter Priority (Applied in Order):**
1. Parent filter
2. Completion filter
3. Sorting

### 4. Expand/Collapse System

**Single Expand/Collapse Mechanism:**

- **Trigger**: Click on chevron icon (▶/▼) OR anywhere on task header
- **Icon**: ▶ (collapsed) / ▼ (expanded)
- **Shows When Expanded**: All task details in the details panel:
  - Task notes (if present)
  - Comments (if present)
  - Subtasks (rendered as nested task items)
- **State**: Persisted to localStorage
- **Data Loading**: 
  - Subtasks already loaded during initial data fetch (instant display)
  - Comments lazy-loaded on first expansion (if not cached)

```javascript
async function toggleTaskExpand(event, taskGid) {
    const detailsDiv = document.getElementById('details-' + taskGid);
    const icon = taskElement.querySelector('.expand-icon');
    
    if (detailsDiv.style.display === 'none') {
        // EXPAND
        detailsDiv.style.display = 'block';
        icon.textContent = '▼';
        expandStates[taskGid] = true;
        
        // Load comments if not already loaded
        if (!task.comments && !task.isProject) {
            await loadComments(taskGid);
        }
        
        // Load subtasks if not already loaded
        if (!task.subtasksLoaded) {
            await loadSubtasksFiltered(task.gid, isAssignedToUser);
        }
        
        // Render everything: notes, comments, and subtasks
        await renderTaskDetails(taskGid);
    } else {
        // COLLAPSE
        detailsDiv.style.display = 'none';
        icon.textContent = '▶';
        expandStates[taskGid] = false;
    }
    
    saveExpandStates();  // Persist to localStorage
}
```

**Expand State Persistence:**
- All tasks load collapsed by default on first visit
- Expand state persisted to localStorage after manual expand/collapse
- State restored from localStorage on subsequent page loads

> **📝 TODO**: When task is marked complete, it will automatically collapse to reduce visual clutter

### 5. Edit System (WYSIWYG)

> **📝 TODO**: Multiple changes planned:
> - Notes/Comments will initially show only 3 lines with "Show More" expansion
> - Edit button will move to top-right corner of text box
> - Bullet point indentation bug will be fixed
> - "Rename" and "Add Subtask" will use inline editing instead of current method

**Three Editable Components:**

#### A. Task Notes
```javascript
// Uses Quill.js rich text editor
async function editTaskNotes(event, taskGid) {
    const editor = new Quill('#editor-' + taskGid, {
        theme: 'snow',
        modules: { toolbar: [...] }
    });
    
    // Load existing HTML notes
    if (task.html_notes) {
        editor.clipboard.dangerouslyPasteHTML(task.html_notes);
    }
    
    activeEditor = editor;  // Store reference
}

async function saveTaskNotes(taskGid) {
    const htmlNotes = activeEditor.root.innerHTML;
    const plainNotes = activeEditor.getText();
    
    // Save to Asana
    await fetch(`https://app.asana.com/api/1.0/tasks/${taskGid}`, {
        method: 'PUT',
        body: JSON.stringify({ data: { notes: plainNotes } })
    });
    
    // Update local cache
    task.notes = plainNotes;
    task.html_notes = htmlNotes;
    saveCache(...);
}
```

#### B. Comments (Add/Edit/Delete)
```javascript
// Add Comment
async function saveComment(taskGid) {
    const htmlText = commentEditor.root.innerHTML;
    const plainText = commentEditor.getText();
    
    await fetch(`https://app.asana.com/api/1.0/tasks/${taskGid}/stories`, {
        method: 'POST',
        body: JSON.stringify({ data: { text: plainText } })
    });
    
    // Reload comments
    task.comments = null;
    await loadComments(taskGid);
    await renderTaskDetails(taskGid);
}

// Edit Comment
async function saveCommentEdit(taskGid, commentGid) {
    // Similar flow but with PUT to /stories/{commentGid}
}

// Delete Comment
async function deleteComment(event, taskGid, commentGid) {
    if (!confirm('Delete this comment?')) return;
    
    await fetch(`https://app.asana.com/api/1.0/stories/${commentGid}`, {
        method: 'DELETE'
    });
    
    // Reload comments
    task.comments = null;
    await loadComments(taskGid);
}
```

#### C. Task Completion
```javascript
async function toggleTaskCompletion(event, taskGid) {
    const checkbox = event.target;
    const completed = checkbox.checked;
    
    // Update Asana
    await fetch(`https://app.asana.com/api/1.0/tasks/${taskGid}`, {
        method: 'PUT',
        body: JSON.stringify({ data: { completed } })
    });
    
    // Update local state
    task.completed = completed;
    
    // Update UI (strikethrough, etc.)
    updateTaskStyling(taskGid, completed);
    
    // Update parent styling if parent is a project/section
    updateParentCompletion(task.parent);
    
    // Update cache
    saveCache(...);
}
```

**Edit Safety:**
- All edits immediately update Asana API
- Local cache updated only AFTER successful API response
- On error, UI state reverted
- Cache saved after every successful edit

---

## Function Reference

### Authentication Functions

#### `authenticate()`
- **Purpose**: Login with PAT from input field
- **Trigger**: "Login" button click
- **Flow**: Read input → Save to localStorage → Call authenticateAndLoad()

#### `authenticateAndLoad()`
- **Purpose**: Verify PAT and initialize app
- **Flow**: 
  1. Call Asana API /users/me to verify PAT
  2. Extract currentUserId
  3. Hide login form, show main UI
  4. Load filter state from localStorage
  5. Load cache OR fetch fresh data

#### `logout()`
- **Purpose**: Clear authentication and reset app
- **Flow**: Confirm → Clear PAT → Clear cache → Reload page

### Data Loading Functions

#### `loadUserTasks(background = false)`
- **Purpose**: Main data loading function (see detailed flow above)
- **Parameters**: 
  - `background`: If true, don't show progress bar (for background refresh)
- **Side Effects**: Populates allTasksMap, rootItems, allParents
- **Returns**: Nothing (updates global state)

#### `loadSubtasksFiltered(taskGid, parentAssignedToUser)`
- **Purpose**: Recursively load subtasks for a task
- **Parameters**:
  - `taskGid`: Task to load subtasks for
  - `parentAssignedToUser`: Whether parent is assigned to user
- **Logic**: Recursively loads subtasks, including only those assigned to user OR (unassigned subtasks when parent is assigned to user). This means unassigned subtasks are shown in the hierarchy as descendants of user-assigned tasks.
- **Side Effects**: Adds subtasks to allTasksMap, sets subtasksLoaded flag

#### `getTaskDetails(taskGid)`
- **Purpose**: Fetch single task from API
- **Returns**: Task object or null
- **Caching**: Returns from allTasksMap if already fetched

#### `getParentChain(taskGid)`
- **Purpose**: Walk up parent references to build breadcrumb
- **Returns**: Array of parent tasks from root to immediate parent
- **Logic**: Stops when parent not in allTasksMap (true root found)

#### `loadComments(taskGid)`
- **Purpose**: Fetch comments/stories for a task
- **Side Effects**: Sets task.comments array
- **Filtering**: Only includes comments (not system stories) with text

### Display Functions

#### `displayTasks()`
- **Purpose**: Main rendering function (see detailed flow above)
- **Side Effects**: Clears and repopulates #taskList
- **Order**: Parent filter → Completion filter → Sort → Render

#### `renderTask(task, level, isRoot = false)`
- **Purpose**: Create DOM element for a single task
- **Parameters**:
  - `task`: Task object to render
  - `level`: Indentation level (for nesting visual)
  - `isRoot`: Whether this is a root-level task (affects breadcrumb)
- **Returns**: DOM element (div.task-item)
- **Side Effects**: If auto-expand, triggers renderTaskDetails()

#### `renderTaskDetails(taskGid)`
- **Purpose**: Render notes, comments, and subtasks in detail pane
- **Side Effects**: Updates #details-{taskGid} innerHTML
- **Lazy Loading**: Loads comments if not already loaded

#### `updateSummary(tasks)`
- **Purpose**: Update task statistics display
- **Calculations**: Root items, total tasks, completed, remaining
- **Display**: Shows filter info and data source (cached vs fresh)

> **📝 TODO**: Modal system will be refactored for consistency. Currently `showInfoModal` uses `showGenericModal` properly, but filter modal uses hardcoded DOM elements. Logout confirmation will change from dialog to modal with Cancel/Logout buttons.

### Filter Functions

> **📝 TODO**: Filter system changes:
> - Bug fix: Changing filter criteria will immediately update chips display
> - Deleting filter chip will show "Apply Filter Changes" button for confirmation
> - Toolbar will become floating/docked to viewport top when scrolling, minimizing to icons on scroll down

#### `filterTasksByCompletion(tasks)`
- **Purpose**: Apply completion status filter recursively
- **Logic**: Keep task if it matches OR any descendant matches
- **Returns**: Filtered array

#### `filterTasksByDate(tasks)`
- **Purpose**: Apply date range filter (completed tasks only)
- **Logic**: Always include uncompleted, filter completed by date
- **Returns**: Filtered array

#### `applyParentFilter(tasks)`
- **Purpose**: Filter by parent project (client-side)
- **Logic**: Keep if task IS parent OR descendant of parent
- **Returns**: Filtered array

#### `getDateFilter()`
- **Purpose**: Convert dateRange setting to ISO date string
- **Returns**: ISO date string or null
- **Used For**: API query parameters

### UI Functions

#### `showToast(message, type = 'info', duration = 5000)`
- **Purpose**: Show notification message
- **Types**: 'error', 'success', 'info', 'warning'
- **Behavior**: Auto-dismiss after duration, manual close button
- **Stacking**: Multiple toasts stack vertically

#### `showProgress()` / `hideProgress()`
- **Purpose**: Toggle progress bar visibility
- **Used During**: Initial data load

#### `updateProgress(current, total, message)`
- **Purpose**: Update progress bar percentage and message
- **Parameters**:
  - `current`: Current step (0-100)
  - `total`: Total steps (usually 100)
  - `message`: Status message to display

#### `showFilterModal()` / `closeFilterModal()`
- **Purpose**: Toggle filter configuration modal
- **Side Effects**: Populates modal with current filter values

#### `displayFilterBadges()`
- **Purpose**: Show active filter chips
- **Logic**: Only shows non-default filters
- **Behavior**: Each chip has X button to remove filter

### Edit Functions

#### `editTaskNotes(event, taskGid)`
- **Purpose**: Show Quill editor for task notes
- **Side Effects**: Creates editor instance, loads existing notes

#### `saveTaskNotes(taskGid)`
- **Purpose**: Save edited notes to Asana
- **Flow**: Get HTML + plain text → API PUT → Update cache → Re-render

#### `showCommentBox(taskGid)` / `saveComment(taskGid)`
- **Purpose**: Add new comment to task
- **Flow**: Show editor → Save → Reload comments → Re-render

#### `editComment(event, taskGid, commentGid)` / `saveCommentEdit(taskGid, commentGid)`
- **Purpose**: Edit existing comment
- **Flow**: Show editor with existing text → Save → Reload comments

#### `deleteComment(event, taskGid, commentGid)`
- **Purpose**: Delete comment from task
- **Flow**: Confirm → API DELETE → Reload comments → Re-render

#### `toggleTaskCompletion(event, taskGid)`
- **Purpose**: Mark task complete/incomplete
- **Flow**: API PUT → Update task object → Update UI styling → Update parents

#### `renameTask(event, taskGid)`
- **Purpose**: Change task name
- **Flow**: Prompt → API PUT → Update cache → Re-render task

#### `createSubtask(event, parentGid)`
- **Purpose**: Add new subtask to parent
- **Flow**: Prompt → API POST → Reload subtasks → Re-render

#### `setDueDate(event, taskGid)` / `saveDueDate(taskGid, dateInput)`
- **Purpose**: Set or change task due date
- **Flow**: Show date picker → API PUT → Update cache → Re-render

### Utility Functions

#### `escapeHtml(text)`
- **Purpose**: Prevent XSS by escaping HTML special characters
- **Used For**: All user-generated text displayed in DOM

#### `linkifyHTML(html)`
- **Purpose**: Convert URLs in text to clickable links
- **Pattern**: Matches http:// and https:// URLs
- **Returns**: HTML string with `<a>` tags

#### `savePAT(token)` / `loadPAT()` / `clearPAT()`
- **Purpose**: Manage PAT in localStorage
- **Key**: 'asana_pat'

#### `saveCache(data)` / `loadCache()` / `clearCache()`
- **Purpose**: Manage cache in localStorage
- **Key**: 'asana_cache'
- **Validation**: Checks 24-hour age

#### `saveFilterState()` / `loadFilterState()`
- **Purpose**: Persist filter preferences
- **Key**: 'asana_filters'

#### `saveExpandStates()` / `loadExpandStates()`
- **Purpose**: Persist expand/collapse state
- **Key**: 'asana_expand_states'

#### `populateParentFilter()`
- **Purpose**: Fill parent filter dropdown with options
- **Source**: allParents set
- **Sorting**: Alphabetical by name

#### `collapseSingleChildChains()`
- **Purpose**: Simplify hierarchy by collapsing single-child chains
- **Logic**: If parent has only one child, use child as root with breadcrumb
- **Result**: Cleaner display, fewer nesting levels

#### `createMissingParentItems()`
- **Purpose**: Create placeholder items for parents (projects/sections) not assigned to user
- **Source**: task.memberships (project/section info)
- **Result**: Complete hierarchy even when parent projects/sections are not assigned to user

---

## State Management

### Global State Variables

The application uses **global mutable state** (not React/Redux style). All state lives in global `let` variables.

#### Mutation Rules

**✅ Safe to Mutate:**
- `allTasksMap` - Add/update tasks freely
- `rootItems` - Replace array entirely during hierarchy building
- `filterState` - Update properties then call saveFilterState()
- `expandStates` - Update then call saveExpandStates()

**⚠️ Mutate Carefully:**
- `personalAccessToken` - Only change during login/logout
- `currentUserId` - Only set during authentication
- `cachedData` - Only set once on page load
- `WORKSPACE_ID` - Only set during authentication/workspace selection

### State Synchronization

**localStorage Persistence:**
```javascript
// These states persist across page loads:
- personalAccessToken → 'asana_pat'
- filterState → 'asana_filters'
- expandStates → 'asana_expand_states'
- {allTasksMap, rootItems, allParents} → 'asana_cache'

// These states are session-only:
- currentUserId (re-fetched on login)
- activeEditor (editor instances)
- tasksBeingLoaded (temporary UI state)
```

**Cache Synchronization:**
```javascript
// Cache must be updated after these operations:
✅ Task completion toggled
✅ Task notes saved
✅ Task renamed
✅ Subtask created
✅ Due date changed
✅ Full data reload completed

// Cache does NOT update after:
❌ Comment added/edited/deleted (comments not cached)
❌ Filter changed (filters applied to cached data)
❌ Sort changed (sorting applied to cached data)
```

### State Consistency Rules

**Rule 1: allTasksMap is Source of Truth**
- ALL task operations read from allTasksMap
- rootItems is a VIEW into allTasksMap (contains references, not copies)
- Never modify task properties without updating allTasksMap

**Rule 2: Hierarchy Building Order**
```javascript
// CORRECT ORDER:
1. Populate allTasksMap with all tasks
2. Load ALL subtasks (adds more to allTasksMap)
3. Build hierarchy (determines rootItems from allTasksMap)

// INCORRECT (will cause missing tasks):
1. Populate allTasksMap
2. Build hierarchy  // ❌ Subtasks not loaded yet!
3. Load subtasks    // ❌ Too late, hierarchy already built!
```

**Rule 3: Cache Invalidation**
```javascript
// API-level filters changed → Must reload from API
if (oldCompletion !== filterState.completion || 
    oldDateRange !== filterState.dateRange) {
    clearCache();
    await loadUserTasks();  // Fresh fetch
}

// Client-side filter changed → Use cached data
if (onlyParentFilterChanged) {
    displayTasks();  // Re-render with existing data
}
```

---

## API Integration Patterns

### Asana REST API Basics

**Base URL**: `https://app.asana.com/api/1.0`

**Authentication**: Bearer token (Personal Access Token)
```javascript
headers: {
    'Authorization': `Bearer ${personalAccessToken}`
}
```

**Rate Limits**: 
- 150 requests per minute per user
- This app generally stays well under (except on initial load with many subtasks)

### Common Endpoints Used

#### 1. Verify Authentication
```javascript
GET /users/me

Response:
{
    "data": {
        "gid": "1234567890",
        "name": "John Doe",
        "email": "john@example.com"
    }
}
```

#### 2. Fetch Assigned Tasks
```javascript
GET /tasks?assignee={userId}&workspace={workspaceId}&opt_fields=...

Query Parameters:
- assignee: User GID
- workspace: Workspace GID
- opt_fields: Comma-separated field list
- completed: true/false (filter by completion)
- completed_since: ISO date (only completed tasks after this date)
- modified_since: ISO date (only tasks modified after this date)
- offset: Pagination token (for next page)

Response:
{
    "data": [ /* array of tasks */ ],
    "next_page": {
        "offset": "eyJ0eXAiOiJKV1Q...",  // Token for next page
        "path": "/tasks?offset=...",
        "uri": "https://app.asana.com/api/1.0/tasks?offset=..."
    }
}
```

**Important**: Default page size is 100. Must paginate!

#### 3. Fetch Single Task
```javascript
GET /tasks/{taskGid}?opt_fields=...

Response:
{
    "data": {
        "gid": "1234567890",
        "name": "Task name",
        "completed": false,
        // ... other fields
    }
}
```

#### 4. Fetch Subtasks
```javascript
GET /tasks/{taskGid}/subtasks?opt_fields=...

Response:
{
    "data": [ /* array of subtask objects */ ]
}
```

#### 5. Update Task
```javascript
PUT /tasks/{taskGid}
Content-Type: application/json

Body:
{
    "data": {
        "completed": true,
        // OR
        "notes": "Updated notes",
        // OR
        "name": "New name",
        // OR
        "due_on": "2025-12-31"
    }
}

Response:
{
    "data": { /* updated task object */ }
}
```

#### 6. Fetch Comments (Stories)
```javascript
GET /tasks/{taskGid}/stories?opt_fields=type,text,html_text,created_at,created_by

Response:
{
    "data": [
        {
            "gid": "9876543210",
            "type": "comment",  // or "system"
            "text": "Plain text comment",
            "html_text": "<body>HTML comment</body>",
            "created_at": "2025-12-05T10:30:00.000Z",
            "created_by": {
                "gid": "1234567890",
                "name": "John Doe"
            }
        }
    ]
}
```

**Filtering**: App only shows type='comment' with text/html_text

#### 7. Add Comment
```javascript
POST /tasks/{taskGid}/stories
Content-Type: application/json

Body:
{
    "data": {
        "text": "Comment text"
    }
}
```

#### 8. Update Comment
```javascript
PUT /stories/{storyGid}
Content-Type: application/json

Body:
{
    "data": {
        "text": "Updated comment"
    }
}
```

#### 9. Delete Comment
```javascript
DELETE /stories/{storyGid}

Response: 204 No Content
```

#### 10. Create Subtask
```javascript
POST /tasks/{parentGid}/subtasks
Content-Type: application/json

Body:
{
    "data": {
        "name": "New subtask name",
        "workspace": "{workspaceId}"
    }
}
```

### opt_fields Strategy

**Purpose**: Reduce API payload size by requesting only needed fields

**Common Fields**:
```javascript
const commonFields = 'name,completed,assignee,due_on,modified_at,parent,notes,html_notes';

// For tasks with memberships:
const membershipFields = 'memberships,memberships.project,memberships.project.name,memberships.section,memberships.section.name';

// Full URL:
const url = `/tasks?opt_fields=${commonFields},${membershipFields}`;
```

**Why Not Fetch Everything?**
- Asana API returns LOTS of fields by default
- Custom fields, followers, etc. not needed
- Reduces bandwidth and parse time

### Error Handling Pattern

```javascript
try {
    const response = await fetch(url, { headers });
    
    if (!response.ok) {
        // HTTP error (4xx, 5xx)
        const errorData = await response.json();
        throw new Error(errorData.errors?.[0]?.message || 'Request failed');
    }
    
    const data = await response.json();
    // Success path
    
} catch (error) {
    console.error('Error:', error);
    showToast('Failed: ' + error.message, 'error');
    // Revert UI state if needed
}
```

**Common Errors:**
- 401 Unauthorized: Invalid or expired PAT
- 404 Not Found: Task/resource deleted or no access
- 429 Too Many Requests: Rate limit exceeded
- 500 Server Error: Asana API issue

---

## Performance Optimizations

### 1. Caching Strategy (Primary Optimization)

**Problem**: Full data load takes 5-10 seconds with pagination and subtask loading

**Solution**: 24-hour cache with stale-while-revalidate
- **Initial Load**: < 100ms (from localStorage)
- **Background Refresh**: Happens after display (invisible to user)
- **Trade-off**: Data may be up to 24 hours old on first view

**Impact**: 99% faster perceived load time

### 2. API-Level Filtering

**Problem**: Fetching all tasks (including very old completed) wastes bandwidth

**Solution**: Use API query parameters to filter at source
```javascript
// Instead of fetching everything:
GET /tasks?assignee={userId}

// Fetch only what's needed:
GET /tasks?assignee={userId}&completed=true&completed_since=2025-11-05
```

**Impact**: 50-90% reduction in API payload for completed-only or date-filtered views

### 3. Lazy Loading of Comments

> **📝 TODO**: Additional lazy loading improvements planned:
> - Notes/Comments will show only 3 lines initially with "Show More" option
> - Highest hierarchy tasks will load and render first, subtasks progressively in background

**Problem**: Comments require separate API call per task (N+1 problem)

**Solution**: Only load comments when task details expanded
```javascript
async function toggleTaskExpand(event, taskGid) {
    // ... expansion logic ...
    
    // Lazy load comments
    if (!task.comments && !task.isProject) {
        await loadComments(taskGid);
    }
}
```

**Impact**: Reduces initial API calls from ~200 to ~50

### 4. Subtask Pre-Loading (Trade-off)

> **📝 TODO**: Subtask loading will be optimized with batching (up to 10 tasks per API request using Promise.all) to reduce network calls. Also investigating Asana API capabilities for fetching tasks with subtasks in single request.

**Decision**: Load ALL subtasks during initial load (not lazy)

**Why?**
- Hierarchy building REQUIRES knowing all subtasks
- Prevents "pop-in" effect when expanding tasks
- Allows instant expand/collapse (no loading spinner)

**Trade-off**: Slower initial load, but better UX after

### 5. Progress Bar Chunking

**Problem**: UI freezes during long operations

**Solution**: Update progress every N iterations
```javascript
for (let i = 0; i < tasks.length; i++) {
    await loadSubtasksFiltered(tasks[i].gid);
    
    // Only update UI every 5 iterations
    if (i % 5 === 0) {
        updateProgress(...);
        // Allows UI to repaint
    }
}
```

**Impact**: UI stays responsive during 5-10 second loads

### 6. Duplicate Prevention

> **📝 TODO**: API calls will check for existing data in allTasksMap before making network requests (e.g., when building parent chains)

**Problem**: Same task appearing multiple times in list

**Solution**: Three-layer deduplication
1. **During building**: `rootGids` Set prevents duplicate additions
2. **After building**: Deduplication pass removes any that slipped through
3. **Cache loading**: Deduplicates cached data before display

**Why Three Layers?**
- Defense in depth (hierarchy building is complex)
- Historical bug (Session 9 in docs) caused duplicates
- Cheap operation compared to rendering duplicates

### 7. Expand State Persistence

**Problem**: Losing expand/collapse state on refresh is frustrating

**Solution**: Save state to localStorage
```javascript
// Save after every expand/collapse
expandStates[taskGid] = isExpanded;
saveExpandStates();

// Restore on render
const isExpanded = expandStates[taskGid] || false;
```

**Impact**: Better UX, faster task navigation

### 8. Single-Child Chain Collapsing

**Problem**: Deep hierarchies with single children create wasted vertical space

**Solution**: Collapse chains into breadcrumbs
```
Before:
- Project A
  └── Section B
      └── Task C (the only real task)

After:
- Task C
  Breadcrumb: "Project A → Section B → Task C"
```

**Impact**: Reduces visual clutter, easier scanning

---

## Known Limitations & Edge Cases

### 1. localStorage Size Limit

**Limitation**: Browsers limit localStorage to ~10MB

**Impact**: 
- Typically handles 1000-2000 tasks
- Large teams may exceed limit
- Symptoms: Cache fails to save, tasks reload every time

**Workaround**: Clear cache periodically, adjust date filters

**No Solution**: Fundamental browser limitation

### 3. Personal Access Token Security

**Limitation**: PAT stored in plaintext in localStorage

**Risk**: 
- Anyone with file access can read PAT
- XSS attacks could steal PAT
- Shared computer = shared PAT

**Mitigation**:
- App runs client-side only (no server to compromise)
- User should understand risk
- Use least-privilege PAT (read/write tasks only, not admin)

**No Solution**: True secure storage requires backend

### 4. Subtask Filtering Rules

**Current Logic**: Only show subtasks if:
- Assigned to current user, OR
- Unassigned AND parent task is assigned to user

**Behavior**: This creates a hierarchical view where unassigned subtasks appear as descendants of user-assigned parent tasks. The app shows all tasks directly assigned to the user, plus any unassigned child tasks beneath them (which allows users to see and potentially claim unassigned work under their assigned tasks).

**Edge Case**: Subtask assigned to different user is hidden

**Example**:
```
Task A (assigned to you)
├── Subtask B (assigned to you) ← Visible (assigned to user)
├── Subtask C (unassigned, parent=A) ← Visible (unassigned but parent assigned to user)
└── Subtask D (assigned to coworker) ← Hidden (assigned to someone else)
```

**Rationale**: User-centric view shows user's assigned work plus unassigned subtasks they might want to claim

**Impact**: May confuse users expecting to see all subtasks

### 5. Pagination Limit

**Current Logic**: Paginate until `next_page.offset` is null

**Assumption**: All tasks fit within pagination limits

**Unknown Limit**: Asana doesn't document max pages

**Potential Issue**: If user has > 10,000 tasks, pagination might fail

**Mitigation**: Unlikely scenario, date filters reduce load

### 6. Cache Invalidation Edge Cases

**Scenario 1**: User changes tasks in Asana native UI
- Cache shows stale data until 24 hours OR manual refresh
- Workaround: Use "Refresh" button

**Scenario 2**: Multiple browser tabs
- Each tab has own cache state
- Changes in one tab don't reflect in other
- Workaround: Refresh other tabs

**Scenario 3**: Filter changes
- Completion/date filter changes reload API (correct)
- Parent filter uses cached data (correct)
- BUT: If new tasks added that match filter, won't show until cache refresh

### 7. Comment Editing Limitations

**Limitation**: Only show edit/delete for user's own comments

**Logic**: `comment.created_by.gid === currentUserId`

**Edge Case**: If current user is org admin, may want to edit others' comments

**No Solution**: Intentional design (users should only edit own content)

### 8. Collapse Logic Quirks

**Issue**: Single-child chain collapsing can be confusing

**Example**:
```
Before collapse:
- Project (has 1 child)
  └── Task A (has 3 children)

After collapse:
- Task A (shown as root)
  Breadcrumb: "Project → Task A"
  ├── Subtask 1
  ├── Subtask 2
  └── Subtask 3
```

**Confusion**: Where did the Project go?

**Rationale**: Reduces nesting, Project shown in breadcrumb

**Impact**: Generally positive, but some users may prefer full nesting

### 9. Date Filter Confusion

**Behavior**: Date filter only applies to completed tasks

**Why Confusing**: Users expect "Last 30 days" to filter ALL tasks

**Rationale**: Uncompleted tasks don't have completion dates

**Example**:
```
Filter: "Completed Only" + "Last 30 days"
Shows: Tasks completed in last 30 days ✓

Filter: "Uncompleted Only" + "Last 30 days"
Shows: ALL uncompleted tasks (date ignored) ← Confusing!
```

**TODO**: Add UI hint explaining this behavior

---

## Development Guidelines

### Making Changes Safely

#### Rule 1: Understand the Data Flow
Before changing any code, trace the data flow:
```
1. Where does this data come from? (API, cache, user input)
2. Where is it stored? (allTasksMap, rootItems, filterState)
3. What operations transform it? (filtering, sorting, hierarchy building)
4. Where is it displayed? (which render function)
5. What invalidates it? (user actions, time)
```

#### Rule 2: Test the Hierarchy
After ANY change to hierarchy building, test these scenarios:
- ✅ Task with no parent (true root)
- ✅ Task with parent assigned to user (nested)
- ✅ Task with parent NOT assigned to user (should be root)
- ✅ Task with subtasks (expand works)
- ✅ Task deep in hierarchy (breadcrumb correct)
- ✅ Task in multiple projects (which parent shown?)

#### Rule 3: Preserve Cache Compatibility
If changing data structures:
```javascript
// BAD: Breaks existing caches
task.subtasks = new Map(); // Changed from Array to Map

// GOOD: Migrate old format
if (Array.isArray(task.subtasks)) {
    // Keep as array (compatible)
} else if (task.subtasks instanceof Map) {
    // New format
}
```

Alternative: Bump cache version, clear on version mismatch

#### Rule 4: Update Multiple Places
Common change areas that need updates together:
- Task object structure → Update cache save/load + API fetch
- Filter logic → Update displayTasks() + applyFilters() + filter modal
- State persistence → Update save function + load function + clear function

#### Rule 5: Test Without Cache
Always test with cache disabled:
```javascript
// Temporarily disable cache for testing
function loadCache() {
    return null; // Force fresh load every time
}
```

Why? Cached data may hide bugs in loading logic

### Adding New Features

#### Adding a New Filter

**Example: Add "Priority" filter**

```javascript
// 1. Add to filterState
let filterState = {
    completion: 'uncompleted',
    dateRange: 'last30',
    parentFilter: '',
    priority: 'all' // NEW: 'all' | 'high' | 'medium' | 'low'
};

// 2. Add to filter modal HTML
<select id="priorityFilter">
    <option value="all">All Priorities</option>
    <option value="high">High Priority</option>
    <option value="medium">Medium Priority</option>
    <option value="low">Low Priority</option>
</select>

// 3. Add filter function
function filterTasksByPriority(tasks) {
    if (filterState.priority === 'all') return tasks;
    
    return tasks.filter(task => {
        // Asana priority: High=1, Medium=2, Low=3
        if (filterState.priority === 'high' && task.priority === 1) return true;
        if (filterState.priority === 'medium' && task.priority === 2) return true;
        if (filterState.priority === 'low' && task.priority === 3) return true;
        return false;
    });
}

// 4. Call in displayTasks()
function displayTasks() {
    let tasksToDisplay = rootItems;
    tasksToDisplay = applyParentFilter(tasksToDisplay);
    tasksToDisplay = filterTasksByCompletion(tasksToDisplay);
    tasksToDisplay = filterTasksByPriority(tasksToDisplay); // NEW
    tasksToDisplay = sortTasks(tasksToDisplay);
    // ... render
}

// 5. Add to applyFilters()
async function applyFilters() {
    filterState.priority = document.getElementById('priorityFilter').value;
    saveFilterState();
    displayTasks(); // Client-side only, no API reload needed
}

// 6. Update opt_fields
const url = '/tasks?opt_fields=name,completed,priority,...'; // Add priority

// 7. Add filter badge
function displayFilterBadges() {
    // ... existing chips ...
    
    if (filterState.priority !== 'all') {
        chips.push({
            label: `Priority: ${filterState.priority}`,
            onDelete: () => {
                filterState.priority = 'all';
                saveFilterState();
                displayTasks();
            }
        });
    }
}
```

#### Adding a New Task Action

**Example: Add "Duplicate Task" button**

```javascript
// 1. Add button to renderTask()
<button class="btn-action" onclick="duplicateTask(event, '${task.gid}')" title="Duplicate">📋 Duplicate</button>

// 2. Implement action
async function duplicateTask(event, taskGid) {
    event.stopPropagation();
    
    const task = allTasksMap.get(taskGid);
    if (!task) return;
    
    try {
        // Create new task with same name + " (Copy)"
        const response = await fetch('https://app.asana.com/api/1.0/tasks', {
            method: 'POST',
            headers: {
                'Authorization': `Bearer ${personalAccessToken}`,
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({
                data: {
                    name: task.name + ' (Copy)',
                    notes: task.notes,
                    workspace: WORKSPACE_ID,
                    assignee: currentUserId,
                    parent: task.parent ? task.parent.gid : null
                }
            })
        });
        
        if (!response.ok) throw new Error('Failed to duplicate task');
        
        showToast('Task duplicated!', 'success', 2000);
        
        // Reload to show new task
        await loadUserTasks();
        
    } catch (error) {
        showToast('Failed to duplicate: ' + error.message, 'error');
    }
}
```

### Debugging Tips

#### Console Logging Strategy

The app already has extensive logging. Key points:
```javascript
// Hierarchy building debug points
console.log(`Fetched ${allTasks.length} tasks`);
console.log(`After date filter: ${filteredTasks.length} tasks`);
console.log(`Total root items found: ${rootItems.length}`);

// Check for duplicates
console.log(`rootGids size: ${rootGids.size}`);
console.log(`rootItems length: ${rootItems.length}`);
// If different, duplicates exist!

// Task inspection
console.log('Sample tasks:', tasks.slice(0, 3).map(t => ({
    name: t.name,
    completed: t.completed,
    hasParent: !!t.parent
})));
```

#### Using window.taskDebugInfo

App stores debug info globally:
```javascript
// Open browser console and type:
window.taskDebugInfo

// Returns:
{
    completionFilter: "uncompleted",
    dateFilter: "last30",
    rootItems: 45,
    afterFilters: 42,
    totalTasks: 156,
    completedTasks: 23,
    remainingTasks: 133,
    dataSource: "Loaded from API at 10:30 AM"
}
```

#### Inspecting allTasksMap

```javascript
// See all tasks
Array.from(allTasksMap.values())

// Find specific task
allTasksMap.get('1234567890')

// Find tasks matching criteria
Array.from(allTasksMap.values()).filter(t => !t.completed)

// Check parent relationships
Array.from(allTasksMap.values()).filter(t => t.parent && !allTasksMap.has(t.parent.gid))
// ^ These should be root items!
```

#### Testing Filters

```javascript
// Manually change filter
filterState.completion = 'completed';
displayTasks();

// Check filter application
console.log('Before filter:', rootItems.length);
const filtered = filterTasksByCompletion(rootItems);
console.log('After filter:', filtered.length);
```

#### Simulating Cache

```javascript
// See what's in cache
const cache = JSON.parse(localStorage.getItem('asana_cache'));
console.log('Cache age (hours):', (Date.now() - cache.timestamp) / (1000 * 60 * 60));
console.log('Cached tasks:', cache.data.allTasksMap.length);

// Invalidate cache
localStorage.removeItem('asana_cache');
location.reload();
```

---

## Anti-Patterns to Avoid

### ❌ Anti-Pattern 1: Modifying Task During Render

**BAD:**
```javascript
function renderTask(task, level) {
    // Modifying task during render!
    task.displayName = task.name.toUpperCase();
    
    return `<div>${task.displayName}</div>`;
}
```

**Why Bad:**
- Render functions should be read-only
- Side effects make debugging hard
- Breaks cache (object modified, cache out of sync)

**GOOD:**
```javascript
function renderTask(task, level) {
    // Compute display value without modifying task
    const displayName = task.name.toUpperCase();
    
    return `<div>${displayName}</div>`;
}
```

### ❌ Anti-Pattern 2: Building Hierarchy Before Loading Subtasks

**BAD:**
```javascript
// Load assigned tasks
const tasks = await fetchAssignedTasks();

// Build hierarchy (subtasks not loaded yet!)
rootItems = determineRoots(tasks);

// Load subtasks (too late!)
for (const task of tasks) {
    await loadSubtasks(task.gid);
}
```

**Why Bad:**
- Hierarchy building needs ALL tasks to determine roots
- Missing subtasks means wrong roots
- Historical bug (Session 13)

**GOOD:**
```javascript
// Load assigned tasks
const tasks = await fetchAssignedTasks();

// Load ALL subtasks FIRST
for (const task of tasks) {
    await loadSubtasks(task.gid);
}

// NOW build hierarchy (allTasksMap is complete)
rootItems = determineRoots(tasks);
```

### ❌ Anti-Pattern 3: Forgetting to Update Cache

**BAD:**
```javascript
async function toggleTaskCompletion(taskGid) {
    // Update Asana
    await updateTaskInAsana(taskGid, { completed: true });
    
    // Update local state
    task.completed = true;
    
    // Forgot to update cache!
    // Next page load will show old state!
}
```

**GOOD:**
```javascript
async function toggleTaskCompletion(taskGid) {
    await updateTaskInAsana(taskGid, { completed: true });
    task.completed = true;
    
    // Update cache
    saveCache({
        allTasksMap: Array.from(allTasksMap.entries()),
        rootItems: rootItems,
        allParents: Array.from(allParents)
    });
}
```

### ❌ Anti-Pattern 4: Clearing Cache When Not Needed

**BAD:**
```javascript
function applyFilters() {
    // Parent filter changed
    filterState.parentFilter = newValue;
    
    // Clearing cache unnecessarily!
    clearCache();
    await loadUserTasks(); // Slow full reload
}
```

**Why Bad:**
- Parent filter is client-side only
- No need to fetch from API again
- Wastes bandwidth and time

**GOOD:**
```javascript
function applyFilters() {
    filterState.parentFilter = newValue;
    
    // Just re-display with existing data
    displayTasks(); // Fast
}
```

### ❌ Anti-Pattern 5: Synchronous API Calls in Loops

**BAD:**
```javascript
// Blocks UI for entire duration
for (const task of tasks) {
    const subtasks = await loadSubtasks(task.gid); // Sequential
}
```

**Why Bad:**
- Each call waits for previous to finish
- UI freezes (no progress updates)
- Slow (5-10 seconds feels like forever)

**BETTER (Current Approach):**
```javascript
for (let i = 0; i < tasks.length; i++) {
    await loadSubtasks(tasks[i].gid);
    
    // Update progress every 5 iterations
    if (i % 5 === 0) {
        updateProgress(i, tasks.length, 'Loading...');
        // Allows UI to repaint
    }
}
```

**BEST (Future Optimization):**
```javascript
// Parallel with Promise.all, chunked
const chunkSize = 10;
for (let i = 0; i < tasks.length; i += chunkSize) {
    const chunk = tasks.slice(i, i + chunkSize);
    await Promise.all(chunk.map(t => loadSubtasks(t.gid)));
    updateProgress(i, tasks.length, 'Loading...');
}
```

### ❌ Anti-Pattern 6: Creating Multiple Editor Instances

**BAD:**
```javascript
function editTaskNotes(taskGid) {
    // Creates new editor every time!
    const editor = new Quill('#editor', {...});
    // Old editor not cleaned up (memory leak)
}
```

**GOOD:**
```javascript
let activeEditor = null; // Global reference

function editTaskNotes(taskGid) {
    // Clean up old editor
    if (activeEditor) {
        activeEditor.container = null;
    }
    
    // Create new editor
    activeEditor = new Quill('#editor', {...});
}
```

### ❌ Anti-Pattern 7: Not Escaping User Content

**BAD:**
```javascript
function renderTask(task) {
    // XSS vulnerability!
    return `<div>${task.name}</div>`;
}

// If task.name = "<script>alert('XSS')</script>"
// Script executes!
```

**GOOD:**
```javascript
function renderTask(task) {
    // Escape HTML
    return `<div>${escapeHtml(task.name)}</div>`;
}

function escapeHtml(text) {
    const div = document.createElement('div');
    div.textContent = text; // textContent auto-escapes
    return div.innerHTML;
}
```

### ❌ Anti-Pattern 8: Infinite Loops in Parent Chain Walking

**BAD:**
```javascript
function getParentChain(taskGid) {
    const chain = [];
    let current = allTasksMap.get(taskGid);
    
    // No loop detection!
    while (current.parent) {
        current = allTasksMap.get(current.parent.gid);
        chain.push(current);
        // If circular reference, infinite loop!
    }
    
    return chain;
}
```

**GOOD:**
```javascript
function getParentChain(taskGid) {
    const chain = [];
    const visited = new Set(); // Prevent infinite loops
    let current = allTasksMap.get(taskGid);
    
    while (current.parent && !visited.has(current.gid)) {
        visited.add(current.gid);
        current = allTasksMap.get(current.parent.gid);
        if (current) chain.push(current);
    }
    
    return chain;
}
```

---

## Conclusion

This documentation covers the complete architecture, data flow, and implementation details of TaskRaptor. Key takeaways:

1. **Single-file architecture** trades code organization for deployment simplicity
2. **Global state pattern** works because app is small and single-user
3. **Caching strategy** is critical for performance (instant load vs 5-10 second load)
4. **Hierarchy building** is the most complex logic - understand the flow before modifying
5. **API integration** uses standard REST patterns with careful pagination
6. **User-centric view** fundamentally changes how tasks are displayed vs Asana native

When making changes:
- Always test hierarchy building with various scenarios
- Understand cache invalidation rules
- Follow the data flow from API → Map → Render
- Use console logging to debug state issues
- Test without cache to verify loading logic

The app is production-ready but has room for improvements (see TODO.md). Major architectural changes should preserve the single-file structure and client-side-only design unless there's a compelling reason to change.
