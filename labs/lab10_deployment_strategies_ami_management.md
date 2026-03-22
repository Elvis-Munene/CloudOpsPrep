# Lab 10: Deployment Strategies and AMI Management

## Estimated Time: 1 hour 30 minutes
## Objectives
- Build automated AMI pipelines with EC2 Image Builder
- Implement blue/green and rolling deployment strategies
- Manage Lambda deployments with versions and aliases
- Share resources across accounts with RAM and StackSets
- Design effective tagging strategies

## Prerequisites
- AWS account with admin access
- AWS CLI configured
- Existing ALB + ASG setup (from Lab 6 or create new)

---

## Part 1: EC2 Image Builder

### Overview

EC2 Image Builder automates the creation, testing, and distribution of AMIs.

```
Recipe (Base AMI + Components) -> Pipeline -> Build -> Test -> Distribute AMI
```

### Step 1: Create a Build Component

```bash
# Component that installs and configures a web server
aws imagebuilder create-component \
  --name "install-webserver" \
  --semantic-version "1.0.0" \
  --platform "Linux" \
  --data '
name: InstallWebServer
schemaVersion: 1.0
phases:
  - name: build
    steps:
      - name: InstallHTTPD
        action: ExecuteBash
        inputs:
          commands:
            - yum update -y
            - yum install -y httpd php
            - systemctl enable httpd
            - echo "<?php phpinfo(); ?>" > /var/www/html/index.php
      - name: InstallCloudWatchAgent
        action: ExecuteBash
        inputs:
          commands:
            - yum install -y amazon-cloudwatch-agent
      - name: InstallSSMAgent
        action: ExecuteBash
        inputs:
          commands:
            - yum install -y amazon-ssm-agent
            - systemctl enable amazon-ssm-agent
  - name: validate
    steps:
      - name: TestHTTPD
        action: ExecuteBash
        inputs:
          commands:
            - systemctl start httpd
            - curl -f http://localhost/ || exit 1
            - systemctl stop httpd
'
```

### Step 2: Create an Image Recipe

```bash
# Get the latest Amazon Linux 2 AMI ARN
BASE_AMI=$(aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2 \
  --query 'Parameter.Value' --output text)

aws imagebuilder create-image-recipe \
  --name "webserver-recipe" \
  --semantic-version "1.0.0" \
  --parent-image $BASE_AMI \
  --components '[
    {"componentArn": "arn:aws:imagebuilder:us-east-1:ACCOUNT:component/install-webserver/1.0.0"}
  ]' \
  --block-device-mappings '[{
    "deviceName": "/dev/xvda",
    "ebs": {"volumeSize": 20, "volumeType": "gp3", "encrypted": true}
  }]'
```

### Step 3: Create Infrastructure Configuration

```bash
aws imagebuilder create-infrastructure-configuration \
  --name "webserver-infra" \
  --instance-types '["t3.micro"]' \
  --instance-profile-name "EC2ImageBuilderRole" \
  --subnet-id "subnet-abc123" \
  --security-group-ids '["sg-abc123"]' \
  --terminate-instance-on-failure true \
  --sns-topic-arn "arn:aws:sns:us-east-1:ACCOUNT:image-builder-notifications"
```

### Step 4: Create Distribution Settings

```bash
# Distribute AMI to multiple regions and accounts
aws imagebuilder create-distribution-configuration \
  --name "webserver-distribution" \
  --distributions '[
    {
      "region": "us-east-1",
      "amiDistributionConfiguration": {
        "name": "webserver-{{imagebuilder:buildDate}}",
        "amiTags": {"CreatedBy": "ImageBuilder", "Version": "{{imagebuilder:buildVersion}}"},
        "launchPermission": {"userIds": ["111111111111", "222222222222"]}
      }
    },
    {
      "region": "us-west-2",
      "amiDistributionConfiguration": {
        "name": "webserver-{{imagebuilder:buildDate}}",
        "amiTags": {"CreatedBy": "ImageBuilder"}
      }
    }
  ]'
```

### Step 5: Create and Run Pipeline

