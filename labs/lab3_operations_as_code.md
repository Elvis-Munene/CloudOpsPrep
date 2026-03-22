# Lab 3: Operations as Code

## Estimated Time: 2 hours
## Objectives
- Execute commands across EC2 fleets using SSM Run Command
- Create and run SSM Automation runbooks (built-in and custom)
- Enforce desired state with SSM State Manager
- Use Parameter Store for configuration management
- Connect to instances securely with Session Manager
- Integrate EventBridge with SSM for auto-remediation

## Prerequisites
- AWS account with EC2 instances running SSM Agent (Amazon Linux 2 has it pre-installed)
- IAM instance profile with `AmazonSSMManagedInstanceCore` policy
- AWS CLI configured

---

## Part 1: SSM Run Command

### Step 1: Verify Managed Instances

```bash
# List all SSM-managed instances
aws ssm describe-instance-information \
  --query 'InstanceInformationList[*].[InstanceId,PlatformType,PingStatus,AgentVersion]' \
  --output table
```

> **Exam Tip:** If an instance doesn't appear in SSM, check: (1) SSM Agent is installed and running, (2) Instance has an IAM role with `AmazonSSMManagedInstanceCore`, (3) Instance has outbound internet access or a VPC endpoint for SSM.

### Step 2: Run Commands Across Multiple Instances

```bash
# Run a shell script on instances tagged Environment=Production
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Environment,Values=Production" \
  --parameters 'commands=["uptime","free -m","df -h","cat /etc/os-release"]' \
  --comment "System health check" \
  --timeout-seconds 60 \
  --max-concurrency "50%" \
  --max-errors "25%" \
  --output-s3-bucket-name my-ssm-output-bucket \
  --output-s3-key-prefix run-command-logs

# Get command ID from output, then check status
aws ssm list-command-invocations \
  --command-id "COMMAND_ID" \
  --details \
  --query 'CommandInvocations[*].[InstanceId,Status,CommandPlugins[0].Output]'
```

### Key Run Command Parameters

| Parameter | Description |
|-----------|-------------|
| `--targets` | Target by tag, instance ID, or resource group |
| `--max-concurrency` | How many instances run simultaneously (number or %) |
| `--max-errors` | Stop execution if this many instances fail (number or %) |
| `--timeout-seconds` | Per-instance command timeout |
| `--output-s3-bucket-name` | Store output in S3 (useful for large outputs) |
| `--cloud-watch-output-config` | Stream output to CloudWatch Logs |

### Common SSM Documents

| Document | Purpose |
|----------|---------|
| `AWS-RunShellScript` | Run bash commands on Linux |
| `AWS-RunPowerShellScript` | Run PowerShell on Windows |
| `AWS-InstallApplication` | Install applications |
| `AWS-ConfigureAWSPackage` | Install AWS packages (CloudWatch agent, etc.) |
| `AWS-RunPatchBaseline` | Apply OS patches |
| `AWS-UpdateSSMAgent` | Update the SSM Agent itself |

> **Exam Tip:** Run Command is for one-time or on-demand execution. For recurring tasks, use State Manager associations.

---

## Part 2: SSM Automation Runbooks

### Step 1: Run Built-in Automations

```bash
# Create an AMI from an instance
aws ssm start-automation-execution \
  --document-name "AWS-CreateImage" \
  --parameters "InstanceId=i-0123456789abcdef0,NoReboot=true"

# Stop an instance
aws ssm start-automation-execution \
  --document-name "AWS-StopEC2Instance" \
  --parameters "InstanceId=[i-0123456789abcdef0]"

# Check automation status
aws ssm describe-automation-executions \
  --filters Key=ExecutionId,Values=EXECUTION_ID
```

### Step 2: Create a Custom Automation Runbook

Save as `backup-and-stop.yaml`:

```yaml
description: Create AMI backup, stop instance, and tag it
schemaVersion: '0.3'
assumeRole: '{{ AutomationAssumeRole }}'
parameters:
  InstanceId:
    type: String
    description: EC2 Instance ID to backup and stop
  AutomationAssumeRole:
    type: String
    default: ''
    description: IAM role for automation

mainSteps:
  - name: createImage
    action: aws:createImage
    maxAttempts: 3
    timeoutSeconds: 600
    inputs:
      InstanceId: '{{ InstanceId }}'
      ImageName: 'Backup-{{ InstanceId }}-{{ global:DATE_TIME }}'
      NoReboot: true
    outputs:
      - Name: ImageId
        Selector: $.ImageId
        Type: String

  - name: verifyImageAvailable
    action: aws:waitForAwsResourceProperty
    timeoutSeconds: 600
    inputs:
      Service: ec2
      Api: DescribeImages
      ImageIds:
        - '{{ createImage.ImageId }}'
      PropertySelector: '$.Images[0].State'
      DesiredValues:
        - available

  - name: stopInstance
    action: aws:changeInstanceState
    inputs:
      InstanceIds:
        - '{{ InstanceId }}'
      DesiredState: stopped

  - name: tagInstance
    action: aws:createTags
    inputs:
      ResourceType: EC2
      ResourceIds:
        - '{{ InstanceId }}'
      Tags:
        - Key: LastBackupAMI
          Value: '{{ createImage.ImageId }}'
        - Key: LastBackupDate
          Value: '{{ global:DATE_TIME }}'
        - Key: InstanceState
          Value: StoppedForMaintenance
```

```bash
# Register the custom document
aws ssm create-document \
  --content file://backup-and-stop.yaml \
  --name "Custom-BackupAndStop" \
  --document-type Automation \
  --document-format YAML

# Execute it
aws ssm start-automation-execution \
  --document-name "Custom-BackupAndStop" \
  --parameters "InstanceId=i-0123456789abcdef0"
```

### Step 3: Conditional Logic with aws:branch

```yaml
  - name: checkInstanceState
    action: aws:branch
    inputs:
      Choices:
        - NextStep: stopInstance
          Variable: '{{ describeInstance.State }}'
          StringEquals: running
        - NextStep: startInstance
          Variable: '{{ describeInstance.State }}'
          StringEquals: stopped
      Default: exitWithError
```

### Step 4: Human Approval Step

```yaml
  - name: approveAction
    action: aws:approve
    inputs:
      NotificationArn: arn:aws:sns:us-east-1:123456789012:approval-topic
      Message: "Please approve stopping instance {{ InstanceId }}"
      MinRequiredApprovals: 1
      Approvers:
        - arn:aws:iam::123456789012:role/SysOpsAdmin
```

### Automation Execution Modes

| Mode | Description |
|------|-------------|
| **Auto** | Runs all steps automatically |
| **Interactive** | Pauses before each step for approval |
| **Manual** | User must approve and advance each step |

> **Exam Tip:** Automation runbooks are for multi-step workflows (backup -> stop -> tag). Run Command is for single commands. Automation supports branching, approval, and error handling that Run Command doesn't.

---

## Part 3: SSM State Manager

### Step 1: Create an Association

```bash
# Enforce software inventory collection every 30 minutes
aws ssm create-association \
  --name "AWS-GatherSoftwareInventory" \
  --targets "Key=tag:Environment,Values=Production" \
  --schedule-expression "rate(30 minutes)" \
  --association-name "Inventory-Production"

# Enforce patch compliance weekly
aws ssm create-association \
  --name "AWS-RunPatchBaseline" \
  --targets "Key=tag:PatchGroup,Values=WebServers" \
  --schedule-expression "cron(0 2 ? * SUN *)" \
  --parameters "Operation=Scan" \
  --association-name "PatchScan-WebServers"

# Apply patches (not just scan)
aws ssm create-association \
  --name "AWS-RunPatchBaseline" \
  --targets "Key=tag:PatchGroup,Values=WebServers" \
  --schedule-expression "cron(0 4 ? * SUN *)" \
  --parameters "Operation=Install" \
  --association-name "PatchInstall-WebServers" \
  --max-concurrency "25%" \
  --max-errors "10%"
```

### Step 2: View Compliance

