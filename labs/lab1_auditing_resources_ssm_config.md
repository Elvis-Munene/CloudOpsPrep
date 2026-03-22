# Lab 1: Auditing Your AWS Resources with AWS Systems Manager and AWS Config

## Estimated Time: 1 hour 30 minutes

---

## Overview

This hands-on lab teaches you how to audit and ensure compliance of your AWS resources using AWS Config and AWS Systems Manager (SSM). You'll learn to track configuration changes, detect non-compliant resources, and implement automated remediation—critical skills for the AWS Certified CloudOps Engineer (SOA-C03) exam.

**Key Exam Topics Covered:**
- Domain 1: Monitoring, Logging, and Remediation
- Domain 4: Security and Compliance
- AWS Config rules and compliance monitoring
- Systems Manager inventory and compliance
- Automated remediation workflows

---

## Learning Objectives

By the end of this lab, you will be able to:

1. Configure AWS Config to track resource configuration changes
2. Implement managed Config rules to evaluate resource compliance
3. Set up AWS Systems Manager Inventory for EC2 instance tracking
4. Use SSM Run Command to audit software packages across instances
5. Create custom Lambda-backed Config rules for specialized compliance checks
6. Configure automatic remediation using SSM Automation documents
7. Analyze compliance dashboards and generate audit reports

---

## Prerequisites

### AWS Account Requirements
- Active AWS account with administrative access
- Free Tier eligible (most services used qualify for Free Tier)
- Estimated cost if beyond Free Tier: $2-5 for lab duration

### Required IAM Permissions
Your IAM user/role needs:
- `AWSConfigRole` (managed policy)
- `AmazonSSMFullAccess` (managed policy)
- `AmazonEC2FullAccess` (for launching test instances)
- `IAMFullAccess` (to create service roles)
- `AWSLambdaFullAccess` (for custom Config rules)

### Tools Needed
- AWS CLI v2 installed and configured (`aws configure`)
- SSH client (for optional EC2 access)
- Web browser (for AWS Console)

### Knowledge Prerequisites
- Basic understanding of EC2 instances
- Familiarity with IAM roles and policies
- Basic JSON/YAML knowledge

---

## Architecture Overview

In this lab, you'll build the following architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                         AWS Account                          │
│                                                               │
│  ┌──────────────┐         ┌─────────────────┐              │
│  │  AWS Config  │────────▶│  S3 Bucket      │              │
│  │  Recorder    │         │  (Config Logs)  │              │
│  └──────┬───────┘         └─────────────────┘              │
│         │                                                    │
│         │ Evaluates                                         │
│         ▼                                                    │
│  ┌──────────────────────────────────────┐                  │
│  │       Config Rules                    │                  │
│  │  • ec2-instance-managed-by-ssm       │                  │
│  │  • required-tags                      │                  │
│  │  • restricted-ssh                     │                  │
│  │  • Custom Lambda-backed rule          │                  │
│  └──────┬───────────────────────────────┘                  │
│         │                                                    │
│         │ Triggers (on non-compliance)                      │
│         ▼                                                    │
│  ┌──────────────────────────────────────┐                  │
│  │   SSM Automation Document            │                  │
│  │   (Automatic Remediation)            │                  │
│  └──────┬───────────────────────────────┘                  │
│         │                                                    │
│         │ Remediates                                        │
│         ▼                                                    │
│  ┌──────────────────────────────────────┐                  │
│  │         EC2 Instances                 │                  │
│  │    ┌──────────────────────┐          │                  │
│  │    │   SSM Agent          │          │                  │
│  │    │   • Inventory         │          │                  │
│  │    │   • Compliance        │          │                  │
│  │    │   • Run Command       │          │                  │
│  │    └──────────────────────┘          │                  │
│  └───────────────────────────────────────┘                  │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**How It Works:**
1. AWS Config continuously records resource configurations
2. Config Rules evaluate resources against compliance standards
3. SSM Agent on EC2 instances reports inventory and compliance data
4. Non-compliant resources trigger automatic remediation via SSM Automation
5. All configuration data is stored in S3 for auditing

---

## Part 1: Setting Up AWS Config

AWS Config provides a detailed view of resource configurations and how they change over time. This is essential for compliance auditing, security analysis, and troubleshooting.

### Step 1.1: Create an S3 Bucket for Config Data

**Why:** AWS Config needs an S3 bucket to store configuration snapshots and history files.

#### Using AWS CLI:

```bash
# Set your preferred region
export AWS_REGION=us-east-1

# Create a unique bucket name (replace 'your-initials' with your initials)
export CONFIG_BUCKET=config-bucket-$(aws sts get-caller-identity --query Account --output text)-$AWS_REGION

# Create the S3 bucket
aws s3 mb s3://$CONFIG_BUCKET --region $AWS_REGION

# Create bucket policy to allow Config to write
cat > /tmp/config-bucket-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSConfigBucketPermissionsCheck",
      "Effect": "Allow",
      "Principal": {
        "Service": "config.amazonaws.com"
      },
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::$CONFIG_BUCKET"
    },
    {
      "Sid": "AWSConfigBucketExistenceCheck",
      "Effect": "Allow",
      "Principal": {
        "Service": "config.amazonaws.com"
      },
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::$CONFIG_BUCKET"
    },
    {
      "Sid": "AWSConfigBucketPutObject",
      "Effect": "Allow",
      "Principal": {
        "Service": "config.amazonaws.com"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::$CONFIG_BUCKET/AWSLogs/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control"
        }
      }
    }
  ]
}
EOF

# Apply the bucket policy
aws s3api put-bucket-policy --bucket $CONFIG_BUCKET --policy file:///tmp/config-bucket-policy.json
```

#### Using AWS Console:

1. Navigate to **S3** console
2. Click **Create bucket**
3. Bucket name: `config-bucket-[ACCOUNT-ID]-[REGION]`
4. Region: Select your preferred region (e.g., us-east-1)
5. Block Public Access: Keep all boxes checked (recommended)
6. Click **Create bucket**
7. Select the bucket → **Permissions** tab → **Bucket Policy**
8. Paste the policy from above (replace `$CONFIG_BUCKET` with your bucket name)

**Exam Tip:** Config requires specific S3 bucket permissions. The service principal `config.amazonaws.com` needs `GetBucketAcl`, `ListBucket`, and `PutObject` permissions.

### Step 1.2: Create IAM Role for AWS Config

**Why:** Config needs permissions to access AWS resources and write to S3.

#### Using AWS CLI:

```bash
# Create trust policy for Config
cat > /tmp/config-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "config.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the IAM role
aws iam create-role \
  --role-name AWSConfigRole \
  --assume-role-policy-document file:///tmp/config-trust-policy.json

# Attach AWS managed policy for Config
aws iam attach-role-policy \
  --role-name AWSConfigRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/ConfigRole

# Create inline policy for S3 access
cat > /tmp/config-s3-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketVersioning",
        "s3:PutObject",
        "s3:GetObject"
      ],
      "Resource": [
        "arn:aws:s3:::$CONFIG_BUCKET",
        "arn:aws:s3:::$CONFIG_BUCKET/*"
      ]
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name AWSConfigRole \
  --policy-name ConfigS3Access \
  --policy-document file:///tmp/config-s3-policy.json

# Get the role ARN for later use
export CONFIG_ROLE_ARN=$(aws iam get-role --role-name AWSConfigRole --query 'Role.Arn' --output text)
echo "Config Role ARN: $CONFIG_ROLE_ARN"
```

#### Using AWS Console:

1. Navigate to **IAM** console → **Roles** → **Create role**
2. Trusted entity type: **AWS service**
3. Use case: **Config**
4. Click **Next**
5. Attach policy: **ConfigRole** (AWS managed)
6. Role name: `AWSConfigRole`
7. Click **Create role**
8. Select the role → **Add permissions** → **Create inline policy**
9. JSON tab → paste the S3 policy above
10. Policy name: `ConfigS3Access`

### Step 1.3: Enable AWS Config Recorder

**Why:** The Configuration Recorder captures resource configurations and changes.

#### Using AWS CLI:

```bash
# Create configuration recorder
aws configservice put-configuration-recorder \
  --configuration-recorder name=default,roleARN=$CONFIG_ROLE_ARN \
  --recording-group allSupported=true,includeGlobalResourceTypes=true

# Create delivery channel
aws configservice put-delivery-channel \
  --delivery-channel name=default,s3BucketName=$CONFIG_BUCKET

# Start the configuration recorder
aws configservice start-configuration-recorder \
  --configuration-recorder-name default

# Verify recorder status (should show "recording": true)
aws configservice describe-configuration-recorder-status
```

**Expected Output:**
```json
{
    "ConfigurationRecordersStatus": [
        {
            "name": "default",
            "lastStatus": "SUCCESS",
            "recording": true
        }
    ]
}
```

#### Using AWS Console:

1. Navigate to **AWS Config** console
2. If first time: Click **Get started** → **1-click setup**
3. For manual setup:
   - Click **Settings** in the left menu
   - **Edit** Configuration recorder
   - Recording strategy: **Record all resource types**
   - Include global resources: **Yes** (checked)
   - S3 bucket: Select the bucket created earlier
   - IAM role: Select `AWSConfigRole`
   - Click **Save**

**Exam Tip:** Including global resources (IAM, CloudFront, etc.) should only be enabled in ONE region to avoid duplication and extra costs.

### Step 1.4: Add Managed Config Rules

AWS provides pre-built managed rules for common compliance scenarios.

#### Rule 1: ec2-instance-managed-by-ssm

**Purpose:** Ensures EC2 instances are managed by Systems Manager.

```bash
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "ec2-instance-managed-by-ssm",
  "Description": "Checks if EC2 instances are managed by AWS Systems Manager",
  "Source": {
    "Owner": "AWS",
    "SourceIdentifier": "EC2_INSTANCE_MANAGED_BY_SSM"
  },
  "Scope": {
    "ComplianceResourceTypes": ["AWS::EC2::Instance"]
  }
}'
```

**Console Method:**
1. AWS Config → **Rules** → **Add rule**
2. Search: `EC2_INSTANCE_MANAGED_BY_SSM`
3. Click rule → **Next**
4. Leave defaults → **Next** → **Add rule**

#### Rule 2: required-tags

**Purpose:** Ensures resources have required tags (e.g., Environment, Owner).

```bash
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "required-tags",
  "Description": "Checks if resources have required tags",
  "Source": {
    "Owner": "AWS",
    "SourceIdentifier": "REQUIRED_TAGS"
  },
  "InputParameters": "{\"tag1Key\":\"Environment\",\"tag2Key\":\"Owner\"}",
  "Scope": {
    "ComplianceResourceTypes": [
      "AWS::EC2::Instance",
      "AWS::EC2::Volume"
    ]
  }
}'
```

**Console Method:**
1. AWS Config → **Rules** → **Add rule**
2. Search: `REQUIRED_TAGS`
3. Configure:
   - tag1Key: `Environment`
   - tag2Key: `Owner`
4. Resource types: `AWS::EC2::Instance`, `AWS::EC2::Volume`
5. **Next** → **Add rule**

#### Rule 3: restricted-ssh

**Purpose:** Detects security groups allowing unrestricted SSH (0.0.0.0/0) access.

```bash
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "restricted-ssh",
  "Description": "Checks if security groups allow unrestricted SSH access",
  "Source": {
    "Owner": "AWS",
    "SourceIdentifier": "INCOMING_SSH_DISABLED"
  },
  "Scope": {
    "ComplianceResourceTypes": ["AWS::EC2::SecurityGroup"]
  }
}'
```

**Console Method:**
1. AWS Config → **Rules** → **Add rule**
2. Search: `INCOMING_SSH_DISABLED`
3. **Next** → **Add rule**

#### Verify Rules are Active

```bash
# List all Config rules
aws configservice describe-config-rules

# Check compliance status
aws configservice describe-compliance-by-config-rule
```

**Exam Tip:** Config rules evaluate resources either:
- **Configuration changes** (triggered when resource changes)
- **Periodic** (runs on a schedule, e.g., every 24 hours)
- **Hybrid** (both triggers)

---

## Part 2: AWS Systems Manager Inventory & Compliance

Systems Manager provides operational insights into EC2 instances and on-premises servers.

### Step 2.1: Launch EC2 Instances with SSM Agent

**Why:** We need test instances to audit with SSM and Config.

#### Create IAM Instance Profile for SSM

```bash
# Create trust policy for EC2
cat > /tmp/ec2-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create IAM role for EC2 instances
aws iam create-role \
  --role-name SSMManagedInstanceRole \
  --assume-role-policy-document file:///tmp/ec2-trust-policy.json

# Attach AWS managed policy for SSM
aws iam attach-role-policy \
  --role-name SSMManagedInstanceRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# Create instance profile
aws iam create-instance-profile \
  --instance-profile-name SSMManagedInstanceProfile

# Add role to instance profile
aws iam add-role-to-instance-profile \
  --instance-profile-name SSMManagedInstanceProfile \
  --role-name SSMManagedInstanceRole

# Wait for instance profile propagation
echo "Waiting 10 seconds for IAM propagation..."
sleep 10
```

#### Launch Test EC2 Instances

