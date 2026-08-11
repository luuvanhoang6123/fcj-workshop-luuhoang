---
title: "Blog 3 - Optimizing Amazon S3 Storage Costs"
date: 2026-08-09
weight: 3
chapter: false
pre: " <b> 3.3. </b> "
---

# Optimizing Amazon S3 Storage Costs: Understanding Storage Classes and Lifecycle Policies

## Overview

When first learning about AWS, Amazon S3 is often understood as a simple storage service: upload a file to a bucket, store it, and download it when needed.

However, data in a system does not always have the same access frequency or level of importance. Some files are accessed regularly, while others are retained only for backup, auditing, or legal compliance.

If all data remains in **S3 Standard** throughout its lifecycle, the system may continue paying the storage rate intended for frequently accessed data, even when most objects are rarely used.

Amazon S3 provides multiple **Storage Classes** and **Lifecycle Policies** that help organizations select suitable storage tiers and automatically move data to lower-cost classes over time.

{{% notice note %}}
The actual cost savings depend on the AWS Region, storage volume, object size, access frequency, retrieval fees, and retention period. Therefore, no fixed savings percentage applies to every workload.
{{% /notice %}}

## The Problem: Not All Data Is Accessed in the Same Way

Suppose an S3 bucket stores:

* Application logs.
* Images uploaded by users.
* Database backup files.
* System audit reports.
* Documents that must be retained for a long period.

These data types may have different access patterns:

```text
Logs from the current week
→ Frequently accessed for monitoring and troubleshooting.

Logs from the previous month
→ Occasionally accessed for investigation.

Logs from six months ago
→ Rarely accessed but retained for auditing.

Older backup files
→ Retrieved only when system recovery is required.
```

If all these objects remain in S3 Standard, the system continues paying for a storage class intended for frequently accessed data, even though many objects are rarely used.

The appropriate Storage Class should therefore be selected based on:

* Data access frequency.
* Required retention period.
* Acceptable restoration time.
* Whether the data can be recreated.
* Durability and availability requirements.
* Storage and retrieval costs.
* Audit or legal requirements.

## Overview of S3 Storage Classes

Amazon S3 provides several storage classes for different workload requirements.

| Storage Class | Access frequency | Retrieval time | Typical use case |
| --- | --- | --- | --- |
| **S3 Standard** | Frequent | Immediate | Websites, applications, and active data |
| **S3 Intelligent-Tiering** | Unknown or changing | Immediate for online access tiers | Data with unpredictable access patterns |
| **S3 Standard-IA** | Infrequent | Immediate | Backups and data that still requires rapid retrieval |
| **S3 One Zone-IA** | Infrequent | Immediate | Re-creatable data stored in one Availability Zone |
| **S3 Glacier Instant Retrieval** | Rare | Immediate | Archived data that occasionally requires instant access |
| **S3 Glacier Flexible Retrieval** | Very rare | Minutes to several hours | Backups and long-term archives |
| **S3 Glacier Deep Archive** | Almost never accessed | Several hours | Long-term retention and regulatory archives |

## S3 Standard

**S3 Standard** is designed for frequently accessed data that requires low latency and high throughput.

It is suitable for:

* Static website resources.
* Frequently accessed images.
* Active application data.
* Frequently read or updated files.
* Content distributed through Amazon CloudFront.

Advantages include:

* Low access latency.
* High throughput.
* No restoration delay.
* Support for many application workloads.

Its main disadvantage is that its storage cost is generally higher than storage classes designed for infrequently accessed data.

## S3 Intelligent-Tiering

**S3 Intelligent-Tiering** is suitable for data with changing or unpredictable access patterns.

It monitors object access activity and automatically moves objects between suitable access tiers. This reduces the need for administrators to determine the exact transition date for every object.

This storage class is useful when:

* The future access frequency is unknown.
* Access patterns change over time.
* Automatic cost optimization is required.
* Complex lifecycle transition rules should be avoided.

S3 Intelligent-Tiering includes monitoring and automation charges for eligible objects. Therefore, object count and size should be considered before using it.

## S3 Standard-IA

**S3 Standard-Infrequent Access (Standard-IA)** is intended for data that is accessed less frequently but still requires immediate retrieval.

It is suitable for:

* Backup files.
* Disaster recovery data.
* Older documents that are occasionally required.
* Data that is no longer actively used.

S3 Standard-IA has a lower storage cost than S3 Standard, but retrieval charges apply when objects are accessed.

If an object is accessed frequently, its total cost may be higher than keeping it in S3 Standard.

## S3 One Zone-IA

**S3 One Zone-IA** is similar to Standard-IA, but the data is stored in only one Availability Zone.

Advantages include:

* Lower storage cost than Standard-IA.
* Low-latency object retrieval.

Limitations include:

* It does not provide the same multi-Availability Zone resiliency as Standard-IA.
* It is not suitable for the only copy of critical data.

