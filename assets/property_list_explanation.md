# Property List Fetching and Rendering Explanation

The primary file responsible for fetching the list of properties visible on the `properties.html` page and rendering the individual property items is:

`js/lazy-load-properties.js`

Here's a breakdown of its responsibilities:

## 1. Fetching Properties

*   **Function**: `fetchProperties(offset, limit)`
*   **Data Source**: It asynchronously fetches property data from the Supabase backend using `window._supabase.from('properties').select(...)`. The specific details fetched include `id`, `property_name`, `address`, `property_image_url`, `property_type`, and `qr_code_image_url`.
*   **Filtering & Ordering**: While RLS (Row Level Security) in Supabase is expected to filter properties based on the logged-in user, the script orders the received properties by `created_at` in descending order.
*   **Pagination (Lazy Loading)**:
    *   The function fetches properties in batches (e.g., 9 properties per call, defined by `propertiesPerPage`).
    *   It uses an `offset` to get the next set of properties for lazy loading.
*   **State Management**: It manages loading states like `isLoading` (to prevent multiple simultaneous fetches) and `allPropertiesLoaded` (to stop fetching when all properties have been retrieved).
*   **Error Handling**: Basic error handling is in place to log issues during the fetch process and optionally display a message to the user.

## 2. Rendering Properties

*   **Function**: `displayProperties(properties)`
*   **Target Container**: This function takes an array of property objects (fetched by `fetchProperties`) and generates HTML to display them. The generated HTML is injected into the `<div id="propertiesContainer">` element on the `properties.html` page.
*   **Property Card Structure**: For each property, it creates a "card" that includes:
    *   Property image (`property.property_image_url`). A placeholder is used if no image is available.
    *   Property name (`property.property_name`).
    *   Property address (`property.address`).
    *   Property type (`property.property_type`).
    *   A "View Details" button that links to the specific property's detail page (`property-details.html?id=<property_id>`).
    *   A button to display a QR code for the property, if a `qr_code_image_url` is available.
*   **Empty State**: If no properties are fetched on the initial load, it displays a "You currently have no properties..." message.

## 3. Lazy Loading Trigger

*   **Function**: `lazyScrollHandler()`
*   **Mechanism**: This function is attached to the window's scroll event.
*   **Action**: When the user scrolls close to the bottom of the page, it triggers a call to `fetchProperties` to load the next batch of properties, provided that not all properties are already loaded and a fetch isn't currently in progress.

## 4. Refresh Functionality

*   **Global Function**: `window.refreshPropertiesList()`
*   **Purpose**: This function can be called to reset the loading state (offset, flags) and re-fetch the properties from the very beginning. This is useful for updating the list if properties have been added, edited, or deleted elsewhere in the application.

## 5. QR Code Modal Interaction

*   The script also includes event listeners on the property cards to handle clicks on the QR code buttons. When clicked, it displays the property's QR code in a modal window (`qrCodeDisplayModal`).

In essence, `js/lazy-load-properties.js` manages the complete lifecycle of the property list on `properties.html`, from data retrieval and dynamic HTML generation to user interactions like lazy loading and accessing QR codes.