```bash
# Get the latest Amazon Linux 2023 AMI
export AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023*-x86_64" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text)

# Get default VPC and subnet
export VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' \
  --output text)

export SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[0].SubnetId' \
  --output text)

# Create security group
export SG_ID=$(aws ec2 create-security-group \
  --group-name SSMTestSG \
  --description "Security group for SSM test instances" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

# No need to add SSH rule - SSM Session Manager provides access
# This demonstrates security best practice

# Launch instance 1 (compliant - with tags)
aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --iam-instance-profile Name=SSMManagedInstanceProfile \
  --security-group-ids $SG_ID \
  --subnet-id $SUBNET_ID \
  --tag-specifications \
    'ResourceType=instance,Tags=[{Key=Name,Value=SSM-Test-Instance-1},{Key=Environment,Value=Dev},{Key=Owner,Value=CloudOps}]' \
    'ResourceType=volume,Tags=[{Key=Environment,Value=Dev},{Key=Owner,Value=CloudOps}]' \
  --query 'Instances[0].InstanceId' \
  --output text

# Launch instance 2 (non-compliant - missing tags)
aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --iam-instance-profile Name=SSMManagedInstanceProfile \
  --security-group-ids $SG_ID \
  --subnet-id $SUBNET_ID \
  --tag-specifications \
    'ResourceType=instance,Tags=[{Key=Name,Value=SSM-Test-Instance-2}]' \
  --query 'Instances[0].InstanceId' \
  --output text

echo "Instances launched. Wait 2-3 minutes for SSM agent registration."
```

#### Using AWS Console:

1. **EC2** → **Launch Instance**
2. Name: `SSM-Test-Instance-1`
3. AMI: Amazon Linux 2023
4. Instance type: t2.micro
5. **Advanced details** → IAM instance profile: `SSMManagedInstanceProfile`
6. **Add tags**:
   - `Environment`: `Dev`
   - `Owner`: `CloudOps`
7. **Launch instance**
8. Repeat for Instance 2 without Environment/Owner tags

**Important:** Amazon Linux 2023, Amazon Linux 2, Ubuntu, and Windows AMIs have SSM Agent pre-installed.

### Step 2.2: Verify SSM Managed Instances

Wait 2-3 minutes after instance launch, then verify:

```bash
# List managed instances
aws ssm describe-instance-information \
  --query 'InstanceInformationList[*].[InstanceId,PlatformName,PingStatus]' \
  --output table
```

**Expected Output:**
```
-------------------------------------------------------
|            DescribeInstanceInformation             |
+-----------------------+------------------+----------+
|  i-0123456789abcdef0 |  Amazon Linux    |  Online  |
|  i-0fedcba9876543210 |  Amazon Linux    |  Online  |
+-----------------------+------------------+----------+
```

**Console:** Systems Manager → **Fleet Manager** → View managed instances

**Troubleshooting:** If instances don't appear:
- Verify IAM instance profile is attached
- Check VPC has internet access (or VPC endpoints for SSM)
- Review instance logs: `aws ssm get-console-output --instance-id <instance-id>`

### Step 2.3: Configure SSM Inventory Collection

**Why:** Inventory collects metadata about instances (OS, applications, network config, etc.).

```bash
# Create inventory association (applies to all managed instances)
aws ssm create-association \
  --name AWS-GatherSoftwareInventory \
  --targets "Key=InstanceIds,Values=*" \
  --schedule-expression "rate(30 minutes)" \
  --parameters '{
    "applications": ["Enabled"],
    "awsComponents": ["Enabled"],
    "networkConfig": ["Enabled"],
    "files": ["Enabled"],
    "services": ["Enabled"]
  }'
```

**Console Method:**
1. Systems Manager → **Inventory** → **Setup Inventory**
2. Targets: **All managed instances**
3. Schedule: **Every 30 minutes**
4. Parameters (check all):
   - Applications
   - AWS Components
   - Network Configuration
   - Files
   - Services
5. **Setup Inventory**

**Exam Tip:** Inventory data is stored in S3 and can be queried using Amazon Athena for advanced analytics.

### Step 2.4: View Inventory Data

Wait 5-10 minutes for first inventory collection, then:

```bash
# Get inventory for a specific instance
export INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=SSM-Test-Instance-1" \
  --query 'Reservations[0].Instances[0].InstanceId' \
  --output text)

# View inventory entries
aws ssm list-inventory-entries \
  --instance-id $INSTANCE_ID \
  --type-name "AWS:Application" \
  --query 'Entries[0:5]' \
  --output table
```

**Console:** Systems Manager → **Inventory** → Click instance ID → View detailed inventory

### Step 2.5: Use SSM Run Command to Audit Packages

**Why:** Run Command executes scripts on managed instances without SSH.

#### List Installed Packages

```bash
# Run command on all instances
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Name,Values=SSM-Test-Instance-1" \
  --parameters 'commands=["rpm -qa | head -20"]' \
  --comment "Audit installed packages" \
  --output text \
  --query 'Command.CommandId'

# Save the command ID from output, then check results:
export COMMAND_ID=<command-id-from-above>

# Wait 10 seconds for command execution
sleep 10

# Get command output
aws ssm get-command-invocation \
  --command-id $COMMAND_ID \
  --instance-id $INSTANCE_ID \
  --query 'StandardOutputContent' \
  --output text
```

#### Check for Specific Security Package

```bash
# Check if fail2ban is installed (security best practice)
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=InstanceIds,Values=*" \
  --parameters 'commands=["rpm -qa | grep -i fail2ban || echo \"Not installed\""]' \
  --output json
```

**Console Method:**
1. Systems Manager → **Run Command** → **Run command**
2. Document: `AWS-RunShellScript`
3. Targets: Select instances or tags
4. Commands: `rpm -qa | head -20`
5. **Run**
6. View output in Command history

**Exam Tip:** Run Command vs Session Manager:
- **Run Command**: Execute one-off scripts/commands on multiple instances
- **Session Manager**: Interactive shell access (SSH replacement)

### Step 2.6: View SSM Compliance Dashboard

```bash
# View compliance summary
aws ssm list-compliance-summaries --query 'ComplianceSummaryItems[*]' --output table

# View compliance by resource
aws ssm list-resource-compliance-summaries \
  --filters "Key=ComplianceType,Values=Association" \
  --query 'ResourceComplianceSummaryItems[*].[ResourceId,ComplianceType,Status]' \
  --output table
```

**Console:** Systems Manager → **Compliance** → View dashboard

---

## Part 3: Creating Custom Config Rules

While AWS provides many managed rules, you may need custom rules for organization-specific requirements.

### Step 3.1: Create Lambda Function for Custom Rule

**Scenario:** Create a rule that checks if EC2 instances use approved AMI IDs.

#### Create Lambda Execution Role

