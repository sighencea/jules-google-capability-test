# Staff Invitation and Onboarding Flow

This document details the end-to-end process for an administrator inviting a new staff member to join their company, and the subsequent steps for the staff member to accept the invitation and activate their account.

## Phase 1: Admin Invites Staff Member

This phase is initiated and handled by an existing administrator of a company.

1.  **Trigger**:
    *   The administrator navigates to the "Staff Management" page (`pages/staff.html`).
    *   They click the **"+ Staff"** button (ID: `addNewStaffMemberBtn`).

2.  **Client-Side Actions (`js/staff-management.js`)**:
    *   The "Add New Staff Member" modal (ID: `addStaffModal`) is displayed.
    *   The administrator fills in the required details for the new staff member:
        *   First Name (`addStaffFirstName`)
        *   Last Name (`addStaffLastName`)
        *   Email (`addStaffEmail`)
        *   Role (`addStaffRole` - e.g., Electrician, Plumber)
    *   Upon clicking the **"Send Invite"** button (ID: `saveNewStaffBtn` within the modal), the `processStaffInvitation()` function is executed.
    *   This function gathers the form data and makes an asynchronous call to a Supabase Edge Function named `invite-staff-member`.

3.  **Server-Side - Supabase Edge Function (`supabase/functions/invite-staff-member/index.ts`)**:
    *   **Authentication & Authorization**:
        *   The function first validates the calling user (the administrator) using their JWT.
        *   It checks if the administrator has an `is_admin` status and retrieves their `company_id` from their profile in the `profiles` table. If these checks fail, an error is returned.
    *   **Core Invitation Logic**:
        *   It calls the Supabase Admin Auth method: `supabase.auth.admin.inviteUserByEmail(email, options)`.
        *   **`email`**: The email address of the staff member to be invited.
        *   **`options.redirectTo`**: This is set to `https://www.afiaro.com/pages/set-password.html`. After the invitee clicks the link in their email, they will be redirected to this URL, and Supabase will automatically append necessary tokens (like `access_token`, `refresh_token`, etc.) to the URL fragment.
        *   **`options.data`**: This metadata object is crucial for the onboarding process. It includes:
            *   `first_name`: Invitee's first name.
            *   `last_name`: Invitee's last name.
            *   `user_role`: The role assigned by the admin.
            *   `company_id`: The `company_id` of the inviting administrator's company. This links the new staff member to the correct company.
            *   `is_admin: false`: Staff members are explicitly set as non-admins.
            *   `user_status: 'Invited'`: An initial status marker.
    *   **Email Dispatch**: Supabase Auth handles the generation of a unique invitation token and sends an email to the invitee. The email contains a link that includes the `redirectTo` URL and the necessary tokens.
    *   **Response**: The function returns a success message if the invitation email was sent successfully, or an error message (e.g., if the user is already registered).

## Phase 2: Staff Member Accepts Invitation & Onboards

This phase begins when the invited staff member receives and acts upon the invitation email.

1.  **Email Interaction & Redirection**:
    *   The invitee receives the invitation email.
    *   They click the unique invitation link.
    *   This action directs them to the `redirectTo` URL specified during the invite: `https://www.afiaro.com/pages/set-password.html`. The browser URL will now include a fragment (e.g., `#access_token=...&refresh_token=...&type=recovery...`) containing the necessary tokens.

2.  **Password Setting Page (`pages/set-password.html` & `js/set-password.js`)**:
    *   **Token Processing**: The Supabase client library (`supabase-client.js`, loaded on the page) automatically detects and processes the tokens from the URL fragment. This establishes a temporary authenticated session for the user, specifically for the purpose of updating their password (often referred to as a "recovery" flow, which invitations leverage). The `onAuthStateChange` event in `js/set-password.js` may fire, indicating this state (e.g., event type `PASSWORD_RECOVERY`).
    *   **User Input**: The invitee sees a form where they must enter and confirm their desired password.
    *   **Password Update**: Upon form submission:
        *   Client-side validation (password length, match) is performed.
        *   The script calls `supabase.auth.updateUser({ password: new_password })`.
        *   This updates the user's record in the `auth.users` table with the new password and marks the user as confirmed (if they weren't already). The `raw_user_meta_data` (containing `company_id`, `role`, etc., from the invitation) is now permanently associated with this authenticated user record.

