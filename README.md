# AWS VPC Creation with CloudFormation and CI/CD Pipeline

This repository contains an AWS CloudFormation template for creating a Virtual Private Cloud (VPC) with a subnet for an EC2 instance. It includes VPC Endpoints for SSM, SSM Messages, and EC2 Messages. A GitHub Actions CI/CD pipeline is provided for automated deployment, validation, and security checks. The pipeline also triggers a parent stack deployment via webhooks.

## Table of Contents

- [Introduction](#introduction)
- [CloudFormation Template](#cloudformation-template)
  - [Resources Created](#resources-created)
  - [Parameters](#parameters)
  - [Outputs](#outputs)
- [CI/CD Pipeline](#cicd-pipeline)
  - [Workflow Triggers](#workflow-triggers)
  - [Pipeline Overview](#pipeline-overview)
  - [Environment Variables and Secrets](#environment-variables-and-secrets)
- [Usage](#usage)
  - [Clone the Repository](#clone-the-repository)
  - [Set Up AWS Credentials](#set-up-aws-credentials)
  - [Configure the VPC](#configure-the-vpc)
  - [Branch Strategy](#branch-strategy)
  - [Manual Approval](#manual-approval)
  - [Triggering the Parent Stack](#triggering-the-parent-stack)
- [Notes](#notes)

## Introduction

This project automates the deployment of an AWS VPC with a subnet for hosting an EC2 instance using CloudFormation. It includes VPC Endpoints for AWS Systems Manager (SSM), SSM Messages, and EC2 Messages to enable communication with AWS services without traversing the internet.

The GitHub Actions CI/CD pipeline automates validation, security scanning, deployment processes, and triggers the deployment of a parent stack via webhooks.

The CI/CD pipeline is designed to:

- Validate and test CloudFormation templates on pull requests.
- Deploy infrastructure on pushes to specific branches.
- Perform security checks using CloudFormation Guard.
- Require manual approval before deployment.
- Trigger a parent stack deployment after successful deployment.

## CloudFormation Template

### Resources Created

The CloudFormation template (`template.yaml`) provisions the following resources:

1. **AWS VPC:**

   - **Name:** `VPC_{Environment}`
   - **CIDR Block:** Defined by `VpcCIDR` parameter (default is `192.168.0.0/16`)
   - **DNS Support and Hostnames:** Enabled
   - **Tags:** Includes `Name` and `Environment`

2. **AWS Subnet:**

   - **Name:** `EC2Subnet_{Environment}`
   - **CIDR Block:** Defined by `EC2SubnetCIDR` parameter (default is `192.168.1.0/24`)
   - **Availability Zone:** Defined by `EC2SubnetAZ` parameter (default is `us-west-2a`)
   - **Tags:** Includes `Name` and `Environment`

3. **Security Group for VPC Endpoints:**

   - **Name:** `{EpSGName}_{Environment}`
   - **Description:** Security Group for VPC Endpoints
   - **Ingress Rule:** Allows TCP port 443 from the VPC CIDR
   - **Associated with:** The created VPC

4. **VPC Endpoints:**

   - **SSM Endpoint (`VPCEndpointSSM`):** Interface endpoint for AWS Systems Manager
   - **SSM Messages Endpoint (`VPCEndpointSSMMessages`):** Interface endpoint for SSM messages
   - **EC2 Messages Endpoint (`VPCEndpointEC2Messages`):** Interface endpoint for EC2 messages
   - **Properties:**
     - **Service Name:** Corresponding AWS service in the region
     - **VPC ID:** The created VPC
     - **Subnet IDs:** The created subnet
     - **Security Group IDs:** The created security group
     - **Private DNS Enabled:** `true`

### Parameters

The template accepts the following parameters:

- **Environment:**

  - **Type:** String
  - **Description:** The environment name (provided via Github Workflows pipeline)
  - **Allowed Values:** `development`, `production`, `default`

- **VpcCIDR:**

  - **Type:** String
  - **Description:** The IP Address range of the VPC
  - **Default:** `192.168.0.0/16`

- **EC2SubnetCIDR:**

  - **Type:** String
  - **Description:** The IP Address range of the subnet that hosts the EC2 instance
  - **Default:** `192.168.1.0/24`

- **EC2SubnetAZ:**

  - **Type:** String
  - **Description:** Availability Zone of the EC2 Subnet
  - **Default:** `us-west-2a`

- **EpSGName:**

  - **Type:** String
  - **Description:** Name of the security group attached to the EC2 instance
  - **Default:** `S2SEpSG`

### Outputs

- **VPCId:**
  - **Description:** The ID of the created VPC
  - **Value:** Reference to `MyVPC`

- **SubnetId:**
  - **Description:** The ID of the created Subnet
  - **Value:** Reference to `MySubnet`

- **VPCCIDR:**
  - **Description:** The CIDR range of the VPC
  - **Value:** Reference to `VpcCIDR`

## CI/CD Pipeline

The CI/CD pipeline is defined in the GitHub Actions workflow file `.github/workflows/aws-cloudformation.yml`. It automates the deployment process, ensures code quality and security, and triggers the deployment of a parent stack.

### Workflow Triggers

The pipeline is triggered on:

- **Pull Requests** to the following branches:
  - `development`
  - `production`
  - `testing`
- **Pushes** to the following branches:
  - `development`
  - `production`

### Pipeline Overview

The pipeline consists of two primary jobs:

1. **Validate and Test (`validate-and-test`):**

   - **Checkout Code:** Retrieves the repository code.
   - **Configure AWS Credentials:** Assumes an IAM role using OpenID Connect (OIDC) with the provided credentials.
   - **Validate CloudFormation Template:** Validates the syntax of the CloudFormation template.
   - **Run CloudFormation Guard:** Performs security checks using CloudFormation Guard against CIS benchmarks.
   - **Skip Apply in Pull Requests:** Ensures that deployment does not occur on pull requests.

2. **Upload and Trigger (`upload-and-trigger`):**

   - **Depends On:** The `validate-and-test` job must succeed.
   - **Runs On:** Not triggered on pull requests.
   - **Checkout Code:** Retrieves the repository code.
   - **Configure AWS Credentials:** Assumes an IAM role using OIDC with the provided credentials.
   - **Set Environment Variable:** Determines the environment and S3 bucket based on the branch.
   - **Manual Approval:** Requires manual approval via GitHub Issues before proceeding.
   - **Upload Template to S3:** Uploads the CloudFormation template to the specified S3 bucket.
   - **Trigger Parent Stack Deployment:** Makes an API call to trigger the parent stack's workflow via webhooks.

### Environment Variables and Secrets

The pipeline uses the following secrets and environment variables:

- **Secrets (Stored in GitHub Secrets):**
  - `AWS_ROLE_TO_ASSUME`: ARN of the IAM role to assume for deployment.
  - `TOKEN_PARENT_TRIGGER`: Personal Access Token (PAT) with repository permissions to trigger workflows in the parent repository.
  - `github_TOKEN`: Automatically provided by GitHub for authentication in workflows.

- **Environment Variables:**
  - `ENVIRONMENT`: Set based on the branch (`development`, `production`, or `default`).
  - `S3_BUCKET`: The S3 bucket URL, determined by the environment.

## Usage

### Clone the Repository

```bash
git clone https://github.com/CommittingLearning/Site2Site-AWS-VPC.git
```

### Set Up AWS Credentials

Ensure that the following secrets are added to your GitHub repository under **Settings > Secrets and variables > Actions**:

- `AWS_ROLE_TO_ASSUME`
- `TOKEN_PARENT_TRIGGER`

- **`AWS_ROLE_TO_ASSUME`**: The ARN of an IAM role that the GitHub Actions workflow can assume, with the necessary permissions to deploy CloudFormation stacks and interact with S3.

- **`TOKEN_PARENT_TRIGGER`**: A GitHub Personal Access Token (PAT) with permissions to trigger workflows in the parent repository.

### Configure the VPC

- **VPC CIDR**: Adjust the `VpcCIDR` parameter in the `template.yaml` file if you need a different IP range for the VPC.
- **Subnet CIDR**: Adjust the `EC2SubnetCIDR` parameter for the subnet's IP range.
- **Availability Zone**: Change the `EC2SubnetAZ` parameter if deploying in a different availability zone.
- **Environment Parameter**: The environment (`development`, `production`, or `default`) is determined based on the branch name.

### Branch Strategy

- **Development Environment:** Use the `development` branch to deploy to the development environment.
- **Production Environment:** Use the `production` branch to deploy to the production environment.
- **Default Environment:** Any other branches will use the `default` environment settings.

### Manual Approval

The pipeline requires manual approval before applying changes:

- A GitHub issue will be created prompting for approval.
- Approvers need to approve the issue to proceed with deployment.

### Triggering the Parent Stack

After successful deployment, the pipeline triggers the parent stack deployment via a webhook:

- The last step in the `upload-and-trigger` job makes an API call to the parent repository (`Site2Site-AWS-ParentStack`) to trigger its workflow.
- This ensures that the parent stack is deployed with the latest updates.

## Notes

- **Security Checks:**
  - The pipeline includes security checks using CloudFormation Guard to ensure compliance with CIS benchmarks.

- **Nested CloudFormation Templates:**
  - The VPC template is intended to be uploaded to an S3 bucket and used as a child template by a parent stack.

- **Testing:**
  - Pull requests to `development`, `production`, or `testing` branches will trigger the validation and testing steps without applying changes.

- **IAM Role Configuration:**
  - Ensure that the IAM role specified in `AWS_ROLE_TO_ASSUME` has permissions to deploy CloudFormation stacks, interact with S3, and invoke API calls to other repositories.

- **S3 Bucket Configuration:**
  - The S3 bucket URL is set based on the environment. Ensure that the bucket exists and is accessible.

- **Webhook Trigger:**
  - The `TOKEN_PARENT_TRIGGER` must have appropriate permissions (usually `repo` scope) to trigger workflows in the parent repository.
  - Adjust the API endpoint and payload in the `Trigger Parent Stack Deployment` step if your repository names or structures differ.

---

**Disclaimer:** This repository is accessible in a read only format, and therefore, only the admin has the privileges to perform a push on the branches.