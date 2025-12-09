# TaskRaptor Action Plan

## Priority Analysis

Based on impact, user value, and dependencies, here's the recommended prioritization for implementing TODO items.

---

## **🔴 CRITICAL PRIORITY (Fix First)**

### **1. Race Condition: Background Loading Flag** (Potential Bug #1)
- **Why**: Causes persistent loading spinners, breaks app functionality
- **Impact**: High - directly affects UX
- **Effort**: Low - simple try/finally fix
- **Risk**: Low
- **Location**: `displayCachedData` and `loadUserTasks`
- **Issue**: `window.backgroundLoadingInProgress` flag not always cleared
- **Fix**: Add try/finally to ensure flag cleared

### **2. localStorage Quota Exceeded Not Handled** (Potential Bug #4)
- **Why**: Silent failure means users lose data without knowing
- **Impact**: High - data loss
- **Effort**: Low - add try/catch blocks
- **Risk**: Low
- **Location**: All `localStorage.setItem` calls
- **Issue**: No try/catch for quota exceeded errors
- **Fix**: Wrap in try/catch, show toast warning user

### **3. Memory Leak: Editor References** (Potential Bug #2)
- **Why**: Grows over time, eventually slows app
- **Impact**: Medium - performance degrades
- **Effort**: Low - cleanup on collapse
- **Risk**: Low
- **Location**: Multiple editor creation points
- **Issue**: `commentEditors` Map grows but entries never removed
- **Fix**: Clean up editors when tasks collapsed or navigated away

---

## **🟡 HIGH PRIORITY (User-Facing Improvements)**

### **4. Task Completion Modal for Subtasks** (Tasks > Task Completion)
- **Why**: Prevents accidental incomplete subtasks
- **Impact**: High - workflow improvement
- **Effort**: Medium - modal + logic
- **Risk**: Low
- **TODO Item**: If the task has incomplete subtasks, open a modal to prompt the user if they'd like to also mark all subtasks as complete

### **5. Auto-collapse Completed Tasks** (Tasks > Task Completion)
- **Why**: Reduces visual clutter immediately
- **Impact**: Medium - better UX
- **Effort**: Low - one line of code
- **Risk**: Low
- **Dependency**: Works well with #4
- **TODO Item**: Automatically collapse the completed task to reduce visual clutter

### **6. Rename Filter Labels** (Toolbar)
- **Why**: "Incomplete Only" is clearer than "Uncompleted"
- **Impact**: Low - but easy win
- **Effort**: Very Low - string replacement
- **Risk**: None
- **TODO Item**: Rename "Uncompleted" to "Incomplete Only" and "Completed" to "Completed Only" for clarity

---

## **🟢 MEDIUM PRIORITY (Performance & Polish)**

### **7. Batch API Requests for Subtasks** (Performance)
- **Why**: 50-70% load time reduction
- **Impact**: High - speed improvement
- **Effort**: Medium - refactor loading logic
- **Risk**: Medium - must test thoroughly
- **TODO Item**: Implement batching of API requests when loading subtasks to reduce the number of network calls and improve load times. For example, load subtasks for up to 10 tasks in a single API request using Promise.all to parallelize requests.

### **8. Progressive Loading** (Performance)
- **Why**: Users can interact sooner
- **Impact**: High - perceived performance
- **Effort**: High - significant refactor
- **Risk**: Medium
- **Dependency**: Should do after #7
- **TODO Item**: Load highest hierarchy level tasks first and render them immediately, then progressively load and render subtasks in the background to improve perceived performance and allow users to start interacting with the task list sooner.

### **9. Notes/Comments 3-Line Limit** (Text Entry Fields)
- **Why**: Cleaner UI, less scrolling
- **Impact**: Medium - UX improvement
- **Effort**: Medium - expand/collapse logic
- **Risk**: Low
- **TODO Item**: Limit initial display of Notes and Comments to 3 lines of text with a "Show More" option. Clicking "Show More" expands to show the full content. When expanded, provide a "Show Less" option to collapse back to 3 lines.