3.  **Profile Creation & Enrichment (Database Trigger - `public.handle_new_user_profile_setup`)**:
    *   **Context**: This SQL function is executed by an `AFTER INSERT ON public.profiles FOR EACH ROW` trigger (named, for example, `on_new_user_profile_created`).
    *   **Initial Profile Row**: An initial, possibly minimal, row in `public.profiles` must be created for this trigger to fire. This might occur due to a more generic trigger on `auth.users` (e.g., `AFTER INSERT ON auth.users`) that creates a basic profile shell.
    *   **Trigger Action**:
        *   The `handle_new_user_profile_setup` function retrieves the `auth.users` record corresponding to the new profile's `id`.
        *   It then accesses `auth_user_record.raw_user_meta_data`.
        *   **Invited User Logic**: It specifically checks if this metadata contains `user_role`, `company_id`, and `user_status: 'Invited'`.
        *   If these invite markers are present, it extracts all the necessary details (`first_name`, `last_name`, `user_role`, `company_id`, `is_admin`, `user_status`) from the metadata.
        *   It then **`UPDATE`s** the newly inserted `public.profiles` row, populating it with these extracted, correct details. The `user_status` in `profiles` is set to `'Invited'`.

4.  **Profile Activation - Client Call (`js/set-password.js`)**:
    *   After the `supabase.auth.updateUser()` call is successful (password is set), the `js/set-password.js` script makes another call: `supabase.functions.invoke('activate-profile')`.

5.  **Profile Activation - Server-Side (`supabase/functions/activate-profile/index.ts`)**:
    *   **Authentication**: The function authenticates the user using their JWT (now that they've set a password, they have a valid session).
    *   **Action**: It updates the user's record in the `public.profiles` table.
        *   It sets `user_status = 'Active'`.
        *   It updates `updated_at`.
        *   **Condition**: This update is conditional: it only applies if the current `user_status` for that user in `profiles` is `'Invited'`.
    *   **Outcome**: If successful, the staff member's profile is now marked as 'Active'.

## Phase 3: Post-Onboarding

1.  **Sign In**: The staff member can now sign in to the application using their email and the password they just set, via the main login page (`../index.html`).
2.  **User State**:
    *   Their record in `auth.users` is fully active.
    *   Their record in `public.profiles` is populated with their name, email, assigned `user_role`, the correct `company_id`, `is_admin: false`, and `user_status: 'Active'`.
3.  **Access Control**: Row Level Security (RLS) policies in Supabase should now grant them appropriate access to data and features based on their `company_id` and `user_role`.

## Relevant Files and Functions:

*   **Pages**:
    *   `pages/staff.html`: Admin interface for managing staff and initiating invites.
    *   `pages/set-password.html`: Page where invited users set their password.
*   **Client-Side JavaScript**:
    *   `js/staff-management.js`: Handles admin actions on the staff page, including triggering the invite flow.
    *   `js/set-password.js`: Handles password setting for invited users and calls profile activation.
    *   `js/supabase-client.js` & `js/supabase-config.js`: Configure and provide the Supabase client instance.
*   **Supabase Edge Functions**:
    *   `supabase/functions/invite-staff-member/index.ts`: Backend logic for sending invitations via `supabase.auth.admin.inviteUserByEmail()` and embedding metadata.
    *   `supabase/functions/activate-profile/index.ts`: Backend logic to change user status from 'Invited' to 'Active' in the `profiles` table.
*   **Database (SQL)**:
    *   `supabase/migrations/..._update_handle_new_user_profile_setup_function.sql` (or similar name): Contains the `public.handle_new_user_profile_setup()` SQL function.
    *   The trigger definition on `public.profiles` table (e.g., `CREATE TRIGGER on_new_user_profile_created AFTER INSERT ON public.profiles ... EXECUTE FUNCTION public.handle_new_user_profile_setup();`).

This flow ensures that new staff members are correctly associated with their company and role, and their accounts are securely activated.
