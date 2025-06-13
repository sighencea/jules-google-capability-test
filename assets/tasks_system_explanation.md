# Tasks System Explanation

This document provides a detailed explanation of the tasks system, including task creation, visibility, display, editing, and the underlying data model and security policies.

## 1. Core Concepts and Data Model

The tasks system revolves around several key database tables and a view:

*   **`tasks`**: The central table holding core task information.
    *   *Key Columns*: `task_id`, `property_id`, `task_title`, `task_description`, `task_status`, `task_priority`, `task_due_date`, `company_id`, `created_by`, `task_notes`.
*   **`task_assignments`**: Links tasks to users (staff members).
    *   *Key Columns*: `id`, `task_id`, `user_id`, `assigned_at`.
*   **`task_files`**: Stores metadata for files attached to tasks.
    *   *Key Columns*: `id`, `task_id`, `file_name`, `storage_path`, `mime_type`, `uploaded_by`, `is_deleted`.
*   **`detailed_task_assignments` (View)**: A view that likely combines `tasks` and `task_assignments` with `profiles` to provide richer details about assignees (e.g., first name, last name, email) for easier display.

Row Level Security (RLS) policies are heavily used to control data access.

## 2. Task Creation

The ability to create tasks is restricted to administrators.

*   **Initiation (Admin Only)**:
    *   The "Create Task" button (ID: `addNewTaskBtn` on `pages/tasks.html`) is only made visible to users identified as administrators.
    *   The `js/tasks-display.js` script checks the `is_admin` flag from the user's profile (`getCurrentUserProfile()` function) and dynamically shows/hides this button.

*   **Client-Side Process (`js/tasks-display.js` - Add New Task Modal)**:
    1.  When an admin clicks "Create Task", the "Add New Task" modal (ID: `addNewTaskModal`) is displayed.
    2.  **Dropdown Population**: The `populateCreateTaskModalDropdowns()` function is called. It fetches:
        *   Properties associated with the admin's `company_id` to populate the "Property" select list (`taskPropertySelect`).
        *   Staff members (profiles) belonging to the admin's `company_id` (including the admin themselves if applicable) to populate the "Assign to Staff" select list (`taskStaffSelect`).
    3.  **Admin Input**: The admin fills in the task details: Title, Description, Due Date, Priority, Status, and selects a Property and an Assignee.
    4.  **Submission**: Clicking "Save Task" (ID: `saveNewTaskBtn`):
        *   Collects all form data.
        *   Performs basic client-side validation.
        *   Invokes the `create-task` Supabase Edge Function with the task payload.

*   **Server-Side Process (`supabase/functions/create-task/index.ts`)**:
    1.  **Authentication & Authorization**:
        *   Ensures the calling user is authenticated.
        *   Verifies the user is an administrator by checking `is_admin` in their `profiles` record.
        *   Retrieves the admin's `company_id`.
    2.  **Payload Validation**: Checks for required fields in the received payload (title, property ID, staff ID for assignment, status, priority).
    3.  **Cross-Company Validation (Security)**:
        *   Verifies that the `property_id` submitted belongs to the admin's `company_id`.
        *   Verifies that the `staff_id` (assignee) submitted also belongs to the admin's `company_id`.
        This prevents admins from creating tasks for properties or assigning tasks to staff outside their own company.
    4.  **Task Insertion**:
        *   A new task record is prepared, including `task_title`, `property_id`, `task_status`, `task_priority`, `company_id` (admin's company), and `created_by` (admin's user ID).
        *   This record is inserted into the `tasks` table.
    5.  **Task Assignment Insertion**:
        *   After the task is successfully inserted and its `task_id` is retrieved, a new record is inserted into the `task_assignments` table, linking the `task_id` with the selected `staff_id`.
    6.  **Response**: Returns a success or error message.

*   **RLS Policy for `tasks` Table (INSERT)**:
    *   Policy: `"Allow creation of tasks for owned companies"`
    *   The Edge Function `create-task` effectively pre-enforces this by ensuring the `company_id` in the task record matches the admin's company. The RLS policy on the database provides a secondary layer of defense if direct DB access were attempted.

