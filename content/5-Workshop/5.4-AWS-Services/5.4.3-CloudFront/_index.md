---
title: "Amazon CloudFront"
date: 2026-07-27
weight: 3
chapter: false
pre: " <b> 5.4.3. </b> "
---

## Overview

The **Live Auction** system uses **Amazon CloudFront** to distribute the two frontend applications and the media files associated with auction items.

CloudFront operates as a Content Delivery Network (CDN). It receives requests from users, retrieves content from Amazon S3, and delivers the content through AWS Edge Locations.

The system's CloudFront Distributions are created and configured using Terraform in the following module:

```text
infra/09-edge
```

The system uses three CloudFront Distributions:

* A Distribution for the User Frontend.
* A Distribution for the Admin Frontend.
* A Distribution for Item Media content.

## Role of Amazon CloudFront

In the Live Auction system, Amazon CloudFront is used to:

* Deliver the User Frontend to users.
* Deliver the Admin Frontend to administrators.
* Deliver images of auction items.
* Reduce content loading time through AWS Edge Locations.
* Cache static content.
* Redirect HTTP requests to HTTPS.
* Prevent direct public access to S3 Buckets.
* Access Amazon S3 through Origin Access Control.
* Support routing for Single Page Applications.
* Separate the delivery of the User Frontend, Admin Frontend, and Item Media content.

## CloudFront Distributions of the system

| CloudFront Distribution         | Origin                   | Role                                                           |
| ------------------------------- | ------------------------ | -------------------------------------------------------------- |
| **User Frontend Distribution**  | User Frontend S3 Bucket  | Delivers the user-facing application.                          |
| **Admin Frontend Distribution** | Admin Frontend S3 Bucket | Delivers the administrator-facing application.                 |
| **Item Media Distribution**     | Item Media S3 Bucket     | Delivers images and media files associated with auction items. |

Each Distribution has a default domain name provided by CloudFront with the following structure:

```text
xxxxxxxxxxxxxx.cloudfront.net
```

The User Frontend and Admin Frontend use separate Distributions. Therefore, the team can deploy, update, and manage the two applications independently.

## Content delivery flow

The frontend content delivery flow operates as follows:

1. A user accesses the CloudFront Domain Name.
2. CloudFront checks whether the requested content is available in its cache.
3. If the content is available in the cache, CloudFront returns it directly to the user.
4. If the content is not available, CloudFront sends a request to the S3 Origin.
5. CloudFront uses Origin Access Control to sign the request sent to Amazon S3.
6. The S3 Bucket Policy verifies the CloudFront Distribution that sent the request.
7. Amazon S3 returns the requested object to CloudFront.
8. CloudFront caches the appropriate content and returns the result to the user's browser.

The Item Media delivery flow operates in the same manner, but its origin is the Item Media Bucket instead of a frontend bucket.

## Origin Access Control

Each CloudFront Distribution has a corresponding **Origin Access Control (OAC)**:

```text
User Frontend OAC
Admin Frontend OAC
Item Media OAC
```

Origin Access Control is configured with:

```text
Origin type: S3
Signing behavior: Always
Signing protocol: SigV4
```

OAC allows CloudFront to access private S3 Buckets through signed requests. Users are not allowed to access the objects directly through Amazon S3 URLs.

This configuration helps to:

* Keep the S3 Buckets private.
* Prevent unauthorized access to the buckets.
* Allow only the corresponding CloudFront Distribution to read data.
* Separate access permissions between the three S3 Origins.
* Prevent content from being published directly through Amazon S3.

## User Frontend Distribution configuration

The User Frontend Distribution is configured with:

```text
Default root object: index.html
Viewer protocol policy: Redirect HTTP to HTTPS
Allowed methods: GET, HEAD
Cached methods: GET, HEAD
Compress objects automatically: Enabled
Price class: PriceClass_100
```

The Distribution origin is the User Frontend S3 Bucket.

CloudFront allows only the `GET` and `HEAD` methods because the frontend primarily reads and downloads static resources.

Automatic compression is enabled to reduce the size of content delivered to browsers and improve page loading performance.

## Admin Frontend Distribution configuration

The Admin Frontend Distribution is deployed separately and uses a configuration similar to the User Frontend Distribution:

```text
Default root object: index.html
Viewer protocol policy: Redirect HTTP to HTTPS
Allowed methods: GET, HEAD
Cached methods: GET, HEAD
Compress objects automatically: Enabled
Price class: PriceClass_100
```

The Distribution origin is the Admin Frontend S3 Bucket.