```bash
aws imagebuilder create-image-pipeline \
  --name "webserver-pipeline" \
  --image-recipe-arn "arn:aws:imagebuilder:us-east-1:ACCOUNT:image-recipe/webserver-recipe/1.0.0" \
  --infrastructure-configuration-arn "arn:aws:imagebuilder:us-east-1:ACCOUNT:infrastructure-configuration/webserver-infra" \
  --distribution-configuration-arn "arn:aws:imagebuilder:us-east-1:ACCOUNT:distribution-configuration/webserver-distribution" \
  --schedule '{"scheduleExpression": "cron(0 2 ? * MON *)", "pipelineExecutionStartCondition": "EXPRESSION_MATCH_ONLY"}' \
  --status ENABLED

# Manually trigger a build
aws imagebuilder start-image-pipeline-execution \
  --image-pipeline-arn "arn:aws:imagebuilder:us-east-1:ACCOUNT:image-pipeline/webserver-pipeline"
```

> **Exam Tip:** Image Builder automates the "golden AMI" process: build -> test -> distribute. It replaces manual AMI creation with a repeatable pipeline. Common exam scenario: "How to ensure all instances use a hardened, up-to-date AMI?"

---

## Part 2: Deployment Strategies

### Strategy 1: Blue/Green with ALB

```bash
# Current setup: ALB -> Blue Target Group (v1)
# Goal: Deploy v2 with zero downtime

# Step 1: Create Green Target Group
GREEN_TG=$(aws elbv2 create-target-group \
  --name "app-green" \
  --protocol HTTP --port 80 \
  --vpc-id vpc-abc123 \
  --health-check-path "/health" \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

# Step 2: Create new ASG with v2 launch template
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name "app-green-asg" \
  --launch-template LaunchTemplateId=lt-green123,Version='$Latest' \
  --min-size 2 --max-size 4 --desired-capacity 2 \
  --vpc-zone-identifier "subnet-private1,subnet-private2" \
  --target-group-arns "$GREEN_TG" \
  --health-check-type ELB \
  --health-check-grace-period 300

# Step 3: Wait for green instances to be healthy
aws elbv2 describe-target-health --target-group-arn $GREEN_TG

# Step 4: Switch ALB listener to green target group
LISTENER_ARN=$(aws elbv2 describe-listeners \
  --load-balancer-arn $ALB_ARN \
  --query 'Listeners[0].ListenerArn' --output text)

aws elbv2 modify-listener \
  --listener-arn $LISTENER_ARN \
  --default-actions "Type=forward,TargetGroupArn=$GREEN_TG"

# Step 5: After validation, delete the blue ASG
# aws autoscaling delete-auto-scaling-group --auto-scaling-group-name "app-blue-asg" --force-delete
```

### Strategy 2: Rolling Update with ASG Instance Refresh

```bash
# Update the launch template to v2
aws ec2 create-launch-template-version \
  --launch-template-id lt-abc123 \
  --source-version 1 \
  --launch-template-data '{"ImageId": "ami-newversion123"}'

# Set the default version
aws ec2 modify-launch-template \
  --launch-template-id lt-abc123 \
  --default-version 2

# Start instance refresh (rolling replacement)
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name "my-asg" \
  --preferences '{
    "MinHealthyPercentage": 90,
    "InstanceWarmup": 120,
    "MaxHealthyPercentage": 110,
    "SkipMatching": true
  }'

# Monitor progress
aws autoscaling describe-instance-refreshes \
  --auto-scaling-group-name "my-asg" \
  --query 'InstanceRefreshes[0].[Status,PercentageComplete,StatusReason]'
```

### Strategy 3: CloudFormation Update Policies

```yaml
Resources:
  AutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    UpdatePolicy:
      # Rolling update: replace instances in batches
      AutoScalingRollingUpdate:
        MinInstancesInService: 1
        MaxBatchSize: 1
        PauseTime: PT10M
        WaitOnResourceSignals: true
        SuspendProcesses:
          - HealthCheck
          - ReplaceUnhealthy
          - AlarmNotification
          - AZRebalance
          - ScheduledActions

      # OR: Full replacement (blue/green via CloudFormation)
      # AutoScalingReplacingUpdate:
      #   WillReplace: true

    CreationPolicy:
      ResourceSignal:
        Timeout: PT10M
        Count: 1
```

### CloudFormation Rollback Triggers

```yaml
  AutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      # ...
    # Rollback if alarm triggers during/after update
    RollbackConfiguration:
      RollbackTriggers:
        - Arn: !GetAtt HighErrorRateAlarm.Arn
          Type: AWS::CloudWatch::Alarm
      MonitoringTimeInMinutes: 10
```

