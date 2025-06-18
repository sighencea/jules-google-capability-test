# Data Access Control Explained

This document details how the application ensures that users can only see and manage information relevant to them and their company, preventing unauthorized access to data from other companies or users.

## 1. Core Principles of Data Access

The application's data security model is built on these foundational principles:

*   **User Authentication**: Every user must log in to the system. Anonymous access to data is generally not permitted. Supabase Authentication handles this, identifying each user uniquely.
*   **Company Association**: This is the cornerstone of data segregation. Nearly every piece of significant data (properties, tasks, staff profiles) is linked to a specific company.
    *   A user's primary link to a company is the `company_id` field in their `public.profiles` record.
    *   **For Company Creators/Admins**: Their `company_id` is established when they set up their company (see `account.html` functionality and the `save-company-details` Edge Function). The `companies` table stores an `owner_id` linking the company to its creator/admin.
    *   **For Invited Staff**: Their `company_id` is inherited from the inviting admin via metadata in the invitation process and is automatically populated into their `profiles` record by the `handle_new_user_profile_setup` database trigger.
*   **Row Level Security (RLS)**: Supabase's RLS is the primary enforcement mechanism at the database layer. RLS policies are rules defined on database tables that automatically filter which rows a user can view, insert, update, or delete based on their user ID, their `company_id`, their role (`is_admin`), or other criteria from their session. **Even if client-side code tries to fetch broader data, RLS will restrict what the database returns.**
*   **Role-Based Logic (Application Layer)**:
    *   The `is_admin` flag in a user's `profiles` record grants them elevated privileges within their company's context, often enabling additional UI elements (e.g., "Create Task" button, "Company Settings" section) or allowing them to pass checks in Edge Functions.
    *   Specific user roles (e.g., 'Electrician') are more for classification than for distinct data access patterns, which are primarily governed by company association and admin status.

## 2. Data Access for `companies` Table

*   **Direct Interaction**: Most users (especially staff) do not interact directly with the `companies` table.
*   **Admin Control**: Company owners/administrators manage their company's details. This is typically done through the "Company Settings" section of the `account.html` page. The `save-company-details` Edge Function handles the logic for creating or updating their company record in the `companies` table, linking it via their `user.id` as the `owner_id`.
*   **RLS**: RLS policies on the `companies` table would likely ensure an admin can only modify the company record where their `user.id` matches the `owner_id`.

## 3. Data Access for `profiles` Table

*   **Own Profile**: Users can generally read and update their own profile information (name, contact details, language preference, avatar). This is handled by `js/account-details.js` which calls Supabase `UPDATE` on the `profiles` table, restricted by RLS to the current user's ID (`profiles.id = auth.uid()`).
*   **Other Profiles (Staff Management)**:
    *   Administrators can view a list of staff members within their company on the `pages/staff.html` page. The `js/staff-management.js` script fetches profiles where `profiles.company_id` matches the admin's company. RLS policies would enforce this, ensuring an admin can only see profiles from their own company.
    *   Regular staff members likely cannot see a list of all other staff profiles unless RLS explicitly allows it (e.g., for a team directory, though current evidence points more to admin-only staff listing).
*   **`is_admin` Flag**: This boolean field in `profiles` is critical. Client-side JavaScript (`js/account-details.js`, `js/tasks-display.js`) and Edge Functions (`create-task`, `invite-staff-member`) check this flag to enable or restrict administrative functionalities.
*   **Profile Creation (Invited Staff)**: The `handle_new_user_profile_setup` database trigger plays a key role by correctly populating `company_id`, `user_role`, and `user_status: 'Invited'` from the invitation metadata (`auth.users.raw_user_meta_data`) into the new staff member's `profiles` record.

## 4. Data Access for `properties` Table

*   **Creation**: When a user (typically an admin or a staff member with permissions, though creation seems admin-centric in current flows) adds a new property using `js/addProperty.js`, the `company_id` of that logged-in user (derived from their profile) is assigned to the new property's `company_id` field.
*   **Visibility (RLS)**:
    *   RLS policies on the `properties` table ensure that users can only `SELECT` (view) properties where the `properties.company_id` matches their own `profiles.company_id`.
    *   This means staff and admins of "Company A" cannot see properties belonging to "Company B".
    *   `js/lazy-load-properties.js` fetches properties, and RLS filters the results automatically.
*   **Modification/Deletion (RLS)**:
    *   Permissions to `UPDATE` or `DELETE` properties are likely restricted by RLS to users within the same company. Admins might have full CRUD rights within their company, while staff might have more limited or no direct modification rights on properties themselves (their interaction might be primarily through tasks related to properties).

## 5. Data Access for Tasks System

This is a multi-table system with specific RLS policies provided by the user.

