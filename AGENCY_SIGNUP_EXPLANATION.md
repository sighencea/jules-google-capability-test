# Agency Signup and Onboarding Workflow

The signup process for a new agency in Property Hub is a multi-step workflow that involves user registration, email verification, security checks, and detailed company setup. Below is a step-by-step breakdown of how it works.

## 1. Initial Signup and Email Verification

The process begins on the main landing page, which serves both sign-in and sign-up functions.

1.  **Select Account Type**: A new user who wants to register an agency starts by clicking the "Sign up" link. They are then presented with a choice of account types. To proceed, they must select **"Agency Account"** from the dropdown menu.

2.  **Submit Registration Details**: After selecting the account type, a form appears asking for the following information:
    *   First Name
    *   Email Address
    *   A password (must be at least 6 characters long)

3.  **User Creation in Supabase**: When the user submits this form, the system calls the `supabase.auth.signUp()` function. This action creates a new user record in the Supabase `auth.users` table. Crucially, it also stores the `first_name` and `account_type: 'agency'` in the user's `user_metadata`. This metadata is essential for differentiating an agency administrator from a regular user in later steps.

4.  **Email Verification**: Upon successful registration, Supabase automatically sends a confirmation email to the address provided. This email contains a verification link that the user **must click** to confirm their account. This step is mandatory and ensures that the email address is valid and owned by the user. The user cannot sign in until their email is verified.

## 2. First Sign-In, Profile Creation, and Code Verification: A Detailed Breakdown

This phase is critical, as it bridges the gap between a user existing in the authentication system and having a functional profile within the application itself. Here is a granular, step-by-step look at what happens when a new, email-verified agency user signs in for the first time.

1.  **User Initiates Sign-In**:
    *   **Action**: The user navigates to `index.html` and enters their email and password into the sign-in form.
    *   **Code**: The `signInForm` element in `js/main.js` captures this event.

2.  **Authentication with Supabase**:
    *   **Action**: The browser sends the credentials to Supabase.
    *   **Code**: `window._supabase.auth.signInWithPassword({ email, password })` is called.
    *   **Result**: Supabase verifies the credentials. If successful, it returns a session object containing a JSON Web Token (JWT) for the user.

3.  **Profile Check**:
    *   **Action**: With the user now authenticated, the application immediately queries the database to see if a profile exists for this user.
    *   **Code**: A `select` query is performed on the `profiles` table, filtering by the user's ID: `window._supabase.from('profiles').select(...).eq('id', userId).single()`.
    *   **Result**: For a first-time sign-in, this query returns `null` and a `PGRST116` error, which specifically means "No rows found."

4.  **Triggering Profile Creation**:
    *   **Action**: The application logic detects the `PGRST116` error, confirming that no profile exists. This is the cue to create one.
    *   **Code**: The `create-initial-profile` Supabase Edge Function is invoked: `window._supabase.functions.invoke('create-initial-profile', ...)`
    *   **Behind the Scenes**: This is a secure, server-side operation. The user's JWT is sent with the request, allowing the edge function to verify the user's identity.

5.  **Inside the `create-initial-profile` Edge Function**:
    *   **Authentication**: The function first validates the user's JWT using the `supabaseAdminClient`.
    *   **Metadata Retrieval**: It accesses the `user_metadata` that was stored during the initial signup. This is where it finds `first_name` and, most importantly, `account_type: 'agency'`.
    *   **Database Insert**: The function constructs a new record to be inserted into the `profiles` table. This record includes:
        *   `id`: The user's unique ID from `auth.users`.
        *   `email`: The user's email.
        *   `first_name`: The user's first name.
        *   `is_admin`: Set to `true` because `account_type` is 'agency'.
        *   `has_company_set_up`: Set to `false`.
        *   `is_verified_by_code`: Set to `false` by default.
        *   `verification_code`: Set to the hardcoded value `22446688`.
    *   **Result**: A new row is created in the `profiles` table, officially giving the user a profile in the application.