### Deployment Strategy Comparison

| Strategy | Downtime | Rollback | Cost | Risk |
|----------|----------|----------|------|------|
| **In-place** | Yes | Slow (redeploy old) | Low | High |
| **Rolling** | No | Medium (stop and rollback) | Low | Medium |
| **Blue/Green** | No | Fast (switch back) | 2x during deploy | Low |
| **Canary** | No | Fast (shift back) | Low | Very Low |
| **Immutable** | No | Fast (terminate new) | 2x during deploy | Low |

> **Exam Tip:** "LEAST downtime" -> Blue/Green or Rolling. "FASTEST rollback" -> Blue/Green. "MOST cost-effective" -> Rolling or In-place. "Safest" -> Canary (test with small % first).

---

## Part 3: Lambda Deployment

### Versions and Aliases

```bash
# Publish a version (immutable snapshot)
VERSION=$(aws lambda publish-version \
  --function-name my-function \
  --description "v1 - initial release" \
  --query 'Version' --output text)

# Create alias pointing to version
aws lambda create-alias \
  --function-name my-function \
  --name "production" \
  --function-version $VERSION

# Deploy new code and publish v2
aws lambda update-function-code \
  --function-name my-function \
  --zip-file fileb://function-v2.zip
aws lambda wait function-updated --function-name my-function

VERSION_2=$(aws lambda publish-version \
  --function-name my-function \
  --description "v2 - new feature" \
  --query 'Version' --output text)
```

### Canary Deployment with Weighted Alias

```bash
# Shift 10% of traffic to v2
aws lambda update-alias \
  --function-name my-function \
  --name "production" \
  --function-version $VERSION \
  --routing-config "AdditionalVersionWeights={\"$VERSION_2\"=0.1}"

# Monitor errors for v2 in CloudWatch
# If healthy, shift 100% to v2
aws lambda update-alias \
  --function-name my-function \
  --name "production" \
  --function-version $VERSION_2 \
  --routing-config "AdditionalVersionWeights={}"

# If errors, rollback to v1
aws lambda update-alias \
  --function-name my-function \
  --name "production" \
  --function-version $VERSION \
  --routing-config "AdditionalVersionWeights={}"
```

### CodeDeploy with Lambda

| Deployment Preference | Behavior |
|-----------------------|----------|
| `Canary10Percent5Minutes` | 10% for 5 min, then 100% |
| `Canary10Percent10Minutes` | 10% for 10 min, then 100% |
| `Canary10Percent15Minutes` | 10% for 15 min, then 100% |
| `Linear10PercentEvery1Minute` | Add 10% every minute |
| `Linear10PercentEvery2Minutes` | Add 10% every 2 minutes |
| `Linear10PercentEvery3Minutes` | Add 10% every 3 minutes |
| `Linear10PercentEvery10Minutes` | Add 10% every 10 minutes |
| `AllAtOnce` | Immediate 100% shift |

> **Exam Tip:** Lambda aliases with weighted routing enable canary deployments. CodeDeploy automates the traffic shifting and can auto-rollback based on CloudWatch alarms.

---

## Part 4: Multi-Account Resource Sharing

### AWS Resource Access Manager (RAM)

```bash
# Share a subnet with another account
aws ram create-resource-share \
  --name "shared-networking" \
  --resource-arns "arn:aws:ec2:us-east-1:ACCOUNT:subnet/subnet-abc123" \
  --principals "111111111111" \
  --allow-external-principals

# Share with entire Organization
aws ram create-resource-share \
  --name "shared-networking-org" \
  --resource-arns "arn:aws:ec2:us-east-1:ACCOUNT:subnet/subnet-abc123" \
  --principals "arn:aws:organizations::MGMT_ACCOUNT:organization/o-abc123"
```

### Common RAM-Shareable Resources

| Resource | Use Case |
|----------|----------|
| VPC Subnets | Centralized networking, shared VPC |
| Transit Gateways | Hub-and-spoke connectivity |
| Route 53 Resolver Rules | Shared DNS resolution |
| License Manager | Centralized license tracking |
| Aurora DB Clusters | Cross-account database access |
| AWS CodeBuild Projects | Shared build infrastructure |

### CloudFormation StackSets