Using a separate Distribution prevents the Admin Frontend from sharing its origin and cached content with the User Frontend.

## Item Media Distribution configuration

The Item Media Distribution uses the Item Media Bucket as its origin.

The Distribution is configured with:

```text
Viewer protocol policy: Redirect HTTP to HTTPS
Allowed methods: GET, HEAD
Cached methods: GET, HEAD
Compress objects automatically: Enabled
Price class: PriceClass_100
IPv6: Enabled
```

This Distribution delivers images and media content associated with auction items without making the Item Media Bucket publicly accessible.

## Cache Behavior configuration

The Default Cache Behavior of the Distributions allows:

```text
GET
HEAD
```

The methods stored in the cache include:

```text
GET
HEAD
```

The current configuration does not forward query strings or cookies to the S3 Origin:

```text
Query strings: None
Cookies: None
```

This configuration is suitable for static content because Amazon S3 returns the same object based on the requested path.

Using CloudFront caching helps to:

* Reduce the number of requests sent directly to Amazon S3.
* Reduce frontend loading latency.
* Improve the delivery speed of auction item images.
* Reduce load on the origin.
* Improve the user experience.

## HTTPS redirection

The Distributions are configured with:

```text
Viewer protocol policy: Redirect HTTP to HTTPS
```

When a user sends a request using HTTP, CloudFront automatically redirects the request to HTTPS.

Using HTTPS helps to:

* Encrypt data transferred between the browser and CloudFront.
* Reduce the risk of network eavesdropping.
* Ensure that content is not modified during transmission.
* Improve the security of system access.

The system currently uses the default CloudFront certificate for domains with the following format:

```text
xxxxxxxxxxxxxx.cloudfront.net
```

## Single Page Application support

The User Frontend and Admin Frontend are Single Page Applications.

When a user directly accesses a path such as:

```text
/profile
/my-auctions
/admin/users
```

Amazon S3 may not find a corresponding object and may return a `403` or `404` error.

To allow the frontend router to process these paths, the two frontend Distributions are configured with Custom Error Responses:

| Error Code | Response Code | Response Page |
| ---------- | ------------- | ------------- |
| `403`      | `200`         | `/index.html` |
| `404`      | `200`         | `/index.html` |

CloudFront returns the `index.html` file to the browser, after which the frontend router displays the corresponding page.

The Item Media Distribution does not use this mechanism because a media request must point to an existing object in Amazon S3.

## Verifying CloudFront on the AWS Management Console

### Step 1: Access Amazon CloudFront

Sign in to the **AWS Management Console**.

Enter the following value in the search bar:

```text
CloudFront
```

Select **CloudFront — Content Delivery Network**.

CloudFront is a global service. Therefore, the Distribution list is not completely dependent on the AWS Region selected in the navigation bar.

### Step 2: Verify the Distribution list

From the navigation menu, select:

```text
Distributions
```

Verify the three Distributions of the system:

* User Frontend Distribution.
* Admin Frontend Distribution.
* Item Media Distribution.

The following information should be verified:

* Distribution ID.
* Description.
* Status.
* Last modified time.
* Distribution Domain Name.
* Corresponding origin.
* Enabled state.

The deployment should display the following states:

```text
Enabled
Deployed
```

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.3-CloudFront/cloudfront-distribution-list.png"
    title="Figure 5.4.3.1: CloudFront Distributions of the Live Auction system"
    width="100%"
>}}

### Step 3: Verify the User Frontend Distribution

Select the Distribution whose Description corresponds to the User Frontend.

Open the following tab:

```text
General
```

Verify:

* Distribution status.
* Distribution Domain Name.
* Distribution ID.
* Price class.
* Supported HTTP versions.
* IPv6 configuration.
* Default root object.

The Default Root Object should be:

```text
index.html
```

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.3-CloudFront/cloudfront-user-frontend-general.png"
    title="Figure 5.4.3.2: User Frontend CloudFront Distribution details"
    width="100%"
>}}

### Step 4: Verify the User Frontend Origin

In the User Frontend Distribution, open:

```text
Origins
```

Verify:

* The Origin Domain points to the User Frontend S3 Bucket.
* The Origin Type is Amazon S3.
* Origin Access Control is configured.
* The Origin Path, if used.
* The Origin does not incorrectly point to the Admin Frontend Bucket or Media Bucket.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.3-CloudFront/cloudfront-user-frontend-origin.png"
    title="Figure 5.4.3.3: S3 Origin of the User Frontend Distribution"
    width="100%"
>}}

### Step 5: Verify Cache Behavior

