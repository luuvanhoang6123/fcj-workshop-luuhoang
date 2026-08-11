---
title: "S3, Image Upload, and CloudFront Testing"
date: 2026-08-03
weight: 7
chapter: false
pre: " <b> 5.5.7. </b> "
---

#### Objectives

This testing section verifies the storage and distribution of static content through Amazon S3 and Amazon CloudFront, including:

- The User Frontend is distributed through CloudFront.
- The Admin Frontend is distributed through CloudFront.
- S3 buckets do not allow direct public access.
- CloudFront Origin Access Control (OAC) can read authorized objects correctly.
- Authorized users can upload auction item images to the Item Media Bucket.
- CORS allows only the correct origins and HTTP methods.
- The system rejects unsupported file types or files exceeding the size limit.
- Uploaded images can be displayed correctly on the frontend.
- S3 Versioning creates a new version when an object is replaced.
- Lambda cannot write to buckets outside the scope of its assigned IAM permissions.

The following three buckets must be tested:

| Bucket | Purpose |
| --- | --- |
| User Frontend Bucket | Stores the React build for regular users |
| Admin Frontend Bucket | Stores the React build for administrators |
| Item Media Bucket | Stores images of auction items |

---

#### Frontend Distribution Flow

```text
Browser
-> CloudFront
-> Origin Access Control
-> Private S3 Frontend Bucket
-> CloudFront
-> Browser
````

#### Auction Item Image Upload Flow

```text
Authenticated User
-> Backend API or Lambda
-> Generate presigned URL/presigned POST
-> Browser uploads image directly to Item Media Bucket
-> S3 stores the object
-> CloudFront or a distribution URL serves the image
-> Frontend displays the image
```

---

#### General Test Prerequisites

Before testing, the system must meet the following conditions:

* The User Frontend Bucket exists.
* The Admin Frontend Bucket exists.
* The Item Media Bucket exists.
* Block Public Access is enabled for all three buckets.
* User Frontend and Admin Frontend have been deployed to S3.
* Two CloudFront distributions have been created or configured to distribute the correct frontend applications.
* CloudFront uses OAC instead of exposing S3 buckets publicly.
* Bucket policies allow only the correct CloudFront distribution to read objects.
* The React build contains `index.html`, JavaScript, CSS, and all required assets.
* React route fallback is configured.
* CORS is enabled for the Item Media Bucket.
* Versioning is enabled for the Item Media Bucket if required by the system.
* The API or Lambda responsible for generating presigned uploads has been deployed.
* Authentication and authorization for uploads have been implemented.
* File type and file size are validated server-side or through a presigned POST policy.
* Lambda uses a dedicated IAM role with least-privilege permissions.
* CloudFront and S3 access logs or CloudTrail Data Events are enabled if detailed evidence is required.
* The testing environment is isolated from production.

If a required component has not been implemented, the corresponding test case must be marked as `BLOCKED`.

---

#### Determine the Image Upload Method Before Testing

Before running CORS test cases, the team must identify exactly which upload method the frontend uses.

##### Case 1: Presigned PUT URL

The frontend sends the file content directly using:

```http
PUT /object-key HTTP/1.1
Content-Type: image/jpeg
```

CORS must allow at least:

```text
PUT
```

##### Case 2: Presigned POST

The frontend submits a `multipart/form-data` form directly to S3 using:

```http
POST / HTTP/1.1
Content-Type: multipart/form-data
```

CORS must allow at least:

```text
POST
```

---

#### Test Data

| Data                    | Description                                                              |
| ----------------------- | ------------------------------------------------------------------------ |
| User Frontend URL       | CloudFront URL for regular users                                         |
| Admin Frontend URL      | CloudFront URL for administrators                                        |
| User Bucket Object URL  | Direct URL to an object in the User Frontend Bucket                      |
| Admin Bucket Object URL | Direct URL to an object in the Admin Frontend Bucket                     |
| Item Media Object URL   | Direct URL to an image in the Item Media Bucket                          |
| Trusted User Origin     | Allowed origin for the User Frontend                                     |
| Trusted Admin Origin    | Allowed origin for the Admin Frontend                                    |
| Untrusted Origin        | Origin not included in the CORS allowlist                                |
| Valid User              | Authenticated User with permission to upload images                      |
| Invalid User            | User who is not logged in, has an invalid token, or has an expired token |
| Valid Image             | Valid JPEG, PNG, or WebP file according to system rules                  |
| Invalid File            | `.exe`, `.html`, `.js`, `.pdf`, or another prohibited file type          |
| Oversized Image         | File larger than the upload size limit                                   |
| Existing Object Key     | Existing object key used to test Versioning                              |
| Allowed Lambda          | Lambda allowed to write to the Item Media Bucket                         |
| Restricted Lambda       | Lambda that is not allowed to write to frontend buckets                  |

Production data must not be used during testing.

---

### STORAGE-01 — Access User Frontend Through CloudFront

| Field               | Content                                                                                                                                                                               |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `STORAGE-01`                                                                                                                                                                          |
| **Test Name**       | Access User Frontend through CloudFront                                                                                                                                               |
| **Objective**       | Verify that users can open the User Frontend from the CloudFront distribution.                                                                                                        |
| **Prerequisites**   | User Frontend has been built and deployed; the CloudFront distribution is in `Deployed` state.                                                                                        |
| **Test Steps**      | 1. Open a browser in incognito mode. 2. Access the User Frontend CloudFront URL. 3. Check the HTTP response. 4. Verify the interface and browser console. 5. Check response headers.  |
| **Expected Result** | The page returns `200 OK`; the interface renders correctly; no XML Access Denied response appears; no critical asset loading errors occur; the response is served through CloudFront. |
| **Actual Result**   | Record the URL, status code, response time, and display result.                                                                                                                       |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                         |
| **Evidence**        | Interface screenshot, Network tab, and response headers.                                                                                                                              |

---

### STORAGE-02 — Access Admin Frontend Through CloudFront

| Field               | Content                                                                                                                                                                                                        |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `STORAGE-02`                                                                                                                                                                                                   |
| **Test Name**       | Access Admin Frontend through CloudFront                                                                                                                                                                       |
| **Objective**       | Verify that the Admin Frontend is distributed from the correct CloudFront distribution.                                                                                                                        |
| **Prerequisites**   | Admin Frontend has been built and deployed; the CloudFront distribution is ready.                                                                                                                              |
| **Test Steps**      | 1. Open the Admin Frontend CloudFront URL. 2. Check the status code. 3. Verify the login page or default page. 4. Check the Network tab and browser console. 5. Confirm which distribution serves the request. |
| **Expected Result** | The page returns `200 OK`; the Admin interface renders correctly; the User Frontend is not served by mistake; assets are distributed through the correct CloudFront distribution.                              |
| **Actual Result**   | Record the URL, distribution, status code, and interface result.                                                                                                                                               |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                  |
| **Evidence**        | Admin interface screenshot, Network tab, and CloudFront headers.                                                                                                                                               |

---

### STORAGE-03 — Load `index.html`, JavaScript, and CSS

| Field               | Content                                                                                                                                                                                                       |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `STORAGE-03`                                                                                                                                                                                                  |
| **Test Name**       | Load all frontend assets                                                                                                                                                                                      |
| **Objective**       | Verify that `index.html` and JavaScript/CSS bundles load successfully with the correct content types.                                                                                                         |
| **Prerequisites**   | The frontend build has been fully uploaded to S3.                                                                                                                                                             |
| **Test Steps**      | 1. Open the User Frontend and Admin Frontend. 2. Open Developer Tools -> Network. 3. Reload the page. 4. Filter by `Doc`, `JS`, and `CSS`. 5. Check status code, `Content-Type`, and response body.           |
| **Expected Result** | `index.html`, JavaScript, and CSS all return `200 OK`; JavaScript has the appropriate content type; CSS returns `text/css`; JavaScript/CSS files are not replaced by `index.html`; no MIME type errors occur. |
| **Actual Result**   | Record the object name, status code, content type, and size.                                                                                                                                                  |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                 |
| **Evidence**        | Network waterfall, response headers, and browser console.                                                                                                                                                     |

---

### STORAGE-04 — Reload a React Route

| Field               | Content                                                                                                                                                                                                                                    |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Test Case ID**    | `STORAGE-04`                                                                                                                                                                                                                               |
| **Test Name**       | Reload a client-side React route                                                                                                                                                                                                           |
| **Objective**       | Verify that React routes work when accessed directly or when the page is reloaded.                                                                                                                                                         |
| **Prerequisites**   | The application uses client-side routing; CloudFront is configured with an appropriate fallback.                                                                                                                                           |
| **Test Steps**      | 1. Access a valid route, for example `/auction-items/item-001`. 2. Reload the page using `Ctrl+R` or `F5`. 3. Paste the route URL directly into a new tab. 4. Check the response and interface. 5. Attempt to access a non-existent asset. |
| **Expected Result** | A valid route renders correctly instead of returning S3 `AccessDenied` or another unintended error; React initializes successfully; a missing asset is not incorrectly masked as a successful HTML response outside the intended design.   |
| **Actual Result**   | Record the route, status code, response type, and display result.                                                                                                                                                                          |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                              |
| **Evidence**        | Route URL, Network tab, CloudFront configuration, and interface screenshot.                                                                                                                                                                |

---

### STORAGE-05 — Reject Direct Access to Private S3 Objects

| Field               | Content                                                                                                                                                                                              |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `STORAGE-05`                                                                                                                                                                                         |
| **Test Name**       | Block direct access to private S3 buckets                                                                                                                                                            |
| **Objective**       | Verify that Internet users cannot bypass CloudFront to read objects directly from S3.                                                                                                                |
| **Prerequisites**   | Block Public Access is enabled; the bucket does not expose direct public website hosting.                                                                                                            |
| **Test Steps**      | 1. Copy the direct S3 object URL for `index.html`. 2. Open the URL in an incognito browser. 3. Repeat with JavaScript, CSS, or image objects. 4. Check the bucket policy and public access settings. |
| **Expected Result** | Direct requests are rejected with `403 AccessDenied` or an equivalent result; the object is not returned; the bucket and objects are not public.                                                     |
| **Actual Result**   | Record the object URL with the bucket name masked if necessary, status code, and response code.                                                                                                      |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                        |
| **Evidence**        | `403` response, Block Public Access settings, and bucket policy.                                                                                                                                     |

---

### STORAGE-06 — CloudFront OAC Can Read the Object

| Field               | Content                                                                                                                                                                                                                   |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `STORAGE-06`                                                                                                                                                                                                              |
| **Test Name**       | CloudFront accesses private S3 through OAC                                                                                                                                                                                |
| **Objective**       | Verify that OAC allows only the intended CloudFront distribution to read objects from the private bucket.                                                                                                                 |
| **Prerequisites**   | OAC is attached to the S3 origin; the bucket policy is restricted using the CloudFront service principal and distribution ARN.                                                                                            |
| **Test Steps**      | 1. Access an object through its CloudFront URL. 2. Confirm that the request succeeds. 3. Access the same object directly using the S3 URL. 4. Check origin configuration. 5. Check the bucket policy and `AWS:SourceArn`. |
| **Expected Result** | CloudFront successfully returns the object; direct S3 access is rejected; OAC is active; the bucket policy does not grant public read access and allows only the intended distribution.                                   |
| **Actual Result**   | Record the CloudFront status, direct S3 status, OAC ID, and masked distribution ARN if necessary.                                                                                                                         |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                             |
| **Evidence**        | Two HTTP responses, CloudFront origin configuration, and bucket policy.                                                                                                                                                   |

---

### STORAGE-07 — Authorized User Uploads an Auction Item Image

| Field               | Content                                                                                                                                                                                                                                                                                           |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `STORAGE-07`                                                                                                                                                                                                                                                                                      |
| **Test Name**       | Image upload by an authorized User                                                                                                                                                                                                                                                                |
| **Objective**       | Verify that an authorized User can upload an auction item image to the correct Item Media Bucket.                                                                                                                                                                                                 |
| **Prerequisites**   | The User is authenticated; the User has permission to edit the item; the presigned upload API is operational.                                                                                                                                                                                     |
| **Test Steps**      | 1. Log in as Valid User. 2. Select an item the User is allowed to edit. 3. Select a Valid Image. 4. Request a presigned upload. 5. Upload using the correct `POST` or `PUT` method. 6. Check the S3 response. 7. Verify the object and its metadata in the bucket.                                |
| **Expected Result** | The backend verifies the User and item-level permission; the presigned upload is generated for the correct bucket/key; the upload succeeds; the object appears under the correct prefix; content type and size are correct; the URL expires and does not grant broader permissions than required. |
| **Actual Result**   | Record the masked User ID, Item ID, method, object key, status code, and size.                                                                                                                                                                                                                    |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                                     |
| **Evidence**        | API response with signature masked, Network request, S3 object metadata, and CloudWatch Logs.                                                                                                                                                                                                     |

Do not include a full active presigned URL in the test documentation because it contains signature information that may still be used to upload an object.

---

### STORAGE-08 — CORS Allows the Correct Origin and HTTP Method

| Field               | Content                                                                                                                                                                                                                           |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `STORAGE-08`                                                                                                                                                                                                                      |
| **Test Name**       | Allow uploads from a trusted origin                                                                                                                                                                                               |
| **Objective**       | Verify that the Item Media Bucket allows only the intended frontend origin and actual upload method.                                                                                                                              |
| **Prerequisites**   | The application has been confirmed to use either presigned `POST` or presigned `PUT`.                                                                                                                                             |
| **Test Steps**      | 1. Open the frontend from a trusted origin. 2. Perform an upload. 3. Check the `OPTIONS` request if present. 4. Check `Access-Control-Allow-Origin`. 5. Check `Access-Control-Allow-Methods`. 6. Check the actual upload request. |
| **Expected Result** | The preflight succeeds; the response allows only the trusted origin; the actual `POST` or `PUT` method is allowed; required headers such as `Content-Type` are accepted; the browser completes the upload without a CORS error.   |
| **Actual Result**   | Record the origin, method, request headers, and CORS response headers.                                                                                                                                                            |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                     |
| **Evidence**        | Network tab showing the `OPTIONS` request and upload request.                                                                                                                                                                     |

---

### STORAGE-09 — Reject an Untrusted Origin

| Field               | Content                                                                                                                                                                                                                                                                   |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `STORAGE-09`                                                                                                                                                                                                                                                              |
| **Test Name**       | Prevent uploads from an untrusted origin                                                                                                                                                                                                                                  |
| **Objective**       | Verify that an origin outside the allowlist is not permitted by CORS.                                                                                                                                                                                                     |
| **Prerequisites**   | A test origin exists that is not included in the CORS configuration.                                                                                                                                                                                                      |
| **Test Steps**      | 1. Send a preflight request with `Origin` set to the untrusted origin. 2. Use the same requested method used by the application. 3. Check response headers. 4. Attempt the upload from a browser running on the untrusted origin. 5. Verify whether an object is created. |
| **Expected Result** | The response does not return `Access-Control-Allow-Origin` for the untrusted origin; the browser blocks the request through CORS; the upload does not complete through the normal browser flow; no unintended object is created.                                          |
| **Actual Result**   | Record the origin, preflight status, CORS headers, and object result.                                                                                                                                                                                                     |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                             |
| **Evidence**        | Preflight response, browser console, and S3 object verification.                                                                                                                                                                                                          |

CORS is not an authentication mechanism. A request executed outside a browser may still use a valid presigned URL. Therefore, presigned URLs must have short expiration times and be restricted to the correct key, method, content type, and required conditions.

---

### STORAGE-10 — Reject Unsupported File Types

| Field               | Content                                                                                                                                                                                                                                                        |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `STORAGE-10`                                                                                                                                                                                                                                                   |
| **Test Name**       | Reject unsupported file types                                                                                                                                                                                                                                  |
| **Objective**       | Verify that the system does not accept files outside the approved image format allowlist.                                                                                                                                                                      |
| **Prerequisites**   | An image format allowlist and file type validation mechanism have been implemented.                                                                                                                                                                            |
| **Test Steps**      | 1. Select a `.exe`, `.html`, `.js`, or another prohibited file. 2. Try changing its extension to `.jpg`. 3. Request a presigned upload. 4. If a URL is still issued, attempt the upload. 5. Check S3 and logs.                                                 |
| **Expected Result** | The file is rejected with a code such as `UNSUPPORTED_FILE_TYPE`; the system does not rely only on the file extension or client-declared `Content-Type`; the object is not published as a valid image; executable content is not served from the media domain. |
| **Actual Result**   | Record the file name, extension, detected MIME type, and error code.                                                                                                                                                                                           |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                  |
| **Evidence**        | API response, file validation logs, S3 object listing, and frontend.                                                                                                                                                                                           |

---

### STORAGE-11 — Reject Files Exceeding the Size Limit

| Field               | Content                                                                                                                                                                                                                               |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `STORAGE-11`                                                                                                                                                                                                                          |
| **Test Name**       | Reject images exceeding the size limit                                                                                                                                                                                                |
| **Objective**       | Verify that images larger than the allowed limit are rejected.                                                                                                                                                                        |
| **Prerequisites**   | The file size limit has been defined, for example `5 MB`.                                                                                                                                                                             |
| **Test Steps**      | 1. Prepare an image smaller than the limit. 2. Prepare an image exactly at the limit. 3. Prepare an image larger than the limit. 4. Attempt to upload each file. 5. Check the response, S3, and logs.                                 |
| **Expected Result** | Images below or exactly at the limit are handled according to policy; oversized images are rejected with `FILE_TOO_LARGE` or an equivalent code; oversized objects are not stored or are quarantined/deleted according to the design. |
| **Actual Result**   | Record the configured limit, size of each file, status, and error code.                                                                                                                                                               |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                         |
| **Evidence**        | File sizes, API/S3 responses, object metadata, and logs.                                                                                                                                                                              |

With presigned `POST`, size can be controlled using the `content-length-range` policy condition. With presigned `PUT`, the actual size-control mechanism must be verified because frontend-only validation is not sufficiently secure.

---

### STORAGE-12 — Uploaded Image Displays Correctly on the Frontend

| Field               | Content                                                                                                                                                                                                                                                                 |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `STORAGE-12`                                                                                                                                                                                                                                                            |
| **Test Name**       | Distribute and display an auction item image                                                                                                                                                                                                                            |
| **Objective**       | Verify that an uploaded image is correctly associated with the item and displayed through an allowed distribution URL.                                                                                                                                                  |
| **Prerequisites**   | `STORAGE-07` has succeeded; item data stores the correct object key or media URL.                                                                                                                                                                                       |
| **Test Steps**      | 1. Complete the image upload. 2. Open the item detail page. 3. Do not use the S3 Console URL to display the image. 4. Check the image request in the Network tab. 5. Check content type, size, and cache headers. 6. Reopen the page or verify from another browser.    |
| **Expected Result** | The image displays correctly; no broken image appears; the request returns `200 OK`; the content matches the uploaded file; the object is retrieved through CloudFront or another designed URL mechanism; users are not granted permission to browse the entire bucket. |
| **Actual Result**   | Record the Item ID, masked object key, distribution URL, status, and display result.                                                                                                                                                                                    |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                           |
| **Evidence**        | Interface screenshot, Network response, and S3 object metadata.                                                                                                                                                                                                         |

---

### STORAGE-13 — S3 Versioning Creates a New Version

| Field               | Content                                                                                                                                                                                                                                  |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `STORAGE-13`                                                                                                                                                                                                                             |
| **Test Name**       | Create a new version when replacing an object                                                                                                                                                                                            |
| **Objective**       | Verify that overwriting the same object key does not immediately destroy the previous version.                                                                                                                                           |
| **Prerequisites**   | Versioning is enabled on the bucket being tested.                                                                                                                                                                                        |
| **Test Steps**      | 1. Upload image A to a specific object key. 2. Record the first `VersionId`. 3. Upload image B to the same object key. 4. Record the second `VersionId`. 5. List object versions. 6. Check the current version and the previous version. |
| **Expected Result** | Two different `VersionId` values exist; image B is the current version; image A still exists as an older version; no `null` Version ID appears if Versioning was properly enabled before the upload.                                     |
| **Actual Result**   | Record the object key, Version ID 1, Version ID 2, and current version.                                                                                                                                                                  |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                            |
| **Evidence**        | S3 Versions view, object metadata, and CLI output if used.                                                                                                                                                                               |

If the object is distributed through CloudFront using the same URL, cache invalidation or a versioned/hashed object key strategy must also be verified. S3 Versioning does not automatically invalidate an existing CloudFront cache entry.

---

### STORAGE-14 — Lambda Cannot Write Outside Its IAM Scope

| Field               | Content                                                                                                                                                                                                                                                                                                               |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `STORAGE-14`                                                                                                                                                                                                                                                                                                          |
| **Test Name**       | Restrict Lambda S3 write permissions                                                                                                                                                                                                                                                                                  |
| **Objective**       | Verify that Lambda can operate only on the bucket and prefix allowed by IAM.                                                                                                                                                                                                                                          |
| **Prerequisites**   | The Lambda execution role follows least privilege; one allowed bucket/prefix and one out-of-scope bucket/prefix are available for testing.                                                                                                                                                                            |
| **Test Steps**      | 1. Invoke Lambda to write to the allowed Item Media Bucket and prefix. 2. Confirm success. 3. Attempt to write to the User Frontend Bucket. 4. Attempt to write to the Admin Frontend Bucket. 5. Attempt to write to another prefix inside the Item Media Bucket. 6. Check the result and CloudTrail/CloudWatch Logs. |
| **Expected Result** | Lambda successfully writes only to the allowed bucket/prefix; out-of-scope operations are rejected with `AccessDenied`; no overly broad write permission such as `arn:aws:s3:::*/*` exists; the error is logged without exposing sensitive information.                                                               |
| **Actual Result**   | Record the role, action, masked resource, result, and error code.                                                                                                                                                                                                                                                     |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                                                         |
| **Evidence**        | IAM policy, Lambda logs, CloudTrail event, and S3 object listing.                                                                                                                                                                                                                                                     |

Negative testing should be performed with a dedicated test function or controlled test environment. Production Lambda Functions should not be modified intentionally to write to out-of-scope buckets.

---

### Expected Access Permission Matrix

| Component                     | User Frontend Bucket | Admin Frontend Bucket | Item Media Bucket                                 |
| ----------------------------- | -------------------- | --------------------- | ------------------------------------------------- |
| User accesses S3 directly     | Denied               | Denied                | Denied, except through a valid signed URL         |
| User CloudFront Distribution  | Read allowed         | Denied                | According to design                               |
| Admin CloudFront Distribution | Denied               | Read allowed          | According to design                               |
| Upload-generation Lambda      | No write             | No write              | Only generate URLs or write to the allowed prefix |
| Image-processing Lambda       | No write             | No write              | Read/write only within the allowed prefix         |
| Unauthenticated User          | No upload            | No upload             | No presigned upload issued                        |
| Authorized User               | No direct write      | No direct write       | Upload through a restricted presigned request     |

---

### Upload Verification Matrix

| Scenario                                   | Presigned Upload Issued        | S3 Stores Object                    | Frontend Displays |
| ------------------------------------------ | ------------------------------ | ----------------------------------- | ----------------- |
| Authorized User, valid image type and size | Yes                            | Yes                                 | Yes               |
| User not logged in                         | No                             | No                                  | No                |
| User without item permission               | No                             | No                                  | No                |
| Trusted origin                             | According to User permission   | Upload may succeed                  | Yes               |
| Untrusted origin                           | Browser CORS does not allow it | Not through the normal browser flow | No                |
| Invalid file type                          | No or quarantined              | Not stored as a valid image         | No                |
| Oversized file                             | No or rejected by S3           | No                                  | No                |
| Expired presigned URL                      | Not applicable                 | Rejected                            | No                |
| Object key outside signed URL scope        | Not applicable                 | Rejected                            | No                |

---

### CORS Verification Guidelines

CORS for the Item Media Bucket should satisfy the following:

* Only trusted origins are included.
* User Frontend and Admin Frontend are treated separately if their upload permissions differ.
* Only the actual upload method is allowed: `POST` or `PUT`.
* Only necessary headers are allowed.
* `AllowedOrigins: ["*"]` is not used when the design requires restricted origins.
* Unnecessary methods such as `DELETE` are not allowed.
* `OPTIONS` preflight returns the correct CORS headers.
* CORS is not confused with authentication or authorization.
* CORS changes are retested in a real browser.

Example for presigned `PUT`:

```json
[
  {
    "AllowedOrigins": [
      "https://user.example.com"
    ],
    "AllowedMethods": [
      "PUT"
    ],
    "AllowedHeaders": [
      "Content-Type"
    ],
    "ExposeHeaders": [
      "ETag"
    ],
    "MaxAgeSeconds": 3000
  }
]
```

Example for presigned `POST`:

```json
[
  {
    "AllowedOrigins": [
      "https://user.example.com"
    ],
    "AllowedMethods": [
      "POST"
    ],
    "AllowedHeaders": [
      "*"
    ],
    "ExposeHeaders": [
      "ETag"
    ],
    "MaxAgeSeconds": 3000
  }
]
```

The origins and headers in these examples must be replaced with the actual values used by the system. Example configurations should not be copied directly into production without comparing them against real browser requests.

---

### CloudFront Cache Verification Guidelines

* `index.html` should not remain cached for too long after each deployment.
* Files with content hashes in their names, such as JavaScript and CSS, may use long-term caching.
* Object `Content-Type` values must be correct.
* User Frontend and Admin Frontend must not use the wrong origin.
* CloudFront must not expose private S3 URLs.
* Cache keys should not contain unnecessary data.
* Images should be cached according to an appropriate policy.
* When the same object key is replaced, CloudFront should be invalidated or a new object key should be used.
* CloudFront custom error responses should not unintentionally convert asset errors into successful `index.html` responses.
* HTTPS should be enforced or HTTP should redirect to HTTPS.

Recommended cache policy:

| Object Type                                       | Suggested Cache-Control               |
| ------------------------------------------------- | ------------------------------------- |
| `index.html`                                      | `no-cache` or a short cache duration  |
| JavaScript/CSS with content hash                  | `public, max-age=31536000, immutable` |
| Images with immutable object keys                 | `public, max-age=31536000, immutable` |
| Images that may be overwritten using the same key | Short cache duration or invalidation  |

---

### Upload Security Verification Guidelines

The upload flow should verify at minimum:

* The User is authenticated.
* The User has permission to edit the correct item.
* The object key is controlled by the server.
* The User cannot modify the key to overwrite another User's image.
* The input file name is not used directly as the object key if this introduces path manipulation risk.
* Presigned URLs have short expiration times.
* Presigned URLs are valid only for the correct bucket, key, and HTTP method.
* Tokens and full active presigned URLs are not written to logs.
* File extension and `Content-Type` are not the only evidence used to determine file type.
* File size is enforced server-side or through S3 policy conditions.
* Images should be inspected or processed before publication if the system accepts untrusted content.
* Objects must not use a `public-read` ACL.
* Bucket owner enforced should be used when appropriate.
* Server-side encryption is enabled according to requirements.
* Lifecycle rules are configured for old versions or incomplete uploads if necessary.

---

### IAM Verification Guidelines

Lambda IAM roles should follow the principle of least privilege:

* Grant only required actions.
* Grant access only to the correct bucket.
* Restrict access to the required prefix where possible.
* Separate read and write permissions when appropriate.
* Do not use `s3:*`.
* Do not use resource `*` for object operations unless necessary.
* Do not grant write access to the User Frontend or Admin Frontend buckets for media-processing Lambdas.
* Generating a presigned URL does not mean users receive AWS credentials.
* Explicit Deny rules from bucket policies, permission boundaries, or SCPs must still take effect.

Example expected resource scope:

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:PutObject"
  ],
  "Resource": "arn:aws:s3:::item-media-bucket/items/*"
}
```

The bucket name must be replaced with the actual system resource.

---

### Log and Evidence Verification Guidelines

Logs should contain:

* Correlation ID or Request ID.
* Verified User ID, masked where necessary.
* Item ID.
* Object key.
* Upload HTTP method.
* Content type.
* File size.
* Authentication result.
* Authorization result.
* Presigned upload generation result.
* Image processing result.
* S3 error code.
* Lambda Request ID.
* Processing time.

Logs must not contain:

* Access Token.
* ID Token.
* Refresh Token.
* AWS access key or secret key.
* Full active presigned URL.
* Signature values in the query string.
* Login cookies.
* Base64 image data.
* Unnecessary personal information.

---

### Test Result Summary

| ID           | Test Content                      | Main Expected Result                                     | Status |
| ------------ | --------------------------------- | -------------------------------------------------------- | ------ |
| `STORAGE-01` | User Frontend through CloudFront  | Returns `200`, interface displays correctly              | Tested |
| `STORAGE-02` | Admin Frontend through CloudFront | Returns `200`, correct Admin UI                          | Tested |
| `STORAGE-03` | Load HTML, JavaScript, and CSS    | Assets load with the correct content type                | Tested |
| `STORAGE-04` | Reload React route                | Route renders without unintended errors                  | Tested |
| `STORAGE-05` | Direct S3 access                  | Rejected                                                 | Tested |
| `STORAGE-06` | CloudFront OAC reads S3           | CloudFront can read, direct access is blocked            | Tested |
| `STORAGE-07` | Authorized User uploads image     | Image is stored in the correct bucket and prefix         | Tested |
| `STORAGE-08` | CORS trusted origin/method        | Upload succeeds                                          | Tested |
| `STORAGE-09` | CORS untrusted origin             | Browser rejects request                                  | Tested |
| `STORAGE-10` | Invalid file type                 | Rejected                                                 | Tested |
| `STORAGE-11` | Oversized file                    | Rejected                                                 | Tested |
| `STORAGE-12` | Display image on frontend         | Image displays correctly                                 | Tested |
| `STORAGE-13` | S3 Versioning                     | Creates a new Version ID                                 | Tested |
| `STORAGE-14` | Lambda IAM restrictions           | Writes only within scope; out-of-scope writes are denied | Tested |