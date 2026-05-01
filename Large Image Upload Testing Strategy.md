# Large Image Upload Testing Strategy

## Problem Statement

Uploading small images works perfectly, but uploading large images freezes the app without any error message. No crash logs appear on the server.

## Key Assumptions

* The application freeze is occurring on the client side, not due to a server crash, since no server crash logs are generated.
* Large image files require significantly more memory and processing power, which may cause the UI thread to become unresponsive.
* The client application may be loading the full image into memory (for preview, encoding, or conversion) before uploading, increasing memory usage.
* The image upload is likely being sent as a single large request, not in chunks, increasing risk of timeout or memory issues.
* There may be a maximum request body size limit configured at the API server, load balancer, CDN, or reverse proxy level.
* Large uploads may exceed client-side or server-side timeout limits, causing silent failures.
* Requests might be blocked or dropped before reaching the main server application, resulting in no server logs.
* The application may lack proper error handling or user feedback for large payload failures.
* Image decoding or processing may temporarily consume more memory than the file size itself.

## Test Strategy for Large Image Upload Issue

### 1. Memory Usage Testing

* Monitor client-side memory usage during upload of small vs large images.
* Test on low-memory devices to observe freezing or crashes.
* Upload images of increasing size (e.g., 20MB, 50MB, 100MB).
* Observe memory spikes during:

  * Image preview
  * Encoding
  * Compression

---

### 2. API Request Size Limit Testing

* Verify maximum request body size at:

  * API server
  * Load balancer
  * Reverse proxy
  * CDN (if applicable)
* Test boundary values:

  * Just below limit
  * Just above limit
* Ensure proper error responses are returned.
* Validate client displays meaningful error messages.

---

### 3. Timeout Behavior Testing

* Simulate slow network conditions (3G, high latency).
* Upload large files and observe:

  * Client-side timeout
  * Server-side timeout
  * Idle connection timeout
* Validate proper timeout error handling.
* Check retry mechanisms if available.

---

### 4. Client-Side Handling Testing

* Ensure UI remains responsive during upload.
* Verify loading indicators or progress bars.
* Test cancel upload functionality.
* Confirm error messages are shown properly.
* Test background behavior (minimize app, tab switching).

---

### 5. Server-Side Processing Testing

* Monitor CPU and memory usage during large uploads.
* Check logs for partial uploads or aborted connections.
* Measure image processing performance (resize/compression).

---

### 6. Payload Handling Testing

* Test different formats:

  * JPG
  * PNG
  * HEIC
  * WebP
* Compare base64 upload vs multipart upload.
* Test corrupted or partially uploaded files.

---

## Edge Cases

* File size just below and above limit
* High-resolution images with small file size
* Different image formats
* Corrupted images
* Non-image file renamed as image
* Very long file names or special characters
* Multiple simultaneous uploads
* Slow internet connection
* Network disconnection during upload
* Network switching (Wi-Fi to mobile data)
* App backgrounding during upload
* Cancel upload mid-process
* Server returning Payload Too Large error
* Low-memory or low-storage devices
* Screen rotation during upload

---

## Risks Identified

### 1. User Experience Risks

* App freeze without error reduces usability
* Users may assume app is broken

### 2. Trust & Retention Risks

* Repeated failures can reduce user trust
* Potential app uninstall

### 3. Data Loss Risks

* Selected images or progress may be lost

### 4. Memory Risks

* High RAM usage may crash low-end devices

### 5. Server Load Risks

* Increased CPU and memory usage

### 6. API Limit Risks

* Requests may exceed configured size limits

### 7. Timeout Risks

* Uploads may fail silently due to timeout

### 8. Network Risks

* Slow or unstable networks increase failure rate

### 9. Security Risks

* Large or malformed files may be used for abuse (DoS-like behavior)

### 10. Infrastructure Risks

* Inconsistent limits across CDN, load balancer, and server

### 11. Scalability Risks

* System degradation under concurrent large uploads

### 12. Error Handling Risks

* Missing or unclear error feedback hides root cause
