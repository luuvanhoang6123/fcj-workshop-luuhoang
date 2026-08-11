---
title: "Amazon S3"
date: 2026-07-27
weight: 2
chapter: false
pre: " <b> 5.4.2. </b> "
---

## Overview

The **Live Auction** system uses **Amazon Simple Storage Service (Amazon S3)** to store two frontend applications and media files associated with auction items.

The three main content groups stored in Amazon S3 include:

* The user-facing application — User Frontend.
* The administrator-facing application — Admin Frontend.
* Images and media files of auction items.

The S3 Buckets are created and configured using Terraform in the following two modules:

```text
infra/04-data
infra/09-edge
```

The `04-data` module creates the Media Bucket used to store item images. The `09-edge` module creates the two frontend buckets and connects the buckets to Amazon CloudFront.

## Role of Amazon S3

Amazon S3 is a highly scalable object storage service provided by AWS.

In the Live Auction system, Amazon S3 is used to:

* Store the HTML, CSS, JavaScript, and static resources of the User Frontend.
* Store the Admin Frontend files separately.
* Store images and media files of auction items.
* Serve as the origin for CloudFront Distributions.
* Support frontend content delivery to users.
* Support the management of multiple object versions.
* Encrypt data stored in the buckets.
* Prevent direct public access to the buckets.
* Allow CloudFront to access objects through Origin Access Control.

## S3 Buckets of the system

The system uses three main S3 Buckets.

| S3 Bucket                 | Role                                                             |
| ------------------------- | ---------------------------------------------------------------- |
| **User Frontend Bucket**  | Stores the build output of the user-facing application.          |
| **Admin Frontend Bucket** | Stores the build output of the administrator-facing application. |
| **Item Media Bucket**     | Stores images and media files associated with auction items.     |

Bucket names follow the project naming convention and include the AWS Account ID to ensure that each bucket name is globally unique in Amazon S3.

The general naming structure is as follows:

```text
<name-prefix>-<environment>-frontend-<account-id>
<name-prefix>-<environment>-admin-frontend-<account-id>
<name-prefix>-item-media-<account-id>-<aws-region>
```

Examples with the AWS Account ID masked:

```text
la-dev-frontend-************
la-dev-admin-frontend-************
la-item-media-************-ap-southeast-1
```

## User Frontend Bucket

The User Frontend Bucket stores the build output of the user-facing application.

Common files and directories in the bucket include:

```text
index.html
assets/
favicon.ico
```

The `assets` directory can contain:

* Compiled JavaScript files.
* CSS files.
* Static images.
* Fonts and other interface resources.

Users do not access the S3 Bucket directly. Instead, its content is delivered through Amazon CloudFront.

## Admin Frontend Bucket

The Admin Frontend Bucket is deployed separately to store the administrator-facing application.

Using a separate bucket provides the following benefits:

* Separates the User Frontend from the Admin Frontend.
* Allows the two applications to be deployed independently.
* Allows the content of each application to be managed separately.
* Supports separate CloudFront Distribution configurations.
* Reduces the risk of mixing User and Admin resources.
* Allows one frontend to be updated without affecting the other.

The Admin Frontend Bucket also blocks public access and delivers its content only through CloudFront.

## Item Media Bucket

The Item Media Bucket stores images and media files for items added to auction sessions.

The bucket name follows this structure:

```text
<name-prefix>-item-media-<account-id>-<aws-region>
```

The Media Bucket is configured to:

* Block all public access.
* Use Bucket Owner Enforced ownership.
* Enable Versioning.
* Apply server-side encryption using `AES256`.
* Configure CORS for the required operations.
* Remove incomplete multipart uploads after seven days.
* Allow CloudFront to read objects through the Bucket Policy.
* Prevent Terraform from automatically deleting a non-empty bucket.

The `force_destroy = false` configuration helps prevent accidental bucket deletion while the bucket still contains data.

## S3 access mechanism

The two frontend applications and media content are not made publicly accessible directly from Amazon S3.

The content access flow operates as follows:

1. A user accesses the domain name of a CloudFront Distribution.
2. CloudFront receives the request.
3. CloudFront uses Origin Access Control to request content from Amazon S3.
4. The S3 Bucket Policy verifies the CloudFront Distribution that sent the request.
5. If the request is valid, Amazon S3 returns the requested object to CloudFront.
6. CloudFront delivers the content to the user's browser.

This mechanism keeps the S3 Buckets private while allowing their content to be distributed through CloudFront.

{{% notice info %}}
The two frontend buckets do not need **Static website hosting** because CloudFront uses the S3 Regional Domain Name and Origin Access Control to access them. Enabling Static website hosting would not be appropriate for the OAC mechanism used by the system.
{{% /notice %}}

## Security configurations

### Block Public Access

All Block Public Access settings are enabled for the system buckets:

```text
Block public ACLs
Ignore public ACLs
Block public bucket policies
Restrict public buckets
```

This configuration prevents users from accessing content directly through an Amazon S3 URL.

### Server-side Encryption

The buckets are configured to use server-side encryption with:

```text
Amazon S3 managed keys — SSE-S3
AES256
```

Amazon S3 automatically encrypts objects when they are stored and decrypts them when they are retrieved through an authorized request.

### Bucket Versioning

Versioning is enabled so that Amazon S3 can retain multiple versions of the same object.

This configuration supports:

* Restoring files after they are overwritten.
* Reducing the impact of accidental object deletion.
* Tracking different versions of frontend content.
* Improving the recoverability of media data.

### Bucket Policy and Origin Access Control

The User Frontend Bucket, Admin Frontend Bucket, and Media Bucket use Bucket Policies that allow CloudFront to read objects.

