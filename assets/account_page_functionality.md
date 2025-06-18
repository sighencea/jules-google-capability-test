# Account Page (`account.html`) Functionality

This document details the functionality of the Account Settings page (`pages/account.html`), including how user profile information and company details are fetched, displayed, and updated.

## 1. Initialization and Overall Structure

When `pages/account.html` is loaded, the `js/account-details.js` script orchestrates its dynamic behavior.

*   **Initial Setup (`DOMContentLoaded`)**:
    *   References to key HTML elements for profile display, modals, company settings, and messages are obtained.
    *   A loading indicator (`profileLoadingIndicator`) is initially shown for the profile section while data is fetched.
    *   The script immediately calls `checkAdminStatusAndApplyUI()` to determine if the current user is an administrator.
*   **Admin Check (`checkAdminStatusAndApplyUI`)**:
    *   Fetches the current user's ID from `supabase.auth.getUser()`.
    *   Queries the `profiles` table to get the `is_admin` status for this user.
    *   The "Company Settings" section (`companySettingsSection`) on the page is shown only if `is_admin` is `true`. Otherwise, it remains hidden.
*   **Data Loading**:
    *   `window.loadAndDisplayAccountDetails()` is called to fetch and display the user's personal profile information.
    *   `loadCompanySettings()` is called to fetch and display company information (this function internally also checks if the company settings section should be visible).

## 2. Profile Information Display

The "Profile Information" section on the page is populated as follows by `loadAndDisplayAccountDetails()`:

*   **Data Source**:
    *   The current user is retrieved using `window._supabase.auth.getUser()`.
    *   Their corresponding record is fetched from the `public.profiles` table, selecting fields like `first_name`, `last_name`, `email`, `phone_number`, `preferred_ui_language`, and `avatar_url`.
*   **Display Logic**:
    *   The user's full name (concatenation of `first_name` and `last_name`), email, and phone number are displayed in read-only input fields (`fullName`, `emailAddress`, `phoneNumber`).
    *   The `preferred_ui_language` is converted to a human-readable format (e.g., 'en' to 'English') and shown in `languageDisplay`.
    *   If an `avatar_url` exists in the profile, it's displayed in the `currentProfileImage` image tag; otherwise, a default person icon (`defaultProfileIcon`) is shown.
    *   Appropriate messages like "Not provided" or "Error loading profile" are displayed if data is missing or issues occur.
    *   The loading indicator is hidden, and the profile data container is shown once fetching is complete.

## 3. Profile Information Update

Users can update their profile information via the "Edit Profile" modal.

*   **Modal Trigger**:
    *   Clicking the "Edit Profile" button (ID: `editProfileButton`) on the main page triggers the "Edit Profile Modal" (ID: `editProfileModal`).
    *   Before the modal is shown, an event listener fetches the latest profile data for the current user and populates the modal's form fields (`modalFirstName`, `modalLastName`, `modalEmailAddress` (read-only), `modalPhoneNumber`, `modalLanguageSelector`). The profile image upload and preview are reset.
*   **Image Handling (Client-Side)**:
    *   The `profileImageUploadElement` (a file input) allows users to select a new profile image.
    *   A change listener on this input reads the selected file and displays a preview in `profileImagePreviewModalElement` using `FileReader`.
*   **Saving Changes (Logic for `saveProfileChanges` button)**:
    1.  **Validation**: Ensures "First Name" and "Last Name" fields are not empty.
    2.  **User Session Check**: Verifies the user is still authenticated.
    3.  **Profile Image Upload (if a new image is selected)**:
        *   The system first fetches the current `avatar_url` from the user's profile to mark it for deletion if the new upload and profile update are successful.
        *   The new image file is uploaded to the `profile-images` Supabase Storage bucket. The path is constructed as `user_{user.id}/profile_image_{timestamp}.{extension}`. The `upsert: true` option is used.
        *   The public URL of the newly uploaded image (`newAvatarUrl`) is retrieved from storage.
    4.  **Data Preparation**: An `updates` object is created containing `first_name`, `last_name`, `phone_number` (or `null`), `preferred_ui_language`, `updated_at` (current timestamp), and `avatar_url` (if `newAvatarUrl` is available).
    5.  **Database Update**: The user's record in the `public.profiles` table is updated with the `updates` object using `window._supabase.from('profiles').update(updates).eq('id', user.id)`.
    6.  **Old Avatar Deletion**: If a new avatar was successfully uploaded and the profile update was successful, the script attempts to delete the previously stored `oldAvatarUrlToDelete` from the `profile-images` storage bucket. This is a best-effort deletion.
    7.  **Feedback & Page Reload**: A success or error message is displayed in `editProfileMessageElement`. On successful update, the entire page is reloaded (`window.location.reload()`) to reflect all changes, including the potentially new avatar.