```bash
# Check compliance summary
aws ssm list-compliance-summaries

# Get detailed compliance for an instance
aws ssm list-compliance-items \
  --resource-ids i-0123456789abcdef0 \
  --resource-types ManagedInstance \
  --filters "Key=ComplianceType,Values=Patch"
```

> **Exam Tip:** State Manager = desired state enforcement (recurring). Run Command = one-time execution. State Manager associations run on a schedule and report compliance status.

---

## Part 4: SSM Parameter Store

### Step 1: Create Parameters

```bash
# String parameter
aws ssm put-parameter \
  --name "/app/prod/db/endpoint" \
  --type String \
  --value "mydb.cluster-abc123.us-east-1.rds.amazonaws.com"

# StringList parameter
aws ssm put-parameter \
  --name "/app/prod/allowed-origins" \
  --type StringList \
  --value "https://example.com,https://api.example.com"

# SecureString parameter (encrypted with KMS)
aws ssm put-parameter \
  --name "/app/prod/db/password" \
  --type SecureString \
  --value "MyS3cr3tP@ss!" \
  --key-id "alias/aws/ssm"

# With custom KMS key
aws ssm put-parameter \
  --name "/app/prod/api-key" \
  --type SecureString \
  --value "sk-abc123xyz" \
  --key-id "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123"
```

### Step 2: Retrieve Parameters

```bash
# Get single parameter
aws ssm get-parameter --name "/app/prod/db/endpoint"

# Get SecureString (decrypted)
aws ssm get-parameter \
  --name "/app/prod/db/password" \
  --with-decryption

# Get all parameters in a hierarchy
aws ssm get-parameters-by-path \
  --path "/app/prod" \
  --recursive \
  --with-decryption
```

### Step 3: Reference in CloudFormation

```yaml
Parameters:
  DBEndpoint:
    Type: AWS::SSM::Parameter::Value<String>
    Default: /app/prod/db/endpoint

# Or use dynamic references
Resources:
  MyInstance:
    Type: AWS::EC2::Instance
    Properties:
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          DB_ENDPOINT={{resolve:ssm:/app/prod/db/endpoint}}
          DB_PASSWORD={{resolve:ssm-secure:/app/prod/db/password}}
```

### Parameter Store vs Secrets Manager

| Feature | Parameter Store | Secrets Manager |
|---------|----------------|-----------------|
| **Cost** | Free tier (Standard), $0.05/10K for Advanced | $0.40/secret/month |
| **Rotation** | No built-in rotation | Built-in automatic rotation with Lambda |
| **Max size** | 4KB (Standard), 8KB (Advanced) | 64KB |
| **Cross-account** | No native sharing | Resource-based policies for sharing |
| **Versioning** | Yes (labels) | Yes (staging labels) |
| **Best for** | Config values, non-rotating secrets | Database credentials, API keys that rotate |

> **Exam Tip:** If the question mentions "automatic rotation" of credentials, the answer is Secrets Manager. If it's about configuration values or hierarchical storage, it's Parameter Store.

---

## Part 5: Session Manager

### Step 1: Start a Session

```bash
# Start interactive session (no SSH key needed, no port 22 needed)
aws ssm start-session --target i-0123456789abcdef0

# Port forwarding (access RDS through EC2 bastion)
aws ssm start-session \
  --target i-0123456789abcdef0 \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"portNumber":["3306"],"localPortNumber":["3306"],"host":["mydb.abc123.us-east-1.rds.amazonaws.com"]}'
```

### Step 2: Configure Session Logging

```bash
# Update SSM Session Manager preferences
aws ssm update-document \
  --name "SSM-SessionManagerRunShell" \
  --document-version '$LATEST' \
  --content '{
    "schemaVersion": "1.0",
    "description": "Session Manager settings",
    "sessionType": "Standard_Stream",
    "inputs": {
      "s3BucketName": "my-session-logs-bucket",
      "s3KeyPrefix": "session-logs",
      "cloudWatchLogGroupName": "/ssm/session-logs",
      "cloudWatchEncryptionEnabled": true,
      "idleSessionTimeout": "20",
      "runAsEnabled": true,
      "runAsDefaultUser": "ec2-user"
    }
  }'
```