## 3. Task Visibility and Display

What tasks a user sees is governed by RLS policies and implemented in the client-side display logic.

*   **Client-Side Fetching & Rendering (`js/tasks-display.js`)**:
    1.  **`fetchTasksAndRelatedData()`**: This function is responsible for fetching the list of tasks to display.
        *   It performs a `SELECT` query on the `tasks` table.
        *   Crucially, it also selects related data: `properties ( property_name )` and `detailed_task_assignments ( assignee_first_name, assignee_last_name, ... )`. This implies that the RLS policies on `tasks`, `properties`, and `detailed_task_assignments` (or its underlying tables) will collectively determine what data is returned based on the authenticated user.
    2.  **`renderTasks(tasks)`**:
        *   Takes the fetched (and RLS-filtered) tasks and dynamically creates HTML table rows in `tasksTableBody` on `pages/tasks.html`.
        *   Displays key information: title, property, primary assignee, status (with a colored badge), and due date.
        *   Includes "View" and "Edit" action buttons for each task.
    3.  **Filtering**:
        *   The UI provides options to filter tasks by search term, status, and property.
        *   For administrators, an additional filter for staff members is available. This client-side filtering operates on the data already fetched (and thus already RLS-filtered).

*   **RLS Policies for `tasks` Table (SELECT)**:
    *   `"Allow read access to tasks of owned companies"`: Users can see all tasks associated with their `company_id`. The application logic implies a user's `company_id` is derived from their `profiles` record.
    *   `"Allow users to view their assigned tasks (v2)"`: Users can see tasks for which they have an entry in the `task_assignments` table.
    *   **Combined Effect**: A regular staff member will typically see tasks from their company that are either assigned to them or generally visible to everyone in the company (if the first policy is broad enough). Admins, also being part of a company, would see all tasks in their company.

*   **RLS Policies for `task_assignments` Table (SELECT)**:
    *   `"Admin can read task assignments in own company"`
    *   `"Allow user to see their own task assignments"`
    *   These ensure that when task details are fetched (especially via `detailed_task_assignments`), users only see assignment details they are permitted to.

*   **RLS for `detailed_task_assignments` View**: While not explicitly provided, the RLS for this view would need to be consistent with the underlying `tasks`, `task_assignments`, and `profiles` tables to prevent data leakage. It likely allows users to see assignment details for tasks they can already see.

## 4. Viewing Task Details (Details Modal)

Users can view more detailed information about a specific task.

*   **Client-Side (`js/tasks-display.js` - View Task Modal)**:
    1.  Clicking the "View" button on a task row triggers this.
    2.  **`fetchSingleTaskDetails(taskId)`**: This function is called to get all details for the specified `taskId`.
        *   It fetches the task record, its related property name, detailed assignee information, and any associated (non-deleted) `task_files`.
    3.  **Modal Population**: The "View Task Modal" (ID: `viewTaskModal`) is populated with all retrieved information, including title, property, assignees, status, priority, due date, description, notes, and lists of attached images and documents.

*   **RLS Policy for `task_files` Table (SELECT)**:
    *   Intended Policy: `"Allow users to see files for tasks they can access"`
    *   This ensures that users can only view files attached to tasks they are otherwise permitted to see (as per task visibility rules).
    *   *(Note: User mentioned RLS for `task_files` might be currently disabled but these are the intended rules).*

## 5. Task Editing

Task editing capabilities are available, with permissions governed by RLS.

