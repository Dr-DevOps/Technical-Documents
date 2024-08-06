Certainly. I'll provide a more detailed guide, breaking down each step further for a comprehensive understanding of the Disaster Recovery (DR) system setup for AWS and EKS.

# Comprehensive Disaster Recovery System Guide for AWS and EKS

## 1. Introduction

Disaster Recovery (DR) is a critical strategy that ensures business continuity in the face of unexpected events. This guide will walk you through setting up a DR environment using Amazon Web Services (AWS) and Elastic Kubernetes Service (EKS), mirroring your production environment to ensure seamless operations in case of a disaster.

## 2. Prerequisites

Before beginning the DR setup process, ensure you have the following:

1. An AWS account with administrative permissions
2. Terraform installed on your local machine (version 1.2.0 or higher)
3. AWS Command Line Interface (CLI) installed and configured
4. Access to your organization's GitHub repository containing Terraform configuration files
5. Basic familiarity with command-line operations

## 3. Setting Up Your Tools

### 3.1 Installing Terraform (for Mac users)

1. Open Terminal on your Mac
2. Install Homebrew if not already installed:
   ```
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
   ```
3. Add Hashicorp tap:
   ```
   brew tap hashicorp/tap
   ```
4. Install Terraform:
   ```
   brew install hashicorp/tap/terraform
   ```
5. Verify the installation:
   ```
   terraform -version
   ```
   Ensure the version is 1.2.0 or higher.

### 3.2 Configuring AWS CLI

1. Open Terminal
2. Run the AWS configure command:
   ```
   aws configure
   ```
3. When prompted, enter the following information:
   - AWS Access Key ID
   - AWS Secret Access Key
   - Default region name (e.g., us-east-1)
   - Default output format (leave blank for default)

### 3.3 Cloning the Terraform Repository

1. Open Terminal
2. Navigate to the directory where you want to clone the repository
3. Clone the repository:
   ```
   git clone <your_organization's_github_repo_url>
   ```
4. Navigate into the cloned repository:
   ```
   cd <repo_directory>
   ```

## 4. Configuring Your DR Environment

### 4.1 Setting Up the Terraform Backend

1. Open the `backend.tf` file in a text editor
2. Ensure it contains the following configuration:
   ```hcl
   terraform {
     backend "local" {
       path = "terraform.tfstate"
     }
   }
   ```
   This configuration stores the Terraform state locally.

### 4.2 Configuring the AWS Provider

1. Open the `provider.tf` file in a text editor
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
   This sets up the AWS provider and defines the region variable.

### 4.3 Setting Up the Virtual Private Cloud (VPC)

Ensure your Terraform configuration includes a VPC setup that mirrors the production environment. This typically involves:

1. Defining the VPC CIDR block
2. Creating public and private subnets
3. Setting up Internet Gateway
4. Configuring route tables

### 4.4 Setting Up a Bastion Host

Configure a Bastion Host in your Terraform files:

1. Define an EC2 instance in a public subnet
2. Configure security groups to allow SSH access
3. Add user data script to install necessary tools (Helm, AWS CLI, kubectl)

### 4.5 Creating the EKS Cluster

Set up an EKS cluster (version 1.28) in your Terraform configuration:

1. Define the EKS cluster resource
2. Configure node groups
3. Set up necessary IAM roles and policies

## 4. Existing DR Environment Configuration

### 4.6 Elastic Container Registry (ECR)

Cross Region Replication for ECR is already configured and operational:

- The production backend core ECR repository (prod-be-core) is replicated to the DR region.
- Repository URL: 832726350816.dkr.ecr.us-east-1.amazonaws.com/prod-be-core

No additional setup is required. The DR environment will use these replicated images automatically.

### 4.7 Secrets Manager

AWS Secrets Manager replication is fully configured:

- All necessary secrets are being replicated from US-EAST-1 to the DR region.
- The DR environment is set up to use these replicated secrets.

No further action is needed. Your applications in the DR environment will access the correct, replicated secrets.

### 4.8 S3 Buckets

Cross Region Replication for S3 is in place:

- Both internal and external system buckets are being replicated from US-EAST-1 to the DR region.
- Buckets with the prefix "drs" are used for backend applications in the DR environment.

The DR environment is configured to use these replicated buckets. No additional setup is required.

### 4.9 Database (RDS)

RDS Cross Region Replication is active:

- The production database is being replicated to the DR region.
- The DR environment is configured to use this replicated database.

Important: While replication is ongoing, for the most up-to-date data in case of failover:
1. Use the most recent AWS Backup snapshot of your production database.
2. Restore this snapshot in the DR region if needed.
3. The backend application manifests are already updated with the correct database endpoint for the DR environment.

### 4.10 Configuring Amazon Managed Streaming for Apache Kafka (MSK)

Set up an MSK cluster in your Terraform configuration:

1. Define the MSK cluster resource
2. Configure topics and partitions to match production

### 4.11 Setting Up Redis ElastiCache

Configure a Redis ElastiCache cluster in your Terraform files:

1. Define the ElastiCache cluster resource
2. Set up subnet groups and security groups

## 5. Deploying Your DR Environment

1. Open Terminal
2. Navigate to your Terraform directory
3. Initialize Terraform:
   ```
   terraform init
   ```
4. Preview the changes:
   ```
   terraform plan
   ```
5. Apply the changes:
   ```
   terraform apply
   ```
6. When prompted, type 'yes' to confirm the changes

## 6. Setting Up Argo CD

### 6.1 Connecting to Your EKS Cluster

1. Open Terminal
2. Run the following command:
   ```
   aws eks update-kubeconfig --name drs-prod-vi
   ```
   Replace "drs-prod-vi" with your actual EKS cluster name

3. If you encounter connection issues, you may need to update the cluster's security group in the AWS console to allow inbound traffic from your IP address

### 6.2 Installing Argo CD

1. Add the Argo CD Helm repository:
   ```
   helm repo add argo https://argoproj.github.io/argo-helm
   helm repo update
   ```

2. Install Argo CD:
   ```
   helm upgrade --install argocd argo/argo-cd --version 5.27.1 --namespace argocd --values drs-prod-vi/argocd-values.yaml
   ```

3. Get the initial Argo CD password:
   ```
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
   ```

### 6.3 Accessing Argo CD

1. Set up port forwarding:
   ```
   kubectl port-forward service/argocd-server -n argocd 8080:443
   ```

2. SSH to your Bastion host:
   ```
   ssh -i drs-production-bastion.pem ubuntu@<bastion_ip> -L 8080:localhost:8080
   ```

3. Open a web browser and go to `http://localhost:8080`

### 6.4 Configuring Argo CD

1. Log in using the username 'admin' and the password obtained earlier
2. Navigate to 'Settings' > 'Repositories'
3. Add your Helm chart repository
4. Create applications for each of your services

## 7. Final Configuration Steps

### 7.1 Updating DNS Records

1. Log in to your Cloudflare account
2. Navigate to the DNS settings for your domain
3. Update CNAME records to point to your new DR environment ingresses

### 7.2 Configuring Content Delivery Network (CDN)

1. Remove the CNAME from the existing production CloudFront distribution
2. Create a new CloudFront distribution for your DR environment
3. Update the S3 bucket for your frontend build
4. Adjust IAM policies to allow access to the DR S3 buckets

## 8. Testing and Maintenance

1. Regularly test your DR environment:
   - Simulate failover scenarios
   - Verify data integrity
   - Ensure all applications are functioning correctly

2. Keep your DR environment up-to-date:
   - Sync any changes from production to DR
   - Regularly update and patch systems

3. Document and review your DR plan:
   - Keep detailed documentation of the DR setup and processes
   - Regularly review and update the DR plan as business needs change

4. Train your team:
   - Ensure all relevant team members understand the DR process
   - Conduct regular drills to familiarize the team with failover procedures

By following this detailed guide, you will have set up a comprehensive Disaster Recovery environment that mirrors your production setup. Remember, the key to an effective DR strategy is regular testing, updating, and continuous improvement to ensure your business can quickly recover from any unforeseen events.
