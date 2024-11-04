# AWS Parent Stack Deployment with CloudFormation and CI/CD Pipeline

This repository contains an AWS CloudFormation parent stack template that orchestrates the deployment of child stacks for VPC, EC2, and VPN Gateway. It includes a GitHub Actions CI/CD pipeline that automates the deployment process, handles manual approvals, and ensures that the infrastructure is provisioned and deprovisioned in a controlled manner.

## Table of Contents

- [Introduction](#introduction)
- [CloudFormation Template](#cloudformation-template)
  - [Resources Created](#resources-created)
  - [Parameters](#parameters)
- [CI/CD Pipeline](#cicd-pipeline)
  - [Workflow Triggers](#workflow-triggers)
  - [Pipeline Overview](#pipeline-overview)
  - [Environment Variables and Secrets](#environment-variables-and-secrets)
- [Usage](#usage)
  - [Clone the Repository](#clone-the-repository)
  - [Set Up AWS Credentials](#set-up-aws-credentials)
  - [Configure the Parent Stack](#configure-the-parent-stack)
  - [Branch Strategy](#branch-strategy)
  - [Manual Approval](#manual-approval)
- [Notes](#notes)

## Introduction

This project automates the deployment of an AWS Parent Stack that orchestrates the creation of:

- **VPC Stack:** Deploys a Virtual Private Cloud and associated resources.
- **EC2 Stack:** Launches an EC2 instance within the VPC.
- **VPN Gateway Stack:** Sets up a VPN Gateway to establish a site-to-site VPN connection.

The GitHub Actions CI/CD pipeline automates the deployment, manual approval, and teardown processes. It is triggered via webhooks from child repositories after the child templates are uploaded to S3.

The CI/CD pipeline is designed to:

- Validate the CloudFormation template.
- Deploy the parent stack upon manual approval.
- Wait for a specified period before automatically deleting the stack to prevent unnecessary costs.

## CloudFormation Template

### Resources Created

The `template.yaml` CloudFormation template defines the following resources:

1. **VPC Stack (`VPCStack`):**

   - Type: `AWS::CloudFormation::Stack`
   - Deploys the VPC child stack using the provided `VPCStackURL`.
   - Parameters:
     - `Environment`

2. **EC2 Stack (`EC2Stack`):**

   - Type: `AWS::CloudFormation::Stack`
   - Deploys the EC2 child stack using the provided `EC2StackURL`.
   - Parameters:
     - `VPCId`: Retrieved from `VPCStack` outputs.
     - `SubnetId`: Retrieved from `VPCStack` outputs.
     - `VPCCIDR`: Retrieved from `VPCStack` outputs.
     - `Environment`

3. **VPN Gateway Stack (`VGWStack`):**

   - Type: `AWS::CloudFormation::Stack`
   - Deploys the VPN Gateway child stack using the provided `VGWStackURL`.
   - Parameters:
     - `VPCId`: Retrieved from `VPCStack` outputs.
     - `SubnetId`: Retrieved from `VPCStack` outputs.
     - `PreSharedKey`: Provided via the `PreSharedKey` parameter.
     - `Environment`

### Parameters

The template accepts the following parameters:

- **Environment:**

  - **Type:** String
  - **Description:** The environment name (e.g., development, production)
  - **Allowed Values:** `development`, `production`, `default`

- **VPCStackURL:**

  - **Type:** String
  - **Description:** The S3 URL of the VPC child template.

- **EC2StackURL:**

  - **Type:** String
  - **Description:** The S3 URL of the EC2 child template.

- **VGWStackURL:**

  - **Type:** String
  - **Description:** The S3 URL of the VPN Gateway child template.

- **PreSharedKey:**

  - **Type:** String
  - **Description:** The pre-shared key used to establish the VPN connection (provided via Secrets).
  - **NoEcho:** `true` (Sensitive information)

## CI/CD Pipeline

The CI/CD pipeline is defined in the GitHub Actions workflow file `.github/workflows/automatic-webhook-push.yml`. It automates the deployment process, handles manual approvals, and ensures proper teardown.

### Workflow Triggers

The pipeline is triggered on:

- **Repository Dispatch Events:** Specifically, when a `trigger-parent-stack` event is received via webhook from child repositories.

### Pipeline Overview

The pipeline consists of a single job:

1. **Deploy:**

   - **Checkout Code:** Retrieves the repository code.
   - **Configure AWS Credentials:** Assumes an IAM role using OpenID Connect (OIDC) with the provided credentials.
   - **Set Environment Variables:** Determines the environment and child template URLs based on the branch from the child repository.
   - **Validate CloudFormation Template:** Validates the syntax of the parent CloudFormation template.
   - **Manual Approval:** Requires manual approval via GitHub Issues before proceeding.
   - **Deploy CloudFormation Stack:** Deploys the parent stack using AWS CLI.
   - **Wait Period:** Waits for 20 minutes (adjustable) to allow for demonstration or testing.
   - **Delete CloudFormation Stack:** Automatically deletes the stack to prevent unnecessary costs.
   - **Confirm Deletion:** Waits until the stack deletion is complete.

### Environment Variables and Secrets

The pipeline uses the following secrets and environment variables:

- **Secrets (Stored in GitHub Secrets):**

  - `AWS_ROLE_TO_ASSUME`: ARN of the IAM role to assume for deployment.
  - `S2SSharedKey`: The pre-shared key for the VPN connection.
  - `github_TOKEN`: Automatically provided by GitHub for authentication in workflows.

- **Environment Variables:**

  - `ENVIRONMENT`: Set based on the branch from the child repository (`development`, `production`, or `default`).
  - `VPCSTACKURL`: S3 URL of the VPC child template.
  - `EC2STACKURL`: S3 URL of the EC2 child template.
  - `VGWSTACKURL`: S3 URL of the VPN Gateway child template.

## Usage

### Clone the Repository

```bash
git clone https://github.com/CommittingLearning/Site2Site-AWS-ParentStack.git
```

### Set Up AWS Credentials

Ensure that the following secrets are added to your GitHub repository under **Settings > Secrets and variables > Actions**:

- `AWS_ROLE_TO_ASSUME`

- `S2SSharedKey`

- **`AWS_ROLE_TO_ASSUME`**: The ARN of an IAM role that the GitHub Actions workflow can assume, with the necessary permissions to deploy CloudFormation stacks.

- **`S2SSharedKey`**: The pre-shared key used for establishing the VPN connection.

### Configure the Parent Stack

- **Child Template URLs**: Ensure that the child templates (VPC, EC2, VGW) are uploaded to the correct S3 buckets and that the URLs are correctly set in the pipeline based on the environment.

- **Environment Parameter**: The environment (`development`, `production`, or `default`) is determined based on the branch name from the child repositories.

### Branch Strategy

- **Development Environment**: Use the `development` branch in child repositories to deploy to the development environment.

- **Production Environment**: Use the `production` branch in child repositories to deploy to the production environment.

- **Default Environment**: Any other branches will use the `default` environment settings.

### Manual Approval

The pipeline requires manual approval before deploying the parent stack:

- A GitHub issue will be created prompting for approval.

- Approvers need to approve the issue to proceed with deployment.

## Notes

- **Pipeline Interaction**:

  - The parent stack deployment is triggered via webhooks from child repositories after they upload their templates to S3.

  - The `repository_dispatch` event carries the branch information, which the parent pipeline uses to determine the environment and child template URLs.

- **Automatic Teardown**:

  - After deployment, the pipeline waits for 20 minutes (default) before automatically deleting the stack to prevent unnecessary costs.

- **IAM Role Configuration**:

  - Ensure that the IAM role specified in `AWS_ROLE_TO_ASSUME` has permissions to deploy CloudFormation stacks and delete them.

- **Testing**:

  - The pipeline is designed to be triggered automatically and does not run on pull requests or manual pushes.

- **Security Considerations**:

  - Sensitive parameters like `PreSharedKey` are handled securely using GitHub Secrets.

- **Adjustable Wait Time**:

  - The wait period before stack deletion can be adjusted by modifying the `sleep` command in the pipeline.

---

**Disclaimer:** This repository is accessible in a read only format, and therefore, only the admin has the privileges to perform a push on the branches.