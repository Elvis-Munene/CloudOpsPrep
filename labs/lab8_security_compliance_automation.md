# Lab 8: Security and Compliance Automation

**Domain:** Domain 4 - Security and Compliance
**Estimated Time:** 1 hour 30 minutes
**Difficulty:** Intermediate to Advanced

## Objectives

By the end of this lab, you will be able to:
- Implement and evaluate IAM policies with permission boundaries
- Configure encryption using AWS KMS across multiple services
- Manage secrets using AWS Secrets Manager and SSM Parameter Store
- Enable and configure security monitoring services (GuardDuty, Security Hub, Inspector)
- Implement Service Control Policies (SCPs) in AWS Organizations
- Automate security compliance checks and remediation

## Prerequisites

- AWS CLI configured with appropriate credentials
- AWS account with administrative access
- jq installed for JSON parsing (`sudo apt-get install jq` or `brew install jq`)
- Basic understanding of IAM, encryption concepts, and AWS Organizations

---

## Part 1: IAM Deep Dive

### 1.1 Create Custom IAM Policies

**Identity-Based Policy Example:**

```bash
# Create a policy that allows read-only access to S3 with conditions
cat > s3-readonly-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3ReadOnlyAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-company-data-*",
        "arn:aws:s3:::my-company-data-*/*"
      ],
      "Condition": {
        "StringEquals": {
          "s3:ExistingObjectTag/Environment": "Production"
        }
      }
    }
  ]
}
EOF

# Create the policy
aws iam create-policy \
  --policy-name S3ReadOnlyWithConditions \
  --policy-document file://s3-readonly-policy.json \
  --description "S3 read-only access with tag-based conditions"
```

**Resource-Based Policy Example:**

```bash
# Create an S3 bucket with a resource-based policy
BUCKET_NAME="security-lab-bucket-$(date +%s)"
aws s3api create-bucket --bucket $BUCKET_NAME --region us-east-1

# Apply bucket policy (resource-based)
cat > bucket-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSpecificAccountAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::$BUCKET_NAME/*"
    }
  ]
}
EOF

aws s3api put-bucket-policy --bucket $BUCKET_NAME --policy file://bucket-policy.json
```

### 1.2 Permission Boundaries

**Exam Concept:** Permission boundaries set the maximum permissions an entity can have, even if the identity-based policy grants more.

```bash
# Create a permission boundary policy
cat > permission-boundary.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ServiceBoundary",
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "ec2:Describe*",
        "cloudwatch:*"
      ],
      "Resource": "*"
    }
  ]
}
EOF

# Create the boundary policy
BOUNDARY_ARN=$(aws iam create-policy \
  --policy-name DeveloperBoundary \
  --policy-document file://permission-boundary.json \
  --query 'Policy.Arn' \
  --output text)

echo "Permission Boundary ARN: $BOUNDARY_ARN"

# Create a user with this permission boundary
aws iam create-user --user-name developer-user

aws iam attach-user-policy \
  --user-name developer-user \
  --policy-arn arn:aws:iam::aws:policy/PowerUserAccess

# Apply the permission boundary
aws iam put-user-permissions-boundary \
  --user-name developer-user \
  --permissions-boundary $BOUNDARY_ARN

# Even though the user has PowerUserAccess, they can only use S3, EC2 Describe, and CloudWatch
```

**Policy Evaluation Logic:**
1. **Explicit Deny** - Always wins (SCP, permission boundary, identity policy, resource policy)
2. **Explicit Allow** - Must exist in identity-based policy AND not denied by boundaries
3. **Implicit Deny** - Default if no explicit allow

### 1.3 IAM Access Analyzer

```bash
# Create an IAM Access Analyzer
ANALYZER_NAME="security-lab-analyzer"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws accessanalyzer create-analyzer \
  --analyzer-name $ANALYZER_NAME \
  --type ACCOUNT \
  --tags Key=Purpose,Value=SecurityLab

# List analyzers
aws accessanalyzer list-analyzers

# Wait for findings (this takes a few minutes)
echo "Waiting for Access Analyzer to generate findings..."
sleep 60

# List findings (external access)
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:access-analyzer:us-east-1:$ACCOUNT_ID:analyzer/$ANALYZER_NAME

# Get detailed finding
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:access-analyzer:us-east-1:$ACCOUNT_ID:analyzer/$ANALYZER_NAME \
  --query 'findings[0]' \
  --output json
```

### 1.4 Cross-Account Access with IAM Roles

```bash
# Create a cross-account access role (to be assumed by another account)
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111122223333:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "unique-external-id-12345"
        }
      }
    }
  ]
}
EOF

aws iam create-role \
  --role-name CrossAccountS3ReadRole \
  --assume-role-policy-document file://trust-policy.json \
  --description "Role for cross-account S3 read access"

# Attach permissions to the role
aws iam attach-role-policy \
  --role-name CrossAccountS3ReadRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# To assume this role from the trusted account:
# aws sts assume-role --role-arn arn:aws:iam::ACCOUNT_ID:role/CrossAccountS3ReadRole \
#   --role-session-name cross-account-session --external-id unique-external-id-12345
```

### 1.5 IAM Credential Report and Access Advisor

```bash
# Generate credential report
aws iam generate-credential-report

# Wait for report generation
sleep 10

# Get credential report
aws iam get-credential-report --query Content --output text | base64 -d > credential-report.csv

echo "Credential report saved to credential-report.csv"
cat credential-report.csv

# Get last accessed information for a user
aws iam generate-service-last-accessed-details \
  --arn arn:aws:iam::$ACCOUNT_ID:user/developer-user \
  --query JobId \
  --output text > job-id.txt

JOB_ID=$(cat job-id.txt)

# Wait for job completion
sleep 10

# Get service last accessed details
aws iam get-service-last-accessed-details --job-id $JOB_ID
```

