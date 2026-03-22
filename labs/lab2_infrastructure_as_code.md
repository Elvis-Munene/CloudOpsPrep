# Lab 2: Infrastructure as Code

## Estimated Time: 2 hours
## Objectives
- Deploy AWS infrastructure using CloudFormation templates
- Use advanced CloudFormation features (parameters, mappings, conditions, change sets, drift detection)
- Understand AWS CDK basics and compare with CloudFormation
- Deploy across accounts/regions with StackSets
- Understand Terraform basics for the exam

## Prerequisites
- AWS account with admin access
- AWS CLI configured
- Node.js installed (for CDK section)

---

## Part 1: CloudFormation Basics

### Step 1: Create Your First Template

Save this as `vpc-ec2-stack.yaml`:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Lab 2 - Basic VPC with EC2 Instance

Parameters:
  EnvironmentType:
    Description: Environment type
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - staging
      - prod
    ConstraintDescription: Must be dev, staging, or prod

  KeyPairName:
    Description: EC2 Key Pair name
    Type: AWS::EC2::KeyPair::KeyName

  InstanceType:
    Description: EC2 instance type
    Type: String
    Default: t3.micro
    AllowedValues:
      - t3.micro
      - t3.small
      - t3.medium

Mappings:
  RegionAMI:
    us-east-1:
      HVM64: ami-0c02fb55956c7d316
    us-west-2:
      HVM64: ami-0892d3c7ee96c0bf7
    eu-west-1:
      HVM64: ami-0d71ea30463e0ff8d

Conditions:
  IsProduction: !Equals [!Ref EnvironmentType, prod]

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentType}-vpc'

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentType}-igw'

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentType}-public-subnet'

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachGateway
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  SubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet
      RouteTableId: !Ref PublicRouteTable

  WebSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTP and SSH
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0

  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      ImageId: !FindInMap [RegionAMI, !Ref 'AWS::Region', HVM64]
      KeyName: !Ref KeyPairName
      SubnetId: !Ref PublicSubnet
      SecurityGroupIds:
        - !Ref WebSecurityGroup
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentType}-web-server'
        - Key: Environment
          Value: !Ref EnvironmentType

  # Only created in production
  EIPForProd:
    Type: AWS::EC2::EIP
    Condition: IsProduction
    Properties:
      InstanceId: !Ref WebServer

Outputs:
  VPCId:
    Description: VPC ID
    Value: !Ref VPC
    Export:
      Name: !Sub '${EnvironmentType}-VPCId'

  PublicSubnetId:
    Description: Public Subnet ID
    Value: !Ref PublicSubnet
    Export:
      Name: !Sub '${EnvironmentType}-PublicSubnetId'

  WebServerPublicIP:
    Description: Public IP of the web server
    Value: !GetAtt WebServer.PublicIp

  SecurityGroupId:
    Description: Security Group ID
    Value: !Ref WebSecurityGroup
    Export:
      Name: !Sub '${EnvironmentType}-SGId'
```

### Step 2: Deploy the Stack

```bash
# Validate the template first
aws cloudformation validate-template \
  --template-body file://vpc-ec2-stack.yaml

# Create the stack
aws cloudformation create-stack \
  --stack-name lab2-vpc-ec2 \
  --template-body file://vpc-ec2-stack.yaml \
  --parameters \
    ParameterKey=EnvironmentType,ParameterValue=dev \
    ParameterKey=KeyPairName,ParameterValue=my-key-pair \
    ParameterKey=InstanceType,ParameterValue=t3.micro

# Monitor stack creation
aws cloudformation describe-stack-events \
  --stack-name lab2-vpc-ec2 \
  --query 'StackEvents[*].[Timestamp,ResourceType,ResourceStatus,ResourceStatusReason]' \
  --output table

# Wait for completion
aws cloudformation wait stack-create-complete --stack-name lab2-vpc-ec2

# View outputs
aws cloudformation describe-stacks \
  --stack-name lab2-vpc-ec2 \
  --query 'Stacks[0].Outputs'
```

> **Exam Tip:** Stack events show the order of resource creation and any errors. If a stack fails, read events from BOTTOM to TOP to find the root cause.

### Step 3: Update with Change Sets

```bash
# Create a change set to add HTTPS to the security group
# First, modify the template to add port 443 to SecurityGroupIngress, then:

aws cloudformation create-change-set \
  --stack-name lab2-vpc-ec2 \
  --change-set-name add-https \
  --template-body file://vpc-ec2-stack-v2.yaml \
  --parameters \
    ParameterKey=EnvironmentType,UsePreviousValue=true \
    ParameterKey=KeyPairName,UsePreviousValue=true

# Review what will change BEFORE applying
aws cloudformation describe-change-set \
  --stack-name lab2-vpc-ec2 \
  --change-set-name add-https

# Execute only if changes look correct
aws cloudformation execute-change-set \
  --stack-name lab2-vpc-ec2 \
  --change-set-name add-https
```

> **Exam Tip:** Change sets let you preview changes BEFORE applying. They show whether a resource will be Modified, Replaced, or Added. Always use change sets for production stacks.

---

## Part 2: CloudFormation Advanced Features

### Cross-Stack References

Create a second stack that imports values from the first:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Second stack using cross-stack references

Resources:
  AppSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: App tier security group
      VpcId: !ImportValue dev-VPCId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 8080
          ToPort: 8080
          SourceSecurityGroupId: !ImportValue dev-SGId
```

> **Exam Tip:** `Fn::ImportValue` creates a dependency between stacks. You CANNOT delete the exporting stack while another stack imports its values.

### cfn-init and cfn-signal (Instance Bootstrapping)

```yaml
Resources:
  WebServer:
    Type: AWS::EC2::Instance
    CreationPolicy:
      ResourceSignal:
        Timeout: PT10M
        Count: 1
    Metadata:
      AWS::CloudFormation::Init:
        config:
          packages:
            yum:
              httpd: []
          files:
            /var/www/html/index.html:
              content: |
                <h1>Hello from CloudFormation!</h1>
              mode: '000644'
          services:
            sysvinit:
              httpd:
                enabled: true
                ensureRunning: true
    Properties:
      # ... (other properties)
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash -xe
          yum update -y aws-cfn-bootstrap
          /opt/aws/bin/cfn-init -v \
            --stack ${AWS::StackName} \
            --resource WebServer \
            --region ${AWS::Region}
          /opt/aws/bin/cfn-signal -e $? \
            --stack ${AWS::StackName} \
            --resource WebServer \
            --region ${AWS::Region}
```

> **Exam Tip:** `CreationPolicy` + `cfn-signal` ensures CloudFormation waits for the instance to fully configure before marking it CREATE_COMPLETE. Without this, the stack succeeds even if bootstrapping fails.

### Stack Policies

```bash
# Prevent accidental replacement of the database
aws cloudformation set-stack-policy \
  --stack-name lab2-vpc-ec2 \
  --stack-policy-body '{
    "Statement": [
      {
        "Effect": "Allow",
        "Action": "Update:*",
        "Principal": "*",
        "Resource": "*"
      },
      {
        "Effect": "Deny",
        "Action": "Update:Replace",
        "Principal": "*",
        "Resource": "LogicalResourceId/WebServer"
      }
    ]
  }'
```

> **Exam Tip:** Stack policies protect specific resources from accidental updates. By default, all resources can be updated. Once set, a stack policy CANNOT be removed, only modified.

### Drift Detection

```bash
# Make a manual change in the console (e.g., add a tag to the EC2 instance)
# Then detect the drift:

aws cloudformation detect-stack-drift --stack-name lab2-vpc-ec2

# Check drift status
aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id <detection-id>

# View drifted resources
aws cloudformation describe-stack-resource-drifts \
  --stack-name lab2-vpc-ec2 \
  --stack-resource-drift-status-filters MODIFIED DELETED
```

> **Exam Tip:** Drift detection identifies resources whose actual configuration differs from the template. Not all resource types support drift detection.

### DeletionPolicy

```yaml
Resources:
  Database:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Snapshot      # Creates final snapshot before deletion
    UpdateReplacePolicy: Snapshot  # Creates snapshot if resource must be replaced
    Properties:
      # ...

  ImportantBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain  # Keeps the bucket even when stack is deleted
```

> **Exam Tip:** `DeletionPolicy: Snapshot` works for RDS, ElastiCache, Neptune, Redshift, and EBS volumes. `Retain` works for any resource type.

---

## Part 3: AWS CDK

### Initialize a CDK Project

```bash
# Install CDK
npm install -g aws-cdk

# Initialize project
mkdir cdk-lab && cd cdk-lab
cdk init app --language typescript

# Install constructs
npm install @aws-cdk/aws-ec2 @aws-cdk/aws-s3
```

