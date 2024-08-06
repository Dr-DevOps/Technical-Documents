# Disaster Recovery (DR) Setup Guide

This guide assumes you're replicating from a primary region (e.g., `ap-southeast-1`) to a DR region (e.g., `us-east-1`). Follow these steps to ensure your disaster recovery environment is robust and ready to go.

## Table of Contents
1. [Amazon Elastic Container Registry (ECR)](#1-amazon-elastic-container-registry-ecr)
2. [AWS Backup](#2-aws-backup)
   - [Amazon RDS Read Replicas](#21-amazon-rds-read-replicas)
3. [AWS Secrets Manager](#3-aws-secrets-manager)
4. [Amazon S3](#4-amazon-s3)
5. [Additional Steps for DR Setup](#additional-steps-for-dr-setup)
   - [Application Configurations](#application-configurations)
   - [IAM Roles and Permissions](#iam-roles-and-permissions)
   - [Testing](#testing)
   - [Monitoring](#monitoring)
   - [Documentation](#documentation)
   - [Failover Process](#failover-process)
   - [Regular Reviews](#regular-reviews)

---

### 1. Amazon Elastic Container Registry (ECR)

1. **Open the Amazon ECR console in your primary region.**
2. **Select the repository** you want to replicate (e.g., `prod-be-core`).
3. In the navigation pane, choose **Replication**.
4. Click **Edit replication rules**.
5. Choose **Add replication rule**.
6. Configure the replication rule:
   - **Destination Region:** Select your DR region (e.g., `us-east-1`)
   - **Destination Account ID:** Enter your AWS account ID (leave blank if the same account)
7. Click **Save** to create the replication rule.
8. **Verify replication:**
   - Switch to the DR region in the ECR console
   - Confirm that the repository exists and images are being replicated

---

### 2. AWS Backup

#### To configure AWS Backup for your RDS instances:

1. **Open the AWS Backup Console:** Navigate to the AWS Backup console in your primary region.
2. **Create a Backup Plan:** 
   - Go to **Backup plans** and click **Create backup plan**.
   - Choose a predefined plan or create a custom plan.
3. **Configure Backup Plan:**
   - **Name:** Give your backup plan a name.
   - **Backup Vault:** Choose an existing backup vault or create a new one.
   - **Backup Frequency:** Set the backup schedule and retention policy.
   - **Backup Window:** Define the time window for backups.
4. **Assign Resources:** 
   - Go to **Assign resources** and select the resources you want to back up.
   - For RDS, select the DB instances you want to include.
5. **Configure Backup Options:** Choose any additional backup options, such as lifecycle policies.
6. **Review and Create:** Review your backup plan settings and click **Create backup plan**.
7. **Monitor Backups:** Check the backup status and history in the AWS Backup console.

---

### 2.1 Amazon RDS Read Replicas

#### To set up read replicas in a disaster recovery (DR) region:

1. **Open the Amazon RDS Console:** Navigate to the Amazon RDS console in your primary region.
2. **Select Your Source DB Instance:** Choose the DB instance you want to replicate.
3. **Create Read Replica:** Choose **Actions** > **Create read replica**.
4. **Configure the Read Replica:**
   - **Destination Region:** Select your DR region.
   - **DB Instance Identifier:** Provide a name (e.g., `drs-prod-db-replica`).
   - **DB Instance Class:** Choose the appropriate size for DR.
   - **Multi-AZ Deployment:** Enable for high availability.
   - **VPC:** Select the VPC in your DR region.
5. **Configure Advanced Settings:** Adjust settings for storage, backup retention, maintenance, etc.
6. **Create Read Replica:** Click **Create read replica**.
7. **Monitor Creation:** Track the progress in the DR region's RDS console.
8. **Promote Read Replica:** In the event of a disaster, promote the read replica to a standalone DB instance as needed.

---

### 3. AWS Secrets Manager

1. **Open the AWS Secrets Manager console** in your primary region.
2. Select the secret you want to replicate.
3. Choose **Actions** > **Edit**.
4. Scroll to the **Replication** section.
5. Click **Add replica**.
6. Choose your DR region from the dropdown.
7. Click **Save** to start replication.
8. **Verify in the DR region:**
   - Switch to the DR region in the Secrets Manager console
   - Confirm that replicated secrets are present and up-to-date

---

### 4. Amazon S3

1. **Open the Amazon S3 console.**
2. Select the source bucket in your primary region.
3. Go to the **Management** tab.
4. Scroll to **Replication rules** and click **Create replication rule**.
5. Configure the replication rule:
   - **Rule name:** Give it a descriptive name
   - **Status:** Enable
   - **Source bucket:** Choose **Entire bucket**
   - **Destination:** Choose **Buckets in this account** and select the destination bucket in your DR region (create one if it doesn't exist)
   - **IAM role:** Create a new role or choose an existing one with necessary permissions
6. (Optional) Configure additional options like storage class, encryption, etc.
7. Review and click **Save** to create the replication rule.
8. **Verify replication:**
   - Check the destination bucket in the DR region
   - Confirm that objects are being replicated

---

### Additional Steps for DR Setup

#### Application Configurations
- Modify your Helm charts and Kubernetes manifests in the `drs-prod-vi` directory to use the replicated resources in the DR region.

#### IAM Roles and Permissions
- Ensure that necessary IAM roles in the DR region have permissions to access the replicated resources.

#### Testing
- Regularly test accessing these replicated resources from your DR environment to ensure everything is set up correctly.

#### Monitoring
- Set up CloudWatch alarms to monitor the replication status for each service.

#### Documentation
- Keep detailed documentation of all replicated resources, including their ARNs and endpoints in the DR region.

#### Failover Process
- Develop and document a clear process for failing over to these replicated resources in a DR scenario.

#### Regular Reviews
- Periodically review and update your replication settings to ensure they still meet your DR requirements.

---

By following these steps, you'll have set up cross-region replication for ECR, RDS, Secrets Manager, and S3, creating a robust foundation for your Disaster Recovery environment. Remember to regularly test and update your DR processes to ensure they remain effective and aligned with your business needs.

---