```bash
# Create trust policy for Lambda
cat > /tmp/lambda-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create IAM role
aws iam create-role \
  --role-name ConfigRuleLambdaRole \
  --assume-role-policy-document file:///tmp/lambda-trust-policy.json

# Attach policies
aws iam attach-role-policy \
  --role-name ConfigRuleLambdaRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSConfigRulesExecutionRole

aws iam attach-role-policy \
  --role-name ConfigRuleLambdaRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Get role ARN
export LAMBDA_ROLE_ARN=$(aws iam get-role --role-name ConfigRuleLambdaRole --query 'Role.Arn' --output text)
echo "Lambda Role ARN: $LAMBDA_ROLE_ARN"
```

#### Create Lambda Function Code

```bash
# Create Lambda function code
cat > /tmp/check_approved_ami.py <<'EOF'
import json
import boto3

# Define approved AMI IDs (update with your AMIs)
APPROVED_AMIS = [
    "ami-0c55b159cbfafe1f0",  # Example: Amazon Linux 2
    "ami-0dc2d3e4c0f9ebd18",  # Example: Ubuntu
    # Add your approved AMIs here
]

def evaluate_compliance(configuration_item):
    """
    Evaluate if EC2 instance uses an approved AMI
    """
    if configuration_item['resourceType'] != 'AWS::EC2::Instance':
        return 'NOT_APPLICABLE'

    if configuration_item['configurationItemStatus'] == 'ResourceDeleted':
        return 'NOT_APPLICABLE'

    ami_id = configuration_item['configuration'].get('imageId')

    if ami_id in APPROVED_AMIS:
        return 'COMPLIANT'
    else:
        return 'NON_COMPLIANT'

def lambda_handler(event, context):
    """
    Main handler for Config rule evaluation
    """
    invoking_event = json.loads(event['invokingEvent'])
    configuration_item = invoking_event['configurationItem']

    compliance = evaluate_compliance(configuration_item)

    config_client = boto3.client('config')

    response = config_client.put_evaluations(
        Evaluations=[
            {
                'ComplianceResourceType': configuration_item['resourceType'],
                'ComplianceResourceId': configuration_item['resourceId'],
                'ComplianceType': compliance,
                'OrderingTimestamp': configuration_item['configurationItemCaptureTime']
            }
        ],
        ResultToken=event['resultToken']
    )

    return response
EOF

# Zip the function
cd /tmp
zip check_approved_ami.zip check_approved_ami.py
cd -
```

#### Deploy Lambda Function

```bash
# Wait for IAM role propagation
sleep 10

# Create Lambda function
aws lambda create-function \
  --function-name CheckApprovedAMI \
  --runtime python3.11 \
  --role $LAMBDA_ROLE_ARN \
  --handler check_approved_ami.lambda_handler \
  --zip-file fileb:///tmp/check_approved_ami.zip \
  --timeout 60 \
  --description "Config rule to check if EC2 instances use approved AMIs"

# Get Lambda function ARN
export LAMBDA_ARN=$(aws lambda get-function --function-name CheckApprovedAMI --query 'Configuration.FunctionArn' --output text)
echo "Lambda ARN: $LAMBDA_ARN"
```

#### Grant Config Permission to Invoke Lambda

```bash
aws lambda add-permission \
  --function-name CheckApprovedAMI \
  --statement-id AllowConfigInvoke \
  --action lambda:InvokeFunction \
  --principal config.amazonaws.com \
  --source-account $(aws sts get-caller-identity --query Account --output text)
```

### Step 3.2: Create Custom Config Rule

```bash
# Create custom Config rule
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "check-approved-ami",
  "Description": "Checks if EC2 instances use approved AMI IDs",
  "Source": {
    "Owner": "CUSTOM_LAMBDA",
    "SourceIdentifier": "'$LAMBDA_ARN'",
    "SourceDetails": [
      {
        "EventSource": "aws.config",
        "MessageType": "ConfigurationItemChangeNotification"
      }
    ]
  },
  "Scope": {
    "ComplianceResourceTypes": ["AWS::EC2::Instance"]
  }
}'

echo "Custom Config rule created. It will evaluate existing instances."
```

**Console Method:**
1. AWS Config → **Rules** → **Add rule**
2. **Create custom Lambda rule**
3. Lambda function ARN: Paste `$LAMBDA_ARN`
4. Trigger type: **Configuration changes**
5. Resource type: `AWS::EC2::Instance`
6. Rule name: `check-approved-ami`
7. **Save**

### Step 3.3: Trigger Rule Evaluation

```bash
# Manually trigger rule evaluation
aws configservice start-config-rules-evaluation \
  --config-rule-names check-approved-ami

# Wait 30 seconds for evaluation
sleep 30

# Check compliance results
aws configservice describe-compliance-by-config-rule \
  --config-rule-names check-approved-ami
```

**Exam Tip:** Lambda-backed rules offer flexibility but have higher latency and cost compared to managed rules. Use managed rules when available.

---

## Part 4: Remediation

Automatic remediation reduces manual intervention and speeds up compliance.

### Step 4.1: Create SSM Automation Document for Remediation

**Scenario:** Auto-stop EC2 instances that are non-compliant (e.g., missing required tags).

#### Create Custom Automation Document

```bash
cat > /tmp/stop-noncompliant-instance.json <<'EOF'
{
  "schemaVersion": "0.3",
  "description": "Stops EC2 instances that are non-compliant",
  "assumeRole": "{{ AutomationAssumeRole }}",
  "parameters": {
    "InstanceId": {
      "type": "String",
      "description": "EC2 Instance ID to stop"
    },
    "AutomationAssumeRole": {
      "type": "String",
      "description": "IAM role for automation"
    }
  },
  "mainSteps": [
    {
      "name": "StopInstance",
      "action": "aws:executeAwsApi",
      "inputs": {
        "Service": "ec2",
        "Api": "StopInstances",
        "InstanceIds": [
          "{{ InstanceId }}"
        ]
      },
      "description": "Stops the non-compliant EC2 instance"
    }
  ]
}
EOF

# Create automation document
aws ssm create-document \
  --name StopNonCompliantInstance \
  --document-type Automation \
  --content file:///tmp/stop-noncompliant-instance.json

echo "Automation document created: StopNonCompliantInstance"
```

**Console Method:**
1. Systems Manager → **Documents** → **Create document**
2. Name: `StopNonCompliantInstance`
3. Document type: **Automation**
4. Paste JSON content above
5. **Create document**

### Step 4.2: Create IAM Role for Remediation

