# Deployment Guide - Amazon EKS using Jenkins & Terraform

## Overview

This project automates the provisioning of an Amazon Elastic Kubernetes Service (EKS) cluster using **Jenkins** and **Terraform**.

The Jenkins pipeline allows operators to execute Terraform actions (`plan`, `apply`, and `destroy`) through build parameters, enabling repeatable and controlled infrastructure deployments.

Infrastructure state is stored remotely in an **Amazon S3 backend**, while **Amazon DynamoDB** is used for state locking to prevent concurrent Terraform executions.

---

# Architecture

```text
                    +----------------+
                    |    Developer   |
                    +-------+--------+
                            |
                            | Trigger Build
                            |
                    +-------v--------+
                    |    Jenkins     |
                    +-------+--------+
                            |
                 Terraform Pipeline
                            |
        +-------------------+-------------------+
        |                   |                   |
     terraform init   terraform validate   terraform plan/apply/destroy
                            |
                            |
                    +-------v--------+
                    | Terraform IaC  |
                    +-------+--------+
                            |
                Remote Terraform Backend
                +-----------------------+
                |  S3 State File        |
                |  DynamoDB Lock Table  |
                +-----------------------+
                            |
                            |
                    +-------v--------+
                    | AWS Resources  |
                    +----------------+
                            |
        +---------------------------------------------+
        | VPC                                         |
        | Public & Private Subnets                    |
        | Internet Gateway                            |
        | NAT Gateway                                 |
        | Route Tables                                |
        | Security Groups                             |
        | IAM Roles                                   |
        | Amazon EKS Control Plane                    |
        | Managed Node Groups                         |
        +---------------------------------------------+
```

---

# Repository Structure

```text
EKS-TF-Jenkins/
│
├── Jenkinsfile
├── eks/
│   ├── backend.tf
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── dev.tfvars
│   └── prod.tfvars
│
└── module/
    ├── vpc.tf
    ├── subnet.tf
    ├── route-table.tf
    ├── internet-gateway.tf
    ├── nat-gateway.tf
    ├── security-group.tf
    ├── iam.tf
    ├── eks.tf
    ├── nodegroup.tf
    └── outputs.tf
```

---

# Prerequisites

Before running the pipeline, ensure the following components are installed and configured.

## AWS

* AWS Account
* IAM User with sufficient permissions to provision EKS resources
* AWS CLI configured

```bash
aws configure
```

Verify credentials:

```bash
aws sts get-caller-identity
```

---

## Jenkins

Install Jenkins and configure the following plugins:

* Pipeline
* Git
* Terraform
* AWS Steps Plugin
* Credentials Binding Plugin

---

## Terraform

Verify installation.

```bash
terraform version
```

---

# Remote Terraform Backend

Terraform state is stored remotely using Amazon S3.

```hcl
backend "s3" {
    bucket         = "dev-asadjvd-tf-bucket"
    key            = "eks/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "Lock-Files"
    encrypt        = true
}
```

### Benefits

* Shared remote state
* State locking using DynamoDB
* Prevents simultaneous infrastructure modifications
* State persistence across Jenkins builds

---

# Jenkins Parameters

The pipeline accepts two parameters.

| Parameter        | Description                                              |
| ---------------- | -------------------------------------------------------- |
| Environment      | Selects the Terraform variable file (e.g., `dev.tfvars`) |
| Terraform_Action | Selects one of `plan`, `apply`, or `destroy`             |

---

# Pipeline Workflow

## Stage 1 - Preparing

Verifies that the Jenkins agent is available.

```bash
echo Preparing
```

---

## Stage 2 - Git Pulling

Downloads the latest infrastructure code from GitHub.

Repository:

```text
https://github.com/asadjvd/jenkins-projects.git
```

---

## Stage 3 - Terraform Init

Initializes Terraform and downloads all required providers.

```bash
terraform -chdir=eks/ init
```

This step also initializes the remote S3 backend.

---

## Stage 4 - Terraform Validate

Validates the Terraform configuration.

```bash
terraform -chdir=eks/ validate
```

This ensures the configuration is syntactically correct before deployment.

---

## Stage 5 - Terraform Action

The pipeline executes one of three actions depending on the selected Jenkins parameter.

### Plan

Generates an execution plan.

```bash
terraform -chdir=eks/ plan \
-var-file=dev.tfvars
```

---

### Apply

Creates the AWS infrastructure.

```bash
terraform -chdir=eks/ apply \
-var-file=dev.tfvars \
-auto-approve
```

---

### Destroy

Deletes all AWS resources created by Terraform.

```bash
terraform -chdir=eks/ destroy \
-var-file=dev.tfvars \
-auto-approve
```

---

# Infrastructure Provisioned

The Terraform module provisions the following AWS resources.

* Amazon VPC
* Public Subnets
* Private Subnets
* Internet Gateway
* NAT Gateway
* Elastic IP
* Route Tables
* Security Groups
* IAM Roles
* Amazon EKS Cluster
* Amazon EKS Managed Node Groups
* EKS Add-ons

---

# Verify the Deployment

After a successful deployment, configure kubectl.

```bash
aws eks update-kubeconfig \
--region us-east-1 \
--name <cluster-name>
```

Verify the worker nodes.

```bash
kubectl get nodes
```

Example output:

```text
NAME                             STATUS   ROLES    AGE
ip-10-16-131-21.ec2.internal     Ready    <none>   5m
ip-10-16-150-40.ec2.internal     Ready    <none>   5m
```

---

# Cleanup

To remove all infrastructure, run the Jenkins pipeline with:

```text
Terraform_Action = destroy
```

or execute manually:

```bash
terraform destroy \
-var-file=dev.tfvars \
-auto-approve
```

Before destroying the infrastructure, ensure that:

* Argo CD applications have been deleted.
* Kubernetes Ingress resources have been removed.
* AWS Application Load Balancers have been deleted.
* Route 53 records are no longer required.

This prevents Terraform dependency issues during resource deletion.

---

# Troubleshooting

## Remote Backend Bucket Not Found

```
S3 bucket does not exist
```

Create the S3 bucket before running `terraform init`.

---

## State Lock Error

```
ConditionalCheckFailedException
```

Verify the DynamoDB lock table exists and no stale lock remains.

---

## Unsupported Terraform Version

```
Unsupported Terraform Core version
```

Install the Terraform version specified in `backend.tf`.

---

## VPC Limit Exceeded

```
VpcLimitExceeded
```

Delete unused VPCs or request a VPC service quota increase.

---

## Cluster Not Found

```
No cluster found
```

Verify that the Terraform apply stage completed successfully and update the kubeconfig using the correct cluster name.

---

# Technologies Used

* Jenkins
* Terraform
* Amazon EKS
* Amazon EC2
* Amazon VPC
* Amazon IAM
* Amazon S3
* Amazon DynamoDB
* AWS CLI
* kubectl

---

# Outcome

This pipeline provides an automated and repeatable method for provisioning Amazon EKS infrastructure using Infrastructure as Code. By combining Jenkins, Terraform, Amazon S3, and DynamoDB, the deployment process becomes version-controlled, parameterized, and suitable for production-style DevOps workflows.