```bash
# Deploy a standard Config rule to all accounts in an OU
aws cloudformation create-stack-set \
  --stack-set-name "org-config-rules" \
  --template-body file://config-rules.yaml \
  --permission-model SERVICE_MANAGED \
  --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=false

aws cloudformation create-stack-instances \
  --stack-set-name "org-config-rules" \
  --deployment-targets OrganizationalUnitIds=ou-abc123-defg456 \
  --regions us-east-1 us-west-2 eu-west-1
```

### AWS Service Catalog

```bash
# Create a portfolio
PORTFOLIO_ID=$(aws servicecatalog create-portfolio \
  --display-name "Approved Infrastructure" \
  --provider-name "Cloud Team" \
  --description "Pre-approved, compliant infrastructure templates" \
  --query 'PortfolioDetail.Id' --output text)

# Create a product (CloudFormation template)
PRODUCT_ID=$(aws servicecatalog create-product \
  --name "Standard Web Server" \
  --product-type CLOUD_FORMATION_TEMPLATE \
  --provisioning-artifact-parameters '{
    "Name": "v1.0",
    "Info": {"LoadTemplateFromURL": "https://s3.amazonaws.com/templates/webserver.yaml"},
    "Type": "CLOUD_FORMATION_TEMPLATE"
  }' --query 'ProductViewDetail.ProductViewSummary.ProductId' --output text)

# Associate product with portfolio
aws servicecatalog associate-product-with-portfolio \
  --product-id $PRODUCT_ID \
  --portfolio-id $PORTFOLIO_ID

# Grant access to an IAM principal
aws servicecatalog associate-principal-with-portfolio \
  --portfolio-id $PORTFOLIO_ID \
  --principal-arn "arn:aws:iam::ACCOUNT:role/DeveloperRole" \
  --principal-type IAM
```

> **Exam Tip:** Service Catalog = self-service provisioning with governance. Developers launch pre-approved templates without needing CloudFormation knowledge or broad IAM permissions.

---

## Part 5: Tagging Strategy

### Tag Policies with AWS Organizations

```json
{
  "tags": {
    "Environment": {
      "tag_key": {
        "@@assign": "Environment"
      },
      "tag_value": {
        "@@assign": ["Production", "Staging", "Development", "Testing"]
      },
      "enforced_for": {
        "@@assign": ["ec2:instance", "rds:db", "s3:bucket"]
      }
    },
    "Owner": {
      "tag_key": {
        "@@assign": "Owner"
      }
    },
    "CostCenter": {
      "tag_key": {
        "@@assign": "CostCenter"
      },
      "tag_value": {
        "@@assign": ["CC-100", "CC-200", "CC-300"]
      }
    }
  }
}
```

### Cost Allocation Tags

```bash
# Activate cost allocation tags in Billing console
aws ce update-cost-allocation-tags-status \
  --cost-allocation-tags-status '[
    {"TagKey": "Environment", "Status": "Active"},
    {"TagKey": "CostCenter", "Status": "Active"},
    {"TagKey": "Owner", "Status": "Active"}
  ]'
```

> **Exam Tip:** Cost allocation tags must be ACTIVATED in the billing console to appear in Cost Explorer reports. They're not automatic even if resources are tagged.

### Attribute-Based Access Control (ABAC)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:RebootInstances"
      ],
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Environment": "${aws:PrincipalTag/Environment}"
        }
      }
    }
  ]
}
```

This policy allows users to manage only instances that have the same `Environment` tag as their IAM principal. Dev team members (tagged `Environment=Development`) can only manage dev instances.

### Resource Groups

```bash
# Create a resource group based on tags
aws resource-groups create-group \
  --name "production-resources" \
  --resource-query '{
    "Type": "TAG_FILTERS_1_0",
    "Query": "{\"ResourceTypeFilters\":[\"AWS::AllSupported\"],\"TagFilters\":[{\"Key\":\"Environment\",\"Values\":[\"Production\"]}]}"
  }'

# List resources in the group
aws resource-groups list-group-resources --group-name "production-resources"
```

### Tag Editor for Bulk Operations

```bash
# Find all untagged EC2 instances
aws resourcegroupstaggingapi get-resources \
  --resource-type-filters "ec2:instance" \
  --tag-filters '[]' \
  --query 'ResourceTagMappingList[?Tags==`[]`].ResourceARN'

