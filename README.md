
<img width="800" height="400" alt="CloudTrail-feat-img-2" src="https://github.com/user-attachments/assets/1bc24a28-3c3b-4c57-a894-0702dcbac721" />

### Project Overview

This project deploys AWS CloudTrail to automatically capture and log every API call made within an AWS account, storing the audit trail securely in S3 with real-time streaming to CloudWatch Logs for monitoring and alerting.

**Why this matters in the real world:** Without an audit trail, security teams have no way to investigate incidents, detect unauthorized access, or prove compliance during an audit. CloudTrail is one of the first things any serious AWS environment turns on. It's often the difference between answering "who did this and when?" in five minutes versus never knowing at all.

### **Problem**

Organizations struggle to maintain comprehensive audit trails of AWS API activities for security monitoring and compliance requirements. Without proper API logging, security teams cannot track user actions, investigate incidents, or demonstrate compliance with regulatory frameworks. Manual log collection is time-consuming and often incomplete, leaving security blind spots in cloud environments.

### **Solution**

Implement AWS CloudTrail to automatically capture and log all API calls across your AWS account, storing the audit logs in a secure S3 bucket with proper access controls. This solution provides centralized API logging with CloudWatch integration for real-time monitoring and alerting, ensuring complete visibility into AWS account activities while meeting compliance requirements.

### **Architecture Diagram**

<img width="504" height="744" alt="image" src="https://github.com/user-attachments/assets/73fbbc8e-d053-46a1-a97f-4b9878aaa0a6" />

Breakdown of what is happening:

1. All AWS account activity (console logins, API calls, service actions) flows through CloudTrail automatically.
2. CloudTrail delivers logs to two destinations in parallel: an S3 bucket for durable, long-term storage, and a CloudWatch Log Group for real-time visibility.
3. A CloudWatch metric filter continuously scans incoming log events for root account usage.
4. When a match occurs, a CloudWatch alarm transitions to ALARM state.
5. The alarm publishes a notification to an SNS topic, which emails the subscriber.

### Prerequisites

- AWS account with administrative permissions for CloudTrail, S3, CloudWatch, IAM, and SNS
- Terraform installed (version 1.0 or higher)
- AWS CLI installed and configured
- Basic understanding of IAM policies and JSON syntax

### Tools & Services Used

- AWS CloudTrail
- Amazon S3
- Amazon CloudWatch (Logs, Metric Filters, Alarms)
- AWS IAM
- Amazon SNS
- Terraform

### Preparation

The infrastructure is organized into four Terraform files:

- **`versions.tf`** — pins the Terraform and provider versions, and configures default resource tags applied account-wide.
- **`variables.tf`** — defines all configurable inputs (trail name, environment, retention settings, encryption options, alarm email, etc.), each with validation rules to prevent misconfiguration.
- **`main.tf`** — the core build: S3 bucket with versioning/encryption/public-access blocking, IAM role and policies for CloudTrail, the CloudTrail trail itself, CloudWatch Logs integration, and the SNS-backed root-account alarm.
- **`outputs.tf`** — surfaces key resource identifiers (ARNs, names) plus a set of pre-built AWS CLI commands for verification after deployment.

A `terraform.tfvars` file supplies environment-specific values not suited to defaults — in this case, the alarm notification email and a `force_destroy` flag for easy teardown during testing.

### Steps

#### **1. Define provider and version requirements**

Set up the Terraform block requiring version 1.0+, declare the AWS and Random providers, and configure default tags applied to every resource for consistent tracking.

<img width="696" height="545" alt="image-2" src="https://github.com/user-attachments/assets/afbd9d2d-e9b1-4f45-abd1-f9110ce7ad35" />


#### **2. Define input variables**

Create configurable variables for trail name, environment, log retention, encryption settings, and alarm notification email. Each protected with validation rules so invalid values fail fast before touching AWS.

<img width="1003" height="711" alt="image-3" src="https://github.com/user-attachments/assets/1cbeee6d-2ced-424e-abdb-d9149c6e492d" />


