---
title: "Initialize the Terraform Environment"
date: 2026-07-13
weight: 3
chapter: false
pre: "<b>5.3.3. </b>"
---

## Introduction

After completing the Infrastructure directory structure, the team initializes the Terraform working environment for each module using the `terraform init` command.

This is the first command that must be executed before using commands such as `terraform validate`, `terraform plan`, or `terraform apply`. It prepares the working directory, downloads the required Providers, and configures the Backend used to store the infrastructure state.

In the Live Auction system, Terraform modules are deployed separately according to their functional layers. Therefore, `terraform init` must be executed inside each corresponding module directory before planning and deploying its resources.

---

## Check the Terraform Installation

Open PowerShell or Terminal in the project directory and check the installed Terraform version:

```powershell
terraform version
```

If Terraform has been installed successfully, the Terminal displays the Terraform version and the current operating platform.

<!--
SCREENSHOT INSTRUCTIONS:
1. Run: terraform version
2. Capture the Terminal showing both the command and its result.
3. Save the image as:
static/images/5-Workshop/5.3-Infrastructure/terraform-version.png
-->

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/terraform-version.png" alt="Check the Terraform version" width="75%">
    <figcaption style="text-align: center;">
        <b>Figure 5.3.12.</b> Checking the Terraform version in the deployment environment.
    </figcaption>
</figure>

{{% notice info %}}
The actual Terraform version may vary depending on when it was installed. The installed version must satisfy the requirements declared in the `versions.tf` file.
{{% /notice %}}

---

## Verify the AWS Connection

Before initializing Terraform, verify the AWS credentials and connection:

```powershell
aws sts get-caller-identity
```

If AWS CLI has been configured correctly, the command returns:

* User ID.
* AWS Account ID.
* ARN of the IAM User or IAM Role currently in use.

Example:

```json
{
    "UserId": "EXAMPLEUSERID",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/example-user"
}
```
---

## Navigate to the Module Directory

From the project root directory, navigate to the Infrastructure directory:

```powershell
cd infra
```

For the initial setup, the team starts with the `03-identity` module:

```powershell
cd 03-identity
```

Check the configuration files in the module:

```powershell
Get-ChildItem
```

The module directory contains the following Terraform configuration files:

```text
backend.tf
main.tf
outputs.tf
providers.tf
variables.tf
versions.tf
```

---

## Initialize the Terraform Backend

The project uses a Terraform Backend to store the infrastructure state remotely. The Backend configuration is declared in the `backend.tf` file of each module.

Example:

```hcl
terraform {
  backend "s3" {
    bucket = "TERRAFORM_STATE_BUCKET_NAME"
    key    = "identity/terraform.tfstate"
    region = "ap-southeast-1"
  }
}
```

The configuration properties have the following purposes:

| Property | Description                                              |
| -------- | -------------------------------------------------------- |
| `bucket` | The S3 Bucket used to store the Terraform State.         |
| `key`    | The path of the State file for the corresponding module. |
| `region` | The AWS Region containing the S3 Bucket.                 |

Using a Remote Backend prevents the Terraform State from depending on one team member’s computer. Team members can share the same infrastructure state during development and deployment.

{{% notice note %}}
Replace `TERRAFORM_STATE_BUCKET_NAME` with the actual S3 Bucket name used in the team's Terraform configuration.
{{% /notice %}}

If the Remote Backend resources have not been created, navigate to the `00-bootstrap` directory:

```powershell
cd ..\00-bootstrap
```

Run the bootstrap script:

```powershell
.\bootstrap-remote-state.ps1
```

This script prepares the resources required to store and manage the Terraform State remotely.

After the bootstrap process is complete, return to the Identity module:

```powershell
cd ..\03-identity
```

---

## Run Terraform Init

Inside the `03-identity` directory, run:

```powershell
terraform init
```

During this process, Terraform performs the following operations:

1. Reads the Terraform configuration files in the current directory.
2. Initializes the Terraform Backend.
3. Connects to the remote Terraform State storage.
4. Downloads the AWS Provider with the declared version.
5. Initializes dependent modules, if any.
6. Creates the `.terraform/` directory.
7. Creates or updates the `.terraform.lock.hcl` file.

When the initialization process is successful, the Terminal displays:

```text
Terraform has been successfully initialized!
```

<!--
SCREENSHOT INSTRUCTIONS:
1. Open the Terminal in infra/03-identity.
2. Run: terraform init
3. Capture the result containing:
   Terraform has been successfully initialized!
