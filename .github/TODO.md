# TaskRaptor ODO List

## Toolbar
- [ ] Rename "Uncompleted" to "Incomplete Only" and "Completed" to "Completed Only" for clarity
- [ ] Fix bug where changing filter criteria does not immediately update the chips display
- [ ] Persist "Sort by" selection across sessions using the same method as other user preferences
- [ ] When user deletes a filter chip, add an "Apply Filter Changes" button to confirm the action before updating the task list, to prevent accidental deletions and allow multiple changes before applying
- [ ] Replace logout confirmation dialog with a modal that includes "Cancel" and "Logout" buttons for better user experience
- [ ] Make toolbar floating and docked to the top of the viewport when scrolling through long task lists, to keep filter and sort options easily accessible. When scrolling down, the toolbar should minimize to a compact version showing only icons, and expand back to full view when scrolling up. Add a subtle shadow effect to the floating toolbar to distinguish it from the content below.

## Text Entry Fields
- [ ] Limit initial display of Notes and Comments to 3 lines of text with a "Show More" option. Clicking "Show More" expands to show the full content. When expanded, provide a "Show Less" option to collapse back to 3 lines. The initial display should show the plain text version of the content (task.notes), and the expanded view should show the full rich text (task.html_notes).
- [ ] Fix bug where bullet points are displayed outside of the box on the left side in Notes and Comments fields due to incorrect indentation handling
- [ ] Move "Edit" button for Notes and Comments to the top-right corner of the text box for cleaner UI and screenspace optimization

## Tasks

### Tasks with Breadcrumbs
- [ ] Move breadcrumb trail below the title text to keep title area clearly visible and stylistically consistent with other items
- [ ] Extend clickable area of title bar to include breadcrumb trail

### Task Completion
- [ ] If the task has incomplete subtasks, open a modal to prompt the user if they'd like to also mark all subtasks as complete
- [ ] Automatically collapse the completed task to reduce visual clutter

### General Task Features
- [ ] Add a "Copy Link" button to each task for easy sharing
- [ ] Change the text entry process for "Rename" and "Add Subtask" to use inline editing instead of whatever method is currently implemented, for a more seamless user experience
- [ ] When a root-level task does not have a due date, but a subtask does, display the due date of the nearest (soonest due date) subtask in the root task's due date field, to provide better visibility of upcoming deadlines. There must be a visual indicator to show that the due date is inherited from a subtask, and clicking on the due date should navigate to the relevant subtask.

## Performance
- [ ] Implement batching of API requests when loading subtasks to reduce the number of network calls and improve load times. For example, load subtasks for up to 10 tasks in a single API request using Promise.all to parallelize requests.
- [ ] Load highest hierarchy level tasks first and render them immediately, then progressively load and render subtasks in the background to improve perceived performance and allow users to start interacting with the task list sooner.
- [ ] Research Asana API capabilities to see if there is a way to fetch tasks along with their subtasks in a single request, potentially using query parameters or expanding related entities, to reduce the number of API calls needed.
- [ ] Optimize API calls by checking for existing data in memory before making network requests, to reduce redundant data fetching and improve load times. Example: Subtask needs to build parent chain - check if parent is already in allTasksMap before making API call.

## Configuration
- [ ] Make workspace ID configurable instead of hardcoded (currently `219640683144922`). Add workspace selector to UI or prompt on first launch, store in localStorage. Research whether it's possible to get the user's workspace from the PAT.

## Code Refactoring
- [ ] Refactor modal system to use generic modal functions consistently. Currently `showInfoModal` uses `showGenericModal` properly, but `showFilterModal` and `closeFilterModal` are hardcoded to specific DOM elements. All modals should use the generic modal system to improve maintainability and reduce code duplication.

