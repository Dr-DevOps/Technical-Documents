# Comprehensive Disaster Recovery System Guide for AWS and EKS

## 1. Introduction

Disaster Recovery (DR) is a critical strategy for ensuring business continuity during unexpected events. This guide will walk you through setting up a DR environment using Amazon Web Services (AWS) and Elastic Kubernetes Service (EKS), mirroring your production environment to ensure seamless operations in case of a disaster.

## 2. Prerequisites

Before beginning the DR setup process, ensure you have the following:

1. **An AWS account with administrative permissions**
2. **Terraform (version 1.2.0 or higher) installed on your local machine**
3. **AWS Command Line Interface (CLI) installed and configured**
4. **Access to the following GitHub repositories:**
   - [Terraform Repository](https://github.com/viplatform/aws-terraform)
   - [Helm Chart Repository](https://github.com/viplatform/helmcharts)
5. **Basic familiarity with command-line operations**

### Repository Structure

- **Terraform configurations for the DR environment** are located in the `DRs-Prod` directory of the Terraform repository.
- **Helm charts and application manifests for the DR environment** are in the `drs-prod-vi` directory of the Helm chart repository.

### Basic Steps

a. **Clone the repositories:**
   ```bash
   git clone https://github.com/viplatform/aws-terraform.git
   git clone https://github.com/viplatform/helmcharts.git
   ```

b. **Navigate to the DR environment directories:**
   - For Terraform: `cd aws-terraform/DRs-Prod`
   - For Helm charts: `cd helmcharts/drs-prod-vi`

c. **Review and familiarize yourself** with the contents of these directories.

d. **Ensure you have the necessary permissions** to make changes to these repositories and create/modify resources in your AWS account.

e. **Set up your local environment** with the required tools (AWS CLI, Terraform, kubectl, helm, etc.) as detailed in the following sections.

## 3. Setting Up Your Tools

### 3.1 Installing Terraform (for Mac users)

1. Open Terminal on your Mac.
2. Install Homebrew if not already installed:
   ```bash
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
   ```
3. Add Hashicorp tap:
   ```bash
   brew tap hashicorp/tap
   ```
4. Install Terraform:
   ```bash
   brew install hashicorp/tap/terraform
   ```
5. Verify the installation:
   ```bash
   terraform -version
   ```
   Ensure the version is 1.2.0 or higher.

### 3.2 Configuring AWS CLI

1. Open Terminal.
2. Run the AWS configure command:
   ```bash
   aws configure
   ```
3. When prompted, enter the following information:
   - AWS Access Key ID
   - AWS Secret Access Key
   - Default region name (e.g., us-east-1)
   - Default output format (leave blank for default)

### 3.3 Cloning the Terraform Repository

1. Open Terminal.
2. Navigate to the directory where you want to clone the repository.
3. Clone the repository:
   ```bash
   git clone <your_organization's_github_repo_url>
   ```
4. Navigate into the cloned repository:
   ```bash
   cd <repo_directory>
   ```

## 4. Configuring Your DR Environment

### 4.1 Setting Up the Terraform Backend

1. Open the `backend.tf` file in a text editor.
2. Ensure it contains the following configuration:
   ```hcl
   terraform {
     backend "local" {
       path = "terraform.tfstate"
     }
   }
   ```

### 4.2 Configuring the AWS Provider

1. Open the `provider.tf` file in a text editor.
2. Ensure it contains the following configuration:
   ```hcl
   provider "aws" {
     region = var.region
   }

   variable "region" {
     description = "AWS Region"
     type = string
     default = "us-east-1"
   }
   ```

### 4.3 Setting Up the Virtual Private Cloud (VPC)

Ensure your Terraform configuration includes a VPC setup that mirrors the production environment. This typically involves:

1. Defining the VPC CIDR block.
2. Creating public and private subnets.
3. Setting up an Internet Gateway.
4. Configuring route tables.

### 4.4 Setting Up a Bastion Host

Configure a Bastion Host in your Terraform files:

1. Define an EC2 instance in a public subnet.
2. Configure security groups to allow SSH access.
3. Add a user data script to install necessary tools (Helm, AWS CLI, kubectl).

### 4.5 Creating the EKS Cluster

Set up an EKS cluster (version 1.28) in your Terraform configuration:

1. Define the EKS cluster resource.
2. Configure node groups.
3. Set up necessary IAM roles and policies.

## 5. Existing DR Environment Configuration

### 5.1 Elastic Container Registry (ECR)

Cross Region Replication for ECR is already configured and operational:

- The production backend core ECR repository (`prod-be-core`) is replicated to the DR region.
- Repository URL: `832726350816.dkr.ecr.us-east-1.amazonaws.com/prod-be-core`

### 5.2 Secrets Manager

AWS Secrets Manager replication is fully configured:

- All necessary secrets are being replicated to US-EAST-1.
- The DR environment is set up to use these replicated secrets.

### 5.3 S3 Buckets

Cross Region Replication for S3 is in place:

- Both internal and external system buckets are being replicated to US-EAST-1.
- Buckets with the prefix "drs" are used for backend applications in the DR environment.

### 5.4 Database (RDS)

RDS Cross Region Replication is active:

- The production database is being replicated to the DR region.
- The DR environment is configured to use this replicated database.

### 5.5 Restoring RDS in DR Region from AWS Backup

1. Log in to the AWS Management Console.
2. Switch to the DR region (e.g., us-east-1).
3. Navigate to AWS Backup:
   - Go to the AWS Backup console.
   - In the left navigation pane, choose "Backup vaults".
4. Locate the backup:
   - Select the appropriate backup vault.
   - Find the most recent RDS backup for your production database.
5. Initiate the restore process:
   - Select the backup you want to restore.
   - Click "Restore" to start the restoration process.
6. Configure the restore settings:
   - Choose "Restore to new RDS database" as the restore type.
   - Select the appropriate DB engine version (should match your production version).
   - Choose the DB instance class (match or scale as needed for DR).
   - Set up network & security:
     - VPC: Select the VPC created by Terraform for your DR environment.
     - Subnet group: Choose the subnet group for your DR environment.
     - Public accessibility: typically set to 'No' for security.
     - VPC security group: Select the security group created for RDS in your DR environment.
   - Configure instance settings:
     - DB instance identifier: Give it a unique name (e.g., "drs-prod-db").
     - Set the master username and password.
   - Additional configuration:
     - Set parameters like backup retention, monitoring, etc., to match your DR requirements.
7. Review and initiate restore:
   - Review all settings.
   - Click "Restore DB instance".
8. Monitor the restore process:
   - Go to the RDS console.
   - Find your new DR database instance.
   - Wait for the status to change to "Available".
9. Update your application configurations:
   - Once the database is available, go to its details page in the RDS console.
   - Copy the endpoint address.
   - Update your application manifests or environment variables with this new endpoint.
10. Verify database accessibility:
    - From your bastion host or an application instance, try connecting to the new database to ensure it's accessible.
11. (Optional) Set up ongoing replication:
    - If continuous replication from production to DR is required, consider setting up AWS DMS (Database Migration Service) for ongoing replication.

### 5.6 Amazon Managed Streaming for Apache Kafka (MSK)

Set up an MSK cluster in your Terraform configuration:

1. Define the MSK cluster resource.
2. Configure topics and partitions to match production.

### 5.7 Redis ElastiCache

Configure a Redis ElastiCache cluster in your Terraform files:

1. Define the ElastiCache cluster resource.
2. Set up subnet groups and security groups.

## 6. Deploying Your DR Environment

### 6.1 Applying Terraform Configuration

1. Open Terminal.
2. Navigate to your Terraform directory:
   ```bash
   cd aws-terraform/DRs-Prod
   ```
3. Initialize Terraform:
   ```bash
   terraform init
   ```
4. Preview the changes:
   ```bash
   terraform plan
   ```
5. Apply the changes:
   ```bash
   terraform apply
   ```
6. When prompted, type 'yes' to confirm the changes.

### 6.2 Verifying the Deployment

1. Check AWS resources:
   - Log into the AWS Console.
   - Verify that all expected resources (VPC, EKS, ECR, S3, etc.) are created in the DR region.
2. Verify EKS cluster:
   ```bash
   aws eks --

region us-east-1 describe-cluster --name drs-prod-vi
   ```
3. Update kubeconfig:
   ```bash
   aws eks --region us-east-1 update-kubeconfig --name drs-prod-vi
   ```
4. Check nodes:
   ```bash
   kubectl get nodes
   ```
5. Verify other AWS services:
   - Check ECR repositories.
   - Verify S3 bucket creation.
   - Ensure Secrets Manager secrets are replicated.
6. Review security groups and IAM roles to ensure proper access and permissions.

## 7. Setting Up Argo CD

### 7.1 Connecting to Your EKS Cluster

1. Open Terminal.
2. Run the following command:
   ```bash
   aws eks update-kubeconfig --name drs-prod-vi
   ```
   Replace "drs-prod-vi" with your actual EKS cluster name.

3. If you encounter connection issues, you may need to update the cluster's security group in the AWS EKS Subnet console to allow inbound traffic from the CIDR IP range.

### 7.2 Installing Argo CD

1. Add the Argo CD Helm repository:
   ```bash
   helm repo add argo https://argoproj.github.io/argo-helm
   helm repo update
   ```

2. Install Argo CD (go to cloned "helmchart/drs-prod-vi"):
   ```bash
   helm upgrade --install argocd argo/argo-cd --version 5.27.1 --namespace argocd --values drs-prod-vi/argocd-values.yaml
   ```

3. Get the initial Argo CD password:
   ```bash
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
   ```

### 7.3 Accessing Argo CD

1. Set up port forwarding:
   ```bash
   kubectl port-forward service/argocd-server -n argocd 8080:443
   ```

2. SSH to your Bastion host:
   ```bash
   ssh -i drs-production-bastion.pem ubuntu@<bastion_ip> -L 8080:localhost:8080
   ```

3. Open a web browser and go to `http://localhost:8080`.

### 7.4 Configuring Argo CD

1. Log in using the username 'admin' and the password obtained earlier.
2. Navigate to 'Settings' > 'Repositories'.
3. Add your Helm chart repository.
4. Create applications for each of your services.

### 7.5 Reviewing and Updating IAM Roles

1. Log in to the AWS Management Console and navigate to IAM.
2. Review all roles created for the EKS DRs cluster.
3. For each relevant role:
   a. Check the permissions to ensure they match your DR requirements.
   b. Update the trust relationships if necessary.
4. Specifically, check the drs-AWSSecretManagerAccessProd Role:
   a. Go to the role's "Trust relationships" tab.
   b. Edit the trust relationship.
   c. Ensure the following line is present and correct:
      ```json
      "oidc.eks.us-east-1.amazonaws.com/id/CE4F7AC97FCC50606C1AF23D17D0C463:sub": "system:serviceaccount:default:sa-eso"
      ```
   d. If it's not present or needs updating, add or modify it accordingly.
   e. Save the changes.

### 7.6 Updating Application Manifests

1. Navigate to your Helm charts repository.
2. Locate the DRs-prod-vi directory.
3. Review and update all Helm charts and manifests in this directory:
   a. Ensure all references to IAM roles are correct and match the roles you reviewed.
   b. Update any environment-specific configurations to match your DR setup.
   c. Verify that all necessary changes from the production environment are reflected here.

## 8. Final Configuration Steps

### 8.1 Updating DNS Records

1. Log in to your Cloudflare account.
2. Navigate to the DNS settings for your domain.
3. Update CNAME records to point to your new DR environment ingresses.
4. check ingresses for all apps:
   ```bash
   kubectl get ing -A
   ```

### 8.2 Configuring Content Delivery Network (CDN)

1. Remove the CNAME from the existing production CloudFront distribution.
2. Create a new CloudFront distribution for your DR environment.
3. Update the S3 bucket for your front-end build.
4. Adjust IAM policies to allow access to the DR S3 buckets.

## 9. Testing and Maintenance

1. **Regularly test your DR environment:**
   - Simulate failover scenarios.
   - Verify data integrity.
   - Ensure all applications are functioning correctly.

2. **Keep your DR environment up-to-date:**
   - Sync any changes from production to DR.
   - Regularly update and patch systems.

3. **Document and review your DR plan:**
   - Keep detailed documentation of the DR setup and processes.
   - Regularly review and update the DR plan as business needs change.

4. **Train your team:**
   - Ensure all relevant team members understand the DR process.
   - Conduct regular drills to familiarize the team with failover procedures.

By following this detailed guide, you will have set up a comprehensive Disaster Recovery environment that mirrors your production setup. Remember, the key to an effective DR strategy is regular testing, updating, and continuous improvement to ensure your business can quickly recover from any unforeseen events.

---
Happy disaster-proofing! 🚀