---

## Part 2: Encryption & Key Management

### 2.1 Create KMS Keys (Symmetric vs Asymmetric)

**Symmetric KMS Key:**

```bash
# Create a symmetric KMS key (default)
KMS_KEY_ID=$(aws kms create-key \
  --description "Symmetric key for data encryption" \
  --key-usage ENCRYPT_DECRYPT \
  --origin AWS_KMS \
  --tags TagKey=Purpose,TagValue=SecurityLab \
  --query 'KeyMetadata.KeyId' \
  --output text)

echo "Symmetric KMS Key ID: $KMS_KEY_ID"

# Create an alias
aws kms create-alias \
  --alias-name alias/security-lab-symmetric \
  --target-key-id $KMS_KEY_ID

# Test encryption
echo "Secret data for encryption" > plaintext.txt

aws kms encrypt \
  --key-id $KMS_KEY_ID \
  --plaintext fileb://plaintext.txt \
  --query CiphertextBlob \
  --output text > encrypted.txt

# Test decryption
aws kms decrypt \
  --ciphertext-blob fileb://<(cat encrypted.txt | base64 -d) \
  --query Plaintext \
  --output text | base64 -d
```

**Asymmetric KMS Key:**

```bash
# Create an asymmetric key (for signing)
ASYMMETRIC_KEY_ID=$(aws kms create-key \
  --description "Asymmetric key for digital signatures" \
  --key-usage SIGN_VERIFY \
  --key-spec RSA_2048 \
  --tags TagKey=Purpose,TagValue=DigitalSignature \
  --query 'KeyMetadata.KeyId' \
  --output text)

echo "Asymmetric KMS Key ID: $ASYMMETRIC_KEY_ID"

aws kms create-alias \
  --alias-name alias/security-lab-asymmetric \
  --target-key-id $ASYMMETRIC_KEY_ID
```

### 2.2 Key Policies and Grants

```bash
# Get the current key policy
aws kms get-key-policy \
  --key-id $KMS_KEY_ID \
  --policy-name default \
  --query Policy \
  --output text | jq '.' > key-policy.json

# Create a more restrictive key policy
cat > new-key-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::$ACCOUNT_ID:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "Allow use of the key for specific users",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::$ACCOUNT_ID:user/developer-user"
      },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws kms put-key-policy \
  --key-id $KMS_KEY_ID \
  --policy-name default \
  --policy file://new-key-policy.json

# Create a grant (temporary delegation)
GRANT_TOKEN=$(aws kms create-grant \
  --key-id $KMS_KEY_ID \
  --grantee-principal arn:aws:iam::$ACCOUNT_ID:role/CrossAccountS3ReadRole \
  --operations Encrypt Decrypt DescribeKey \
  --query GrantToken \
  --output text)

echo "Grant Token: $GRANT_TOKEN"

# List grants
aws kms list-grants --key-id $KMS_KEY_ID
```

### 2.3 Key Rotation

```bash
# Enable automatic key rotation (annual rotation)
aws kms enable-key-rotation --key-id $KMS_KEY_ID

# Check rotation status
aws kms get-key-rotation-status --key-id $KMS_KEY_ID

# Note: Manual rotation requires creating a new key and updating aliases
```

### 2.4 Encrypt EBS Volumes, S3 Buckets, RDS Instances

**EBS Volume Encryption:**

```bash
# Create an encrypted EBS volume
VOLUME_ID=$(aws ec2 create-volume \
  --availability-zone us-east-1a \
  --size 10 \
  --volume-type gp3 \
  --encrypted \
  --kms-key-id $KMS_KEY_ID \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=EncryptedVolume}]' \
  --query VolumeId \
  --output text)

echo "Encrypted EBS Volume ID: $VOLUME_ID"

# Verify encryption
aws ec2 describe-volumes --volume-ids $VOLUME_ID --query 'Volumes[0].Encrypted'
```

**S3 Bucket Encryption:**

```bash
# SSE-S3 (Server-Side Encryption with S3-managed keys)
aws s3api put-bucket-encryption \
  --bucket $BUCKET_NAME \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      },
      "BucketKeyEnabled": false
    }]
  }'

# SSE-KMS (Server-Side Encryption with KMS)
ENCRYPTION_BUCKET="encryption-test-bucket-$(date +%s)"
aws s3api create-bucket --bucket $ENCRYPTION_BUCKET --region us-east-1

aws s3api put-bucket-encryption \
  --bucket $ENCRYPTION_BUCKET \
  --server-side-encryption-configuration "{
    \"Rules\": [{
      \"ApplyServerSideEncryptionByDefault\": {
        \"SSEAlgorithm\": \"aws:kms\",
        \"KMSMasterKeyID\": \"$KMS_KEY_ID\"
      },
      \"BucketKeyEnabled\": true
    }]
  }"

# Verify encryption configuration
aws s3api get-bucket-encryption --bucket $ENCRYPTION_BUCKET
```

**RDS Instance Encryption:**

```bash
# Create an encrypted RDS instance
DB_INSTANCE_ID="encrypted-db-instance"

aws rds create-db-instance \
  --db-instance-identifier $DB_INSTANCE_ID \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --master-username admin \
  --master-user-password MySecurePass123! \
  --allocated-storage 20 \
  --storage-encrypted \
  --kms-key-id $KMS_KEY_ID \
  --backup-retention-period 7 \
  --no-publicly-accessible

echo "RDS instance $DB_INSTANCE_ID is being created with encryption enabled"

# Check encryption status
aws rds describe-db-instances \
  --db-instance-identifier $DB_INSTANCE_ID \
  --query 'DBInstances[0].[StorageEncrypted,KmsKeyId]' \
  --output table
```

**SSE Comparison Table:**