## Stretch Goal Features
- [ ] "Minimap" feature for quick navigation through long task lists
- [ ] "Dark mode" theme for better usability in low-light environments
- [ ] Drag-and-drop functionality for reordering sub-tasks
- [ ] Convert to Progressive Web App (PWA) for true offline capability with service workers and manifest
- [ ] Replace all UI components with equivalent components from the [Material 3 Design](https://m3.material.io/) system for a more up-to-date and cohesive design language
- [ ] Replace all emojis with equivalent icons from the [Material Symbols (new) Rounded](https://fonts.google.com/icons?icon.set=Material+Symbols&icon.style=Rounded) collection for a consistent and modern look

---

## Code Analysis

### DRY Violations

#### 1. Duplicate API URL Construction Pattern
**Location**: Multiple functions (`loadUserTasks`, `getTaskDetails`, `loadSubtasksFiltered`, etc.)
**Issue**: Each function constructs fetch URLs with similar patterns, repeating headers and error handling
**Example**:
```javascript
// Repeated in ~10 places:
const response = await fetch(url, {
    headers: { 'Authorization': `Bearer ${personalAccessToken}` }
});
if (!response.ok) throw new Error('Failed to ...');
```
**Recommendation**: Create a helper function:
```javascript
async function asanaFetch(endpoint, options = {}) {
    const response = await fetch(`https://app.asana.com/api/1.0${endpoint}`, {
        ...options,
        headers: {
            'Authorization': `Bearer ${personalAccessToken}`,
            ...options.headers
        }
    });
    if (!response.ok) {
        const errorData = await response.json();
        throw new Error(errorData.errors?.[0]?.message || 'Request failed');
    }
    return response.json();
}
```

#### 2. Duplicate Progress Update Logic
**Location**: `loadUserTasks` function
**Issue**: Progress calculation logic is repeated with slight variations:
```javascript
const fetchProgress = Math.min(30, (allTasks.length / 200) * 30);
setProgress(fetchProgress, `Fetched...`);
// Later:
const subtaskProgress = 40 + ((i / filteredTasks.length) * 30);
setProgress(subtaskProgress, `Loading subtasks...`);
```
**Recommendation**: Create progress stage helper:
```javascript
function updateStageProgress(stage, current, total, message) {
    const stages = {
        fetch: { start: 0, end: 30 },
        subtasks: { start: 40, end: 70 },
        hierarchy: { start: 70, end: 90 }
    };
    const { start, end } = stages[stage];
    const percent = start + ((current / total) * (end - start));
    updateProgress(percent, 100, message);
}
```

#### 3. Duplicate Editor Initialization
**Location**: `editTaskNotes`, `showCommentBox`, `editComment`
**Issue**: Quill editor configuration repeated three times with same settings
**Recommendation**: Create `createEditor(elementId, initialContent)` helper function

#### 4. Duplicate Task Styling Update
**Location**: `toggleTaskCompletion` function
**Issue**: Completion styling logic repeated for task element and parent elements
**Recommendation**: Extract to `updateTaskCompletionStyling(taskGid, completed)` function

#### 5. Duplicate Cache Save Calls
**Location**: Multiple edit functions (8+ locations)
**Issue**: Same three-line cache save repeated everywhere:
```javascript
saveCache({
    allTasksMap: Array.from(allTasksMap.entries()),
    rootItems: rootItems,
    allParents: Array.from(allParents)
});
```
**Recommendation**: Create `saveCacheFromGlobalState()` function that reads globals

### Code Smells

#### 1. God Function: `loadUserTasks`
**Issue**: 250+ lines, handles fetching, filtering, hierarchy building, caching, progress updates, error handling
**Severity**: High
**Impact**: Hard to test, debug, and modify
**Recommendation**: Split into separate functions:
- `fetchAllAssignedTasks()` - API pagination
- `filterAndInitializeTasks()` - Date filtering + structure setup
- `buildTaskHierarchy()` - Hierarchy logic
- `loadUserTasks()` - Orchestrates the above with progress tracking

#### 2. God Function: `renderTask`
**Issue**: 140+ lines, handles breadcrumbs, styling, event handlers, auto-expand, placeholders
**Severity**: Medium
**Impact**: Hard to modify rendering logic
**Recommendation**: Split into:
- `buildTaskHeader()` - Create header HTML
- `buildBreadcrumb()` - Breadcrumb generation
- `applyTaskStyling()` - Completion/display styling
- `renderTask()` - Orchestrates the above

#### 3. Magic Numbers
**Issue**: Hardcoded values scattered throughout
**Examples**:
- `if (i % 5 === 0)` - Why 5? (progress update frequency)
- `Math.min(30, ...)` - Why 30? (progress stage end)
- `level * 5` - Why 5? (indentation multiplier)
- `task.commentsShown = 3` - Why 3? (default comment display)
- `ageHours > 24` - Cache expiry time
**Recommendation**: Define constants at top:
```javascript
const CONFIG = {
    PROGRESS_UPDATE_FREQUENCY: 5,
    PROGRESS_STAGES: { FETCH_END: 30, SUBTASKS_START: 40, ... },
    TASK_INDENT_PX: 5,
    DEFAULT_COMMENTS_SHOWN: 3,
    CACHE_EXPIRY_HOURS: 24,
    COMMENTS_LOAD_MORE_COUNT: 5
};
```

#### 4. Inconsistent Error Handling
**Issue**: Some functions show toast + console.error, others just toast, others just console.error
**Example**:
```javascript
// Function A:
catch (error) {
    console.error('Error:', error);
    showToast('Failed: ' + error.message, 'error');
}