```bash
# Create trust policy for SSM Automation
cat > /tmp/ssm-automation-trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "ssm.amazonaws.com",
          "config.amazonaws.com"
        ]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create IAM role
aws iam create-role \
  --role-name ConfigRemediationRole \
  --assume-role-policy-document file:///tmp/ssm-automation-trust.json

# Create permission policy
cat > /tmp/remediation-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:StopInstances",
        "ec2:DescribeInstances",
        "ec2:CreateTags"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetDocument",
        "ssm:DescribeDocument",
        "ssm:StartAutomationExecution"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name ConfigRemediationRole \
  --policy-name RemediationPolicy \
  --policy-document file:///tmp/remediation-policy.json

# Get role ARN
export REMEDIATION_ROLE_ARN=$(aws iam get-role --role-name ConfigRemediationRole --query 'Role.Arn' --output text)
echo "Remediation Role ARN: $REMEDIATION_ROLE_ARN"
```

### Step 4.3: Configure Automatic Remediation

**Example:** Auto-remediate instances missing required tags.

#### Create a Remediation Action for Required Tags Rule

```bash
# First, let's use a simpler remediation: Add required tags automatically
# Use AWS-managed document for tag remediation

aws configservice put-remediation-configurations \
  --remediation-configurations '[
    {
      "ConfigRuleName": "required-tags",
      "TargetType": "SSM_DOCUMENT",
      "TargetIdentifier": "AWS-CreateTags",
      "TargetVersion": "1",
      "Parameters": {
        "AutomationAssumeRole": {
          "StaticValue": {
            "Values": ["'$REMEDIATION_ROLE_ARN'"]
          }
        },
        "ResourceId": {
          "ResourceValue": {
            "Value": "RESOURCE_ID"
          }
        },
        "ResourceType": {
          "StaticValue": {
            "Values": ["AWS::EC2::Instance"]
          }
        },
        "Tags": {
          "StaticValue": {
            "Values": ["{\"Environment\":\"Untagged\",\"Owner\":\"AutoRemediated\"}"]
          }
        }
      },
      "Automatic": true,
      "MaximumAutomaticAttempts": 3,
      "RetryAttemptSeconds": 60
    }
  ]'

echo "Remediation configured for required-tags rule"
```

**Console Method:**
1. AWS Config → **Rules** → Select `required-tags`
2. **Actions** → **Manage remediation**
3. **Automatic remediation**
4. Remediation action: `AWS-CreateTags`
5. Parameters:
   - AutomationAssumeRole: `$REMEDIATION_ROLE_ARN`
   - ResourceId: **RESOURCE_ID** (dynamic)
   - ResourceType: `AWS::EC2::Instance`
   - Tags: `{"Environment":"Untagged","Owner":"AutoRemediated"}`
6. Auto remediation: **Enabled**
7. Retry attempts: 3
8. **Save changes**

### Step 4.4: Test Automatic Remediation

```bash
# Launch a new instance without required tags
export TEST_INSTANCE=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --iam-instance-profile Name=SSMManagedInstanceProfile \
  --security-group-ids $SG_ID \
  --subnet-id $SUBNET_ID \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Auto-Remediation-Test}]' \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "Test instance launched: $TEST_INSTANCE"
echo "Config will detect non-compliance and auto-remediate in 2-5 minutes"

# Monitor compliance status
watch -n 30 'aws configservice describe-compliance-by-resource \
  --resource-type AWS::EC2::Instance \
  --resource-id '$TEST_INSTANCE' \
  --query "ComplianceByResources[0].Compliance.ComplianceType"'
```

**Expected Flow:**
1. Instance launches without Environment/Owner tags → **NON_COMPLIANT**
2. Config rule detects non-compliance (2-3 minutes)
3. Remediation triggers automatically
4. SSM Automation adds default tags
5. Config re-evaluates → **COMPLIANT**

**Exam Tip:** Automatic remediation can be risky in production. Consider:
- Manual approval for critical resources
- Testing in dev/staging first
- Using SNS notifications before remediation
- Setting retry limits to avoid remediation loops

---

## Part 5: Verification & Review Questions

### Verify Your Setup

```bash
# Check Config recorder status
echo "=== Config Recorder Status ==="
aws configservice describe-configuration-recorder-status

# List all Config rules
echo -e "\n=== Config Rules ==="
aws configservice describe-config-rules --query 'ConfigRules[*].[ConfigRuleName,ConfigRuleState]' --output table

# Check SSM managed instances
echo -e "\n=== SSM Managed Instances ==="
aws ssm describe-instance-information --query 'InstanceInformationList[*].[InstanceId,PingStatus]' --output table

# View compliance summary
echo -e "\n=== Compliance Summary ==="
aws configservice describe-compliance-by-config-rule --query 'ComplianceByConfigRules[*].[ConfigRuleName,Compliance.ComplianceType]' --output table
```

### Review Questions

Test your knowledge with these scenario-based questions:

#### Question 1
**You need to ensure all EC2 instances in your organization are managed by Systems Manager. Which AWS Config rule would you use?**

<details>
<summary>Click to reveal answer</summary>

**Answer:** `EC2_INSTANCE_MANAGED_BY_SSM`

**Explanation:** This managed Config rule evaluates whether EC2 instances are managed by AWS Systems Manager. For an instance to be compliant:
- SSM Agent must be installed and running
- Instance must have an IAM instance profile with `AmazonSSMManagedInstanceCore` policy
- Instance must successfully connect to SSM service

**Why this matters for the exam:**
- Know the difference between SSM-managed vs. unmanaged instances
- Understand that SSM requires IAM permissions via instance profile
- Remember that VPC endpoints or internet access is required for SSM communication
</details>

#### Question 2
**Your company requires all production resources to have "Environment" and "CostCenter" tags. How would you enforce this with AWS Config, and what happens if a resource is created without these tags?**

<details>
<summary>Click to reveal answer</summary>

**Answer:** Use the `REQUIRED_TAGS` managed Config rule with parameters:
- `tag1Key`: `Environment`
- `tag2Key`: `CostCenter`

**What happens:**
1. Resource is created without required tags
2. Config evaluates the resource (within minutes)
3. Resource is marked as **NON_COMPLIANT**
4. If automatic remediation is configured:
   - Config triggers an SSM Automation document
   - Document adds default tags or notifies admins
   - Resource becomes compliant
5. If manual remediation:
   - CloudOps team receives SNS notification
   - Team manually adds tags

**Exam Tips:**
- Config evaluates resources AFTER creation (not before)
- To prevent creation without tags, use:
  - AWS Service Control Policies (SCPs)
  - IAM permissions with tag conditions
  - CloudFormation/Terraform with tag requirements
- Config is for detection and remediation, not prevention
</details>

#### Question 3
**You notice that some EC2 instances don't appear in Systems Manager Fleet Manager. What are three possible reasons?**

<details>
<summary>Click to reveal answer</summary>

**Answer:** Three common reasons:

1. **Missing IAM Instance Profile**
   - Instance doesn't have an IAM role attached
   - Role lacks `AmazonSSMManagedInstanceCore` policy
   - Solution: Attach instance profile with correct policy

2. **Network Connectivity Issues**
   - No internet access (no IGW or NAT Gateway)
   - No VPC endpoints for SSM services
   - Security group blocks outbound HTTPS (port 443)
   - Solution: Add NAT Gateway OR create VPC endpoints for:
     - `com.amazonaws.<region>.ssm`
     - `com.amazonaws.<region>.ec2messages`
     - `com.amazonaws.<region>.ssmmessages`

3. **SSM Agent Not Running**
   - Agent not installed (older AMIs)
   - Agent crashed or stopped
   - Solution: Install/restart agent:
     ```bash
     sudo systemctl start amazon-ssm-agent
     sudo systemctl enable amazon-ssm-agent
     ```

**Exam Tip:** Know the three SSM VPC endpoints by heart. Also remember that Amazon Linux 2, Amazon Linux 2023, Ubuntu, and Windows Server 2016+ have SSM Agent pre-installed.
</details>

#### Question 4
**What is the difference between AWS Config Rules and AWS Systems Manager Compliance?**

<details>
<summary>Click to reveal answer</summary>

**Answer:**

| Aspect | AWS Config Rules | SSM Compliance |
|--------|------------------|----------------|
| **Scope** | All AWS resources (EC2, S3, IAM, etc.) | Only SSM-managed instances |
| **Evaluation** | Configuration state compliance | Patch status, association status |
| **Use Case** | Resource configuration auditing | Instance-level operational compliance |
| **Example** | "Is security group open to 0.0.0.0/0?" | "Is instance patched?" |
| **Remediation** | SSM Automation, Lambda | SSM Run Command, Patch Manager |

**Combined Usage:**
- Config rule: Check if instances are SSM-managed
- SSM Compliance: Check patch compliance on those instances
- Together: Complete visibility into instance security posture

**Exam Tip:** Config is broader (all resources), SSM Compliance is deeper (instance details).
</details>

#### Question 5
**You've configured automatic remediation for a Config rule, but it keeps running repeatedly every few minutes. What's likely happening, and how do you fix it?**

<details>
<summary>Click to reveal answer</summary>

**Answer:** This is a **remediation loop**.

**Root Cause:**
The remediation action doesn't actually fix the compliance issue, so:
1. Config detects non-compliance
2. Triggers remediation
3. Remediation runs but doesn't fix root cause
4. Config re-evaluates → still non-compliant
5. Triggers remediation again → loop

**Common Examples:**
- Remediation tries to add tags but lacks IAM permissions
- Automation document has wrong parameters
- Resource is recreated by another process (e.g., Auto Scaling)

**Solutions:**
1. **Fix the remediation action:**
   - Verify IAM permissions are sufficient
   - Test automation document manually first
   - Check document parameters are correct

2. **Set retry limits:**
   - `MaximumAutomaticAttempts`: 3 (prevents infinite loops)
   - `RetryAttemptSeconds`: 60 (adds delay between attempts)

3. **Monitor remediation:**
   - CloudWatch Logs for SSM Automation executions
   - Config timeline for remediation attempts
   - Set up CloudWatch Alarms for failed remediations

**Exam Tip:** Always set `MaximumAutomaticAttempts` to prevent runaway remediation costs.
</details>

#### Question 6
**Your manager wants to query AWS Config data to find all EC2 instances created in the last 30 days that don't have backups enabled. How would you do this efficiently?**

<details>
<summary>Click to reveal answer</summary>

**Answer:** Use AWS Config **Advanced Queries**.

**Steps:**

1. Navigate to AWS Config → **Advanced queries**

2. Use this SQL-like query:
```sql
SELECT
  resourceId,
  resourceType,
  configuration.instanceId,
  configuration.tags,
  configurationItemCaptureTime
WHERE
  resourceType = 'AWS::EC2::Instance'
  AND configurationItemCaptureTime > '2026-02-22'
  AND configuration.tags.Backup NOT EXISTS
```

3. Export results to CSV or process with Lambda

**Alternative Methods:**

**Using AWS CLI:**
```bash
aws configservice select-resource-config \
  --expression "SELECT resourceId, configuration.instanceId WHERE resourceType = 'AWS::EC2::Instance' AND configurationItemCaptureTime > '2026-02-22'"
```

**Using Athena:**
- Config can export to S3
- Use Athena to query S3 data
- More cost-effective for large datasets

**Exam Tips:**
- Config Advanced Queries use SQL-like syntax
- Can query across regions and accounts (with Config aggregator)
- Free tier: 1,000 queries/month
- After: $0.001 per query
- Results return max 100 resources per query (paginate for more)
</details>

#### Question 7
**You want to be notified when a Config rule becomes non-compliant, but you don't want automatic remediation. What's the best approach?**

<details>
<summary>Click to reveal answer</summary>

**Answer:** Use Amazon EventBridge (formerly CloudWatch Events) with SNS.

**Implementation:**

1. **Create SNS Topic:**
```bash
aws sns create-topic --name ConfigComplianceAlerts
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:ACCOUNT-ID:ConfigComplianceAlerts \
  --protocol email \
  --notification-endpoint your-email@example.com
```

2. **Create EventBridge Rule:**
```bash
aws events put-rule \
  --name ConfigNonCompliantAlert \
  --event-pattern '{
    "source": ["aws.config"],
    "detail-type": ["Config Rules Compliance Change"],
    "detail": {
      "newEvaluationResult": {
        "complianceType": ["NON_COMPLIANT"]
      }
    }
  }'

aws events put-targets \
  --rule ConfigNonCompliantAlert \
  --targets "Id"="1","Arn"="arn:aws:sns:us-east-1:ACCOUNT-ID:ConfigComplianceAlerts"
```

3. **Grant EventBridge permission to publish to SNS:**
```bash
aws sns set-topic-attributes \
  --topic-arn arn:aws:sns:us-east-1:ACCOUNT-ID:ConfigComplianceAlerts \
  --attribute-name Policy \
  --attribute-value '{
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "events.amazonaws.com"},
      "Action": "SNS:Publish",
      "Resource": "arn:aws:sns:us-east-1:ACCOUNT-ID:ConfigComplianceAlerts"
    }]
  }'
```

**Result:** When any Config rule changes to NON_COMPLIANT, you receive an email notification.

**Advanced:** Filter for specific rules:
```json
{
  "source": ["aws.config"],
  "detail-type": ["Config Rules Compliance Change"],
  "detail": {
    "configRuleName": ["required-tags", "restricted-ssh"],
    "newEvaluationResult": {
      "complianceType": ["NON_COMPLIANT"]
    }
  }
}
```