#### **3. Generate unique resource identifiers**

Use the `random_id` resource to generate a unique hex suffix, combined with the AWS account ID (via the `aws_caller_identity` data source) to build globally-unique, collision-free resource names.

<img width="527" height="170" alt="image-4" src="https://github.com/user-attachments/assets/6acc21a0-48c3-4925-b685-8138de3e2616" />


#### **4. Create the S3 bucket with security controls**

Provision the S3 bucket with versioning enabled, server-side encryption (AES256 or optional KMS), and a public access block to guarantee the bucket can never be exposed.

<img width="658" height="856" alt="image-17" src="https://github.com/user-attachments/assets/494a4c20-ac0a-4f56-9288-cca0cdbadcbb" />


#### **5. Configure the S3 bucket policy**

Build an IAM policy document granting CloudTrail permission to check the bucket ACL and write log objects. Scoped tightly with a `SourceArn` condition so only this specific trail can write to the bucket.

<img width="556" height="545" alt="image-18" src="https://github.com/user-attachments/assets/a3da2564-40ec-4401-9179-c23984dd30e1" />


<img width="831" height="473" alt="image-19" src="https://github.com/user-attachments/assets/c3b27cc3-c752-43d3-9802-f70a3a23efcf" />


#### **6. Create the CloudWatch Log Group**

Provision a log group with a configurable retention period to hold real-time CloudTrail events for fast querying and alerting.

<img width="595" height="182" alt="image-20" src="https://github.com/user-attachments/assets/d1a02aa7-9839-43d7-b566-6c5db5681d9f" />

#### **7. Create the IAM role for CloudTrail**

Build a trust policy allowing the CloudTrail service to assume a dedicated role, with least-privilege permissions limited to creating log streams and writing log events.

<img width="912" height="797" alt="image-21" src="https://github.com/user-attachments/assets/00ed9274-b000-41cb-82b5-49f4e5a6886e" />


#### **8. Create the CloudTrail trail**

Deploy the trail itself: multi-region, including global service events, with log file validation enabled for tamper detection, and linked to both the S3 bucket and the CloudWatch Log Group.

<img width="633" height="579" alt="image-22" src="https://github.com/user-attachments/assets/b5204625-1068-432b-8d0a-17ae872566da" />


#### **9. Set up the SNS alert pipeline**

Create an SNS topic and an email subscription, then a CloudWatch metric filter that scans for root account API usage, feeding a CloudWatch alarm that publishes to the SNS topic when triggered.

<img width="973" height="621" alt="image-23" src="https://github.com/user-attachments/assets/da5d1390-0496-4eff-a03f-99b3dea5863e" />


### Validation & Testing

#### **1. Verify CloudTrail logging status**

Confirmed `IsLogging: true` via `aws cloudtrail get-trail-status`, validating the trail was actively capturing events.

bash

`aws cloudtrail get-trail-status --name <trail-name>`

<img width="768" height="156" alt="image-24" src="https://github.com/user-attachments/assets/34537c0b-406f-412d-81a1-636f19075260" />


#### **2. Confirm real-time event capture**

Queried CloudTrail's event history directly to confirm recent API activity. Including Terraform's own deployment calls — was being captured.

bash

`aws cloudtrail lookup-events --max-items 10`

<img width="698" height="578" alt="image-25" src="https://github.com/user-attachments/assets/93b7946b-ca36-4410-82c4-c5b7089d2f1b" />


#### **3. Test the alarm and SNS notification pipeline**

Manually forced the root-account alarm into `ALARM` state to validate the end-to-end SNS email notification path without using actual root credentials. Confirmed the email arrived, then reset the alarm to `OK`.

bash

`aws cloudwatch set-alarm-state --alarm-name <alarm-name> --state-value ALARM --state-reason "Manual test"`

<img width="1479" height="753" alt="image-26" src="https://github.com/user-attachments/assets/f8065039-044f-4830-a80a-85704e6ffac5" />

### Links
