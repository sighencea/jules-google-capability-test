# Company ID Retrieval Explanation

## How is the `company_id` obtained when adding a new property?

When a new property is added using the "Add Property" modal on the `properties.html` page, the `company_id` is obtained through the following process:

1.  **Fetching User Information**: The system first identifies the currently logged-in user. This is typically done by calling a function like `window._supabase.auth.getUser()`, which retrieves the authenticated user's details, including their unique user ID.

2.  **Querying the `companies` Table**: Using the logged-in user's ID, the application then queries the `companies` database table. It looks for a company record where a specific column (e.g., `owner_id`) matches the user's ID. This query is designed to find the company associated with or owned by that user.

3.  **Extracting `company_id`**: The `id` from the company record found in the previous step is then used as the `company_id` for the new property being created.

It's important to note what this process is **not**:

*   The `company_id` is **not** stored directly in the user's profile or application metadata (e.g., `user.app_metadata.company_id`).
*   It is **not** a field that the user selects or inputs directly in the "Add Property" form. The system determines it automatically based on the logged-in user's association with a company.

## How is the `companies` table linked to the logged-in user?

The `companies` table is linked to the logged-in user through a specific column that stores the user's unique identifier.

*   **Linking Column**: The `companies` table contains a column named `owner_id`.
*   **User ID**: This `owner_id` stores the ID of the user who owns or is associated with the company. This ID corresponds to the `id` field of the user object managed by the authentication system (e.g., the `id` from the `auth.users` table in Supabase, which is retrieved via functions like `window._supabase.auth.getUser()`).

Therefore, the direct linking mechanism is:

`companies.owner_id` = `auth.users.id` (or the equivalent user ID from the authentication provider).
