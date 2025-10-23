## Build and Audit IAM Resources in AWS

This project demonstrates how to **Build and Audit IAM Resources in AWS** using AWS CLI. It is a basic-level IAM environment with users, groups, and roles configured with least privilege access. Security audits are performed using AWS tools like **Credential Reports**, **CloudTrail**, and **Access Analyzer**. It also suggests **remediation measures** taken in response to audit findings to ensure AWS security best practices are followed.

## Architecture Overview

```text
AWS Account
│
├── IAM
│   ├── Users: dev1-user, dev2-user, ops1-user, auditor-user
│   ├── Groups:
│   │   ├── Developers → AmazonS3ReadOnlyAccess
│   │   ├── Operations → AmazonEC2FullAccess
│   │   └── Auditors   → SecurityAudit
│   └── Roles:
│       ├── EC2S3ReadRole → EC2 assumes for S3 access
│       └── LambdaAuditRole → Lambda assumes for audit logs
│
└── Auditing Tools
    ├── IAM Credential Report
    ├── CloudTrail (Root Account Activity)
    └── IAM Access Analyzer

```
## Step 1 Configure AWS CLI and Profile:

First step was to create an admin user "demoadmin" in AWS console having MFA authentication enabled, and access keys. This admin user had full access to all resources in AWS setup of an organization and instead of using Root account for all operations, admin account shall be used. The admin profile allows us to interact securely with AWS CLI by separating credentials and preparing for IAM management.

- Configure the profile:

aws configure --profile demoadmin

Enter:
AWS Access Key ID → paste demoadmin’s Access key ID

AWS Secret Access Key → paste demoadmin’s Secret Access Key

Default region → your preferred region (e.g., ca-central-1)

Output format → json

- Verify the profile:

aws sts get-caller-identity --profile demoadmin

Output:
{
    "UserId": "------",
    "Account": "-----",
    "Arn": "arn:aws:iam::-------:user/demoadmin"
}

# Step 2. Build IAM resources:

In this step, we created groups, users, roles, attached policies with the groups, and added users to the groups. 

## 2.1 Create IAM Groups:

```markdown
aws iam create-group --group-name Developers --profile demoadmin
aws iam create-group --group-name Operations --profile demoadmin
aws iam create-group --group-name Auditors --profile demoadmin

## 2.2 Attach policies to Groups:

- S3 Read Only Access to Developers: 

```markdown
aws iam attach-group-policy --group-name Developers --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess --profile demoadmin

- EC2 Full Access to Operations:

```markdown
aws iam attach-group-policy --group-name Operations --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --profile demoadmin

- Security Audit Access for Auditors:

markdown
aws iam attach-group-policy --group-name Auditors --policy-arn arn:aws:iam::aws:policy/SecurityAudit --profile demoadmin

## 2.3 Create Users:

```markdown
aws iam create-user --user-name dev1-user --profile demoadmin
aws iam create-user --user-name dev2-user --profile demoadmin
aws iam create-user --user-name ops1-user --profile demoadmin

## 2.4 Add Users to Groups:

```markdown
aws iam add-user-to-group --user-name dev1-user --group-name Developers --profile demoadmin
aws iam add-user-to-group --user-name dev2-user --group-name Developers --profile demoadmin
aws iam add-user-to-group --user-name Ops1-user --group-name Operations --profile demoadmin
aws iam add-user-to-group --user-name Auditor-user --group-name Auditors --profile demoadmin

## 2.5 Create IAM Roles and attach Policies:

Step A: Create Trust Policies

trust-ec2.json

{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}

trust-lambda.json

{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "lambda.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}

Step B: Create Roles and Attach Policies

aws iam create-role --role-name EC2S3ReadRole --assume-role-policy-document file://trust-ec2.json --profile demoadmin
aws iam attach-role-policy --role-name EC2S3ReadRole --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess --profile demoadmin

aws iam create-role --role-name LambdaAuditRole --assume-role-policy-document file://trust-lambda.json --profile demoadmin
aws iam attach-role-policy --role-name LambdaAuditRole --policy-arn arn:aws:iam::aws:policy/SecurityAudit --profile demoadmin

## Step 3. Verify Roles, Users, and Groups:

Lastly, we ran some checks with AWS CLI to confirm all the resources we created so far are working correctly.

3.1. Verify Groups

```markdown
aws iam list-groups --profile demoadmin

3.2. Verify Users

```markdown
aws iam list-users --profile demoadmin

3.3. Verify Users in Groups

```markdown
aws iam get-group --group-name Developers --profile demoadmin
aws iam get-group --group-name Ops --profile demoadmin
aws iam get-group --group-name Auditors --profile demoadmin

3.4. Verify Roles