// Function B:
catch (error) {
    showToast('Failed', 'error');
    // No console.error
}

// Function C:
catch (error) {
    console.error(error);
    // No toast
}
```
**Recommendation**: Standardize with `handleError(error, userMessage)` function

#### 5. Boolean Trap in Function Parameters
**Issue**: `loadUserTasks(background)` - parameter name doesn't explain what true/false means
**Better**: Use options object: `loadUserTasks({ isBackgroundRefresh: true })`

#### 6. Long Parameter Lists
**Issue**: `renderTask(task, level, isRoot = false)` and similar functions
**Recommendation**: Use options object for 3+ parameters:
```javascript
renderTask({ task, level, isRoot, forceExpand })
```

#### 7. Deeply Nested Conditionals
**Location**: `renderTaskDetails`, `filterTasksByCompletion`, `loadUserTasks`
**Issue**: 4-5 levels of nesting make code hard to follow
**Recommendation**: Use early returns and extract sub-conditions into functions

#### 8. String Concatenation for HTML
**Issue**: Building HTML with template literals throughout
**Security Risk**: Easy to forget escaping (though most places do escape)
**Example**: `breadcrumbHtml += '<div>' + something + '</div>'`
**Recommendation**: Use DOM creation methods or JSX-like template system

#### 9. Global State Mutation Everywhere
**Issue**: 15+ global variables modified from anywhere
**Risk**: Hard to track state changes, debugging difficult
**Current Mitigation**: None (inherent to architecture)
**Recommendation**: Consider state manager pattern for future refactor (but contradicts single-file philosophy)

#### 10. Callback Hell in Event Handlers
**Issue**: Inline onclick handlers everywhere: `onclick="saveComment('${taskGid}')"`
**Problems**:
- Breaks CSP (Content Security Policy) if added later
- Hard to track which functions are actually used
- No event handler cleanup
**Recommendation**: Use addEventListener in JavaScript instead

### Dead Code

#### 1. Unused Function: `linkifyText`
**Location**: Line ~3186 (utility section)
**Issue**: Function defined but never called (superseded by `linkifyHTML`)
**Verification**: Grep search shows definition but no calls
**Action**: Remove function

#### 2. Unused CSS: `.task-item.completed .due-date` opacity rules
**Location**: CSS section
**Issue**: Complex opacity/pointer-events rules that may not work as intended:
```css
.task-item.completed .due-date {
    opacity: 0;
    pointer-events: none;
}
.task-item.completed:hover .due-date {
    opacity: 1;
    pointer-events: auto;
}
```
**Verification**: Due date shown with inline style override, making these rules ineffective
**Action**: Clean up or document intended behavior

#### 3. Unused Global: `window.lastRefreshTime`
**Location**: Line ~1441 in `loadUserTasks`
**Issue**: Set but never read:
```javascript
window.lastRefreshTime = new Date();
```
**Action**: Remove or add feature to display last refresh time in UI

#### 4. Unused Parameter: `parentAssignedToUser` edge case
**Location**: `loadSubtasksFiltered` function
**Issue**: Parameter passed but logic may not fully utilize it correctly
**Verification**: Need to trace all call sites
**Action**: Review logic or simplify parameter

#### 5. Unreachable Code Path
**Location**: `createMissingParentItems` function
**Issue**: Function defined and called, but may not work correctly due to timing:
- Called AFTER subtasks loaded
- But subtasks loading also walks parent chains
- May create duplicates or miss items
**Action**: Review if function is actually needed or if subtask loading already handles this

### Potential Bugs

#### 1. Race Condition: Background Loading Flag
**Location**: `displayCachedData` and `loadUserTasks`
**Issue**: `window.backgroundLoadingInProgress` flag not always cleared
**Scenario**: If background load fails, flag stays true forever
**Symptom**: Persistent loading spinners
**Fix**: Add try/finally to ensure flag cleared

#### 2. Memory Leak: Editor References
**Location**: Multiple editor creation points
**Issue**: `commentEditors` Map grows but entries never removed
**Scenario**: After 100 task expansions, Map has 100 editor references
**Fix**: Clean up editors when tasks collapsed or navigated away

#### 3. Infinite Loop Risk: Parent Chain Walking
**Location**: `getParentChain`, `getParentChainFromMap`
**Issue**: While code has visited Set, circular references in data could still cause issues
**Scenario**: If Asana API returns circular parent reference (shouldn't happen, but...)
**Current Mitigation**: visited Set (good)
**Enhancement**: Add max depth limit as safety: `while (current && depth < 100)`

#### 4. localStorage Quota Exceeded Not Handled
**Location**: All `localStorage.setItem` calls
**Issue**: No try/catch for quota exceeded errors
**Symptom**: Silent failure, cache not saved
**Fix**: Wrap in try/catch, show toast warning user

#### 5. XSS Risk: Comment HTML
**Location**: `renderTaskDetails` when displaying comments
**Issue**: Uses `comment.html_text` directly from API
**Current State**: Trusts Asana API to sanitize
**Risk**: If Asana API bug returns malicious HTML, XSS possible
**Mitigation**: Consider DOMPurify library or additional sanitization

#### 6. State Inconsistency: Expand States
**Location**: `expandStates` object
**Issue**: Task GIDs can be removed from allTasksMap (filter changes), but expandStates keeps old GIDs
**Result**: Growing object with stale entries
**Fix**: Periodic cleanup or clear on data reload

### Performance Issues

#### 1. N+1 Query Problem: Subtasks
**Location**: `loadUserTasks` - subtasks loading
**Issue**: One API call per task to load subtasks (can be 100+ calls)
**Current**: Sequential with progress updates
**Optimization**: Batch into groups of 10, use Promise.all:
```javascript
const batchSize = 10;
for (let i = 0; i < tasks.length; i += batchSize) {
    const batch = tasks.slice(i, i + batchSize);
    await Promise.all(batch.map(t => loadSubtasksFiltered(t.gid)));
}
```
**Impact**: Could reduce load time from 10s to 3-4s

#### 2. Repeated DOM Queries
**Location**: Throughout rendering code
**Issue**: `document.getElementById` called multiple times for same element:
```javascript
const detailsDiv = document.getElementById('details-' + taskGid);
// ... 10 lines later ...
const detailsDiv2 = document.getElementById('details-' + taskGid);
```
**Fix**: Cache DOM references in variables

#### 3. Unnecessary Array Conversions
**Location**: Cache save/load
**Issue**: Converting Map to Array and back on every cache operation:
```javascript
allTasksMap: Array.from(allTasksMap.entries())  // Expensive
```
**Optimization**: Only convert when actually saving/loading, not during intermediate operations

#### 4. No Debouncing on Filter Changes
**Location**: Filter modal and chip deletion
**Issue**: Every filter change triggers full re-render
**Scenario**: User rapidly clicks filter chips - renders 5 times
**Fix**: Debounce displayTasks() by 100-200ms

#### 5. Large innerHTML Assignments
**Location**: `displayTasks` and `renderTaskDetails`
**Issue**: Setting innerHTML of large strings causes full reparse
**Current**: Acceptable for current scale
**Future**: If 1000+ tasks, consider virtual scrolling or incremental rendering

### Maintainability Issues

#### 1. No Type Definitions
**Issue**: No JSDoc or TypeScript types
**Impact**: Easy to pass wrong types, hard to know function contracts
**Recommendation**: Add JSDoc comments:
```javascript
/**
 * Load subtasks for a task, filtered by assignee
 * @param {string} taskGid - The task GID to load subtasks for
 * @param {boolean} parentAssignedToUser - Whether parent task is assigned to current user
 * @returns {Promise<void>}
 */