| Feature | SSE-S3 | SSE-KMS | SSE-C |
|---------|--------|---------|-------|
| Key Management | AWS managed | Customer managed in KMS | Customer managed externally |
| Key Rotation | Automatic | Automatic (if enabled) | Manual |
| Audit Trail | Limited | CloudTrail logs | None |
| Additional Cost | No | Yes (KMS API calls) | No |
| Use Case | Simple encryption | Compliance, audit requirements | Full customer control |

### 2.5 AWS Certificate Manager (ACM)

```bash
# Request a public certificate
DOMAIN_NAME="example.com"

CERT_ARN=$(aws acm request-certificate \
  --domain-name $DOMAIN_NAME \
  --subject-alternative-names "*.$DOMAIN_NAME" \
  --validation-method DNS \
  --query CertificateArn \
  --output text)

echo "Certificate ARN: $CERT_ARN"

# Get certificate details
aws acm describe-certificate --certificate-arn $CERT_ARN

# List certificates
aws acm list-certificates --certificate-statuses ISSUED PENDING_VALIDATION

# Import a certificate (if you have your own)
# aws acm import-certificate \
#   --certificate fileb://certificate.pem \
#   --private-key fileb://private-key.pem \
#   --certificate-chain fileb://certificate-chain.pem
```

---

## Part 3: Secrets Management

### 3.1 AWS Secrets Manager - Store and Rotate RDS Credentials

```bash
# Create a secret for RDS credentials
SECRET_NAME="rds-mysql-credentials"

SECRET_ARN=$(aws secretsmanager create-secret \
  --name $SECRET_NAME \
  --description "MySQL RDS database credentials" \
  --secret-string '{
    "username": "admin",
    "password": "MySecurePass123!",
    "engine": "mysql",
    "host": "encrypted-db-instance.abc123.us-east-1.rds.amazonaws.com",
    "port": 3306,
    "dbname": "mydb"
  }' \
  --kms-key-id $KMS_KEY_ID \
  --tags Key=Environment,Value=Production \
  --query ARN \
  --output text)

echo "Secret ARN: $SECRET_ARN"

# Retrieve the secret
aws secretsmanager get-secret-value --secret-id $SECRET_NAME --query SecretString --output text | jq '.'

# Update secret value
aws secretsmanager update-secret \
  --secret-id $SECRET_NAME \
  --secret-string '{
    "username": "admin",
    "password": "NewSecurePass456!",
    "engine": "mysql",
    "host": "encrypted-db-instance.abc123.us-east-1.rds.amazonaws.com",
    "port": 3306,
    "dbname": "mydb"
  }'

# Enable automatic rotation (requires Lambda function)
# aws secretsmanager rotate-secret \
#   --secret-id $SECRET_NAME \
#   --rotation-lambda-arn arn:aws:lambda:us-east-1:123456789012:function:SecretsManagerRotation \
#   --rotation-rules AutomaticallyAfterDays=30
```

### 3.2 SSM Parameter Store - SecureString Parameters

```bash
# Create a SecureString parameter
aws ssm put-parameter \
  --name "/myapp/database/password" \
  --description "Database password for MyApp" \
  --value "MySecurePassword123!" \
  --type SecureString \
  --key-id $KMS_KEY_ID \
  --tags Key=Application,Value=MyApp Key=Environment,Value=Production

# Create a standard parameter
aws ssm put-parameter \
  --name "/myapp/database/host" \
  --description "Database host for MyApp" \
  --value "db.example.com" \
  --type String

# Retrieve parameter
aws ssm get-parameter \
  --name "/myapp/database/password" \
  --with-decryption \
  --query Parameter.Value \
  --output text

# Get multiple parameters
aws ssm get-parameters \
  --names "/myapp/database/password" "/myapp/database/host" \
  --with-decryption

# List parameters by path
aws ssm get-parameters-by-path \
  --path "/myapp" \
  --recursive \
  --with-decryption

# Update parameter
aws ssm put-parameter \
  --name "/myapp/database/password" \
  --value "NewPassword456!" \
  --type SecureString \
  --key-id $KMS_KEY_ID \
  --overwrite

# Get parameter history
aws ssm get-parameter-history --name "/myapp/database/password"
```

### 3.3 Secrets Manager vs Parameter Store Comparison

| Feature | Secrets Manager | Parameter Store (Standard) | Parameter Store (Advanced) |
|---------|----------------|---------------------------|---------------------------|
| Price | $0.40/secret/month + $0.05/10K API calls | Free | $0.05/parameter/month |
| Secret Size | Up to 65,536 bytes | Up to 4,096 bytes | Up to 8,192 bytes |
| Rotation | Built-in with Lambda | Manual | Manual |
| Cross-account Access | Yes | Yes | Yes |
| Versioning | Automatic | Manual | Automatic |
| Parameter Policies | No | No | Yes (expiration, notifications) |
| Best For | Database credentials, API keys | Application config | Large configs, parameter policies |

### 3.4 Automatic Rotation with Lambda

```bash
# Create a Lambda function for secret rotation (pseudo-code)
cat > rotation-function.py << 'EOF'
import boto3
import json

def lambda_handler(event, context):
    service_client = boto3.client('secretsmanager')
    arn = event['SecretId']
    token = event['ClientRequestToken']
    step = event['Step']

    # Implement rotation steps: createSecret, setSecret, testSecret, finishSecret
    if step == "createSecret":
        # Generate new password and store as AWSPENDING
        pass
    elif step == "setSecret":
        # Update the database with new password
        pass
    elif step == "testSecret":
        # Test connection with new password
        pass
    elif step == "finishSecret":
        # Mark AWSPENDING as AWSCURRENT
        pass
EOF

# Create IAM role for Lambda
cat > lambda-trust-policy.json << 'EOF'
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

aws iam create-role \
  --role-name SecretsManagerRotationRole \
  --assume-role-policy-document file://lambda-trust-policy.json

aws iam attach-role-policy \
  --role-name SecretsManagerRotationRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam attach-role-policy \
  --role-name SecretsManagerRotationRole \
  --policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite

echo "Lambda rotation setup complete (function deployment required)"
```