### **10. Fix Filter Chip Bug** (Toolbar)
- **Why**: UI feels broken when chips don't update
- **Impact**: Medium - polish
- **Effort**: Low - debugging + fix
- **Risk**: Low
- **TODO Item**: Fix bug where changing filter criteria does not immediately update the chips display

---

## **⚪ LOW PRIORITY (Nice to Have)**

### **11. Code Refactoring** (Various DRY violations and code smells)
- **Why**: Maintainability, not user-facing
- **Impact**: Low - internal quality
- **Effort**: High - time consuming
- **Risk**: Medium - potential for regressions
- **Recommendation**: Do incrementally alongside feature work
- **Items**:
  - Duplicate API URL Construction Pattern
  - Duplicate Progress Update Logic
  - Duplicate Editor Initialization
  - Duplicate Task Styling Update
  - Duplicate Cache Save Calls
  - God Function: `loadUserTasks`
  - God Function: `renderTask`
  - Magic Numbers
  - Inconsistent Error Handling
  - And more...

### **12. Stretch Goals** (Dark mode, PWA, Material 3, etc.)
- **Why**: Large scope, not core functionality
- **Impact**: Varies
- **Effort**: Very High
- **Risk**: High
- **Recommendation**: Defer until core features stable
- **Items**:
  - "Minimap" feature for quick navigation
  - "Dark mode" theme
  - Drag-and-drop functionality for reordering sub-tasks
  - Convert to Progressive Web App (PWA)
  - Replace UI with Material 3 Design components
  - Replace emojis with Material Symbols icons

---

## **Recommended Implementation Phases**

### **Phase 1: Quick Wins (1-2 hours)**
1. ✅ Fix background loading flag race condition
2. ✅ Add localStorage quota error handling
3. ✅ Fix memory leak in editor references
4. ✅ Rename filter labels ("Incomplete Only")
5. ✅ Auto-collapse completed tasks

**Benefits**: Immediate stability improvements and UX polish with minimal risk

---

### **Phase 2: User Experience (2-4 hours)**
6. ✅ Task completion modal for subtasks
7. ✅ Fix filter chip display bug
8. ✅ Notes/Comments 3-line limit with expand

**Benefits**: Noticeable UX improvements that users will appreciate

---

### **Phase 3: Performance (4-6 hours)**
9. ✅ Batch API requests for subtasks
10. ✅ Progressive loading of tasks

**Benefits**: Significantly faster load times, especially for users with many tasks

---

## **Additional Items from TODO List**

### **Toolbar (Not Yet Prioritized)**
- [ ] Persist "Sort by" selection across sessions
- [ ] "Apply Filter Changes" button when deleting filter chips
- [ ] Replace logout confirmation dialog with modal
- [ ] Make toolbar floating/docked when scrolling

### **Text Entry Fields (Not Yet Prioritized)**
- [ ] Fix bullet point indentation bug
- [ ] Move "Edit" button to top-right corner

### **Tasks (Not Yet Prioritized)**
- [ ] Move breadcrumb trail below title text
- [ ] Extend clickable area to include breadcrumb
- [ ] Add "Copy Link" button
- [ ] Change "Rename" and "Add Subtask" to inline editing
- [ ] Show inherited due dates from subtasks

### **Performance (Not Yet Prioritized)**
- [ ] Research Asana API capabilities for batch fetching
- [ ] Optimize API calls by checking memory before network requests

### **Code Quality (Not Yet Prioritized)**
- [ ] Refactor modal system for consistency
- [ ] Remove dead code (unused functions, CSS, variables)
- [ ] Fix potential bugs (infinite loops, state inconsistencies, XSS risks)
- [ ] Add type definitions (JSDoc)
- [ ] Add test coverage
- [ ] Add version number
- [ ] Add feature flags
- [ ] Add error boundary
- [ ] Fix security issues (CSP, input validation, API response validation)

---

## **Next Steps**

1. Review this plan and confirm priorities align with project goals
2. Start with Phase 1 items (quick wins with high impact)
3. Test each fix thoroughly before moving to next item
4. Update TODO.md to mark completed items
5. Update documentation as needed for each change