**Exam Tip:** EventBridge can trigger:
- SNS (notifications)
- Lambda (custom remediation)
- SQS (queue for processing)
- Step Functions (complex workflows)
- SSM Automation (built-in remediation)
</details>

#### Question 8
**What's the difference between AWS Config's configuration snapshots and configuration streams?**

<details>
<summary>Click to reveal answer</summary>

**Answer:**

**Configuration Snapshot:**
- Point-in-time view of all resources
- Delivered to S3 bucket
- Scheduled delivery (e.g., every 6 hours)
- Use case: Periodic compliance reporting, disaster recovery
- Format: JSON file with all resource configurations
- Access: S3 API, Athena queries

**Configuration Stream:**
- Real-time change notifications
- Delivered to SNS topic
- Triggered on any configuration change
- Use case: Immediate alerting, event-driven automation
- Format: Individual change events
- Access: SNS subscribers (Lambda, SQS, email)

**Example Scenario:**
```
Timeline:
10:00 AM - EC2 instance launched
10:05 AM - Security group modified
10:10 AM - S3 bucket created
12:00 PM - Snapshot scheduled

Configuration Stream:
  ↓ Event at 10:00 AM (EC2 created)
  ↓ Event at 10:05 AM (SG changed)
  ↓ Event at 10:10 AM (S3 created)

Configuration Snapshot:
  ↓ Snapshot at 12:00 PM (all 3 resources)
```

**Exam Tip:**
- Stream = real-time, individual changes
- Snapshot = periodic, all resources
- You can (and should) enable both
</details>

#### Question 9
**You need to audit AWS resources across multiple AWS accounts in your organization. How can you centralize Config data?**

<details>
<summary>Click to reveal answer</summary>

**Answer:** Use **AWS Config Aggregator**.

**Setup Steps:**

1. **Enable AWS Organizations (if not already enabled)**
   - Designate one account as the management account

2. **Create Config Aggregator in central account:**
```bash
aws configservice put-configuration-aggregator \
  --configuration-aggregator-name OrganizationAggregator \
  --organization-aggregation-source '{
    "RoleArn": "arn:aws:iam::ACCOUNT-ID:role/aws-service-role/organizations.amazonaws.com/AWSServiceRoleForOrganizations",
    "AllAwsRegions": true
  }'
```

**Console Method:**
1. AWS Config → **Aggregators** → **Add aggregator**
2. Name: `OrganizationAggregator`
3. Source type: **Add all accounts from AWS Organizations**
4. Regions: **All regions**
5. **Save**

3. **Enable Config in all member accounts** (if not already)

4. **Query aggregated data:**
```bash
aws configservice select-aggregate-resource-config \
  --configuration-aggregator-name OrganizationAggregator \
  --expression "SELECT resourceId, accountId WHERE resourceType = 'AWS::EC2::Instance'"
```

**Benefits:**
- Single dashboard for all accounts
- Cross-account compliance reporting
- Centralized security auditing
- No need to switch between accounts

**Exam Tips:**
- Aggregator can source from:
  - AWS Organizations (all accounts automatically)
  - Individual accounts (manual authorization needed)
- Data stays in source accounts (aggregator just queries)
- No additional cost for aggregator
- Max 50 aggregators per account
</details>

#### Question 10
**Your organization has a compliance requirement to retain all Config data for 7 years. Config only retains data for 7 years by default. How do you ensure compliance?**

<details>
<summary>Click to reveal answer</summary>

**Answer:** AWS Config automatically retains data for 7 years (recently updated from 7 years minimum), but you should implement additional safeguards:

**Best Practices for Long-Term Retention:**

1. **S3 Bucket Configuration:**
```bash
# Enable versioning (prevents accidental deletion)
aws s3api put-bucket-versioning \
  --bucket $CONFIG_BUCKET \
  --versioning-configuration Status=Enabled

# Add lifecycle policy for glacier transition
cat > /tmp/lifecycle-policy.json <<EOF
{
  "Rules": [
    {
      "Id": "ArchiveConfigData",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        },
        {
          "Days": 365,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ],
      "NoncurrentVersionTransitions": [
        {
          "NoncurrentDays": 30,
          "StorageClass": "GLACIER"
        }
      ]
    }
  ]
}
EOF

aws s3api put-bucket-lifecycle-configuration \
  --bucket $CONFIG_BUCKET \
  --lifecycle-configuration file:///tmp/lifecycle-policy.json

# Enable MFA Delete (prevents deletion without MFA)
aws s3api put-bucket-versioning \
  --bucket $CONFIG_BUCKET \
  --versioning-configuration Status=Enabled,MFADelete=Enabled \
  --mfa "SERIAL_NUMBER TOKEN"
```

2. **S3 Object Lock (Compliance Mode):**
```bash
# Enable Object Lock on new bucket
aws s3api create-bucket \
  --bucket $CONFIG_BUCKET-locked \
  --object-lock-enabled-for-bucket \
  --region $AWS_REGION

# Set default retention
aws s3api put-object-lock-configuration \
  --bucket $CONFIG_BUCKET-locked \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {
      "DefaultRetention": {
        "Mode": "COMPLIANCE",
        "Years": 7
      }
    }
  }'
```

**Compliance Mode vs. Governance Mode:**
- **Compliance:** Cannot be deleted by anyone (including root)
- **Governance:** Can be deleted with special permissions

3. **S3 Bucket Policy (Deny Delete):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": [
        "s3:DeleteBucket",
        "s3:DeleteObject",
        "s3:DeleteObjectVersion"
      ],
      "Resource": [
        "arn:aws:s3:::CONFIG_BUCKET",
        "arn:aws:s3:::CONFIG_BUCKET/*"
      ]
    }
  ]
}
```

4. **Enable CloudTrail for S3 Data Events:**
```bash
aws cloudtrail put-event-selectors \
  --trail-name myConfigTrail \
  --event-selectors '[{
    "ReadWriteType": "All",
    "IncludeManagementEvents": true,
    "DataResources": [{
      "Type": "AWS::S3::Object",
      "Values": ["arn:aws:s3:::'$CONFIG_BUCKET'/*"]
    }]
  }]'
```

**Cost Optimization:**
- Days 0-90: S3 Standard ($0.023/GB)
- Days 91-365: Glacier ($0.004/GB)
- Year 2-7: Deep Archive ($0.00099/GB)

**Exam Tips:**
- Config retention is 7 years minimum
- S3 retention can be longer with lifecycle policies
- Object Lock in Compliance mode is **immutable** (perfect for regulatory requirements)
- Always enable versioning and MFA Delete for compliance data
</details>

---

## Cleanup Steps

**Important:** Execute these steps to avoid ongoing charges.

### Step 1: Stop/Terminate EC2 Instances

```bash
# Get all test instance IDs
INSTANCE_IDS=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=SSM-Test-Instance-*,Auto-Remediation-Test" \
  --query 'Reservations[*].Instances[*].InstanceId' \
  --output text)

