# Onboarding Workflow: Agency Owner vs. Staff Member

This document provides a comprehensive, step-by-step description of the onboarding process for both an Agency Owner (Admin) and a Staff Member. It details the user interface, expected behaviors, and the specific database interactions that occur at each stage.

## Differentiating Agency Owners and Staff Members

The fundamental difference between an Agency Owner and a Staff Member lies in their permissions and the data they can access. This is controlled by the `is_admin` flag and their `company_id` in the `profiles` table.

*   **Agency Owner (Admin)**:
    *   **Role**: The creator and manager of the company account.
    *   **Permissions**: Has full access to all company data, including managing properties, tasks, and staff. They are the only ones who can invite new staff members.
    *   **Database Representation**: `is_admin` is `true`.

*   **Staff Member**:
    *   **Role**: An employee or contractor of the agency.
    *   **Permissions**: Has limited access, typically restricted to viewing and managing tasks assigned to them. They cannot manage staff or view company-wide administrative information.
    *   **Database Representation**: `is_admin` is `false`. Their `user_role` (e.g., "Electrician") further defines their function.

---

## Part 1: Agency Owner Onboarding Workflow

This workflow describes the process for a new user creating a company account from scratch.

### Step 1: Initial Registration
*   **User Interface**: The user navigates to `index.html`. They click "Sign up," which reveals the account type selection.
*   **Action**: The user selects "Agency Account" from the dropdown, and fills in their First Name, Email, and a Password (min. 6 characters).
*   **Expected Behavior**: Upon form submission, a success message appears, informing the user that a verification email has been sent.
*   **Database/Backend Interaction**:
    *   **File**: `js/main.js`
    *   **Function**: `supabase.auth.signUp()` is called.
    *   **Table**: `auth.users`
        *   A new user record is created.
        *   The `raw_user_meta_data` column is populated with `{ "first_name": "...", "account_type": "agency" }`.

### Step 2: Email Verification
*   **User Interface**: The user opens their email client and clicks the verification link in the email sent from Supabase.
*   **Action**: Clicking the link confirms their email address.
*   **Expected Behavior**: The user is redirected to a "verification successful" page (`pages/email-verified-success.html`).
*   **Database/Backend Interaction**:
    *   **Table**: `auth.users`
        *   The `email_confirmed_at` timestamp is set for the user's record, marking them as verified.

### Step 3: First Sign-In & Profile Creation
*   **User Interface**: The user returns to `index.html` and signs in with their credentials.
*   **Action**: The application authenticates the user and then immediately checks for a corresponding profile.
*   **Expected Behavior**: Since no profile exists, a server-side function is triggered to create one. This is seamless to the user.
*   **Database/Backend Interaction**:
    *   **Files**: `js/main.js`, `supabase/functions/create-initial-profile/index.ts`
    *   The client attempts to fetch a profile from the `profiles` table and gets no result.
    *   The `create-initial-profile` function is invoked.
    *   **Table**: `profiles`
        *   A new row is inserted with the following key values:
            *   `id`: The user's UUID from `auth.users`.
            *   `is_admin`: `true` (because `account_type` was 'agency').
            *   `is_verified_by_code`: `false` (by default).
            *   `verification_code`: `'22446688'`.
            *   `has_company_set_up`: `false`.

### Step 4: 8-Digit Code Verification
*   **User Interface**: A modal appears on the screen, prompting the user to "Enter Verification Code."
*   **Action**: The user must enter the fixed code `22446688`.
*   **Expected Behavior**: Upon entering the correct code, the modal closes, and the process continues.
*   **Database/Backend Interaction**:
    *   **File**: `js/main.js`
    *   **Table**: `profiles`
        *   An `UPDATE` statement is executed to set `is_verified_by_code = true` for the user's profile.

### Step 5: Language Selection
*   **User Interface**: A new modal appears, asking the user to "Select Your Preferred Language."
*   **Action**: The user selects a language from the dropdown (e.g., English).
*   **Expected Behavior**: The modal closes, the preference is saved, and the user is automatically redirected.
*   **Database/Backend Interaction**:
    *   **File**: `js/main.js`
    *   **Table**: `profiles`
        *   An `UPDATE` statement is executed to set the `preferred_ui_language` column to the selected language code (e.g., 'en').