*   **Client-Side (`js/tasks-display.js` - Edit Task Modal)**:
    1.  Clicking the "Edit" button on a task row triggers this.
    2.  **Data Population**: The `editTaskForm` within the "Edit Task Modal" (ID: `editTaskModal`) is populated using `fetchSingleTaskDetails(taskId)`.
        *   Editable fields include title, description, status, priority, due date, and notes.
        *   The property is display-only (cannot be changed once set).
        *   Assignee selection (`editTaskAssigneeSelect`) is currently a placeholder, showing the current assignee(s) but noting full re-assignment UI is a future enhancement.
    3.  **File Management**:
        *   Existing attached images and documents are listed.
        *   Users can mark existing files for deletion (soft delete). An array `filesToDeleteInEditModal` tracks these.
        *   Users can upload new images (to `task-images` Supabase Storage bucket) and documents (to `task-documents` Supabase Storage bucket).
    4.  **Saving Changes (`handleEditTaskFormSubmit(event)`)**: This complex function orchestrates multiple operations:
        *   **File Deletions**: Iterates through `filesToDeleteInEditModal` and updates the `is_deleted` flag to `true` for those records in the `task_files` table.
        *   **File Uploads**:
            *   Uploads new files to the appropriate Supabase Storage bucket (path: `{user.id}/{taskId}/{timestamp}_{filename}`).
            *   On successful storage upload, inserts new metadata records into the `task_files` table.
        *   **Core Task Update**: Updates the main details (title, description, status, etc., including `task_notes`) in the `tasks` table. Sets `task_updated_at`.
        *   **Assignee Change (Simplified)**: If a new assignee is indicated (from the placeholder UI), it attempts to delete all existing assignments for the task and then insert a new one for the selected user. This is noted as needing more robust logic.
        *   **Error Handling**: Collects and displays any errors encountered during these steps.
        *   On success, refreshes the main task list and hides the modal.

*   **RLS Policies for `tasks` Table (UPDATE)**:
    *   `"Allow assigned users to update task details"`: Users directly assigned to a task can modify it.
    *   `"Allow update access to tasks of owned companies"`: Users (likely admins or those with broader company-level permissions) can update any task within their company.
    *   The client-side script allows users to open the edit modal for any task they can see; however, the actual save operation (`UPDATE` query) will succeed or fail based on these RLS policies.

*   **RLS Policies for `task_files` Table (INSERT/UPDATE)**:
    *   Intended INSERT Policy: `"Allow users to insert file metadata for tasks they can edit"`
    *   Intended UPDATE Policy (for soft delete): `"Allow users to soft delete files for tasks they can edit"`
    *   These tie file management permissions to task editing permissions.

*   **RLS Policies for `task_assignments` Table (INSERT/DELETE - relevant to assignee changes in edit mode)**:
    *   `"Allow company owner to create task assignments in their company"`
    *   `"Allow company owner to delete task assignments in their company"`
    *   `"Allow user to delete their own task assignment"` (Could allow a user to unassign themselves if the edit UI supported it).
    *   The current simplified assignee change in the edit modal would primarily work for company owners due to the delete-all-then-add-one approach.

## 6. Task Deletion

*   **RLS Policy for `tasks` Table (DELETE)**: `"Allow deletion of tasks of owned companies"` exists.
*   **Current UI**: The `pages/tasks.html` and `js/tasks-display.js` (as reviewed) do not show an explicit "Delete Task" button or UI element in the main task list or modals.
*   **Implication**: Task deletion might be a capability reserved for direct database operations by authorized personnel or implemented in a part of the application not yet reviewed, even though the RLS policy permits it for users within their own company.

## 7. Summary of Key Files and Roles

*   **`pages/tasks.html`**: Provides the HTML structure for the tasks page, including the table display area and all modals (Add, View, Edit).
*   **`js/tasks-display.js`**: Contains the bulk of the client-side logic for:
    *   Determining user admin status for UI adjustments.
    *   Fetching and rendering the task list.
    *   Handling filters and search.
    *   Managing the display and data population of "Add New Task", "View Task", and "Edit Task" modals.
    *   Orchestrating client-side form submissions for creating and editing tasks, including file uploads/deletions.
    *   Calling the `create-task` Supabase Edge Function.
*   **`supabase/functions/create-task/index.ts`**: Securely handles the creation of new tasks and their initial assignment, ensuring only admins can create tasks and that tasks/assignments stay within the admin's company.
*   **Database Schema & RLS Policies**: Defined in SQL migrations, these are fundamental to data integrity and security, controlling who can perform what actions (SELECT, INSERT, UPDATE, DELETE) on the `tasks`, `task_assignments`, and `task_files` tables.

This system uses a combination of client-side logic for user experience and Supabase Edge Functions for secure CUD operations, all underpinned by Row Level Security policies for data access control.
