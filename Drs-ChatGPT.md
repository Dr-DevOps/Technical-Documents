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

### **5.8. Checking for Existing ACM Certificates in the DR Region**

1. **Log In to the AWS Management Console:**
   - Go to the [AWS Management Console](https://aws.amazon.com/console/) and log in.

2. **Select the DR Region:**
   - Choose the appropriate DR region from the region dropdown in the top-right corner.

3. **Open ACM Console:**
   - Navigate to the [AWS Certificate Manager (ACM) Console](https://console.aws.amazon.com/acm/home).

4. **View Certificates:**
   - Review the list of certificates to check if there are any for `*.api.virtualinternships.com` or `*.viplatform.net`.

5. **Check Details:**
   - Click on the certificate to verify that the domain names match `*.api.virtualinternships.com` or `*.viplatform.net` and ensure the status is "Issued."

### **2. Requesting a New ACM Certificate**

#### **For `*.api.virtualinternships.com` or `*.viplatform.net`:**

1. **Open ACM Console:**
   - Ensure you are in the correct region where you need the certificate.

2. **Request a Certificate:**
   - Click “Request a certificate” in the ACM dashboard.

3. **Choose Certificate Type:**
   - Select “Request a public certificate.”

4. **Enter Domain Names:**
   - For a wildcard certificate, enter `*.api.virtualinternships.com` or `*.viplatform.net` as the domain names.
   - You can enter both patterns if you need certificates for both domains.

5. **Choose Validation Method:**
   - **DNS Validation:** ACM will provide CNAME records that need to be added to your DNS settings.
   - **Email Validation:** ACM will send validation emails to the domain’s registered contacts.

6. **Review and Confirm:**
   - Confirm the request by reviewing the details and clicking “Confirm and request.”

7. **Add DNS Records (if using DNS Validation):**
   - For DNS validation, add the provided CNAME records to your DNS provider. Once done, return to the ACM console to continue the validation process.

8. **Monitor Status:**
   - Track the status of the certificate request. It will be marked as “Issued” once validation is successful.

9. **Use the Certificate:**
   - Associate the issued certificate with your resources such as CloudFront distributions or Elastic Load Balancers.

### **Additional Tips:**

- **Wildcard Certificates:** Wildcard certificates like `*.api.virtualinternships.com` or `*.viplatform.net` will cover all subdomains, but you must ensure that the domain names match exactly.
- **Domain Ownership:** Ensure you have control over the DNS records for the domains you are requesting certificates for, as you'll need to complete domain validation.

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

### 7.6.1 Updating Application Manifests

It's crucial to update the Kubernetes manifests for your DR environment. Focus on the following files in the `drs-prod-vi` directory:

1. Review Region-Specific Configurations:
   - Open `cluster-autoscaler.yaml`, `external-secret.yaml`, and `aws-ebs-csi-driver.yaml`
   - Locate the `AWS_REGION` or `REGION` environment variable or any region-specific parameters
   - Ensure these are set to the DR region (e.g., `us-east-1`)

2. Update Cluster Name in Load Balancer Controller:
   - Edit `aws-load-balancer-controller.yaml`
   - Find the `--cluster-name` flag in the controller arguments
   - Verify it matches your DR EKS cluster name (e.g., `drs-prod-vi`)

3. Verify SSL Certificate ARN:
   - In all ingress resource definitions, check the annotation:
     `alb.ingress.kubernetes.io/certificate-arn`
   - Confirm this ARN corresponds to a valid ACM certificate in the DR region

4. Validate IAM Role ARNs:
   - For each manifest using IAM roles (e.g., `external-secret.yaml`, `aws-load-balancer-controller.yaml`), verify the role ARNs are correct for the DR region

5. Update Endpoint References:
   - In application manifests, ensure any hardcoded endpoints (e.g., for RDS, ElastiCache, MSK) are updated to use the DR region resources

6. Check Resource Requests and Limits:
   - Review and adjust CPU and memory requests/limits if your DR environment has different capacity constraints

7. Verify ConfigMaps and Secrets:
   - Ensure any environment-specific ConfigMaps or Secrets are updated for the DR context

8. Update Ingress Hostnames:
   - If using different domain names in DR, update the `host` fields in Ingress resources

9. Adjust HPA (Horizontal Pod Autoscaler) Settings:
   - Review and modify HPA configurations if your DR scaling strategy differs

10. Validate PersistentVolumeClaim Configurations:
    - Ensure storage class names and other storage-related configurations are valid for the DR region

After making these changes:

1. Commit the updated manifests to your version control system.
2. Use a diff tool to double-check all changes between production and DR manifests.
3. Consider using Helm's template rendering (`helm template`) to verify the final Kubernetes resources that will be applied.
4. Plan a dry-run deployment in the DR environment to catch any configuration issues before an actual DR scenario.

Remember, maintaining parity between production and DR environments is crucial. Establish a process to review and update these manifests whenever changes are made to the production environment.

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

### Steps to Edit CNAME on CloudFront for Existing Records for Frontend

1. Log into the AWS Management Console

2. Navigate to CloudFront:
   - From the AWS services menu, select "CloudFront" under the "Networking & Content Delivery" section

3. Locate Your Distribution:
   - In the CloudFront dashboard, you'll see a list of your distributions
   - Find the distribution you want to modify

4. Edit the Distribution:
   - Click on the ID of the distribution you want to edit
   - This will take you to the distribution's detail page
   - Click the "Edit" button at the top of the page

5. Navigate to the CNAME Section:
   - Scroll down to the "Alternate Domain Names (CNAMEs)" section

6. Modify CNAME Records:
   - In the text box, you'll see your current CNAME records
   - To add a new CNAME:
     - Type the new domain name on a new line
   - To remove a CNAME:
     - Delete the line containing the CNAME you want to remove
   - To modify a CNAME:
     - Edit the existing domain name as needed

7. SSL Certificate:
   - If you're adding new CNAMEs, ensure that your SSL certificate covers these new domain names
   - You may need to request a new certificate in AWS Certificate Manager if your current one doesn't include the new domains

8. Review Changes:
   - Scroll through the other settings to ensure no unintended changes were made

9. Save Changes:
   - At the bottom of the page, click "Save Changes"

10. Wait for Deployment:
    - CloudFront will now deploy your changes across its network
    - This can take up to 15 minutes to complete

11. Verify Changes:
    - After deployment is complete, test your new or modified CNAMEs to ensure they're working correctly

12. Update DNS Records:
    - If you've added new CNAMEs, don't forget to update your DNS records with your domain registrar
    - Add or modify CNAME records to point to your CloudFront distribution's domain name (e.g., d1234abcd.cloudfront.net)

13. For DR Considerations:
    - If this is part of your DR setup, ensure that your DR plan is updated to reflect these changes
    - Document the process and any new CNAMEs in your DR documentation

Important Notes:
- Removing a CNAME from CloudFront doesn't automatically remove it from your DNS settings. Be sure to update your DNS records accordingly.
- Changes to CNAMEs can affect your site's availability. It's best to perform these changes during a maintenance window.
- Always test thoroughly after making changes to ensure your site is accessible via all intended domain names.
- If you're using this process as part of switching to a DR environment, ensure that you have a plan to quickly update DNS records to point to your DR CloudFront distribution if needed.

### **Steps to Add DNS Records in Cloudflare**

1. **Log In to Cloudflare:**
   - Go to the [Cloudflare login page](https://dash.cloudflare.com) and log in with your credentials.

2. **Select Your Domain:**
   - On the Cloudflare dashboard, select the domain (viplatform.net, or virtualinternships.com ) you want to configure DNS records for.

3. **Go to DNS Settings:**
   - Click on the “DNS” tab in the top navigation bar. This will take you to the DNS management page for your domain.

4. **Add a DNS Record:**
   - Click on the “Add record” button.

5. **Choose the Record Type:**
   - From the dropdown menu, select the type of DNS record you want to add. Common types include:
     - **A:** Maps a domain to an IPv4 address.
     - **AAAA:** Maps a domain to an IPv6 address.
     - **CNAME:** Maps a domain to another domain (canonical name).
     - **MX:** Defines mail exchange servers for your domain.
     - **TXT:** Allows you to add arbitrary text to a domain.
     - **SRV:** Specifies services available at specific ports.

6. **Enter Record Details:**
   - **Name:** Enter the subdomain or domain name for this record (e.g., `www`, `mail`, or `@` for the root domain).
   - **Value:** Enter the value for the record, such as an IP address or hostname.
   - **TTL (Time To Live):** Choose how long the DNS record should be cached by DNS resolvers. You can usually leave this as "Auto".
   - **Proxy Status:** Choose whether to proxy traffic through Cloudflare. “Proxied” enables Cloudflare’s services like CDN and security, while “DNS only” does not use Cloudflare’s proxy.

7. **Save the Record:**
   - Click the “Save” or “Add Record” button to save your changes.

8. **Verify the Record:**
   - Once added, the new record will appear in the list of DNS records. Ensure it is correctly configured.

9. **Check Propagation:**
   - DNS changes can take some time to propagate. Use tools like [DNS Checker](https://dnschecker.org) to verify that the changes are reflected globally.

### **Additional Tips:**
- **TTL Settings:** Lower TTL values (e.g., 300 seconds) can be useful during testing, but increase them for production.
- **Cloudflare Features:** Explore additional Cloudflare features such as SSL/TLS settings, Firewall rules (WAF), and Performance optimizations that can enhance your domain's security and performance.


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