The primary permission used is:

```text
s3:GetObject
```

The Bucket Policy limits access to the corresponding CloudFront Distribution instead of allowing public access.

## Verifying S3 Buckets on the AWS Management Console

### Step 1: Access Amazon S3

Sign in to the **AWS Management Console**.

Enter the following value in the search bar:

```text
S3
```

Select **S3 — Scalable Storage in the Cloud**.

### Step 2: Open the bucket list

From the navigation menu, select:

```text
General purpose buckets
```

Find the buckets with the following prefix:

```text
la-
```

Verify the existence of:

* The User Frontend Bucket.
* The Admin Frontend Bucket.
* The Item Media Bucket.

The following information should be verified:

* Bucket name.
* AWS Region.
* Creation date.
* Whether the bucket follows the project naming convention.
* Whether all three required buckets were created.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.2-S3/s3-bucket-list.png"
    title="Figure 5.4.2.1: S3 Buckets of the Live Auction system"
    width="100%"
>}}

### Step 3: Verify the User Frontend Bucket content

Select the User Frontend Bucket.

Open the following tab:

```text
Objects
```

Verify the frontend files uploaded to the bucket, such as:

```text
index.html
assets/
```

The presence of `index.html` and the build resource directory confirms that the User Frontend has been uploaded to Amazon S3.

Files containing API endpoints, Client IDs, or other information that should not be disclosed should not be opened in the report screenshots.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.2-S3/s3-user-frontend-objects.png"
    title="Figure 5.4.2.2: Content stored in the User Frontend Bucket"
    width="100%"
>}}

### Step 4: Verify the Admin Frontend Bucket content

Return to the bucket list and select the Admin Frontend Bucket.

Open the following tab:

```text
Objects
```

Verify the build files of the Admin Frontend.

The Admin Frontend content is stored independently from the User Frontend, confirming that the two applications are deployed through separate S3 Buckets.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.2-S3/s3-admin-frontend-objects.png"
    title="Figure 5.4.2.3: Content stored in the Admin Frontend Bucket"
    width="100%"
>}}

### Step 5: Verify the Item Media Bucket content

Return to the bucket list and select the Item Media Bucket.

Open the following tab:

```text
Objects
```

Verify the directories or image objects associated with auction items.

If the bucket does not contain any data, the interface may display an empty state. This does not indicate a deployment failure because media data is created only after the system uploads item images to Amazon S3.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.2-S3/s3-item-media-objects.png"
    title="Figure 5.4.2.4: Item data stored in the Item Media Bucket"
    width="100%"
>}}

### Step 6: Verify Block Public Access

Open a project bucket and select:

```text
Permissions
```

Locate the following section:

```text
Block public access (bucket settings)
```

Verify the following status:

```text
Block all public access: On
```

Next, inspect the **Bucket policy** section to confirm that CloudFront is allowed to read objects from the bucket.

The Bucket Policy should not be modified directly through the AWS Console because its configuration is managed by Terraform.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.2-S3/s3-block-public-access.png"
    title="Figure 5.4.2.5: Block Public Access configuration of an S3 Bucket"
    width="100%"
>}}

### Step 7: Verify Versioning and Encryption

On the bucket details page, open:

```text
Properties
```

Verify the following sections:

```text
Bucket Versioning
Default encryption
```

The required states include:

* Bucket Versioning is enabled.
* Default encryption is enabled.
* The encryption type uses Amazon S3 managed keys.
* The encryption algorithm is `SSE-S3` or `AES256`.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.2-S3/s3-bucket-versioning.png"
    title="Figure 5.4.2.6: Versioning configuration of the User Frontend Bucket"
    width="100%"
>}}

The verification result shows that **Bucket Versioning** is `Enabled`. Therefore, Amazon S3 can retain multiple versions of the same object and support data recovery when a file is overwritten or accidentally deleted.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.2-S3/s3-default-encryption.png"
    title="Figure 5.4.2.7: Default encryption configuration of the User Frontend Bucket"
    width="100%"
>}}

The verification result shows that the bucket uses **Server-side encryption with Amazon S3 managed keys — SSE-S3**. New objects uploaded to the bucket are automatically encrypted by Amazon S3 before being stored.

### Step 8: Verify CORS of the Item Media Bucket

Open the Item Media Bucket and select:

```text
Permissions → Cross-origin resource sharing (CORS)
```

Verify the allowed methods:

```text
GET
POST
```

CORS allows the frontend applications to send supported requests to the Media Bucket from the configured origins.

The complete origin list should not be published if it contains internal environment addresses or other information that should not be disclosed.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.2-S3/s3-media-cors.png"
    title="Figure 5.4.2.8: CORS configuration of the Item Media Bucket"
    width="100%"
>}}

## Results

After verifying the resources directly through the AWS Management Console, the team confirmed that:

* The User Frontend Bucket was created and contained the user interface files.
* The Admin Frontend Bucket was created and contained the administrator interface files.
* The two frontend applications were stored in separate buckets.
* The Item Media Bucket was created to store auction item images.
* Versioning was enabled for the buckets.
* Data was encrypted using Amazon S3 managed keys.
* Block Public Access was enabled to prevent direct access from the Internet.
* The Frontend Buckets and Media Bucket were accessed by CloudFront through Origin Access Control.
* Bucket Policies restricted object read access to the corresponding CloudFront Distributions.
* The Media Bucket was configured with CORS for the required methods.
* The S3 Buckets were ready for integration with Amazon CloudFront and the remaining system components.

The configuration and verification of content delivery from Amazon S3 through Amazon CloudFront are presented in **Section 5.4.3 — Amazon CloudFront**.