---

## Part 4: Security Monitoring & Detection

### 4.1 Enable GuardDuty and Review Findings

```bash
# Enable GuardDuty
DETECTOR_ID=$(aws guardduty create-detector \
  --enable \
  --finding-publishing-frequency FIFTEEN_MINUTES \
  --query DetectorId \
  --output text)

echo "GuardDuty Detector ID: $DETECTOR_ID"

# List detectors
aws guardduty list-detectors

# Get detector details
aws guardduty get-detector --detector-id $DETECTOR_ID

# List findings (may take time to generate)
aws guardduty list-findings --detector-id $DETECTOR_ID

# Get finding details
FINDINGS=$(aws guardduty list-findings --detector-id $DETECTOR_ID --query 'FindingIds[0]' --output text)

if [ "$FINDINGS" != "None" ] && [ -n "$FINDINGS" ]; then
  aws guardduty get-findings --detector-id $DETECTOR_ID --finding-ids $FINDINGS
else
  echo "No findings yet. GuardDuty needs time to analyze your environment."
fi

# Generate sample findings for testing
aws guardduty create-sample-findings \
  --detector-id $DETECTOR_ID \
  --finding-types "Recon:EC2/PortProbeUnprotectedPort" "UnauthorizedAccess:IAMUser/TorIPCaller"

# List findings again
aws guardduty list-findings --detector-id $DETECTOR_ID
```

### 4.2 AWS Security Hub - Enable Standards

```bash
# Enable Security Hub
aws securityhub enable-security-hub \
  --enable-default-standards

# Get Security Hub status
aws securityhub describe-hub

# Enable specific standards
# AWS Foundational Security Best Practices
aws securityhub batch-enable-standards \
  --standards-subscription-requests '[
    {
      "StandardsArn": "arn:aws:securityhub:us-east-1::standards/aws-foundational-security-best-practices/v/1.0.0"
    },
    {
      "StandardsArn": "arn:aws:securityhub:us-east-1::standards/cis-aws-foundations-benchmark/v/1.2.0"
    }
  ]'

# List enabled standards
aws securityhub get-enabled-standards

# Get findings
aws securityhub get-findings --max-results 10

# Get findings by severity
aws securityhub get-findings \
  --filters '{
    "SeverityLabel": [
      {"Value": "CRITICAL", "Comparison": "EQUALS"},
      {"Value": "HIGH", "Comparison": "EQUALS"}
    ]
  }' \
  --max-results 10

# Get compliance status
aws securityhub describe-standards-controls \
  --standards-subscription-arn "arn:aws:securityhub:us-east-1:$ACCOUNT_ID:subscription/aws-foundational-security-best-practices/v/1.0.0"
```

### 4.3 Amazon Inspector - Vulnerability Scanning

```bash
# Enable Inspector (EC2 scanning)
aws inspector2 enable \
  --resource-types EC2 ECR

# Get Inspector status
aws inspector2 batch-get-account-status

# List findings
aws inspector2 list-findings --max-results 10

# Get finding details with filters
aws inspector2 list-findings \
  --filter-criteria '{
    "severity": [{"comparison": "EQUALS", "value": "HIGH"}]
  }' \
  --max-results 10

# Note: Inspector requires EC2 instances or ECR images to scan
# For EC2, SSM agent must be installed
```

### 4.4 AWS Trusted Advisor Security Checks

```bash
# Describe Trusted Advisor checks (requires Business or Enterprise Support)
aws support describe-trusted-advisor-checks \
  --language en \
  --query 'checks[?category==`security`].[name,id]' \
  --output table

# Get specific check result (Security Groups - Specific Ports Unrestricted)
# Note: This requires AWS Support API access (Business/Enterprise support plan)
# SECURITY_CHECK_ID="1iG5NDGVre"
# aws support describe-trusted-advisor-check-result --check-id $SECURITY_CHECK_ID

echo "Trusted Advisor security checks require Business or Enterprise support plan"
```

### 4.5 Create EventBridge Rules for Security Findings

```bash
# Create an SNS topic for notifications
TOPIC_ARN=$(aws sns create-topic \
  --name security-findings-notifications \
  --query TopicArn \
  --output text)

echo "SNS Topic ARN: $TOPIC_ARN"

# Subscribe email to topic
aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol email \
  --notification-endpoint your-email@example.com

# Create EventBridge rule for GuardDuty findings
aws events put-rule \
  --name guardduty-high-severity-findings \
  --event-pattern '{
    "source": ["aws.guardduty"],
    "detail-type": ["GuardDuty Finding"],
    "detail": {
      "severity": [7, 7.0, 7.1, 7.2, 7.3, 7.4, 7.5, 7.6, 7.7, 7.8, 7.9, 8, 8.0, 8.1, 8.2, 8.3, 8.4, 8.5, 8.6, 8.7, 8.8, 8.9]
    }
  }' \
  --state ENABLED \
  --description "Alert on high severity GuardDuty findings"

# Add SNS target to EventBridge rule
aws events put-targets \
  --rule guardduty-high-severity-findings \
  --targets "Id"="1","Arn"="$TOPIC_ARN"

# Create EventBridge rule for Security Hub findings
aws events put-rule \
  --name securityhub-critical-findings \
  --event-pattern '{
    "source": ["aws.securityhub"],
    "detail-type": ["Security Hub Findings - Imported"],
    "detail": {
      "findings": {
        "Severity": {
          "Label": ["CRITICAL"]
        }
      }
    }
  }' \
  --state ENABLED

aws events put-targets \
  --rule securityhub-critical-findings \
  --targets "Id"="1","Arn"="$TOPIC_ARN"

# Grant EventBridge permission to publish to SNS
aws sns set-topic-attributes \
  --topic-arn $TOPIC_ARN \
  --attribute-name Policy \
  --attribute-value "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [{
      \"Effect\": \"Allow\",
      \"Principal\": {
        \"Service\": \"events.amazonaws.com\"
      },
      \"Action\": \"SNS:Publish\",
      \"Resource\": \"$TOPIC_ARN\"
    }]
  }"

echo "EventBridge rules created for security monitoring"
```