```markdown
aws iam list-roles --profile demoadmin

3.5. Verify Role Policies

```markdown
aws iam list-attached-role-policies --role-name EC2S3ReadRole --profile demoadmin
aws iam list-attached-role-policies --role-name LambdaAuditRole --profile demoadmin

## Step 4.  Auditing IAM Setup:

We have built the IAM system, and next step is to audit the settings and check for any risks and misconfigurations. There are certain tools and services available at AWS that help to audit the IAM setup and provide an insight in to security posture of the organization. Lets go through the services we used for this project and analyze the findings. 

## 4.1 — Generate a Credential Report:
IAM Credential report is a tool used to access multiple security features enabled in an organization's IAM setup including MFA status, passwords rotation and usage date, user access keys.  

i. Run `aws iam generate-credential-report --profile demoadmin` to generate the report.

ii. run `aws iam get-credential-report --profile demoadmin --query "Content" --output text > report.b64` to download the report. 

iii. Run `certutil -decode report.b64 credential-report.csv` to decode to CSV.

iv. Analyze the Report:

Analyze following columns:

- Username
- ARN
- User Creation time
- MFA active/not
- Access key age
- Password Created
- Password last used

This report detected different security related elements of IAM setup, including the users where MFA was not enabled by marking it as False, keys last rotation date to identify the long lived keys, password last used date to disable or delete the inactive users. Most important part is to check Root account usage and whether MFA was enabled, there should not be active access keys and check for account usage. In real world scenarios, organizations run this report regularly to detect insecure IAM practices, achieve compliance with different security frameworks, and reduce the risk of compromised credentials. 

## 4.2 Check for Long-lived Access Keys

Next secuity check is to see if there are any long lived access keys in the account. From the credential report CSV, we checked the following columns:

- access_key_1_active
- access_key_1_last_rotated
- access_key_1_last_used_date
- access_key_1_last_used_region
- access_key_1_last_used_service
- access_key_1_age

From this information, we can identify if there are any access keys that are older than 90 days. In our demo lab, there were no such access keys but in real scenarios if any older keys are found, then it has to be deleted and create new keys.

## 4.3 Check Root Account Usage:

```markdown
aws cloudtrail lookup-events --lookup-attributes AttributeKey=Username,AttributeValue="root" --profile demoadmin

The output returned multiple entries including list applications, describe regions, list notifications hub, get environment status. This indicated that root account has been used for daily tasks which is against the best security practices. To correct this, following remediation measures were taken immediately:

- Signed in to console and check if MFA is enabled for root account
- Made sure there are no active access keys
- Changed the password for security
- Never use root account for daily tasks in future

 ## 4.4 Check Overbroad Permissions:

```markdown
aws iam list-attached-group-policies --group-name Developers --profile demoadmin
aws iam list-attached-group-policies --group-name Ops --profile demoadmin
aws iam list-attached-group-policies --group-name Auditors --profile demoadmin

The results showed the policies attached to respective groups as we assigned earlier in this project. Only admin group showed Adminstrative access policy as assigned. 

## 4.5 Enable IAM Access Analyzer

IAM Access Analyzer is a cruicial tool in auditing as it scans the AWS account for resources that are accessible publicly and find any misconfigurations. To implement this, we created an Analyzer using the command:

```markdown
aws accessanalyzer create-analyzer --analyzer-name iam-audit-analyzer --type ACCOUNT --profile demoadmin

List the findings using the command:

```markdown
aws accessanalyzer list-findings --analyzer-arn arn:aws:access-analyzer:ca-central-1:123456789012:analyzer/iam-audit-analyzer --profile demoadmin

This listed the findings of Access Analyzer and it identified a S3 bucket that was publicly accessible. To remediate this, the S3 bucket was removed from being public immediately. In real organization, Access Analyzer is enabled continuously to find any publicaly exposed resources and findings are intergrated into Security Hub or Cloud watch dashboard. For automated remediation, a Lambda function may be added to automatically remove the S3 bucket from being public.    

Step 5. Project Cleanup:

aws iam delete-user --user-name dev1-user --profile demoadmin
aws iam delete-user --user-name dev2-user --profile demoadmin
aws iam delete-user --user-name ops1-user --profile demoadmin
aws iam delete-user --user-name auditor-user --profile demoadmin

aws iam delete-group --group-name Developers --profile demoadmin
aws iam delete-group --group-name Operations --profile demoadmin
aws iam delete-group --group-name Auditors --profile demoadmin

aws iam delete-role --role-name EC2S3ReadRole --profile demoadmin
aws iam delete-role --role-name LambdaAuditRole --profile demoadmin

aws accessanalyzer delete-analyzer --analyzer-name iam-audit-analyzer --profile demoadmin


