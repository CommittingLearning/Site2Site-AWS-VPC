# Site2Site-AWS-VPC

This repository manages the infrastructure for creating and managing an **AWS VPC** for hosting an EC2 instance. The VPC is dynamically named based on the environment (development, production, or testing) and is deployed using **AWS CloudFormation**. The repository also implements a full **CI/CD pipeline** for validating and deploying infrastructure changes.

## Repository Structure

```plaintext
Site2Site-AWS-VPC/
│
├── .github/
│   └── workflows/
│       └── aws-cloudformation-pipeline.yml   # GitHub Actions CI/CD pipeline script
│
├── template.yaml                             # CloudFormation template to create VPC and Subnet
├── README.md                                 # Project overview and documentation (this file)
```

## Repository Map

- **.github/workflows/aws-cloudformation-pipeline.yml**:
  This directory contains the **CI/CD pipeline** script, which is responsible for validating, testing, and deploying infrastructure based on branch pushes and pull requests.

- **template.yaml**:
  This is the **CloudFormation template** used to create the VPC and its associated resources, such as subnets. The VPC is named based on the environment (e.g., development, production) and its IP address range is defined in the template.

## Branches

The repository uses three branches to manage different stages of the development and deployment lifecycle:

- **testing**:
  - This branch is used for making and testing changes to the infrastructure configuration before merging to development or production.
  - Any infrastructure changes pushed to this branch will trigger the pipeline for validation and testing but will not be applied.

- **development**:
  - Changes that pass testing are merged into this branch.
  - The pipeline will deploy the infrastructure in a **development environment**, using resources named accordingly (e.g., `VPC_development`).

- **production**:
  - The final branch where fully tested and approved changes are merged.
  - The pipeline will deploy the infrastructure in the **production environment** (e.g., `VPC_production`).

## CI/CD Pipeline

The **CI/CD pipeline** is managed by **GitHub Actions** and is triggered on:

- **Pull requests**: For testing and validating infrastructure changes.
- **Pushes**: For deploying the infrastructure when changes are pushed to the `development` or `production` branches.

### Pipeline Workflow

- **Validate and Test**:
  - The pipeline starts by validating the **CloudFormation template** using `aws cloudformation validate-template`.
  - It then installs and runs **CloudFormation Guard** to validate the template against security and compliance rules.
  - **Unit and Integration Tests**: Simple AWS CLI checks are run to ensure the integrity of the CloudFormation stack.

- **Manual Approval**:
  - After validation, the pipeline pauses for **manual approval**. This step ensures that infrastructure changes are reviewed before they are deployed.

- **Deploy CloudFormation Stack**:
  - Once approved, the pipeline deploys the **CloudFormation stack** using the `aws cloudformation deploy` command, passing the appropriate environment variables (e.g., `Environment=development`).

- **Clean-up**:
  - To prevent unnecessary costs, the pipeline waits for 15 minutes and then automatically deletes the CloudFormation stack using `aws cloudformation delete-stack`.

## CloudFormation Template (`template.yaml`)

The **CloudFormation template** is responsible for creating the **AWS VPC** and its associated resources (subnet). It dynamically names resources based on the environment (development, production, or testing).

### Template Overview

- **Parameters**:
  - The template accepts parameters for `Environment` (e.g., `development`, `production`), the **VPC CIDR block**, and the **Subnet CIDR block**. This allows the flexibility to use different environments and IP ranges.

- **Resources**:
  - **VPC**: The VPC is created with a CIDR block of `192.168.0.0/16` and is dynamically named based on the environment.
  - **Subnet**: A subnet with CIDR block `192.168.1.0/24` is created inside the VPC to host an EC2 instance.

- **Outputs**:
  - The template outputs the **VPC ID** and **Subnet ID** for easy reference after stack creation.

## How the Workflow and Template Work Together

- When a developer pushes to one of the branches, the **CI/CD pipeline** triggers and first validates the CloudFormation template.
- The infrastructure is deployed in either the development or production environment based on the branch name.
- Resources (VPC, subnet) are dynamically named based on the environment, ensuring isolation and easy identification of resources in different stages.