4. Save the image as:
static/images/5-Workshop/5.3-Infrastructure/terraform-init-success.png
-->

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/terraform-init-success.png" alt="Successful Terraform initialization" width="85%">
    <figcaption style="text-align: center;">
        <b>Figure 5.3.13.</b> Successfully initializing the Terraform module with <code>terraform init</code>.
    </figcaption>
</figure>

---

## Reinitialize the Backend

When the contents of `backend.tf` are changed, Terraform may require the Backend to be initialized again.

Run the following command:

```powershell
terraform init -reconfigure
```

The `-reconfigure` option instructs Terraform to ignore the previously stored Backend configuration and load the current configuration again.

To migrate the Terraform State from an existing Backend to a new Backend, use:

```powershell
terraform init -migrate-state
```

{{% notice warning %}}
Only use `-migrate-state` when the Terraform State must be moved to another Backend. Review and back up the State before performing the migration to prevent infrastructure state loss.
{{% /notice %}}

---

## Verify the Initialization Result

After `terraform init` is complete, check the module directory:

```powershell
Get-ChildItem -Force
```

Terraform creates the following items:

```text
.terraform/
.terraform.lock.hcl
```

Their purposes are:

* `.terraform/` stores Providers, modules, and Backend information used by Terraform.
* `.terraform.lock.hcl` locks the selected Provider versions.
* The original `.tf` configuration files remain unchanged.
* The module is ready for the next validation and deployment steps.

<!--
SCREENSHOT INSTRUCTIONS:
1. Open the infra/03-identity directory in VS Code Explorer.
2. Capture the directory structure showing:
   .terraform/
   .terraform.lock.hcl
   backend.tf
   main.tf
   outputs.tf
   providers.tf
   variables.tf
   versions.tf
3. Save the image as:
static/images/5-Workshop/5.3-Infrastructure/terraform-init-files.png
-->

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/terraform-init-files.png" alt="Files created after Terraform Init" width="65%">
    <figcaption style="text-align: center;">
        <b>Figure 5.3.14.</b> The <code>.terraform</code> directory and <code>.terraform.lock.hcl</code> file after initialization.
    </figcaption>
</figure>

---

## Initialize the Remaining Modules

After initializing the Identity module, repeat the process for the remaining modules.

Initialize the Data module:

```powershell
cd ..\04-data
terraform init
```

Initialize the Messaging module:

```powershell
cd ..\05-messaging
terraform init
```

Initialize the Compute module:

```powershell
cd ..\06-compute
terraform init
```

Initialize the API module:

```powershell
cd ..\07-api
terraform init
```

Initialize the Edge module:

```powershell
cd ..\09-edge
terraform init
```

Each module has its own Backend configuration and Terraform State. Separating the State by module allows the team to control the scope of infrastructure changes and reduces the impact between infrastructure layers.

The team initializes the modules in the following order:

1. Identity.
2. Data.
3. Messaging.
4. Compute.
5. API.
6. Edge.

The `terraform init` command only prepares the working environment and does not create AWS resources. Resources are created only after running `terraform apply`.

---

## Common Errors

### Terraform is not recognized

Error message:

```text
terraform: The term 'terraform' is not recognized
```

This error may occur when Terraform has not been installed or its directory has not been added to the `PATH` environment variable.

Verify the installation:

```powershell
terraform version
```

### AWS credentials cannot be found

Error message:

```text
No valid credential sources found
```

Check the AWS CLI configuration:

```powershell
aws configure list
```

Verify the AWS connection:

```powershell
aws sts get-caller-identity
```

### The S3 Backend cannot be found

A possible error message is:

```text
Failed to get existing workspaces
```

Check the following:

* Whether the S3 Bucket used for Terraform State has been created.
* Whether the Bucket name in `backend.tf` is correct.
* Whether the configured AWS Region is correct.
* Whether the IAM User or IAM Role has permission to access the S3 Bucket.

### Insufficient permissions

If an `AccessDenied` error appears, review the IAM Policy attached to the current identity. The account must have permission to access the Terraform Backend and manage the resources declared in the module.

---

## Result

After completing the initialization process:

* Terraform successfully recognized the configuration files in each module.
* The AWS Provider was downloaded with the required version.
* The Terraform Backend was connected successfully.
* The `.terraform/` directory and `.terraform.lock.hcl` file were created.
* The modules were ready for configuration validation.
* The team could proceed to prepare the deployment plan using `terraform plan`.