This storage class should be used for:

* Data that can be recreated.
* Secondary copies of existing data.
* Thumbnails or generated resources.
* Data that does not require multi-AZ resiliency.

## S3 Glacier Instant Retrieval

**S3 Glacier Instant Retrieval** is intended for archived data that is rarely accessed but must be available immediately when requested.

Suitable use cases include:

* Medical images.
* Long-term media archives.
* Documents that are rarely accessed but require immediate retrieval.
* Archived data with low-latency access requirements.

This class provides lower storage costs than frequently accessed classes, but retrieval charges and minimum storage-duration requirements apply.

## S3 Glacier Flexible Retrieval

**S3 Glacier Flexible Retrieval** is suitable for archived data that does not require immediate access.

Depending on the selected retrieval option, restoring data may take from several minutes to several hours.

Suitable use cases include:

* Periodic backups.
* Long-term archived data.
* Older application logs.
* Audit files.
* Data restored only in exceptional situations.

This is an appropriate option when reducing storage costs is more important than immediate retrieval.

## S3 Glacier Deep Archive

**S3 Glacier Deep Archive** is intended for data that is almost never accessed and must be retained for a long period.

It is suitable for:

* Data retained for regulatory requirements.
* Records stored for several years.
* Long-term backups.
* Audit information.
* Data that replaces traditional tape archives.

Glacier Deep Archive provides a very low storage cost, but restoring data may take several hours. Therefore, it is unsuitable for information that must be accessed immediately.

## S3 Lifecycle Policies

Manually moving individual objects between Storage Classes requires time and may introduce configuration errors.

An **S3 Lifecycle Policy** allows Amazon S3 to manage objects automatically throughout their lifecycle.

A Lifecycle Policy can:

* Transition objects to another Storage Class.
* Manage previous object versions.
* Delete objects after the retention period.
* Delete old versions that are no longer required.
* Remove incomplete multipart uploads.
* Apply rules based on object prefixes.
* Apply rules based on object tags.

Amazon S3 performs transitions and expiration automatically. No separate cron job or Lambda function is required.

## Example Lifecycle Policy

A log-retention policy could be designed as follows:

```text
Days 0–30
→ Keep data in S3 Standard for regular troubleshooting.

After 30 days
→ Transition data to S3 Standard-IA.

After 90 days
→ Transition data to S3 Glacier Flexible Retrieval.

After 365 days
→ Transition data to S3 Glacier Deep Archive.

After 7 years
→ Delete the objects automatically when retention ends.
```

The transition flow is:

```text
S3 Standard
     ↓ After 30 days
S3 Standard-IA
     ↓ After 90 days
S3 Glacier Flexible Retrieval
     ↓ After 365 days
S3 Glacier Deep Archive
     ↓ After 7 years
Expiration
```

These periods are examples only. Actual transition dates should be selected according to access requirements, retention policies, and estimated costs.

## Example Lifecycle Configuration with Terraform

A Lifecycle Policy can be managed with Terraform to keep the configuration consistent across deployments.

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "log_lifecycle" {
  bucket = aws_s3_bucket.application_logs.id

  rule {
    id     = "archive-application-logs"
    status = "Enabled"

    filter {
      prefix = "logs/"
    }

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    transition {
      days          = 365
      storage_class = "DEEP_ARCHIVE"
    }

    expiration {
      days = 2555
    }
  }
}
```

In this example:

* The rule applies only to objects with the `logs/` prefix.
* Objects transition to Standard-IA after 30 days.
* Objects transition to Glacier Flexible Retrieval after 90 days.
* Objects transition to Deep Archive after 365 days.
* Objects expire after 2,555 days, which is approximately seven years.

## Applying Lifecycle Rules by Prefix or Tag

The same Lifecycle Policy does not need to apply to the entire bucket.

Suppose a bucket has the following structure:

```text
application-data/
├── logs/
├── backups/
├── user-images/
└── reports/
```

Different rules can be created:

```text
logs/
→ Transition to Glacier and delete after the retention period.

backups/
→ Transition to Deep Archive for long-term retention.

user-images/
→ Remain in Standard or Intelligent-Tiering.