### Session Manager IAM Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:StartSession"
      ],
      "Resource": [
        "arn:aws:ec2:*:*:instance/*"
      ],
      "Condition": {
        "StringLike": {
          "ssm:resourceTag/Environment": ["Production"]
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ssm:TerminateSession",
        "ssm:ResumeSession"
      ],
      "Resource": "arn:aws:ssm:*:*:session/${aws:username}-*"
    }
  ]
}
```

> **Exam Tip:** Session Manager eliminates the need for SSH keys, bastion hosts, and opening port 22. All sessions are logged in CloudTrail. Session Manager + VPC endpoints = fully private access without any internet connectivity.

---

## Part 6: EventBridge + SSM Integration

### Rule 1: Auto-remediate Non-Compliant Config Rules

```bash
aws events put-rule \
  --name "config-noncompliant-remediation" \
  --event-pattern '{
    "source": ["aws.config"],
    "detail-type": ["Config Rules Compliance Change"],
    "detail": {
      "messageType": ["ComplianceChangeNotification"],
      "newEvaluationResult": {
        "complianceType": ["NON_COMPLIANT"]
      }
    }
  }'

# Target: SSM Automation to remediate
aws events put-targets \
  --rule "config-noncompliant-remediation" \
  --targets '[{
    "Id": "ssm-remediation",
    "Arn": "arn:aws:ssm:us-east-1:123456789012:automation-definition/Custom-RemediateConfig",
    "RoleArn": "arn:aws:iam::123456789012:role/EventBridgeSSMRole",
    "InputTransformer": {
      "InputPathsMap": {
        "resource": "$.detail.resourceId",
        "rule": "$.detail.configRuleName"
      },
      "InputTemplate": "{\"ResourceId\": [\"<resource>\"], \"RuleName\": [\"<rule>\"]}"
    }
  }]'
```

### Rule 2: Auto-tag New EC2 Instances

```bash
aws events put-rule \
  --name "auto-tag-ec2" \
  --event-pattern '{
    "source": ["aws.ec2"],
    "detail-type": ["EC2 Instance State-change Notification"],
    "detail": {
      "state": ["running"]
    }
  }'

# Target: SSM Automation to tag
aws events put-targets \
  --rule "auto-tag-ec2" \
  --targets '[{
    "Id": "tag-instance",
    "Arn": "arn:aws:ssm:us-east-1:123456789012:automation-definition/AWS-SetRequiredTags",
    "RoleArn": "arn:aws:iam::123456789012:role/EventBridgeSSMRole",
    "InputTransformer": {
      "InputPathsMap": {
        "instanceId": "$.detail.instance-id"
      },
      "InputTemplate": "{\"InstanceId\": [\"<instanceId>\"], \"RequiredTags\": [\"ManagedBy=Automation\"]}"
    }
  }]'
```

### Rule 3: Scheduled Maintenance

```bash
aws events put-rule \
  --name "weekly-patch-scan" \
  --schedule-expression "cron(0 2 ? * SUN *)" \
  --description "Weekly patch scan every Sunday at 2 AM UTC"

aws events put-targets \
  --rule "weekly-patch-scan" \
  --targets '[{
    "Id": "patch-scan",
    "Arn": "arn:aws:ssm:us-east-1:123456789012:automation-definition/AWS-RunPatchBaseline",
    "RoleArn": "arn:aws:iam::123456789012:role/EventBridgeSSMRole",
    "Input": "{\"Operation\": [\"Scan\"], \"RebootOption\": [\"NoReboot\"]}"
  }]'
