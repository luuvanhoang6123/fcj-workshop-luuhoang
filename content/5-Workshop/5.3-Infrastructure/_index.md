---
title: "Infrastructure Provisioning with Terraform"
date: 2026-07-13
weight: 3
chapter: false
pre: "<b>5.3. </b>"
---

# Infrastructure Provisioning with Terraform

## Introduction

After preparing the development environment, the team provisions the infrastructure of the **Live Auction** system on **Amazon Web Services (AWS)** using **Terraform**.

Terraform follows the **Infrastructure as Code (IaC)** approach, allowing AWS resources to be defined as code instead of being configured manually through the AWS Management Console. This enables automated, consistent, and repeatable infrastructure deployment while simplifying future maintenance and expansion.

This section describes the Terraform workflow, including infrastructure directory structure, initialization, deployment planning, infrastructure provisioning, and deployment verification.

#### Contents

1. [Terraform Overview](5.3.1-Terraform-Overview/)
2. [Infrastructure Directory Structure](5.3.2-Infrastructure-Structure/)
3. [Terraform Initialization](5.3.3-Terraform-Init/)
4. [Terraform Planning](5.3.4-Terraform-Plan/)
5. [Infrastructure Provisioning](5.3.5-Terraform-Apply/)