reports/
→ Transition to Standard-IA when they become inactive.
```

Lifecycle Rules may also use object tags:

```text
DataType = ApplicationLog
Retention = 365Days
Environment = Production
```

Prefixes and tags allow different data groups in the same bucket to follow separate storage policies.

## Cost Optimization Considerations

### Transition and Retrieval Costs

Moving objects to another Storage Class may incur request charges. IA and Glacier classes may also charge for data retrieval.

Transitions should not be configured too frequently if the resulting savings do not offset request and retrieval costs.

### Object Size

Small objects may not produce the expected savings because of:

* Transition request costs.
* Minimum billable object sizes in some Storage Classes.
* Additional metadata for archival classes.
* Large object counts that increase request and management costs.

Under the current default S3 Lifecycle behavior, objects smaller than 128 KB are generally not transitioned unless an appropriate object-size transition configuration is specified.

### Minimum Storage Duration

Some Storage Classes have minimum storage-duration requirements:

| Storage Class | Minimum storage duration |
| --- | --- |
| S3 Standard-IA | 30 days |
| S3 One Zone-IA | 30 days |
| S3 Glacier Instant Retrieval | 90 days |
| S3 Glacier Flexible Retrieval | 90 days |
| S3 Glacier Deep Archive | 180 days |

If an object is deleted or transitioned earlier, charges for the remaining minimum storage duration may still apply.

### Restoration Time

Before transitioning data to a Glacier class, the team must determine the maximum acceptable restoration time.

Data that requires immediate access should not be moved to a class that may require several hours to restore.

### Previous Object Versions

When S3 Versioning is enabled, deleting or replacing an object does not necessarily remove its previous versions.

Noncurrent versions continue consuming storage and generating costs. Therefore, a Lifecycle Policy should consider:

* Current versions.
* Noncurrent versions.
* Delete markers.
* Incomplete multipart uploads.

### Retention Requirements

Before configuring expiration, the team should verify:

* Organizational retention policies.
* Audit requirements.
* Legal requirements.
* Recovery requirements.
* Backup policies.

Data should not be deleted automatically only to reduce costs if its required retention period has not been determined.

## Evaluating Costs Before Deployment

Before applying a Lifecycle Policy, the team should:

1. Classify data according to its access frequency.
2. Determine the total storage volume and object count.
3. Determine the average object size.
4. Identify the required retention period.
5. Define the acceptable restoration time.
6. Check the price for the selected AWS Region.
7. Include storage, request, transition, and retrieval charges.
8. Test the policy with a small prefix first.
9. Monitor costs after applying the policy.
10. Adjust Lifecycle Rules when access patterns change.

**AWS Pricing Calculator**, **AWS Cost Explorer**, and Amazon S3 storage metrics can support this evaluation.

## Application to the Live Auction System

In the Live Auction system, Amazon S3 may store:

* Built frontend files.
* Auction item images.
* Static website resources.
* Exported logs or reports.
* Backup files and long-term archives.

These data types should not necessarily use the same storage policy.

| Data type | Possible storage approach |
| --- | --- |
| Active frontend files | S3 Standard |
| Images for active auctions | S3 Standard or Intelligent-Tiering |
| Images from completed auctions | Standard-IA or Intelligent-Tiering |
| Older logs | Glacier Flexible Retrieval |
| Long-term backups | Glacier Deep Archive |
| Temporary files | Automatic removal through an Expiration Rule |

This is a cost-optimization approach for consideration. The actual Storage Class should be selected according to access requirements, data importance, and the pricing of the deployment Region.

## Lessons Learned

Through this research, the team learned that:

* Amazon S3 is more than a simple file upload and download service.
* Data should be classified by access frequency and retention period.
* Not every object should remain in S3 Standard throughout its lifecycle.
* Lifecycle Policies automate transitions and expiration without a cron job or Lambda function.
* Storage Classes have different storage costs, retrieval fees, and minimum-duration requirements.
* Small objects should be evaluated carefully before transitioning to IA or Glacier.
* Prefixes and tags allow separate policies for different data groups.
* Storage design can be an important area for cost optimization before changing application code.

## Published Article

The article was shared with the **AWS Study Group – First Cloud Journey** community.

**Title:** Optimizing Amazon S3 Storage Costs: Understanding Storage Classes and Lifecycle Policies

**Article link:** [View the article on AWS Study Group](https://www.facebook.com/groups/awsstudygroupfcj/posts/2239357320162561/)

## References

* [Amazon S3 Storage Classes overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html)
* [Amazon S3 Storage Classes](https://aws.amazon.com/s3/storage-classes/)
* [Setting the Storage Class of an S3 Object](https://docs.aws.amazon.com/AmazonS3/latest/userguide/sc-howtoset.html)
* [Managing the Object Lifecycle with S3 Lifecycle](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
* [Lifecycle Configuration Elements](https://docs.aws.amazon.com/AmazonS3/latest/userguide/intro-lifecycle-rules.html)
* [Setting a Lifecycle Configuration on a Bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/how-to-set-lifecycle-configuration-intro.html)
* [Official Amazon S3 Pricing](https://aws.amazon.com/s3/pricing/)

## Results

After completing the article, the team:

* Distinguished between the main Amazon S3 Storage Classes.
* Understood how to select a Storage Class according to access frequency.
* Understood the roles of Lifecycle Transition and Expiration.
* Learned how to apply Lifecycle Rules using prefixes and object tags.
* Identified cost categories beyond basic storage charges.
* Related S3 cost optimization to the data used by the Live Auction system.
* Shared the research findings with the AWS Study Group community.