---

## Part 5: AWS Organizations & SCPs

### 5.1 Organization Structure

```bash
# Create an organization (if not already created)
# aws organizations create-organization --feature-set ALL

# Describe organization
aws organizations describe-organization

# List organizational units
aws organizations list-organizational-units-for-parent \
  --parent-id r-xxxx  # Replace with your root ID

# Create OUs
SECURITY_OU=$(aws organizations create-organizational-unit \
  --parent-id r-xxxx \
  --name "Security" \
  --query 'OrganizationalUnit.Id' \
  --output text)

PRODUCTION_OU=$(aws organizations create-organizational-unit \
  --parent-id r-xxxx \
  --name "Production" \
  --query 'OrganizationalUnit.Id' \
  --output text)

DEVELOPMENT_OU=$(aws organizations create-organizational-unit \
  --parent-id r-xxxx \
  --name "Development" \
  --query 'OrganizationalUnit.Id' \
  --output text)

echo "Created OUs: Security, Production, Development"

# List accounts
aws organizations list-accounts
```

### 5.2 Service Control Policies (SCPs)

**Exam Concept:** SCPs do NOT grant permissions; they set maximum permissions. They affect all users and roles in the account, including the root user.

```bash
# Create a deny policy for specific regions
cat > deny-region-scp.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAllOutsideRequestedRegions",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "us-east-1",
            "us-west-2"
          ]
        },
        "ArnNotLike": {
          "aws:PrincipalArn": [
            "arn:aws:iam::*:role/OrganizationAccountAccessRole"
          ]
        }
      }
    }
  ]
}
EOF

# Create the SCP
REGION_SCP_ID=$(aws organizations create-policy \
  --name "DenyNonApprovedRegions" \
  --description "Deny operations outside us-east-1 and us-west-2" \
  --type SERVICE_CONTROL_POLICY \
  --content file://deny-region-scp.json \
  --query 'Policy.PolicySummary.Id' \
  --output text)

echo "Region Restriction SCP ID: $REGION_SCP_ID"

# Create a policy to deny root user actions
cat > deny-root-user-scp.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRootUserActions",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:root"
        }
      }
    }
  ]
}
EOF

ROOT_SCP_ID=$(aws organizations create-policy \
  --name "DenyRootUser" \
  --description "Deny all actions by root user" \
  --type SERVICE_CONTROL_POLICY \
  --content file://deny-root-user-scp.json \
  --query 'Policy.PolicySummary.Id' \
  --output text)

# Create a policy to require MFA
cat > require-mfa-scp.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAllActionsWithoutMFA",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
EOF

MFA_SCP_ID=$(aws organizations create-policy \
  --name "RequireMFA" \
  --description "Require MFA for all actions" \
  --type SERVICE_CONTROL_POLICY \
  --content file://require-mfa-scp.json \
  --query 'Policy.PolicySummary.Id' \
  --output text)

# List all SCPs
aws organizations list-policies --filter SERVICE_CONTROL_POLICY

# Attach SCP to an OU (example)
# aws organizations attach-policy --policy-id $REGION_SCP_ID --target-id $DEVELOPMENT_OU

# Detach SCP
# aws organizations detach-policy --policy-id $REGION_SCP_ID --target-id $DEVELOPMENT_OU

echo "SCPs created but not attached. Use attach-policy to apply them."
```

### 5.3 Tag Policies

```bash
# Create a tag policy
cat > tag-policy.json << 'EOF'
{
  "tags": {
    "Environment": {
      "tag_key": {
        "@@assign": "Environment"
      },
      "tag_value": {
        "@@assign": ["Production", "Development", "Staging"]
      },
      "enforced_for": {
        "@@assign": ["ec2:instance", "s3:bucket", "rds:db"]
      }
    },
    "CostCenter": {
      "tag_key": {
        "@@assign": "CostCenter"
      },
      "enforced_for": {
        "@@assign": ["ec2:*", "s3:*"]
      }
    }
  }
}
EOF

# Create the tag policy
TAG_POLICY_ID=$(aws organizations create-policy \
  --name "RequiredTags" \
  --description "Enforce required tags on resources" \
  --type TAG_POLICY \
  --content file://tag-policy.json \
  --query 'Policy.PolicySummary.Id' \
  --output text)

echo "Tag Policy ID: $TAG_POLICY_ID"

# List effective tags for an account
# aws organizations describe-effective-policy --policy-type TAG_POLICY --target-id 123456789012
```

### 5.4 Consolidated Billing

```bash
# Get consolidated billing information
aws organizations describe-organization \
  --query 'Organization.[MasterAccountId,FeatureSet]' \
  --output table

# List accounts with their status
aws organizations list-accounts \
  --query 'Accounts[].[Id,Name,Status,JoinedMethod]' \
  --output table

# Note: Actual billing data is retrieved through Cost Explorer API
aws ce get-cost-and-usage \
  --time-period Start=2026-03-01,End=2026-03-22 \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --group-by Type=DIMENSION,Key=LINKED_ACCOUNT

# Get cost by service
aws ce get-cost-and-usage \
  --time-period Start=2026-03-01,End=2026-03-22 \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE
```

---

## Part 6: Review Questions