### Step 6: Agency Setup
*   **User Interface**: The user is redirected to the "Set Up Your Agency Company" page (`pages/agency_setup_page.html`). This is a full-page form.
*   **Action**: The user fills out their company's details (Name, Address, Email, etc.) and clicks "Save Company Information."
*   **Expected Behavior**: A success message appears, and after a brief delay, the user is redirected to the main dashboard.
*   **Database/Backend Interaction**:
    *   **Files**: `js/agency_setup.js`, `supabase/functions/save-company-details/index.ts`
    *   The `save-company-details` function is invoked.
    *   **Table**: `companies`
        *   An `INSERT` statement creates a new company record with all the form data.
        *   The `owner_id` is set to the user's ID.
        *   A unique 8-digit `company_secret_code` is generated and stored.
    *   **Table**: `profiles`
        *   An `UPDATE` statement is executed on the user's profile to:
            *   Set `has_company_set_up = true`.
            *   Set the `company_id` to the ID of the newly created company.
    *   **Table**: `auth.users`
        *   The `raw_user_meta_data` is updated to include `{ "is_admin": true, "company_id": "..." }`. This embeds their admin status and company affiliation directly into their JWT for RLS policies.

---

## Part 2: Staff Member Onboarding Workflow

This workflow begins when an Agency Owner invites a new staff member.

### Step 1: Admin Sends Invitation
*   **User Interface**: The Agency Owner is on the "Staff Management" page (`pages/staff.html`) and clicks the "+ Staff" button. A modal appears.
*   **Action**: The admin fills in the new staff member's First Name, Last Name, Email, and Role, then clicks "Send Invite."
*   **Expected Behavior**: A confirmation message indicates the invite has been sent.
*   **Database/Backend Interaction**:
    *   **Files**: `js/staff-management.js`, `supabase/functions/invite-staff-member/index.ts`
    *   The `invite-staff-member` function is invoked.
    *   **Function**: `supabase.auth.admin.inviteUserByEmail()` is called.
    *   **Table**: `auth.users`
        *   A new, unconfirmed user record is created with an `invitation_token`.
        *   The `raw_user_meta_data` is populated with all the details from the form, including `company_id` (from the admin's profile), `user_role`, and `is_admin: false`.

### Step 2: Staff Member Accepts & Sets Password
*   **User Interface**: The invited staff member receives an email and clicks the "Accept Invitation" link. They are redirected to the "Set Password" page (`pages/set-password.html`).
*   **Action**: The staff member enters and confirms their new password.
*   **Expected Behavior**: A success message appears, and they are prompted to sign in.
*   **Database/Backend Interaction**:
    *   **File**: `js/set-password.js`
    *   The page automatically handles the token from the URL to authenticate the user for this one-time action.
    *   **Function**: `supabase.auth.updateUser()` is called.
    *   **Table**: `auth.users`
        *   The user's record is updated with the new password hash.
        *   The `email_confirmed_at` and `invitation_token` fields are updated, marking the user as fully confirmed.

### Step 3: Profile Creation & Activation (Automated)
*   **User Interface**: This is largely invisible to the user, happening in the background after they set their password.
*   **Action**: A database trigger and a subsequent function call create and activate the staff member's profile.
*   **Database/Backend Interaction**:
    *   **Database Trigger**: A trigger on `auth.users` likely creates a basic shell profile in the `profiles` table.
    *   **Database Function**: The `handle_new_user_profile_setup` SQL function fires. It reads the `raw_user_meta_data` from the `auth.users` record and populates the new `profiles` row with the correct `company_id`, `user_role`, `is_admin: false`, and sets `user_status = 'Invited'`.
    *   **Files**: `js/set-password.js`, `supabase/functions/activate-profile/index.ts`
    *   The `activate-profile` function is invoked from the client.
    *   **Table**: `profiles`
        *   An `UPDATE` statement sets `user_status = 'Active'`.

### Step 4: First Sign-In
*   **User Interface**: The staff member goes to `index.html` and signs in with their email and the password they just created.
*   **Action**: They are authenticated.
*   **Expected Behavior**: They are redirected to their dashboard, which is typically the "Tasks" page (`pages/tasks.html`), showing only tasks assigned to them.
*   **Database/Backend Interaction**:
    *   The application fetches their profile, which is now fully populated and active.
    *   Row Level Security (RLS) policies on tables like `tasks` and `properties` ensure they can only see data where the `company_id` matches their own and, if applicable, where they are assigned as a staff member.