*   **`tasks` Table**
    *   **Creation**:
        *   **Edge Function (`create-task`)**: Strictly enforces that only users with `profiles.is_admin = true` can create tasks. It also validates that the `property_id` for the task and the `staff_id` for assignment belong to the admin's `company_id`. The task is then inserted with the admin's `company_id` and their `user.id` as `created_by`.
        *   **RLS Policy**: `"Allow creation of tasks for owned companies"` acts as a database-level safeguard. The RLS policy likely checks if the `company_id` being inserted into the new task row matches a `company_id` the current user is authorized for (e.g., `EXISTS (SELECT 1 FROM profiles WHERE profiles.id = auth.uid() AND profiles.company_id = NEW.company_id)`).
    *   **Visibility (SELECT RLS Policies on `tasks`)**:
        *   `"Allow read access to tasks of owned companies"`: This policy allows any user (admin or staff) to see all tasks where the `tasks.company_id` matches their own `profiles.company_id`.
        *   `"Allow users to view their assigned tasks (v2)"`: This allows a user to see tasks for which they have an entry in the `task_assignments` table (where `task_assignments.user_id = auth.uid()`).
        *   **Combined Effect**: A staff member will see all tasks for their company. The "assigned to me" RLS ensures they definitely see their tasks, which is usually covered by the company-wide policy anyway. The client-side (`js/tasks-display.js`) fetches tasks, and these RLS policies automatically filter the results. Admins, being part of their company, also see all tasks within it.
    *   **Updates (UPDATE RLS Policies on `tasks`)**:
        *   `"Allow assigned users to update task details"`: If a user's ID is in the `task_assignments` table for a given `task_id`, they can update that task's details.
        *   `"Allow update access to tasks of owned companies"`: This is a broader permission. It likely allows users (especially admins) to update any task as long as the `tasks.company_id` matches their `profiles.company_id`. This gives admins full control over tasks in their company.
    *   **Deletion (DELETE RLS Policy on `tasks`)**:
        *   `"Allow deletion of tasks of owned companies"`: Similar to the broad update access, users (primarily admins) can delete any task within their company.

*   **`task_assignments` Table**
    *   **Creation (INSERT RLS Policy)**: `"Allow company owner to create task assignments in their company"`. This means only users identified as company owners (likely admins where `profiles.is_admin = true` and their `id` matches `companies.owner_id` for the relevant `company_id`) can create new assignments. The `create-task` Edge Function handles the initial assignment as part of task creation, operating as an admin.
    *   **Deletion (DELETE RLS Policies)**:
        *   `"Allow company owner to delete task assignments in their company"`: Admins can remove any assignment in their company.
        *   `"Allow user to delete their own task assignment"`: A staff member can unassign themselves from a task.
    *   **Visibility (SELECT RLS Policies)**:
        *   `"Admin can read task assignments in own company"` / `"Allow company owner to read task assignments in their company"`: Admins see all assignments within their company.
        *   `"Allow user to see their own task assignments"`: Staff members can see the tasks they are assigned to.

*   **`detailed_task_assignments` View**
    *   The RLS policies for this view are not explicitly stated but would logically need to combine permissions from `tasks`, `task_assignments`, and `profiles` to ensure users only see enriched assignment data for tasks and users they are already authorized to see.

*   **`task_files` Table (Intended RLS Policies)**
    *   *(User noted current RLS for this table might be disabled, these are the intended rules)*
    *   **Visibility (SELECT)**: `"Allow users to see files for tasks they can access"`. File visibility is directly tied to task visibility. If a user can see a task, they can see its files.
    *   **Modification (INSERT/UPDATE for soft delete)**:
        *   `"Allow users to insert file metadata for tasks they can edit"`
        *   `"Allow users to soft delete files for tasks they can edit"`
        *   Permission to manage files is linked to the permission to edit the parent task.

## 6. Role of Client-Side Logic and Edge Functions in Access Control

*   **Client-Side (`js/tasks-display.js`, `js/properties.js`, `js/account-details.js`, etc.)**:
    *   **UI Adjustments**: Shows or hides UI elements (like admin-specific buttons or sections) based on the user's `is_admin` status fetched from their profile.
    *   **Data Fetching**: Generally makes Supabase client calls to fetch data (e.g., `supabase.from('tasks').select(...)`). It relies on the RLS policies at the database level to ensure only authorized data is returned. The client doesn't usually try to replicate RLS logic.
*   **Edge Functions (`create-task`, `invite-staff-member`, `save-company-details`, `activate-profile`)**:
    *   **Pre-Database Authorization**: They often perform critical authorization checks *before* attempting database operations. For example, `create-task` verifies the user is an admin and that the property/staff involved belong to the admin's company. This prevents invalid data from even attempting to be inserted.
    *   **Encapsulating Complex Logic**: They handle multi-step operations or logic that is too complex or sensitive to run entirely on the client-side (e.g., sending invitation emails, ensuring data consistency across tables).
    *   **Service Role Key**: Some functions (like `invite-staff-member`) may use the `SERVICE_ROLE_KEY` to perform actions that a regular user might not have direct RLS permission for (e.g., creating an auth invite, or writing to a profile during setup when RLS might be restrictive). This is done carefully for specific, controlled operations.

## 7. Summary: "Who Sees/Manages What"

*   **Administrators (`profiles.is_admin = true`)**:
    *   Have full visibility and CRUD (Create, Read, Update, Delete) access to data (properties, tasks, staff profiles, company settings) **within their own `company_id`**.
    *   Can invite new staff to their company.
    *   Can create tasks and assign them to staff within their company.
    *   Can manage all task assignments within their company.
*   **Staff Members (Non-Admins)**:
    *   Can see properties belonging to their `company_id`.
    *   Can see all tasks belonging to their `company_id` (as per the "read access to tasks of owned companies" RLS). The UI might provide filters for "my tasks."
    *   Can update tasks they are directly assigned to. They might also be able to update other tasks in their company if the "update access to tasks of owned companies" RLS policy is interpreted broadly for any company member (though admin control is more typical for this level of update).
    *   Can see their own task assignments and can unassign themselves.
    *   Can manage files for tasks they can edit.
    *   Can view and edit their own profile.
    *   **Crucially, they cannot see or interact with data from any other company.**

This multi-layered approach (client-side UI cues, Edge Function business logic, and database-level RLS enforcement) provides a robust system for managing data access and ensuring company data remains isolated and secure.