```

> **Exam Tip:** EventBridge + SSM Automation is the go-to pattern for auto-remediation. Config detects non-compliance -> EventBridge rule matches -> SSM Automation fixes it. This is "Operations as Code."

---

## Part 7: Review Questions

### Question 1

A SysOps administrator needs to install the CloudWatch agent on 200 EC2 instances across multiple regions. The agent configuration is stored in SSM Parameter Store. What is the MOST efficient approach?

**A)** SSH into each instance and install the agent manually
**B)** Use SSM Run Command with the `AWS-ConfigureAWSPackage` document, targeting instances by tag
**C)** Create a new AMI with the agent pre-installed and replace all instances
**D)** Use AWS Config to automatically install the agent

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** SSM Run Command with `AWS-ConfigureAWSPackage` can install the CloudWatch agent across hundreds of instances simultaneously using tag-based targeting. The agent can then pull its configuration from Parameter Store. SSH (A) doesn't scale. Creating a new AMI (C) is disruptive and time-consuming. AWS Config (D) evaluates compliance but doesn't install software.
</details>

---

### Question 2

What is the difference between SSM Run Command and SSM Automation?

**A)** Run Command can only target one instance; Automation can target multiple
**B)** Run Command executes commands on instances; Automation orchestrates multi-step workflows across AWS services
**C)** Run Command is free; Automation requires an additional license
**D)** Run Command requires SSH; Automation uses the SSM Agent

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Run Command executes commands ON instances (like running shell scripts). Automation orchestrates workflows ACROSS AWS services (create AMI, stop instance, modify security group, wait for approval, etc.). Both can target multiple instances (A is wrong). Both are included in SSM at no extra cost (C). Neither requires SSH - both use the SSM Agent (D).
</details>

---

### Question 3

A company stores database credentials in SSM Parameter Store as SecureString. They need to implement automatic rotation of these credentials every 30 days. What should they do?

**A)** Create an SSM Automation runbook that rotates the password on a schedule
**B)** Migrate the credentials to AWS Secrets Manager, which has built-in automatic rotation
**C)** Use Parameter Store's built-in rotation feature
**D)** Create a CloudWatch alarm that triggers a Lambda function every 30 days

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** AWS Secrets Manager has built-in automatic rotation using Lambda functions, with pre-built rotation templates for RDS, Redshift, and DocumentDB. Parameter Store does NOT have built-in rotation (C is wrong). While you could build rotation with SSM Automation (A) or Lambda (D), Secrets Manager is the purpose-built, least operational overhead solution.
</details>

---

### Question 4

An operations team wants to ensure all EC2 instances in their production environment have the latest security patches applied. They want compliance reporting and automatic patch installation every Sunday at 3 AM. What combination of SSM features should they use?

**A)** Run Command with a cron schedule
**B)** State Manager association with `AWS-RunPatchBaseline` document on a schedule
**C)** Automation runbook triggered by EventBridge
**D)** Maintenance Window with `AWS-RunPatchBaseline`

<details>
<summary>Show Answer</summary>

**Correct Answer: B** (D is also acceptable)

**Explanation:** State Manager associations with `AWS-RunPatchBaseline` provide both scheduled patching AND compliance reporting. The association runs on a cron schedule and reports compliance status in the SSM Compliance dashboard. Maintenance Windows (D) also work and offer more control (service windows, task priorities). Run Command (A) doesn't have built-in scheduling. Automation + EventBridge (C) works but adds unnecessary complexity for this use case.
</details>

---

### Question 5

A SysOps administrator needs to connect to an EC2 instance in a private subnet with no internet access. The instance has no public IP and no NAT gateway. SSH is not configured. How can they connect?

**A)** It's impossible without internet access
**B)** Set up a bastion host in a public subnet
**C)** Use SSM Session Manager with VPC endpoints for SSM, SSM Messages, and S3
**D)** Configure a VPN connection to the VPC

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** SSM Session Manager can connect to instances in private subnets without internet access by using VPC Interface Endpoints for `ssm`, `ssmmessages`, and `ec2messages` (plus a Gateway Endpoint for S3 if needed). No SSH keys, no bastion host, no NAT gateway required. The instance needs an IAM role with SSM permissions and the SSM Agent running. While bastion hosts (B) and VPN (D) work, they add operational overhead and are not the most efficient answer.
</details>

---

### Question 6

An EventBridge rule is configured to trigger an SSM Automation when AWS Config detects a non-compliant security group (open SSH to 0.0.0.0/0). The automation should revoke the offending rule. However, the automation is not being triggered. What is the MOST likely cause?

**A)** EventBridge cannot target SSM Automation
**B)** The EventBridge rule's IAM role does not have permission to start SSM Automation executions
**C)** AWS Config events are not sent to EventBridge
**D)** SSM Automation cannot modify security groups

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** EventBridge rules need an IAM role with permission to invoke the target. If the role lacks `ssm:StartAutomationExecution` permission, the rule matches but cannot trigger the automation. EventBridge absolutely can target SSM Automation (A is wrong). AWS Config compliance changes do emit events to EventBridge (C is wrong). SSM Automation can call any AWS API via `aws:executeAwsApi` (D is wrong).
</details>

---

### Question 7

Which SSM Parameter Store feature allows you to organize parameters and retrieve all related values with a single API call?

**A)** Parameter tags
**B)** Parameter hierarchies with `GetParametersByPath`
**C)** Parameter labels
**D)** Parameter policies

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Parameter hierarchies (e.g., `/app/prod/db/endpoint`, `/app/prod/db/password`) combined with `GetParametersByPath` allow you to retrieve all parameters under a path prefix in a single call. Tags (A) are for metadata but not used for hierarchical retrieval. Labels (C) are for versioning. Policies (D) are for expiration and notification (Advanced tier only).
</details>

---

### Question 8

A maintenance window is configured to run SSM Automation documents. The window has a duration of 2 hours and allows a maximum of 1 hour for the task. What happens if the automation takes 90 minutes?

**A)** The automation completes normally since it's within the window duration
**B)** The automation is cancelled after 1 hour (the task timeout)
**C)** The maintenance window extends automatically
**D)** The automation runs to completion but is marked as failed

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Maintenance windows enforce the task cutoff time. If a task exceeds its configured timeout (1 hour), it is cancelled even if the maintenance window itself hasn't ended. The window does not extend automatically (C). To avoid this, increase the task timeout or optimize the automation to complete faster.
</details>

---

### Question 9

A company uses SSM Inventory to collect data from managed instances. They want to aggregate inventory data from multiple AWS accounts and regions into a single location for analysis. What should they use?

**A)** SSM Resource Data Sync to an S3 bucket
**B)** CloudWatch cross-account dashboards
**C)** AWS Config aggregator
**D)** CloudTrail organization trail

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** SSM Resource Data Sync exports inventory data and compliance data to a centralized S3 bucket. You can configure data sync from multiple accounts and regions to a single S3 bucket, then use Amazon Athena or QuickSight for analysis. CloudWatch dashboards (B) show metrics, not inventory. Config aggregator (C) aggregates Config rules/compliance, not SSM inventory. CloudTrail (D) logs API calls.
</details>

---

### Question 10

An automation runbook needs human approval before proceeding to stop a production database. Which automation action should be used?

**A)** `aws:sleep` with a long duration
**B)** `aws:approve` with SNS notification and approver list
**C)** `aws:waitForAwsResourceProperty`
**D)** `aws:branch` with manual condition

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** The `aws:approve` action pauses the automation and sends an SNS notification to approvers. The automation only continues when the required number of approvals is received (or it times out and fails). `aws:sleep` (A) just pauses for a fixed time with no approval mechanism. `aws:waitForAwsResourceProperty` (C) waits for a resource to reach a specific state. `aws:branch` (D) handles conditional logic, not approvals.
</details>

---

## Cleanup Steps

```bash
# Delete SSM documents
aws ssm delete-document --name "Custom-BackupAndStop"

# Delete associations
aws ssm delete-association --association-id ASSOCIATION_ID

# Delete parameters
aws ssm delete-parameters --names "/app/prod/db/endpoint" "/app/prod/db/password" "/app/prod/api-key" "/app/prod/allowed-origins"

# Delete EventBridge rules (remove targets first)
aws events remove-targets --rule "config-noncompliant-remediation" --ids "ssm-remediation"
aws events delete-rule --name "config-noncompliant-remediation"
aws events remove-targets --rule "auto-tag-ec2" --ids "tag-instance"
aws events delete-rule --name "auto-tag-ec2"
aws events remove-targets --rule "weekly-patch-scan" --ids "patch-scan"
aws events delete-rule --name "weekly-patch-scan"
```