## 4. Company Settings (Administrators Only)

This section is only visible and usable if the logged-in user has `is_admin: true` in their profile.

*   **Data Fetching (`loadCompanySettings`)**:
    *   Retrieves the current user.
    *   Fetches the company record from the `public.companies` table where the `owner_id` matches the current user's ID. (This implies admins manage the company they own).
    *   If a company record is found, its details (`company_name`, address components, `email`, `phone_number`, `website_url`, `tax_id`, `company_logo_url`) are used to populate the fields within the `companySettingsForm`. The `company_logo_url` is used to display an image in `logoPreview`.
    *   If no company record is found (e.g., for a new admin who hasn't completed company setup), a message is shown, and the form is reset.
*   **Updating Company Settings (Logic for `saveCompanySettingsButton`)**:
    1.  **Validation**: Checks for required fields like company name, address components, and a valid company email format.
    2.  **Company Logo Upload (if a new logo is selected)**:
        *   The new logo file from `companyLogoInput` is uploaded to the `agency-logos` Supabase Storage bucket. The path is `user_{user.id}/company_logo_{timestamp}.{extension}` with `upsert: true`.
        *   The public URL of the uploaded logo is retrieved.
        *   If no new file is selected, the system attempts to use the URL from the currently displayed `logoPreview` (if it's not a temporary data URL).
    3.  **Data Preparation**: A `companyData` object is constructed with all the form values and the `companyLogoUrlToSave`.
    4.  **Edge Function Invocation**: The script calls the `save-company-details` Supabase Edge Function, sending the `companyData` object in the request body.
        *   *(The `save-company-details` function is assumed to handle the actual database update to the `companies` table, potentially performing additional validation or logic on the backend, such as creating the company record if it doesn't exist or updating it if it does).*
    5.  **Feedback & Data Refresh**: Displays success or error messages in `companySettingsMessage`. On successful update (as indicated by the Edge Function's response), `loadCompanySettings()` is called again to refresh the displayed company information.

## 5. Non-Implemented Features

The `pages/account.html` file contains HTML structures for "Change Password" and "Notification Preferences," but the `js/account-details.js` script **does not currently include any JavaScript logic to make these sections functional.**

*   **Change Password**: There are no event listeners or functions to handle the submission of the password change form. Users needing to change their password would likely use a "Forgot Password" flow from the login page, or this functionality needs to be explicitly added to `js/account-details.js`.
*   **Notification Preferences**: The checkboxes for notification preferences are present, but there's no code to load current preferences or save changes. The HTML itself marks these as "Dummy Data."

## 6. Key Files, Tables, and Storage Involved

*   **HTML Page**: `pages/account.html`
*   **Client-Side JavaScript**: `js/account-details.js`
*   **Supabase Edge Function (Invoked)**: `supabase/functions/save-company-details/index.ts` (for saving company settings)
*   **Supabase Tables**:
    *   `public.profiles`: Stores user-specific information including names, contact details, language preference, avatar URL, and admin status.
    *   `public.companies`: Stores company-specific information, linked to an owner.
*   **Supabase Storage Buckets**:
    *   `profile-images`: For storing user profile avatars.
    *   `agency-logos`: For storing company logos.

This page provides users with essential self-service capabilities for their profile and, for administrators, control over their company's core details.