In the Distribution, open:

```text
Behaviors
```

Select the Default Behavior and verify:

* Path pattern.
* Origin.
* Viewer protocol policy.
* Allowed HTTP methods.
* Cached HTTP methods.
* Automatic object compression.
* Query string forwarding.
* Cookie forwarding.

The following values should be confirmed:

```text
Path pattern: Default (*)
Viewer protocol policy: Redirect HTTP to HTTPS
Allowed methods: GET, HEAD
Cached methods: GET, HEAD
Compress objects automatically: Yes
```

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.3-CloudFront/cloudfront-cache-behavior.png"
    title="Figure 5.4.3.4: Default Cache Behavior of a CloudFront Distribution"
    width="100%"
>}}

### Step 6: Verify Custom Error Responses

In the User Frontend Distribution, open:

```text
Error pages
```

Verify the following two configurations:

```text
403 → 200 → /index.html
404 → 200 → /index.html
```

These configurations allow the frontend router to process Single Page Application routes.

The Admin Frontend Distribution should contain the same two configurations.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.3-CloudFront/cloudfront-custom-error-responses.png"
    title="Figure 5.4.3.5: Custom Error Responses supporting the Single Page Application"
    width="100%"
>}}

### Step 7: Verify the Admin Frontend Distribution

Return to the Distribution list and select the Admin Frontend Distribution.

Verify:

* The Distribution is Enabled and Deployed.
* The Origin points to the Admin Frontend S3 Bucket.
* The Default Root Object is `index.html`.
* The Viewer Protocol Policy redirects HTTP to HTTPS.
* The allowed methods are `GET` and `HEAD`.
* Custom Error Responses are configured for `403` and `404`.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.3-CloudFront/cloudfront-admin-frontend.png"
    title="Figure 5.4.3.6: CloudFront Distribution for the Admin Frontend"
    width="100%"
>}}

### Step 8: Verify the Item Media Distribution

Select the Item Media Distribution and open:

```text
Origins
```

Verify:

* The Origin points to the Item Media Bucket.
* Origin Access Control is assigned.
* The Distribution is Enabled.
* The Viewer Protocol Policy redirects HTTP to HTTPS.
* The allowed methods are `GET` and `HEAD`.
* IPv6 is enabled.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.3-CloudFront/cloudfront-item-media.png"
    title="Figure 5.4.3.7: CloudFront Distribution for Item Media"
    width="100%"
>}}

### Step 9: Verify Origin Access Control

From the CloudFront navigation menu, select:

```text
Origin access
```

Verify the Origin Access Controls of the system.

The following information should be confirmed:

* The OAC for the User Frontend.
* The OAC for the Admin Frontend.
* The OAC for Item Media.
* Origin Type is S3.
* Signing Behavior is Always.
* Signing Protocol is SigV4.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.3-CloudFront/cloudfront-origin-access-control.png"
    title="Figure 5.4.3.8: Origin Access Controls of the S3 Origins"
    width="100%"
>}}

## Verifying access through CloudFront

After verifying the configurations on the AWS Management Console, the CloudFront Domain Names of the User Frontend and Admin Frontend can be opened in a browser.

For example:

```text
https://xxxxxxxxxxxxxx.cloudfront.net
```

The expected results include:

* The User Frontend displays the user-facing interface.
* The Admin Frontend displays the administrator sign-in interface.
* The browser uses an HTTPS connection.
* Refreshing a nested frontend route does not return an XML error from Amazon S3.
* Auction item images are loaded through the Media CloudFront Domain.

Complete functional testing of the frontend applications is presented in **Section 5.5 — System Testing**.

## Results

After verifying the resources directly through the AWS Management Console, the team confirmed that:

* Three CloudFront Distributions were successfully created by Terraform.
* The User Frontend and Admin Frontend were delivered through separate Distributions.
* Item Media content was delivered through a dedicated Distribution.
* The Distributions were Enabled and Deployed.
* Each Distribution pointed to the correct S3 Origin.
* Origin Access Control was configured for the private S3 Buckets.
* HTTP requests were redirected to HTTPS.
* The `GET` and `HEAD` methods were allowed and cached.
* Content compression was enabled.
* The User Frontend and Admin Frontend used `index.html` as the Default Root Object.
* Custom Error Responses were configured to support Single Page Application routing.
* The S3 Buckets remained private while their content was delivered through CloudFront.
* The system was ready to deliver both frontend applications and media content to users.

The deployment and verification of Lambda functions used to process application logic are presented in **Section 5.4.4 — AWS Lambda**.