### Question 1
You have an IAM user with the `PowerUserAccess` managed policy attached, and a permission boundary that allows only S3 and EC2 actions. The user tries to create a Lambda function but receives an "Access Denied" error. Why?

<details>
<summary>Click to reveal answer</summary>

**Answer:** Permission boundaries set the maximum permissions an entity can have. Even though `PowerUserAccess` grants Lambda permissions, the permission boundary restricts the user to only S3 and EC2 actions. The effective permissions are the intersection of the identity-based policy and the permission boundary.

**Exam Tip:** Remember the policy evaluation logic:
1. Explicit Deny (always wins)
2. Explicit Allow (must be in BOTH identity policy AND permission boundary)
3. Implicit Deny (default)

</details>

### Question 2
Your organization requires all S3 objects to be encrypted at rest. You need to track which KMS keys are being used for encryption and by whom. Which encryption option provides the best audit trail?

A) SSE-S3
B) SSE-KMS
C) SSE-C
D) Client-side encryption

<details>
<summary>Click to reveal answer</summary>

**Answer: B) SSE-KMS**

**Explanation:** SSE-KMS logs all encryption and decryption operations to CloudTrail, providing a complete audit trail. You can see who accessed which objects and when. SSE-S3 provides limited audit capabilities, SSE-C provides no AWS-based auditing, and client-side encryption happens outside AWS.

**Exam Tip:** Choose SSE-KMS when you need:
- Audit trails (CloudTrail integration)
- Fine-grained access control
- Key rotation management
- Compliance requirements

</details>

### Question 3
You created a KMS key with automatic rotation enabled. After one year, you need to decrypt data that was encrypted before the rotation. What happens?

<details>
<summary>Click to reveal answer</summary>

**Answer:** Decryption works seamlessly. KMS automatically uses the correct version of the key material to decrypt data. When automatic rotation is enabled, KMS keeps all previous versions of the key material and uses the appropriate version based on the ciphertext metadata.

**Exam Tip:**
- Automatic rotation occurs annually
- Old key material is retained for decryption
- Key ID and ARN remain the same
- Rotation only applies to symmetric keys
- Manual rotation requires creating a new key and updating aliases

</details>

### Question 4
When should you use AWS Secrets Manager instead of SSM Parameter Store?

<details>
<summary>Click to reveal answer</summary>

**Answer:** Use Secrets Manager when you need:
- **Automatic rotation** of secrets (built-in Lambda integration)
- **Database credentials** that need regular rotation
- **Cross-account access** to secrets with resource-based policies
- **Larger secrets** (up to 65KB vs 4KB/8KB for Parameter Store)

Use Parameter Store when you need:
- Application configuration values
- Free tier (Standard parameters)
- Parameter policies (expiration, notifications) with Advanced parameters
- Hierarchical parameter organization

**Exam Tip:** Secrets Manager is purpose-built for secrets with rotation. Parameter Store is more general-purpose for configuration and secrets without automatic rotation.

</details>

### Question 5
Your Security Hub shows multiple findings with CRITICAL severity. You need to automatically remediate EC2 security groups that allow unrestricted SSH access (0.0.0.0/0 on port 22). What is the best approach?

<details>
<summary>Click to reveal answer</summary>

**Answer:** Create an EventBridge rule that triggers on Security Hub findings for unrestricted SSH access, then invoke a Lambda function to automatically remove the offending ingress rule.

**Implementation:**
1. EventBridge rule filters Security Hub findings for:
   - `detail.findings.Title` contains "EC2 security group allows 0.0.0.0/0"
   - `detail.findings.Severity.Label` equals "CRITICAL"
2. Lambda function parses the finding, identifies the security group, and uses `revoke-security-group-ingress` to remove the rule
3. Lambda updates the finding status in Security Hub
4. SNS notification sent to security team

**Exam Tip:** Automated remediation pattern:
- Security Hub/GuardDuty finding → EventBridge → Lambda → Remediation action
- Use custom actions for manual approval workflows
- Consider AWS Systems Manager Automation for common remediation tasks

</details>

### Question 6
A GuardDuty finding indicates "UnauthorizedAccess:IAMUser/TorIPCaller" for one of your IAM users. What immediate actions should you take?

<details>
<summary>Click to reveal answer</summary>

**Answer:** Immediate response steps:
1. **Rotate the IAM user's access keys** immediately
2. **Attach an explicit deny-all policy** to the user to prevent any actions
3. **Review CloudTrail logs** to identify what actions were performed
4. **Check IAM Access Advisor** to see which services were accessed
5. **Review recent IAM changes** (new users, roles, policy modifications)
6. **Enable MFA** if not already enabled
7. **Investigate** if credentials were exposed in code repositories, logs, or documents

**Exam Tip:** GuardDuty finding types to know:
- **Recon:** - Reconnaissance (port scanning, unusual API calls)
- **UnauthorizedAccess:** - Compromised credentials
- **Trojan:** - Malware or backdoor
- **Backdoor:** - EC2 instance communicating with known malicious IPs
- **CryptoCurrency:** - Bitcoin mining activity

</details>

### Question 7
You need to prevent developers in the Development OU from launching expensive EC2 instance types. What is the most effective way to enforce this using AWS Organizations?

<details>
<summary>Click to reveal answer</summary>

**Answer:** Create a Service Control Policy (SCP) that denies the ability to launch specific instance types and attach it to the Development OU.

