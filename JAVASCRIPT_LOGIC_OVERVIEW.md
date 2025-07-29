# JavaScript Logic Overview

This document provides a structured overview of the key JavaScript files involved in the user onboarding and setup process.

---

## 1. `js/supabase-client.js` - Supabase Client Initialization

This file is responsible for initializing the Supabase client, which is the foundation for all interactions with the Supabase backend.

### Core Logic:
1.  **Import `createClient`**: It imports the necessary function from the Supabase JavaScript library.
2.  **Resolve Credentials**: It looks for `SUPABASE_URL` and `SUPABASE_ANON_KEY` (which are expected to be defined in `js/supabase-config.js`).
3.  **Error Handling**: If the credentials are not found or are placeholders, it logs a critical error to the console and displays an error message on the page.
4.  **Client Initialization**: It calls `createClient()` with the resolved credentials to create the Supabase client instance.
5.  **Global Availability**: If the initialization is successful, the client instance is assigned to `window._supabase`, making it globally accessible to all other scripts. If it fails, `window._supabase` is set to `null`.

---

## 2. `js/main.js` - Main Sign-In and Sign-Up Logic

This is the central file for handling user authentication and the multi-step onboarding process for new users.

### Key Responsibilities:

*   **DOM Element Selection**: It begins by getting references to all relevant DOM elements, including forms, buttons, modals, and message containers.
*   **View Toggling**:
    *   It manages the display of the "Sign In" vs. "Sign Up" views.
    *   Within the sign-up view, it handles the progression from "Select Account Type" to the "Agency Signup Form" or the "User Account Info" view.
*   **Sign-Up Logic (`signUpForm` event listener)**:
    *   Handles the submission of the agency sign-up form.
    *   Performs basic validation (e.g., password length).
    *   Calls `window._supabase.auth.signUp()`, passing the user's email and password, along with `first_name` and `account_type` in the `options.data` field.
    *   Handles success and error responses from Supabase, including logic to show the "Resend Verification" modal if the user already exists.
*   **Sign-In Logic (`signInForm` event listener)**:
    *   Handles the submission of the sign-in form.
    *   Calls `window._supabase.auth.signInWithPassword()`.
    *   **On successful sign-in, it orchestrates the core onboarding flow**:
        1.  It attempts to fetch the user's profile from the `profiles` table.
        2.  If the profile does not exist (a `PGRST116` error is returned), it invokes the `create-initial-profile` Supabase function to create it.
        3.  It then re-fetches the newly created profile.
        4.  It checks the `is_verified_by_code` flag. If `false`, it shows the **8-Digit Code Modal**.
        5.  It checks the `has_company_set_up` flag. If `false` (and the user is an admin), it shows the **Language Selection Modal**.
        6.  If all checks pass, it redirects the user to the appropriate dashboard (`pages/dashboard.html` for admins, `pages/tasks.html` for non-admins).
*   **Modal Logic**:
    *   **8-Digit Code Modal**: Handles showing the modal and verifying the entered code against the value in the user's profile. On success, it updates the `is_verified_by_code` flag in the `profiles` table.
    *   **Language Selection Modal**: Handles showing the modal and saving the user's language preference to the `preferred_ui_language` column in their profile. On success, it redirects to the `agency_setup_page.html`.
    *   **Resend Verification Modal**: Handles the logic for resending a verification email.
    *   **Company Code Modal**: Handles the multi-step process for a user joining an existing company using an 8-digit company code.

---

## 3. `js/agency_setup.js` - Agency Setup Page Logic

This file is dedicated to handling the final step of the Agency Owner onboarding process: setting up the company details.

### Core Logic:

1.  **DOM Element Selection**: It gets references to all the input fields in the agency setup form (`agencySetupForm`).
2.  **Logo Preview**: It includes logic to display a preview of the company logo when a file is selected.
3.  **Form Submission (`agencySetupForm` event listener)**:
    *   Performs client-side validation to ensure all required fields are filled out.
    *   **Logo Upload**: If a logo file is provided, it uploads the file to Supabase Storage in a bucket named `agency-logo`, under a path that includes the user's ID to keep it unique.
    *   **Get Public URL**: After a successful upload, it retrieves the public URL of the logo.
    *   **Prepare Data**: It constructs a `companyData` object with all the information from the form, including the logo URL.
    *   **Invoke Edge Function**: It calls `window._supabase.functions.invoke('save-company-details', { body: companyData })`. This securely saves the company information on the server.
    *   **Handle Response**: On a successful response from the function, it displays a success message and redirects the user to the dashboard (`pages/dashboard.html`), completing the onboarding process.