### Define Infrastructure (lib/cdk-lab-stack.ts)

```typescript
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as s3 from 'aws-cdk-lib/aws-s3';

export class CdkLabStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // L2 Construct - VPC with best-practice defaults
    const vpc = new ec2.Vpc(this, 'LabVPC', {
      maxAzs: 2,
      natGateways: 1,
    });

    // L2 Construct - S3 Bucket
    const bucket = new s3.Bucket(this, 'LogBucket', {
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: true,
    });

    // Output
    new cdk.CfnOutput(this, 'BucketName', { value: bucket.bucketName });
    new cdk.CfnOutput(this, 'VpcId', { value: vpc.vpcId });
  }
}
```

### CDK Commands

```bash
# Synthesize CloudFormation template (preview)
cdk synth

# Compare deployed stack with current code
cdk diff

# Deploy
cdk deploy

# Destroy
cdk destroy
```

### CDK Construct Levels

| Level | Description | Example |
|-------|-------------|---------|
| L1 (Cfn) | Direct CloudFormation mapping | `CfnBucket` |
| L2 | Curated with sensible defaults | `Bucket` (adds encryption, lifecycle) |
| L3 (Patterns) | Multi-resource patterns | `ApplicationLoadBalancedFargateService` |

> **Exam Tip:** CDK synthesizes to CloudFormation. It uses the same CloudFormation engine for deployment. CDK is ideal when you want programming language features (loops, conditionals, type safety).

---

## Part 4: CloudFormation StackSets

### Create a StackSet

```bash
# Create a StackSet to deploy Config rules across regions
aws cloudformation create-stack-set \
  --stack-set-name config-rules-global \
  --template-body file://config-rules.yaml \
  --permission-model SERVICE_MANAGED \
  --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=false

# Deploy to specific regions
aws cloudformation create-stack-instances \
  --stack-set-name config-rules-global \
  --deployment-targets OrganizationalUnitIds=ou-xxxx-xxxxxxxx \
  --regions us-east-1 us-west-2 eu-west-1 \
  --operation-preferences \
    FailureTolerancePercentage=25,MaxConcurrentPercentage=50
```

### Key StackSet Concepts

| Concept | Description |
|---------|-------------|
| **Self-managed permissions** | You create admin and execution IAM roles manually |
| **Service-managed permissions** | AWS Organizations manages roles automatically |
| **Failure tolerance** | How many accounts/regions can fail before stopping |
| **Max concurrent** | How many accounts/regions deploy simultaneously |
| **Auto-deployment** | Automatically deploy to new accounts added to an OU |

> **Exam Tip:** StackSets with service-managed permissions + auto-deployment is the recommended approach for AWS Organizations. Exam questions often test the difference between self-managed and service-managed.

---

## Part 5: Terraform Basics

