# Deploying the Jenkins Server on AWS

This guide explains how to deploy a Jenkins server on AWS using Terraform and configure it for provisioning an Amazon EKS cluster.

---

# Prerequisites

Before you begin, ensure you have:

- An AWS account
- Ubuntu/Linux machine
- Terraform installed
- AWS CLI installed
- Git installed
- SSH key pair created in AWS
- S3 Bucket for Terraform backend
- DynamoDB table for Terraform state locking

---

# Step 1: Create an IAM User

> **Note**
>
> For learning purposes this guide uses the **AdministratorAccess** policy.
> In production environments, always follow the **principle of least privilege**.

## 1. Open IAM

Navigate to:

```
AWS Console → IAM → Users
```

<img src=

Click **Create User**.

---

## 2. Create the User

Provide a username.

Example:

```
jenkins-user
```

Click **Next**.

---

## 3. Assign Permissions

Select

```
Attach policies directly
```

Search for

```
AdministratorAccess
```

Select the policy.

Click **Next**.

---

## 4. Create User

Review the configuration and click

```
Create User
```

---

## 5. Generate Access Keys

Open the newly created user.

Navigate to

```
Security Credentials
```

Choose

```
Create Access Key
```

Select

```
Command Line Interface (CLI)
```

Accept the confirmation checkbox.

Click **Next**.

Optionally provide a description.

Click

```
Create Access Key
```

Save the following credentials safely:

- Access Key ID
- Secret Access Key

You can also download the CSV file for future reference.

---

# Step 2: Install Terraform and AWS CLI

## Install Terraform

```bash
wget -O- https://apt.releases.hashicorp.com/gpg \
| sudo gpg --dearmor \
-o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
| sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update

sudo apt install terraform -y
```

Verify installation:

```bash
terraform version
```

---

## Install AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" \
-o "awscliv2.zip"

sudo apt install unzip -y

unzip awscliv2.zip

sudo ./aws/install
```

Verify installation:

```bash
aws --version
```

---

# Step 3: Configure AWS Credentials

Run:

```bash
aws configure
```

Provide:

```
AWS Access Key ID
AWS Secret Access Key
Default Region (e.g. us-east-1)
Output format (json)
```

Example:

```text
AWS Access Key ID: AKIA...
AWS Secret Access Key: ****************
Default region name: us-east-1
Default output format: json
```

---

# Step 4: Clone the Repository

Clone the project repository.

```bash
git clone https://github.com/asadjvd/jenkins-projects.git

cd jenkins-projects
```

---

# Step 5: Configure Terraform Backend

Before deploying, update the Terraform backend.

Example:

```hcl
backend "s3" {
  bucket         = "dev-asadjvd-tf-bucket"
  key            = "jenkins/terraform.tfstate"
  region         = "us-east-1"
  dynamodb_table = "Lock-Files"
  encrypt        = true
}
```

Ensure that:

- S3 bucket already exists
- DynamoDB table already exists

Terraform does **not** create backend resources automatically.

---

# Step 6: Update the PEM Key

Modify the Terraform variables to use your EC2 Key Pair.

Example:

```hcl
key_name = "my-keypair"
```

This key will be used to SSH into the Jenkins server after deployment.

---

# Step 7: Initialize Terraform

Initialize the backend.

```bash
terraform init
```

---

# Step 8: Validate the Configuration

```bash
terraform validate
```

---

# Step 9: Review the Infrastructure Plan

```bash
terraform plan \
-var-file=variables.tfvars
```

Terraform displays all AWS resources that will be created.

---

# Step 10: Deploy the Jenkins Server

Deploy the infrastructure.

```bash
terraform apply \
-var-file=variables.tfvars \
--auto-approve
```

Deployment usually completes within a few minutes.

---

# Step 11: Connect to the EC2 Instance

After deployment, copy the public IP or use the generated SSH command.

Example:

```bash
ssh -i my-key.pem ubuntu@<PUBLIC_IP>
```

---

# Step 12: Verify Installed Software

The Jenkins server should already have the required DevOps tools installed.

Verify them using:

```bash
jenkins --version

docker --version

docker ps

terraform --version

kubectl version --client

aws --version

trivy --version

eksctl version
```

---

# Step 13: Access Jenkins

Open your browser.

```
http://<EC2_PUBLIC_IP>:8080
```

Retrieve the initial admin password.

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

---

# Step 14: Complete Jenkins Setup

1. Enter the initial admin password.
2. Click **Install Suggested Plugins**.
3. Wait for plugin installation.
4. Create the administrator account.
5. Click **Save and Finish**.
6. Click **Start using Jenkins**.

You should now see the Jenkins Dashboard.

---

# Next Steps

Once Jenkins is ready, you can:

- Configure AWS credentials
- Install additional plugins
- Create Jenkins Pipelines
- Provision Amazon EKS using Terraform
- Deploy applications using Argo CD
- Configure GitOps workflows
- Integrate SonarQube, Trivy, and OWASP Dependency Check

---

# Useful Commands

## Terraform

```bash
terraform init
terraform validate
terraform plan
terraform apply
terraform destroy
```

## AWS CLI

```bash
aws configure
aws sts get-caller-identity
aws ec2 describe-instances
```

## Docker

```bash
docker ps
docker images
docker system prune -a
```

## Kubernetes

```bash
kubectl get nodes
kubectl get pods -A
kubectl get svc -A
```

---

# Troubleshooting

### Terraform backend errors

Verify:

- S3 bucket exists
- DynamoDB table exists
- AWS credentials are configured

---

### Unable to SSH

Check:

- Security Group allows port **22**
- Correct PEM file is used
- Correct public IP address

---

### Jenkins not accessible

Ensure Security Group allows:

- TCP **8080**

Verify Jenkins service:

```bash
sudo systemctl status jenkins
```

View logs:

```bash
sudo journalctl -u jenkins -f
```

---

### AWS CLI authentication issues

Verify credentials:

```bash
aws sts get-caller-identity
```

---

# Clean Up

Destroy the infrastructure when it is no longer required.

```bash
terraform destroy \
-var-file=variables.tfvars \
--auto-approve
```

This removes all AWS resources created by Terraform.