# Bulk tag resources
aws resourcegroupstaggingapi tag-resources \
  --resource-arn-list "arn:aws:ec2:us-east-1:ACCOUNT:instance/i-abc123" "arn:aws:ec2:us-east-1:ACCOUNT:instance/i-def456" \
  --tags '{"Environment": "Production", "Owner": "ops-team"}'
```

---

## Part 6: Review Questions

### Question 1

A company needs to ensure all EC2 instances use a hardened AMI with the latest security patches, CIS benchmarks applied, and monitoring agents pre-installed. New AMIs should be built weekly. What is the BEST solution?

**A)** Create a script that runs on each instance at boot to apply patches
**B)** Use EC2 Image Builder with a scheduled pipeline that builds, tests, and distributes the AMI weekly
**C)** Manually create AMIs from a reference instance every week
**D)** Use AWS Systems Manager Patch Manager to patch running instances

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** EC2 Image Builder provides an automated pipeline for building, testing, and distributing AMIs on a schedule. It ensures consistency, includes testing phases, and can distribute to multiple regions/accounts. Boot scripts (A) add startup time and can fail. Manual creation (C) doesn't scale and is error-prone. Patch Manager (D) patches running instances but doesn't create golden AMIs.
</details>

---

### Question 2

A SysOps administrator wants to deploy a new application version with zero downtime and the ability to instantly roll back. Cost during deployment is not a concern. Which deployment strategy should they use?

**A)** In-place deployment
**B)** Rolling deployment
**C)** Blue/Green deployment
**D)** Canary deployment

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** Blue/Green deployment runs two identical environments. The new version (green) is fully deployed and tested before switching traffic from the old version (blue). Rollback is instant: switch the ALB listener back to blue. In-place (A) has downtime. Rolling (B) has slower rollback. Canary (D) is slower to complete full deployment.
</details>

---

### Question 3

An ASG instance refresh is configured with `MinHealthyPercentage: 90` and the ASG has 10 instances. How many instances can be replaced simultaneously?

**A)** 1 instance at a time
**B)** Up to 1 instance (must maintain at least 9 healthy = 90% of 10)
**C)** Up to 2 instances
**D)** 10% of instances

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** With 10 instances and MinHealthyPercentage=90%, at least 9 instances must be healthy at all times. That means only 1 instance can be replaced at a time. During replacement, the old instance is terminated and a new one launches, so temporarily only 9 are healthy (90%).
</details>

---

### Question 4

A CloudFormation stack update policy has `AutoScalingRollingUpdate` with `WaitOnResourceSignals: true` and `PauseTime: PT15M`. A new instance launches but never sends a signal. What happens?

**A)** The update proceeds after 15 minutes
**B)** CloudFormation waits indefinitely for the signal
**C)** After 15 minutes (PauseTime), CloudFormation marks the instance as failed and rolls back the update
**D)** The instance is terminated immediately

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** When `WaitOnResourceSignals` is true, `PauseTime` acts as a timeout. CloudFormation waits up to 15 minutes for a cfn-signal from the instance. If no signal arrives, it considers the update failed and initiates a rollback. This prevents bad deployments from completing.
</details>

---

### Question 5

A Lambda function alias "production" points to version 3. The team deploys version 4 and wants to gradually shift 10% of traffic to the new version before going to 100%. How should they configure the alias?

**A)** Create a new alias "canary" pointing to version 4
**B)** Update the "production" alias to point to version 4 with routing config: 90% to v3, 10% to v4
**C)** Delete the alias and create a new one pointing to version 4
**D)** Use API Gateway to split traffic between two Lambda functions

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Lambda aliases support weighted routing via `RoutingConfig`. Update the "production" alias to keep the primary version as v3 and add `AdditionalVersionWeights` with v4 at 0.1 (10%). This provides canary testing without changing the alias ARN that other services reference. Creating a separate alias (A) requires updating all callers. Deleting (C) causes disruption.
</details>

---

### Question 6

A company uses AWS Organizations with multiple accounts. They want developers in member accounts to launch only pre-approved, compliant CloudFormation templates. What service should they use?

**A)** AWS Service Catalog with portfolios shared across accounts
**B)** CloudFormation StackSets from the management account
**C)** AWS RAM to share CloudFormation templates
**D)** SCPs to restrict which CloudFormation resources can be created

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** AWS Service Catalog lets the central team create portfolios of approved products (CloudFormation templates) and share them with member accounts. Developers get a self-service portal to launch compliant infrastructure without needing broad permissions. StackSets (B) are for centralized deployment, not self-service. RAM (C) doesn't share CFN templates. SCPs (D) can restrict but don't provide self-service.
</details>

---

### Question 7

A team activates the "Environment" tag as a cost allocation tag. They tag all resources but don't see the tag in Cost Explorer reports. What is the MOST likely reason?

**A)** Cost allocation tags are not supported for EC2 instances
**B)** It takes up to 24 hours for cost allocation tags to appear in billing reports
**C)** The tag must also be added to the Organization's tag policy
**D)** Cost Explorer is disabled for the account

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** After activating cost allocation tags, it can take up to 24 hours for them to appear in Cost Explorer and billing reports. Tags only apply to costs incurred AFTER activation - they are not retroactive. Cost allocation tags work for most resource types including EC2 (A is wrong). Tag policies (C) enforce tag compliance but aren't required for cost allocation.
</details>

---

### Question 8

What is the benefit of using Attribute-Based Access Control (ABAC) with tags instead of traditional IAM policies?

**A)** ABAC is more secure than traditional IAM policies
**B)** ABAC policies don't need to be updated when new resources are created - access is automatically granted based on matching tags
**C)** ABAC eliminates the need for IAM roles
**D)** ABAC works only with EC2 instances

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** ABAC scales automatically. When a new EC2 instance is tagged `Environment=Development`, developers tagged with the same value automatically get access. With traditional policies, you'd need to update the policy with each new resource ARN. ABAC doesn't replace IAM (C), works with many services (D), and provides the same security level (A) - it's about operational efficiency.
</details>

---

### Question 9

An EC2 Image Builder pipeline builds AMIs successfully but the AMI doesn't appear in the target account (cross-account distribution). What should be checked?

**A)** The target account must have Image Builder enabled
**B)** The distribution configuration must include the target account ID in launch permissions
**C)** AMIs cannot be shared across accounts
**D)** The target account must be in the same Organization

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** The distribution configuration must explicitly grant launch permissions to target account IDs via `amiDistributionConfiguration.launchPermission.userIds`. Without this, the AMI is created but only accessible in the source account. AMIs can be shared cross-account (C is wrong). Organization membership helps but isn't required (D is wrong).
</details>

---

### Question 10

A company shares VPC subnets from a central networking account to application accounts using AWS RAM. An application team launches an EC2 instance in the shared subnet. Who owns the instance and who owns the subnet?

**A)** The networking account owns both the subnet and the instance
**B)** The application account owns the instance; the networking account owns the subnet
**C)** The application account owns both
**D)** AWS manages ownership automatically

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** In a shared VPC model with RAM, the networking account (owner) owns the VPC, subnets, route tables, and gateways. The application account (participant) owns the resources it creates IN those subnets (EC2, RDS, Lambda ENIs, etc.). Each account manages and pays for its own resources. This is a key concept for multi-account networking.
</details>

---

## Cleanup Steps

```bash
# Delete Image Builder resources (in reverse order)
aws imagebuilder delete-image-pipeline --image-pipeline-arn PIPELINE_ARN
aws imagebuilder delete-distribution-configuration --distribution-configuration-arn DIST_ARN
aws imagebuilder delete-infrastructure-configuration --infrastructure-configuration-arn INFRA_ARN
aws imagebuilder delete-image-recipe --image-recipe-arn RECIPE_ARN
aws imagebuilder delete-component --component-build-version-arn COMPONENT_ARN

# Delete green ASG and target group
aws autoscaling delete-auto-scaling-group --auto-scaling-group-name "app-green-asg" --force-delete
aws elbv2 delete-target-group --target-group-arn $GREEN_TG

# Delete Lambda versions/aliases
aws lambda delete-alias --function-name my-function --name production

# Delete RAM resource shares
aws ram delete-resource-share --resource-share-arn SHARE_ARN

# Delete Service Catalog resources
aws servicecatalog disassociate-product-from-portfolio --product-id $PRODUCT_ID --portfolio-id $PORTFOLIO_ID
aws servicecatalog delete-product --id $PRODUCT_ID
aws servicecatalog delete-portfolio --id $PORTFOLIO_ID

# Delete resource groups
aws resource-groups delete-group --group-name "production-resources"
```