### Simple S3 Bucket (main.tf)

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  # Remote state (recommended for teams)
  backend "s3" {
    bucket = "my-terraform-state-bucket"
    key    = "lab2/terraform.tfstate"
    region = "us-east-1"
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "lab_bucket" {
  bucket = "lab2-terraform-bucket-${random_id.suffix.hex}"

  tags = {
    Environment = "dev"
    ManagedBy   = "Terraform"
  }
}

resource "aws_s3_bucket_versioning" "lab_bucket" {
  bucket = aws_s3_bucket.lab_bucket.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "random_id" "suffix" {
  byte_length = 4
}

output "bucket_name" {
  value = aws_s3_bucket.lab_bucket.id
}
```

### Terraform Commands

```bash
terraform init      # Download providers, initialize backend
terraform plan      # Preview changes (like CloudFormation change set)
terraform apply     # Apply changes
terraform destroy   # Tear down all resources
terraform import    # Import existing resources into state
```

### State Management

| Concept | Description |
|---------|-------------|
| **Local state** | `terraform.tfstate` file on disk (default) |
| **Remote state** | S3 + DynamoDB for locking (production recommended) |
| **State locking** | DynamoDB table prevents concurrent modifications |
| **terraform import** | Bring existing resources under Terraform management |

> **Exam Tip:** The exam may ask about Terraform at a high level - state files, plan/apply workflow, and how it compares to CloudFormation. You won't need to write HCL on the exam.

---

## Part 6: Review Questions

### Question 1

A SysOps administrator needs to update an Amazon RDS instance in a CloudFormation stack from `db.t3.medium` to `db.r5.large`. Before applying the change, they want to understand the impact. What should they use?

**A)** Run `aws cloudformation update-stack` with `--dry-run` flag
**B)** Create a change set and review the proposed changes before executing
**C)** Use drift detection to preview the changes
**D)** Delete the stack and recreate it with the new instance type

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Change sets allow you to preview the exact changes CloudFormation will make before executing them. The change set will show whether the RDS instance will be Modified (in-place) or Replaced (creates new, deletes old). There is no `--dry-run` flag for CloudFormation (A). Drift detection (C) compares current state to the template, not previewing future changes. Deleting and recreating (D) causes unnecessary downtime and data loss.
</details>

---

### Question 2

A company uses CloudFormation to manage a production database. An engineer accidentally updated the stack and CloudFormation is about to replace the RDS instance. What feature should have been configured to PREVENT this?

**A)** DeletionPolicy: Retain
**B)** Stack policy denying Update:Replace on the RDS resource
**C)** Termination protection on the stack
**D)** UpdateReplacePolicy: Retain

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** A stack policy with `"Effect": "Deny", "Action": "Update:Replace"` on the RDS resource would prevent CloudFormation from replacing it during updates. DeletionPolicy (A) only applies when the stack is deleted, not during updates. Termination protection (C) prevents stack deletion but doesn't prevent resource replacement during updates. UpdateReplacePolicy (D) controls what happens to the old resource AFTER replacement occurs (snapshot/retain), but doesn't prevent the replacement itself.
</details>

---

### Question 3

A SysOps administrator discovers that someone manually changed the security group rules of an EC2 instance managed by CloudFormation. What is the FASTEST way to identify the discrepancy?

**A)** Review CloudTrail logs for security group API calls
**B)** Compare the CloudFormation template with the current console settings manually
**C)** Run CloudFormation drift detection on the stack
**D)** Delete and recreate the stack to restore the original configuration

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** CloudFormation drift detection automatically compares the actual resource configuration with the expected template configuration and highlights differences. CloudTrail (A) shows who made the change but requires manual investigation to compare. Manual comparison (B) is slow and error-prone. Deleting and recreating (D) causes downtime and is excessive.
</details>

---

### Question 4

A company needs to deploy AWS Config rules across all 50 accounts in their AWS Organization across 4 regions. What is the MOST operationally efficient approach?

**A)** Create a CloudFormation stack in each account and region manually
**B)** Use CloudFormation StackSets with service-managed permissions and auto-deployment
**C)** Use AWS CDK to deploy to each account with a loop
**D)** Write a script using AWS CLI to deploy the template to each account

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** CloudFormation StackSets with service-managed permissions (delegated by AWS Organizations) and auto-deployment is the most operationally efficient approach. It automatically handles IAM roles and will deploy to new accounts added to the OU. Manual deployment (A) doesn't scale. CDK with a loop (C) requires managing credentials for 50 accounts. CLI scripts (D) are less maintainable and don't auto-deploy to new accounts.
</details>

---

### Question 5

During a CloudFormation stack update, the update fails and CloudFormation automatically rolls back. However, the rollback also fails, leaving the stack in UPDATE_ROLLBACK_FAILED state. What should the administrator do?

**A)** Delete the stack and recreate it
**B)** Use `ContinueUpdateRollback` and optionally skip the problematic resources
**C)** Create a new stack with a different name
**D)** Contact AWS Support to fix the stack

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** When a stack is in UPDATE_ROLLBACK_FAILED state, you should use `aws cloudformation continue-update-rollback` with the `--resources-to-skip` parameter to skip resources that are causing the rollback to fail. This allows the rollback to complete. Deleting (A) may not work while in this state and loses all resources. Creating a new stack (C) is wasteful. AWS Support (D) is unnecessary when self-service options exist.
</details>

---

### Question 6

What is the key difference between `DeletionPolicy: Retain` and `DeletionPolicy: Snapshot`?

**A)** Retain keeps the resource running; Snapshot creates a backup and then deletes the resource
**B)** Retain applies to all resources; Snapshot only works with compute resources
**C)** Retain prevents stack deletion; Snapshot allows it
**D)** They are equivalent and interchangeable

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** `DeletionPolicy: Retain` keeps the resource intact (orphaned from CloudFormation) when the stack is deleted. `DeletionPolicy: Snapshot` creates a final snapshot/backup of the resource and then deletes it. Snapshot only works with resources that support snapshots: RDS, ElastiCache, Neptune, Redshift, and EBS volumes (B is wrong because it's storage/database resources, not compute). Neither policy prevents stack deletion (C). They serve different purposes (D).
</details>

---

### Question 7

A developer wants to use CloudFormation to deploy an EC2 instance that installs Apache and starts the web server. The stack shows CREATE_COMPLETE, but the Apache installation failed. How should they ensure the stack reflects the true state of the instance configuration?

**A)** Use detailed CloudWatch monitoring on the instance
**B)** Add `cfn-init` for configuration and `cfn-signal` with a `CreationPolicy` on the resource
**C)** Add a `DependsOn` attribute to the EC2 instance
**D)** Use a CloudFormation WaitCondition with a very long timeout

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** `cfn-init` handles the instance configuration (installing packages, creating files, starting services), and `cfn-signal` reports success or failure back to CloudFormation. The `CreationPolicy` with `ResourceSignal` tells CloudFormation to wait for the signal before marking the resource as complete. If the signal indicates failure (or times out), CloudFormation rolls back. DependsOn (C) only controls creation order between resources. WaitCondition (D) works but CreationPolicy is the modern, recommended approach.
</details>

---

### Question 8

A team is evaluating AWS CDK vs raw CloudFormation for their infrastructure. Which statement is TRUE?

**A)** CDK uses a completely different deployment engine than CloudFormation
**B)** CDK synthesizes to CloudFormation templates and deploys using the CloudFormation service
**C)** CDK can deploy resources that CloudFormation cannot
**D)** CDK templates are not visible or reviewable before deployment

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** CDK is an abstraction layer on top of CloudFormation. Running `cdk synth` generates a standard CloudFormation template, and `cdk deploy` uses the CloudFormation service to deploy it. CDK uses the exact same engine (A is wrong). CDK cannot deploy resources outside CloudFormation's capabilities (C). You can always review the synthesized template with `cdk synth` (D is wrong).
</details>

---

### Question 9

A CloudFormation template uses `!ImportValue prod-VPCId` to reference a VPC from another stack. The team tries to delete the exporting stack but gets an error. Why?

**A)** The exporting stack has termination protection enabled
**B)** Exported outputs cannot be deleted while they are being imported by another stack
**C)** The VPC has running EC2 instances
**D)** The stack has a DeletionPolicy: Retain on all resources

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** CloudFormation prevents deletion of a stack if its exported outputs are being imported by other stacks via `Fn::ImportValue`. You must first remove the import references (update or delete the consuming stack) before you can delete the exporting stack. This is a dependency protection mechanism. Termination protection (A) can be disabled. Running instances (C) would be deleted as part of stack deletion. DeletionPolicy: Retain (D) retains resources but doesn't prevent stack deletion.
</details>

---

### Question 10

A Terraform configuration manages an S3 bucket. Another engineer created a DynamoDB table manually in the console. The team wants to bring this table under Terraform management WITHOUT recreating it. What command should they use?

**A)** `terraform plan`
**B)** `terraform apply`
**C)** `terraform import aws_dynamodb_table.mytable my-table-name`
**D)** `terraform refresh`

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** `terraform import` brings an existing resource into Terraform's state file without recreating it. You must also add the corresponding resource block in your `.tf` file. `terraform plan` (A) only previews changes. `terraform apply` (B) would try to create a new table. `terraform refresh` (D) updates the state for resources already managed by Terraform but cannot add new ones.
</details>

---

## Cleanup Steps

```bash
# Delete the CDK stack
cd cdk-lab && cdk destroy

# Delete CloudFormation stacks (delete importing stack first)
aws cloudformation delete-stack --stack-name lab2-cross-stack
aws cloudformation wait stack-delete-complete --stack-name lab2-cross-stack
aws cloudformation delete-stack --stack-name lab2-vpc-ec2
aws cloudformation wait stack-delete-complete --stack-name lab2-vpc-ec2

# Delete StackSet instances first, then the StackSet
aws cloudformation delete-stack-instances \
  --stack-set-name config-rules-global \
  --deployment-targets OrganizationalUnitIds=ou-xxxx-xxxxxxxx \
  --regions us-east-1 us-west-2 eu-west-1 \
  --no-retain-stacks
aws cloudformation delete-stack-set --stack-set-name config-rules-global

# Terraform
cd terraform-lab && terraform destroy
```

> **Important:** Always delete importing/dependent stacks before exporting stacks. Delete StackSet instances before the StackSet itself.