**Example SCP:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyExpensiveInstanceTypes",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringEquals": {
          "ec2:InstanceType": [
            "p3.16xlarge",
            "p4d.24xlarge",
            "x1e.32xlarge"
          ]
        }
      }
    }
  ]
}
```

**Exam Tip:**
- SCPs affect ALL users and roles in the account, including root
- SCPs do not grant permissions; they set boundaries
- SCPs are inherited down the OU hierarchy
- Use explicit deny for preventive controls
- Remember: Explicit Deny in SCP > Everything else

</details>

### Question 8
Your RDS database credentials are stored in AWS Secrets Manager with automatic rotation enabled every 30 days. After a rotation, your application can no longer connect to the database. What could be wrong?

<details>
<summary>Click to reveal answer</summary>

**Answer:** Common issues:
1. **Application not using Secrets Manager API** - The app might be using hardcoded credentials instead of retrieving from Secrets Manager
2. **Application not handling rotation** - The app should retrieve the secret on each connection or handle the rotation signal
3. **Lambda rotation function failed** - Check CloudWatch Logs for the rotation Lambda function
4. **Network connectivity** - Lambda function might not be able to reach the database (VPC configuration)
5. **Database privileges** - The rotation function needs SUPER user privileges to create/modify users

**Best Practice:**
- Applications should retrieve secrets dynamically, not cache them indefinitely
- Use the `VersionStage` "AWSCURRENT" to always get the latest version
- Implement connection retry logic to handle rotation transitions
- Test rotation in non-production first

**Exam Tip:** Secrets Manager rotation has four stages:
1. **createSecret** - Generate new credentials and store as AWSPENDING
2. **setSecret** - Update the database with new credentials
3. **testSecret** - Verify the new credentials work
4. **finishSecret** - Move AWSPENDING to AWSCURRENT

</details>

### Question 9
You need to ensure that all resources in your AWS account have mandatory tags: "Environment", "CostCenter", and "Owner". Non-compliant resources should be identified automatically. What solution would you implement?

<details>
<summary>Click to reveal answer</summary>

**Answer:** Implement a multi-layered tagging strategy:

**1. AWS Organizations Tag Policies (Preventive):**
- Create tag policies that define required tags and allowed values
- Attach to OUs to enforce at the organization level
- Resources cannot be created without required tags (for supported services)

**2. AWS Config Rules (Detective):**
- Enable `required-tags` Config rule
- Specify required tag keys and optionally allowed values
- Automatically evaluates resources and marks non-compliant ones

**3. Service Control Policies (Preventive):**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Action": ["ec2:RunInstances", "s3:CreateBucket"],
    "Resource": "*",
    "Condition": {
      "Null": {
        "aws:RequestTag/Environment": "true",
        "aws:RequestTag/CostCenter": "true",
        "aws:RequestTag/Owner": "true"
      }
    }
  }]
}
```

**4. Automated Remediation:**
- EventBridge rule triggers on Config compliance change
- Lambda function tags non-compliant resources or sends notifications

**Exam Tip:**
- Tag Policies enforce standardization (case, allowed values)
- SCPs can prevent resource creation without tags
- Config Rules identify non-compliant existing resources
- Not all AWS services support tag enforcement

</details>

### Question 10
Your company needs to comply with PCI DSS requirements. You're using AWS Security Hub. Which of the following are best practices for maintaining compliance?

A) Enable only the PCI DSS standard in Security Hub
B) Enable multiple standards (PCI DSS, CIS, AWS Foundational)
C) Disable automated remediation for compliance controls
D) Aggregate findings from multiple regions and accounts

<details>
<summary>Click to reveal answer</summary>

**Answer: B and D**

**Explanation:**

**B) Enable multiple standards** - Different standards have overlapping controls but also unique requirements. Enabling multiple standards provides comprehensive security coverage:
- **PCI DSS** - Payment card industry requirements
- **CIS AWS Foundations Benchmark** - Industry best practices
- **AWS Foundational Security Best Practices** - AWS-recommended security controls

**D) Aggregate findings from multiple regions and accounts** - Security Hub supports:
- Cross-region aggregation (designate one region as aggregation region)
- Cross-account aggregation (master-member architecture)
- Provides centralized security visibility
- Essential for enterprise compliance

**Why not A?** - PCI DSS alone might miss important security controls covered by other standards.

**Why not C?** - Automated remediation is a best practice for rapid compliance. It reduces mean time to remediation (MTTR) and human error.

**Exam Tip:** Security Hub compliance features:
- Integrates with GuardDuty, Inspector, Macie, IAM Access Analyzer, Firewall Manager
- ASFF (AWS Security Finding Format) - standardized finding format
- Supports custom actions for workflow integration
- Compliance scores by standard and by control
- Findings can be suppressed (archived) if they're false positives or accepted risks

**Additional Best Practices:**
- Enable automatic enablement of new controls
- Create EventBridge rules for CRITICAL/HIGH findings
- Use Security Hub Insights for trending and analysis
- Export findings to S3 for long-term storage and analysis
- Integrate with SIEM tools (Splunk, SIEM, etc.)

</details>

---

## Cleanup Steps

**IMPORTANT:** Run these commands to avoid ongoing charges:

```bash
# Get your account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Part 1: IAM Cleanup
aws iam detach-user-policy --user-name developer-user --policy-arn arn:aws:iam::aws:policy/PowerUserAccess
aws iam delete-user-permissions-boundary --user-name developer-user
aws iam delete-user --user-name developer-user

aws iam detach-role-policy --role-name CrossAccountS3ReadRole --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam delete-role --role-name CrossAccountS3ReadRole

aws iam delete-policy --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/S3ReadOnlyWithConditions
aws iam delete-policy --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/DeveloperBoundary

# Delete IAM Access Analyzer
aws accessanalyzer delete-analyzer --analyzer-name security-lab-analyzer

# Part 2: KMS and Encryption Cleanup
# List and revoke grants
GRANTS=$(aws kms list-grants --key-id $KMS_KEY_ID --query 'Grants[*].GrantId' --output text)
for GRANT in $GRANTS; do
  aws kms revoke-grant --key-id $KMS_KEY_ID --grant-id $GRANT
done

# Delete EBS volume
aws ec2 delete-volume --volume-id $VOLUME_ID

# Delete RDS instance (skip final snapshot)
aws rds delete-db-instance \
  --db-instance-identifier encrypted-db-instance \
  --skip-final-snapshot \
  --delete-automated-backups

# Schedule KMS key deletion (minimum 7 days)
aws kms schedule-key-deletion --key-id $KMS_KEY_ID --pending-window-in-days 7
aws kms schedule-key-deletion --key-id $ASYMMETRIC_KEY_ID --pending-window-in-days 7

# Delete S3 buckets
aws s3 rm s3://$BUCKET_NAME --recursive
aws s3api delete-bucket --bucket $BUCKET_NAME

aws s3 rm s3://$ENCRYPTION_BUCKET --recursive
aws s3api delete-bucket --bucket $ENCRYPTION_BUCKET

# Delete ACM certificate (if DNS validated and issued)
# aws acm delete-certificate --certificate-arn $CERT_ARN

# Part 3: Secrets Management Cleanup
aws secretsmanager delete-secret --secret-id rds-mysql-credentials --force-delete-without-recovery

aws ssm delete-parameter --name "/myapp/database/password"
aws ssm delete-parameter --name "/myapp/database/host"

# Delete Lambda role
aws iam detach-role-policy --role-name SecretsManagerRotationRole --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam detach-role-policy --role-name SecretsManagerRotationRole --policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite
aws iam delete-role --role-name SecretsManagerRotationRole

# Part 4: Security Monitoring Cleanup
# Disable GuardDuty
aws guardduty delete-detector --detector-id $DETECTOR_ID

# Disable Security Hub
aws securityhub disable-security-hub

# Disable Inspector
aws inspector2 disable --resource-types EC2 ECR

# Delete EventBridge rules
aws events remove-targets --rule guardduty-high-severity-findings --ids 1
aws events delete-rule --name guardduty-high-severity-findings

aws events remove-targets --rule securityhub-critical-findings --ids 1
aws events delete-rule --name securityhub-critical-findings

# Delete SNS topic
aws sns delete-topic --topic-arn $TOPIC_ARN

# Part 5: AWS Organizations Cleanup
# Detach and delete SCPs (if attached)
# aws organizations detach-policy --policy-id $REGION_SCP_ID --target-id $DEVELOPMENT_OU
aws organizations delete-policy --policy-id $REGION_SCP_ID
aws organizations delete-policy --policy-id $ROOT_SCP_ID
aws organizations delete-policy --policy-id $MFA_SCP_ID
aws organizations delete-policy --policy-id $TAG_POLICY_ID

# Delete OUs (must be empty - move or delete accounts first)
# aws organizations delete-organizational-unit --organizational-unit-id $SECURITY_OU
# aws organizations delete-organizational-unit --organizational-unit-id $PRODUCTION_OU
# aws organizations delete-organizational-unit --organizational-unit-id $DEVELOPMENT_OU

# Clean up local files
rm -f s3-readonly-policy.json bucket-policy.json permission-boundary.json
rm -f trust-policy.json key-policy.json new-key-policy.json
rm -f plaintext.txt encrypted.txt credential-report.csv job-id.txt
rm -f deny-region-scp.json deny-root-user-scp.json require-mfa-scp.json tag-policy.json
rm -f lambda-trust-policy.json rotation-function.py

echo "Cleanup complete! Note: Some resources have delayed deletion (KMS keys: 7 days minimum)"
```

---

## Key Exam Concepts Summary

### IAM Policy Evaluation Logic
1. **Explicit Deny** (SCP, Permission Boundary, Identity Policy, Resource Policy)
2. **Explicit Allow** (must exist in identity policy AND not blocked by boundaries)
3. **Implicit Deny** (default)

### Encryption Options Comparison
- **SSE-S3**: Simplest, AWS-managed keys, no CloudTrail audit
- **SSE-KMS**: Audit trail, key policies, rotation, envelope encryption
- **SSE-C**: Customer-provided keys, customer manages key lifecycle
- **Client-Side**: Encryption before upload, most control

### Secrets Management Decision Tree
- **Automatic rotation needed?** → Secrets Manager
- **Database credentials?** → Secrets Manager
- **Application config?** → Parameter Store
- **Large secrets (>8KB)?** → Secrets Manager
- **Free tier?** → Parameter Store (Standard)
- **Parameter policies?** → Parameter Store (Advanced)

### Security Services
- **GuardDuty**: Threat detection (VPC Flow Logs, CloudTrail, DNS logs)
- **Security Hub**: Compliance standards, aggregated findings
- **Inspector**: Vulnerability scanning (EC2, ECR, Lambda)
- **Macie**: S3 data classification, PII detection
- **IAM Access Analyzer**: External access, unused access

### AWS Organizations SCPs
- Do NOT grant permissions (only limit them)
- Affect all users/roles including root
- Inherited down OU hierarchy
- Use for preventive controls
- Common patterns: region restrictions, service restrictions, enforce MFA

### KMS Best Practices
- Use separate keys for different data classifications
- Enable automatic rotation for symmetric keys
- Use key policies for cross-account access
- Grant principle of least privilege
- Use grants for temporary access
- Monitor key usage with CloudTrail

---

## Additional Resources

- [AWS Security Best Practices](https://docs.aws.amazon.com/security/latest/userguide/best-practices.html)
- [IAM Policy Evaluation Logic](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html)
- [AWS KMS Best Practices](https://docs.aws.amazon.com/kms/latest/developerguide/best-practices.html)
- [AWS Organizations SCPs](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html)
- [GuardDuty Finding Types](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_finding-types-active.html)
- [Security Hub Standards](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-standards.html)

---

**Congratulations!** You've completed Lab 8: Security and Compliance Automation. This lab covered critical security concepts for the AWS Certified CloudOps Engineer exam, including IAM policies, encryption, secrets management, security monitoring, and organizational governance.
