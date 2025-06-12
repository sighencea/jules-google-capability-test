# QR Code Functionality Explanation

This document explains how QR codes are generated, stored, and displayed within the application.

## 1. QR Code Generation and Storage

This process occurs when a new property is successfully added via the "Add Property" modal.

*   **Responsible File**: `js/addProperty.js`

**Process:**

1.  **Trigger**: After a new property record is successfully created in the database and its `newPropertyId` is available.
2.  **QR Code Library**: The script uses a client-side JavaScript library (identified as `qrcode-generator` via the `qrcode.js` script included in `properties.html`) to generate the QR code. This happens in the `attemptQrCodeGeneration` function.
3.  **Content**: The QR code is generated to encode the direct URL to the property's detail page. The URL is structured as: `${window.location.origin}/pages/property-details.html?id=${newPropertyId}`.
4.  **Image Format**: The library generates a `dataURL` for the QR code image (typically a PNG).
5.  **Blob Conversion**: This `dataURL` is then fetched and converted into a `Blob` object, which is suitable for uploading.
6.  **Storage Path & Bucket**:
    *   The QR code image blob is uploaded to the Supabase Storage bucket named `property-qr-codes`.
    *   The specific path within the bucket is dynamically constructed as: `users/${userId}/qr_codes/qr_${newPropertyId}.png`.
    *   `(userId` is the ID of the currently authenticated user, and `newPropertyId` is the ID of the newly created property).
7.  **RLS Policies Alignment**: This storage path directly aligns with the provided Row Level Security (RLS) policies for the `property-qr-codes` bucket:
    *   `((bucket_id = 'property-qr-codes'::text) AND (owner = auth.uid()) AND ((storage.foldername(name))[1] = 'users'::text) AND ((storage.foldername(name))[2] = (auth.uid())::text) AND ((storage.foldername(name))[3] = 'qr_codes'::text))`
    *   This policy ensures that users can only upload to and view files within their own designated `users/{auth.uid()}/qr_codes/` folder structure.
8.  **Upload**: The upload is performed using `window._supabase.storage.from('property-qr-codes').upload(filePath, qrImageBlob, { upsert: true })`.
9.  **Public URL Retrieval**: Upon successful upload, the script retrieves a public URL for the newly uploaded QR code image using `window._supabase.storage.from('property-qr-codes').getPublicUrl(filePath)`.
10. **Database Update**: This public URL is then saved into the `properties` table in the database, specifically in the `qr_code_image_url` column for the corresponding property record. This is done via an update operation: `window._supabase.from('properties').update({ qr_code_image_url: qrCodeImageUrl, generate_qr_on_creation: true }).eq('id', newPropertyId)`.

## 2. QR Code Display on Property Cards

QR codes are displayed on individual property cards on the `properties.html` page.

*   **Responsible Files**:
    *   `js/lazy-load-properties.js` (for fetching data and rendering logic)
    *   `pages/properties.html` (for the structure of the modal used to display the QR code)

**Process:**

1.  **Data Fetching**: When `js/lazy-load-properties.js` fetches the list of properties, it includes the `qr_code_image_url` for each property.
2.  **Conditional Rendering**: In the `displayProperties` function, if a property object contains a valid `qr_code_image_url`, a small button with a QR code icon (`<i class="bi bi-qr-code"></i>`) is added to the HTML for that property card.
3.  **Data Attribute**: This button stores the actual `qr_code_image_url` in a data attribute, specifically `data-qr-url`.
4.  **Modal Display**:
    *   An event listener is attached to the `propertiesContainer`.
    *   When a QR code button on a card is clicked, this listener retrieves the `qr_code_image_url` from the button's `data-qr-url` attribute.
    *   It then populates this URL into an `<img>` tag within a modal window (`qrCodeDisplayModal`, defined in `pages/properties.html`).
    *   The modal is then shown, displaying the QR code image for that specific property.
    *   A download link for the QR code is also provided in this modal.

## 3. QR Code Display on Property Details Page

This section describes how QR codes are (or are not currently) handled on the dedicated `property-details.html` page.

*   **Responsible Files**:
    *   `js/property-details.js` (for fetching data and page logic)
    *   `pages/property-details.html` (for page structure)

**Process & Current Status:**

1.  **Data Fetching**: The `js/property-details.js` script fetches detailed information for a specific property using the `propertyId` from the URL. The fields fetched include `property_name`, `address`, `property_type`, `property_details`, `property_occupier`, and `property_image_url`.
2.  **Main Image Display**: The fetched `property_image_url` is used to display the main image for the property on this page.
3.  **QR Code Display - Not Implemented**:
    *   Based on the current code in `js/property-details.js` (as of the last review), the `qr_code_image_url` specifically for the property's QR code is **not explicitly fetched nor is it used to display the QR code image** on the `property-details.html` page.
    *   While the data *could* be fetched (by adding `qr_code_image_url` to the `select` statement), there is no current UI element or JavaScript logic in this file to render it.
4.  **Potential Enhancement**: If displaying the QR code directly on this page is desired, the `js/property-details.js` script would need to be updated to:
    *   Fetch the `qr_code_image_url`.
    *   Have a designated HTML element in `pages/property-details.html` where the QR code image would be rendered.
    *   Include JavaScript logic to set the `src` of that image element to the fetched `qr_code_image_url`.