async function loadSubtasksFiltered(taskGid, parentAssignedToUser) {
```

#### 2. No Test Coverage
**Issue**: No unit tests, integration tests, or E2E tests
**Risk**: Regressions when making changes (historical bugs prove this)
**Recommendation**: At minimum, add tests for:
- Hierarchy building logic
- Filter functions
- Date calculations
- Parent chain walking

#### 3. No Version Number
**Issue**: No way to know what version of app user is running
**Impact**: Hard to debug user reports
**Recommendation**: Add version constant and display in UI:
```javascript
const APP_VERSION = '1.0.0';
// Show in footer or debug info
```

#### 4. No Feature Flags
**Issue**: No way to enable/disable experimental features
**Example**: If adding new feature, can't A/B test or gradual rollout
**Recommendation**: Add simple flags object in localStorage

#### 5. No Error Boundary
**Issue**: Uncaught errors crash entire app
**Current**: Some try/catch, but not comprehensive
**Recommendation**: Add global error handler:
```javascript
window.onerror = function(msg, url, line) {
    showToast('An error occurred. Please refresh.', 'error');
    console.error('Global error:', msg, url, line);
};
```

### Security Issues

#### 1. PAT Storage in Plain Text
**Severity**: Medium (documented limitation, user should know)
**Risk**: Anyone with file system access can steal PAT
**Mitigation**: Document clearly, recommend least-privilege PAT
**Future**: Consider optional encryption with user-provided password

#### 2. No CSP (Content Security Policy)
**Issue**: Inline event handlers and styles violate CSP
**Impact**: Can't add CSP headers without refactoring
**Recommendation**: Move all onclick handlers to addEventListener

#### 3. No Input Validation
**Location**: Prompt dialogs for rename, create subtask, etc.
**Issue**: No length limits, no special character handling
**Risk**: Could send malformed data to API (API will reject, but UX is poor)
**Fix**: Add validation before API calls

#### 4. Trusting API Response Structure
**Location**: Throughout (e.g., `data.data.gid`)
**Issue**: No validation of API response shape
**Risk**: If API changes, app breaks with cryptic errors
**Fix**: Add response validation:
```javascript
function validateTaskResponse(data) {
    if (!data?.data?.gid) throw new Error('Invalid task response');
    return data.data;
}
```Test change at 12/08/2025 13:23:10