6.  **Post-Profile Creation & Security Verification**:
    *   **Action**: Back on the client-side, the application now knows the profile exists. It re-fetches the newly created profile data.
    *   **Code**: The logic in `js/main.js` checks the value of `profile.is_verified_by_code`. Since it is `false`, it proceeds to the next security step.
    *   **UI**: The **8-Digit Code Modal** is displayed to the user.

7.  **Code Verification Process**:
    *   **Action**: The user must now enter the `22446688` code into the modal.
    *   **Code**: The `submitSixDigitCodeButton`'s click handler compares the input value with the `profile.verification_code` from the database.
    *   **Database Update**: If the codes match, an `update` query is sent to the `profiles` table to set `is_verified_by_code` to `true`.
    *   **Result**: The user has now passed the security check. The modal closes, and the onboarding process continues to the next step (language selection).

## 3. Language Selection and Agency Setup

With the security verification complete, the user moves on to the final steps of personalizing their account and setting up their agency.

1.  **Language Selection**: Immediately after the 8-digit code is verified, a "Select Your Preferred Language" modal appears. This forces the user to choose a language for the application interface (e.g., English or German). This preference is saved to the `preferred_ui_language` column in their `profiles` record.

2.  **Redirect to Agency Setup**: Once a language is chosen, the user is automatically redirected to the **Agency Company Setup** page (`agency_setup_page.html`).

3.  **Complete Company Profile**: This page contains a comprehensive form where the agency administrator must provide their company's details. The required fields include:
    *   Company Name
    *   Company Address (Street, City, State, Post Code)
    *   Company Email
    *   Optional fields include Company Logo, Phone, Website, and Tax ID.

4.  **Saving Company Details**: When the form is submitted, the `agency_setup.js` script triggers the **`save-company-details`** Supabase Edge Function. This function performs several critical actions:
    *   It creates a new record in the **`companies`** table with all the provided details.
    *   It generates a unique, random **8-digit `company_secret_code`** for the newly created company. This code is what the agency will give to its staff or users so they can join the company.
    *   It links the new company to the user by updating the user's record in the `profiles` table, setting the `company_id` and marking `has_company_set_up` as `true`.
    *   Finally, it updates the user's `app_metadata` in `auth.users` to include `is_admin: true` and the `company_id`. This is crucial for enabling Role-Level Security (RLS) policies, as the user's JWT will now contain these claims, granting them administrative access across the application.

After the company information is successfully saved, the user is redirected to the main agency dashboard, and the onboarding process is complete.

## Summary of the Workflow

A new user starts by creating an "Agency" account with their name, email, and password. After verifying their email, they sign in for the first time, which triggers the creation of their user profile and sets them as an administrator. They must then pass a security check by entering an 8-digit verification code. After this, they select their preferred language and are redirected to a form to enter their company's details. Submitting this form saves the company information, generates a unique company code for inviting others, and finalizes the link between the user and their new company account. Upon completion, they are redirected to the agency dashboard, with full administrative access.

## Files Involved in the Onboarding Process

Here is a list of all the files that are involved in making the onboarding process possible, along with their locations:

*   **`index.html`**: The main landing page that contains the sign-in and sign-up forms.
*   **`js/main.js`**: This file contains the core client-side logic for handling user interactions, such as form submissions, view toggling, and modal displays.
*   **`js/supabase-client.js`**: This file initializes the Supabase client, making it available for use throughout the application.
*   **`js/supabase-config.js`**: This file contains the Supabase URL and anon key.
*   **`pages/agency_setup_page.html`**: This is the page where the user enters their company details.
*   **`js/agency_setup.js`**: This file contains the client-side logic for handling the agency setup form submission.
*   **`supabase/functions/create-initial-profile/index.ts`**: This is the Supabase Edge Function that is responsible for creating a new user profile.
*   **`supabase/functions/save-company-details/index.ts`**: This is the Supabase Edge Function that is responsible for saving the company details to the database.
*   **`css/style.css`**: This file contains the custom styles for the application.