# Terminate instances
if [ ! -z "$INSTANCE_IDS" ]; then
  aws ec2 terminate-instances --instance-ids $INSTANCE_IDS
  echo "Instances terminated: $INSTANCE_IDS"
fi

# Delete security group (wait for instances to terminate first)
sleep 60
aws ec2 delete-security-group --group-id $SG_ID
```

### Step 2: Delete AWS Config Resources

```bash
# Stop Config recorder
aws configservice stop-configuration-recorder --configuration-recorder-name default

# Delete Config rules
aws configservice delete-config-rule --config-rule-name ec2-instance-managed-by-ssm
aws configservice delete-config-rule --config-rule-name required-tags
aws configservice delete-config-rule --config-rule-name restricted-ssh
aws configservice delete-config-rule --config-rule-name check-approved-ami

# Delete remediation configurations
aws configservice delete-remediation-configuration \
  --config-rule-name required-tags

# Delete delivery channel
aws configservice delete-delivery-channel --delivery-channel-name default

# Delete configuration recorder
aws configservice delete-configuration-recorder --configuration-recorder-name default
```

### Step 3: Delete Systems Manager Resources

```bash
# Delete SSM associations
ASSOCIATION_IDS=$(aws ssm list-associations \
  --query 'Associations[*].AssociationId' \
  --output text)

for assoc_id in $ASSOCIATION_IDS; do
  aws ssm delete-association --association-id $assoc_id
done

# Delete SSM automation document
aws ssm delete-document --name StopNonCompliantInstance
```

### Step 4: Delete Lambda Function

```bash
# Delete Lambda function
aws lambda delete-function --function-name CheckApprovedAMI
```

### Step 5: Delete IAM Roles

```bash
# Detach policies and delete Config role
aws iam delete-role-policy --role-name AWSConfigRole --policy-name ConfigS3Access
aws iam detach-role-policy --role-name AWSConfigRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/ConfigRole
aws iam delete-role --role-name AWSConfigRole

# Delete SSM instance profile
aws iam remove-role-from-instance-profile \
  --instance-profile-name SSMManagedInstanceProfile \
  --role-name SSMManagedInstanceRole
aws iam delete-instance-profile --instance-profile-name SSMManagedInstanceProfile
aws iam detach-role-policy --role-name SSMManagedInstanceRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam delete-role --role-name SSMManagedInstanceRole

# Delete Lambda role
aws iam detach-role-policy --role-name ConfigRuleLambdaRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSConfigRulesExecutionRole
aws iam detach-role-policy --role-name ConfigRuleLambdaRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name ConfigRuleLambdaRole

# Delete remediation role
aws iam delete-role-policy --role-name ConfigRemediationRole --policy-name RemediationPolicy
aws iam delete-role --role-name ConfigRemediationRole
```

### Step 6: Empty and Delete S3 Bucket

```bash
# Empty bucket (delete all objects)
aws s3 rm s3://$CONFIG_BUCKET --recursive

# Delete bucket
aws s3 rb s3://$CONFIG_BUCKET

echo "Cleanup complete!"
```

### Step 7: Verify Cleanup

```bash
# Verify Config stopped
aws configservice describe-configuration-recorder-status

# Verify instances terminated
aws ec2 describe-instances --filters "Name=tag:Name,Values=SSM-Test-*"

# Verify S3 bucket deleted
aws s3 ls | grep config-bucket
```

**Console Verification:**
- AWS Config: Recorder should show "Not Recording"
- EC2: No test instances running
- S3: Config bucket deleted
- Lambda: CheckApprovedAMI deleted
- IAM: All test roles deleted

---

## Additional Resources

### AWS Documentation
- [AWS Config Developer Guide](https://docs.aws.amazon.com/config/latest/developerguide/)
- [AWS Systems Manager User Guide](https://docs.aws.amazon.com/systems-manager/latest/userguide/)
- [Config Managed Rules List](https://docs.aws.amazon.com/config/latest/developerguide/managed-rules-by-aws-config.html)
- [SSM Automation Document Reference](https://docs.aws.amazon.com/systems-manager/latest/userguide/automation-actions.html)

### Best Practices
- Enable Config in all regions where you have resources
- Use Config Aggregator for multi-account setups
- Start with managed rules before creating custom rules
- Test remediation actions in dev/staging first
- Set up SNS alerts for critical compliance failures
- Review compliance dashboards weekly
- Archive Config data to Glacier for cost savings

### Estimated Costs (Post Free Tier)
- **AWS Config:**
  - Configuration items: $0.003 per item recorded
  - Rule evaluations: $0.001 per evaluation
  - Example: 100 resources × 10 rules = ~$3/month
- **Systems Manager:** Free (except Run Command with advanced parameters)
- **S3 Storage:** $0.023/GB/month (S3 Standard)
- **Lambda:** Free tier covers most usage
- **EC2:** t2.micro ~$8.40/month (if not Free Tier)

### Exam Study Tips
1. **Understand the difference:** Config (configuration compliance) vs. CloudTrail (API auditing) vs. SSM (operational management)
2. **Know managed rules:** Memorize 10-15 common managed rules
3. **Remediation flow:** Config detects → EventBridge triggers → SSM Automation remediates
4. **IAM permissions:** Config needs broad read access + S3 write; SSM needs EC2/SSM actions
5. **Practice scenarios:** "How would you detect/remediate X?" questions are common

---

## Congratulations!

You've completed Lab 1! You now know how to:
- Set up AWS Config to track resource configurations
- Create and manage Config rules for compliance monitoring
- Use Systems Manager for instance inventory and auditing
- Implement custom Lambda-backed Config rules
- Configure automatic remediation with SSM Automation
- Analyze compliance across your AWS environment

**Next Steps:**
- Lab 2: CloudWatch Metrics, Alarms, and Dashboards
- Lab 3: AWS Backup and Disaster Recovery
- Lab 4: VPC Flow Logs and Network Monitoring

**Practice Exam Questions:** Review the 10 questions above multiple times to reinforce concepts.

---

**Lab Version:** 1.0
**Last Updated:** March 2026
**Exam Alignment:** AWS Certified CloudOps Engineer - Associate (SOA-C03)
**Difficulty Level:** Intermediate

---

## Feedback

Found an issue or have suggestions? This lab is designed to help you pass the SOA-C03 exam. Practice these steps multiple times until you can execute them from memory.

**Key Takeaway:** Config and SSM work together to provide complete visibility and control over your AWS resources. Config tells you *what changed*, SSM tells you *what's running*, and together they enable *automated compliance*.
