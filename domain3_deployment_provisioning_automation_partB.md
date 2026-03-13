# Domain 3: Deployment, Provisioning, and Automation — Part B

## Task 3.2: Automate Management of Existing Resources & Additional Task 3.1 Scenarios

### Practice Questions 26–50

---

### Question 26

A SysOps administrator needs to install a security agent on 500 Amazon EC2 instances across multiple AWS accounts and Regions. The instances are already managed by AWS Systems Manager. The administrator needs a solution that can run the installation command on all target instances simultaneously and provide output tracking.

**A)** Use AWS Systems Manager Run Command with a rate control of 500 concurrent targets and an AWS-RunShellScript document.
**B)** SSH into each instance individually using a bastion host and run the installation script.
**C)** Create an AWS Lambda function that connects to each instance via SSH and runs the installation command.
**D)** Use AWS Systems Manager State Manager to create an association that runs once immediately.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** Systems Manager Run Command is designed to run commands across large fleets of managed instances simultaneously. By using the AWS-RunShellScript document (or AWS-RunPowerShellScript for Windows), the administrator can target hundreds of instances using tags, resource groups, or instance IDs. Rate control settings allow specifying how many instances run the command concurrently. Run Command also provides detailed output and status tracking per instance, making it ideal for one-time fleet-wide operations. SSH-based approaches do not scale and are operationally burdensome.
</details>

---

### Question 27

A company uses AWS Systems Manager Patch Manager to maintain patch compliance across its Amazon EC2 fleet. The security team requires that all critical and important security patches for Amazon Linux 2 be applied within 3 days of release, while all other patches should be approved after 7 days. How should the SysOps administrator configure this?

**A)** Create a single custom patch baseline with two approval rules: one for Critical and Important severity with a 3-day auto-approval delay, and another rule for all remaining severities with a 7-day delay.
**B)** Create two separate patch baselines and assign them both to the same patch group.
**C)** Use the default AWS-provided patch baseline and modify its auto-approval delay to 3 days.
**D)** Create a single patch baseline with a 3-day approval delay and manually approve lower-severity patches after 7 days.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** Custom patch baselines in Patch Manager support multiple approval rules within a single baseline. Each rule can target specific patch severity levels (such as Critical or Important) and define a separate auto-approval delay in days. This allows the administrator to enforce the 3-day window for critical/important patches while applying a 7-day delay for all others, all within one baseline. You cannot assign two patch baselines to the same patch group. Modifying the default baseline is not recommended for custom approval logic, and manual approval does not scale.
</details>

---

### Question 28

A SysOps administrator needs to ensure that the Amazon CloudWatch agent is always installed and running on all Amazon EC2 instances tagged with `Environment=Production`. If the agent is stopped or uninstalled, it should be automatically reinstalled and started. Which approach meets this requirement?

**A)** Create an AWS Systems Manager State Manager association targeting instances with the `Environment=Production` tag, using the `AWS-ConfigureAWSPackage` document to install the CloudWatch agent, with a schedule to run every 30 minutes.
**B)** Write a cron job on each instance that checks for the CloudWatch agent every 30 minutes.
**C)** Create an Amazon EventBridge rule that monitors EC2 instance state changes and triggers a Lambda function to install the agent.
**D)** Use AWS Config to detect when the agent is not running and send an SNS notification to the operations team.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** Systems Manager State Manager is purpose-built for maintaining desired state configuration on managed instances. By creating an association with the `AWS-ConfigureAWSPackage` document targeting the Production tag, State Manager will ensure the CloudWatch agent is installed on all matching instances on a recurring schedule. If the agent is removed or stopped, the next association execution will reinstall and restart it automatically. Cron jobs require per-instance configuration and do not scale. EventBridge instance state changes do not detect agent status. AWS Config with SNS only notifies but does not remediate.
</details>

---

### Question 29

A company's security policy prohibits opening inbound SSH (port 22) or RDP (port 3389) on any Amazon EC2 instances. However, system administrators still need interactive shell access to instances for troubleshooting. Which solution satisfies both requirements?

**A)** Use AWS Systems Manager Session Manager to establish shell sessions through the SSM agent without opening any inbound ports.
**B)** Use EC2 Instance Connect to push a temporary SSH key and connect over port 22.
**C)** Set up an AWS Client VPN endpoint and allow SSH access only through the VPN.
**D)** Deploy a bastion host in a public subnet and restrict SSH access to the bastion's security group.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** Session Manager provides interactive shell access (bash or PowerShell) to managed instances through the Systems Manager agent, which communicates outbound to the Systems Manager service endpoint over HTTPS (port 443). No inbound ports need to be opened in security groups. Session Manager also provides audit logging through AWS CloudTrail and can stream session logs to Amazon S3 or CloudWatch Logs. EC2 Instance Connect still requires port 22 open. VPN and bastion host approaches still require inbound SSH or RDP ports on the target instances.
</details>

---

### Question 30

A SysOps administrator stores database connection strings in AWS Systems Manager Parameter Store as `SecureString` parameters encrypted with a customer managed AWS KMS key. A new Lambda function needs to read these parameters at runtime. The function currently returns an `AccessDeniedException` error. What are the most likely causes? **(Choose TWO.)**

**A)** The Lambda function's execution role does not have the `ssm:GetParameter` permission for the parameter ARN.
**B)** The Lambda function's execution role does not have the `kms:Decrypt` permission for the KMS key used to encrypt the parameter.
**C)** The Parameter Store parameter is stored as a standard parameter instead of an advanced parameter.
**D)** The Lambda function is running in a VPC without a NAT gateway or VPC endpoint for Systems Manager.
**E)** The Parameter Store parameter name does not begin with a forward slash.

<details>
<summary>Show Answer</summary>

**Correct Answer: A, B**

**Explanation:** To retrieve a SecureString parameter, the calling principal needs two permissions: `ssm:GetParameter` (or `ssm:GetParameters`) on the parameter resource, and `kms:Decrypt` on the KMS key that was used to encrypt it. If either permission is missing, the API call returns an AccessDeniedException. Standard vs. advanced tier does not affect access permissions. While VPC networking issues can prevent connectivity, they would result in a timeout error, not an AccessDeniedException. Parameter names do not require a leading slash for access to succeed.
</details>

---

### Question 31

A company has a maintenance window configured in AWS Systems Manager every Sunday from 02:00 to 04:00 UTC for patching. The SysOps administrator notices that some instances are not being patched during the window. The instances are tagged correctly and appear in the Systems Manager console as managed instances. What should the administrator check first?

**A)** Whether the maintenance window has the correct targets registered and that the instances match the target selection criteria (tags or resource groups).
**B)** Whether the instances have the SSM Agent version updated to the latest release.
**C)** Whether the patch baseline includes patches for the correct operating system.
**D)** Whether the maintenance window duration is long enough to patch all instances.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** The most common reason instances are not patched during a maintenance window is that the targets are not correctly registered within the window. Even though instances are managed by Systems Manager and tagged correctly, the maintenance window must have an explicit target registration that matches those instances through tags, resource groups, or specific instance IDs. The administrator should verify the maintenance window target configuration first. While SSM Agent version, patch baseline OS matching, and window duration are all valid considerations, target registration mismatches are the most frequent root cause for missed patching.
</details>

---

### Question 32

A SysOps administrator wants to automatically stop all Amazon EC2 development instances every weekday at 7:00 PM to reduce costs, and start them again at 7:00 AM. Which solution is the most operationally efficient?

**A)** Create two Amazon EventBridge rules with cron expressions: one to trigger an AWS Lambda function at 7:00 PM to stop instances tagged `Environment=Dev`, and another at 7:00 AM to start them.
**B)** Create an AWS Systems Manager maintenance window that runs a stop command at 7:00 PM and a start command at 7:00 AM.
**C)** Use AWS Auto Scaling scheduled actions to set the desired capacity to 0 at 7:00 PM and back to the original count at 7:00 AM.
**D)** Write a shell script on a dedicated EC2 instance that uses cron to call the AWS CLI stop and start commands.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** Using Amazon EventBridge (formerly CloudWatch Events) scheduled rules with a Lambda function is the most operationally efficient and serverless approach for scheduling instance start/stop operations. The Lambda function can filter instances by tags (e.g., Environment=Dev) and call the EC2 StopInstances/StartInstances APIs. This approach requires no infrastructure to maintain and is highly reliable. While Systems Manager maintenance windows could work, they are primarily designed for maintenance tasks like patching. Auto Scaling scheduled actions only apply to Auto Scaling groups. A dedicated EC2 instance running cron adds operational overhead and is a single point of failure.
</details>

---

### Question 33

An organization uses AWS Systems Manager Parameter Store to manage application configuration. The application team needs to store a configuration value that is 12 KB in size. When they attempt to create the parameter, they receive an error. What should the SysOps administrator do?

**A)** Change the parameter tier from Standard to Advanced, which supports values up to 8 KB. Then compress the configuration data to fit within the 8 KB limit.
**B)** Create the parameter as an Advanced tier parameter, which supports values up to 8 KB, and split the configuration into two parameters.
**C)** Create the parameter as an Advanced tier parameter, which supports parameter values up to 8 KB. Store the remaining data in an S3 object and reference its URI.
**D)** Store the configuration in Amazon S3 and reference the S3 URI in a Standard tier parameter.

<details>
<summary>Show Answer</summary>

**Correct Answer: D**

**Explanation:** Standard tier parameters in Parameter Store support values up to 4 KB, while Advanced tier parameters support values up to 8 KB. Since the configuration is 12 KB, it exceeds even the Advanced tier limit. The best approach is to store the large configuration object in Amazon S3 and then store the S3 object URI as the parameter value in Parameter Store. The application can then retrieve the URI from Parameter Store and fetch the actual configuration from S3. This pattern is commonly used for configuration data that exceeds Parameter Store size limits while still leveraging Parameter Store for centralized configuration management.
</details>

---

### Question 34

A SysOps administrator is configuring AWS Config rules to ensure all Amazon EBS volumes are encrypted. If a non-encrypted volume is detected, it should be automatically remediated. Which configuration achieves this?

**A)** Create an AWS Config managed rule `encrypted-volumes` and configure automatic remediation using the `AWS-EnableEBSEncryptionByDefault` Systems Manager Automation document.
**B)** Create an AWS Config managed rule `encrypted-volumes` and set up an SNS notification to alert the team to manually encrypt the volume.
**C)** Create a custom AWS Config rule backed by a Lambda function that automatically encrypts any non-encrypted EBS volume in place.
**D)** Enable EBS encryption by default in the account and use AWS Config to detect and delete any existing non-encrypted volumes.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** AWS Config supports automatic remediation actions that trigger Systems Manager Automation documents when a resource is found non-compliant. The `encrypted-volumes` managed rule evaluates whether EBS volumes are encrypted. The remediation action can invoke the `AWS-EnableEBSEncryptionByDefault` Automation document to enable default encryption for new volumes at the account level. Note that existing unencrypted volumes cannot be encrypted in place — they must be snapshotted and recreated with encryption. SNS notifications only alert and do not remediate. You cannot encrypt an EBS volume in place, and deleting volumes is destructive and inappropriate.
</details>

---

### Question 35

A SysOps administrator uploads an object to an Amazon S3 bucket and needs to automatically trigger an AWS Lambda function to process the file. The function should only be triggered for objects uploaded with a `.csv` suffix in the `incoming/` prefix. How should this be configured?

**A)** Create an S3 Event Notification on the bucket for the `s3:ObjectCreated:*` event type with a prefix filter of `incoming/` and a suffix filter of `.csv`, targeting the Lambda function.
**B)** Create an Amazon EventBridge rule that matches S3 PutObject API calls and use an input transformer to filter by prefix and suffix before invoking Lambda.
**C)** Create an S3 Event Notification for all object creation events and have the Lambda function check the object key for the prefix and suffix before processing.
**D)** Configure S3 Replication to replicate `.csv` files to another bucket that triggers the Lambda function.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** S3 Event Notifications natively support prefix and suffix filtering. By configuring the notification with a prefix of `incoming/` and a suffix of `.csv`, only objects matching both criteria will trigger the Lambda function. This is the most efficient approach because filtering happens at the S3 service level before Lambda is invoked, avoiding unnecessary invocations and costs. While option C would work functionally, it results in the Lambda function being triggered for every object upload, which is wasteful. EventBridge can also receive S3 events, but the native S3 event notification with filters is simpler and more direct for this use case.
</details>

---

### Question 36

A company uses AWS CloudFormation to manage infrastructure. A stack update fails and enters the `UPDATE_ROLLBACK_FAILED` state. The administrator identifies that the rollback failed because a resource was manually deleted outside of CloudFormation. What should the administrator do to recover the stack?

**A)** Use the `ContinueUpdateRollback` API call with the `ResourcesToSkip` parameter to skip the deleted resource, allowing the rollback to complete.
**B)** Delete the entire stack and recreate it from scratch.
**C)** Manually recreate the deleted resource with the exact same physical ID, then retry the rollback.
**D)** Contact AWS Support to reset the stack state.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** When a stack enters the `UPDATE_ROLLBACK_FAILED` state, the `ContinueUpdateRollback` action can be used to retry the rollback. The `ResourcesToSkip` parameter allows the administrator to specify resources that CloudFormation should skip during the rollback, which is essential when the resource no longer exists. Once the rollback completes successfully, the stack returns to the `UPDATE_ROLLBACK_COMPLETE` state and can be updated again. While manually recreating the resource (option C) might work in some cases, it is error-prone and not always possible to match the exact physical resource ID. Deleting and recreating the stack is disruptive and unnecessary.
</details>

---

### Question 37

A SysOps administrator is building a golden AMI pipeline using EC2 Image Builder. The pipeline must produce AMIs that are distributed to three AWS Regions (us-east-1, eu-west-1, and ap-southeast-1) and shared with two other AWS accounts. Which component of EC2 Image Builder handles this requirement?

**A)** Distribution settings that specify the target Regions for AMI replication and the target account IDs for sharing.
**B)** The image recipe, which defines the base image and components along with distribution targets.
**C)** The infrastructure configuration, which specifies the VPC and subnet in each target Region.
**D)** A post-build Lambda function that copies the AMI to other Regions and modifies AMI launch permissions.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** EC2 Image Builder uses distribution settings to control where the output AMI is copied and who it is shared with. The distribution configuration specifies the target Regions for AMI replication and can include target AWS account IDs that will receive launch permissions for the AMI. The image recipe defines the base image and build/test components, not distribution. The infrastructure configuration specifies the build environment (VPC, subnet, instance type, etc.) but not the distribution targets. While a Lambda function could handle distribution, EC2 Image Builder provides this natively through its distribution settings, making it operationally simpler.
</details>

---

### Question 38

A SysOps administrator has a CloudFormation stack that provisions an EC2 instance and an RDS database. The stack creation fails with a `CREATE_FAILED` status on the RDS resource due to an invalid `DBInstanceClass` parameter. What happens to the resources that were successfully created before the failure?

**A)** By default, CloudFormation rolls back the entire stack and deletes all resources that were successfully created, returning the stack to the `ROLLBACK_COMPLETE` state.
**B)** CloudFormation keeps all successfully created resources and marks the stack as `CREATE_FAILED`, allowing the administrator to fix and update the stack.
**C)** CloudFormation deletes only the failed resource and keeps the successfully created resources.
**D)** CloudFormation pauses the stack creation and waits for the administrator to fix the failed resource before continuing.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** By default, CloudFormation follows an all-or-nothing approach. When a resource fails during stack creation, CloudFormation automatically rolls back the entire stack by deleting all resources that were successfully created, and the stack enters the `ROLLBACK_COMPLETE` state. This ensures there are no partially provisioned stacks. However, administrators can disable automatic rollback using the `--disable-rollback` option (or the "Preserve successfully provisioned resources" option in the console) to keep successfully created resources for debugging purposes. In the default behavior, the administrator would need to fix the template and create a new stack.
</details>

---

### Question 39

A SysOps administrator needs to rotate a database password stored in AWS Systems Manager Parameter Store every 90 days. The password is stored as a `SecureString` parameter. Which approach automates this rotation?

**A)** Create an Amazon EventBridge rule with a rate expression of 90 days that triggers an AWS Lambda function to generate a new password, update the SSM parameter, and update the database credentials.
**B)** Enable automatic rotation on the SSM parameter by setting a rotation policy with a 90-day interval.
**C)** Use AWS Secrets Manager instead, which natively supports automatic rotation with configurable intervals and Lambda rotation functions.
**D)** Create an AWS Config rule that checks the parameter's last modified date and triggers remediation after 90 days.

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** While Parameter Store can store secrets as SecureString parameters, it does not natively support automatic rotation. AWS Secrets Manager is specifically designed for secrets management and includes built-in automatic rotation with configurable rotation intervals and Lambda rotation functions. Secrets Manager can automatically generate new credentials, update the secret, and update the target service (such as an RDS database). Although option A would technically work using EventBridge and Lambda, it requires custom development and is less secure and harder to maintain than using the purpose-built rotation capability in Secrets Manager. For secrets that require rotation, Secrets Manager is the recommended service.
</details>

---

### Question 40

An organization requires that all Amazon EC2 instances have the AWS Systems Manager SSM Agent installed and that instances are reporting to Systems Manager. A SysOps administrator finds that several instances are not appearing in the Systems Manager managed instances console. What should the administrator verify? **(Choose THREE.)**

**A)** The instances have an IAM instance profile attached with the `AmazonSSMManagedInstanceCore` managed policy.
**B)** The SSM Agent is installed and running on the instances.
**C)** The instances have outbound network connectivity to the Systems Manager service endpoints (directly, via NAT gateway, or via VPC endpoints).
**D)** The instances are running Amazon Linux 2, as SSM Agent is only supported on Amazon Linux.
**E)** The instances have inbound port 443 open in their security groups.

<details>
<summary>Show Answer</summary>

**Correct Answer: A, B, C**

**Explanation:** For an EC2 instance to appear as a managed instance in Systems Manager, three conditions must be met: (1) the instance must have an IAM instance profile with the necessary permissions, typically the `AmazonSSMManagedInstanceCore` managed policy; (2) the SSM Agent must be installed and running — it is pre-installed on many AMIs including Amazon Linux 2, Windows Server, and Ubuntu, but may need manual installation on others; (3) the instance must have outbound HTTPS (port 443) connectivity to the Systems Manager API endpoints, either through an internet gateway/NAT gateway or through VPC endpoints. SSM Agent supports multiple operating systems including Windows, various Linux distributions, and macOS, not just Amazon Linux. Inbound port 443 is not required because the SSM Agent initiates outbound connections.
</details>

---

### Question 41

A SysOps administrator wants to automatically remediate any Amazon S3 bucket that has public access enabled. The administrator creates an AWS Config rule `s3-bucket-public-read-prohibited` to detect non-compliant buckets. Which remediation approach should be configured?

**A)** Configure automatic remediation on the Config rule using the `AWS-DisableS3BucketPublicReadWrite` Systems Manager Automation document as the remediation action.
**B)** Create an Amazon EventBridge rule that triggers when the Config rule detects non-compliance and invoke a Lambda function to block public access.
**C)** Enable S3 Block Public Access at the account level and delete the Config rule since it is no longer needed.
**D)** Create an SNS topic that sends an email to the administrator whenever a non-compliant bucket is found.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** AWS Config supports automatic remediation using Systems Manager Automation documents. The `AWS-DisableS3BucketPublicReadWrite` document is a pre-built Automation document that disables public read and write access on an S3 bucket. When the Config rule evaluates a bucket as non-compliant, the automatic remediation action triggers the Automation document to fix the configuration. While option B would also work, it requires additional components and custom code. Account-level S3 Block Public Access (option C) is a good practice but does not address individual bucket policies that may have been configured, and removing the Config rule eliminates visibility. SNS notification (option D) is only alerting, not remediation.
</details>

---

### Question 42

A SysOps administrator is configuring AWS Systems Manager Patch Manager for a fleet of Windows Server instances. The organization requires patches to be tested in a staging environment before being applied to production. How should this be implemented?

**A)** Create two patch groups: `Staging` and `Production`. Assign the same custom patch baseline to both groups. Schedule the staging maintenance window one week before the production maintenance window to allow time for testing.
**B)** Create separate patch baselines for staging and production with different approval rules and assign each to its respective patch group.
**C)** Patch staging instances manually and then use the same Run Command to patch production instances.
**D)** Use a single maintenance window and target staging instances first by specifying execution order in the task priority.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** The recommended approach is to create two patch groups (Staging and Production) and assign the same custom patch baseline to both. By scheduling the staging maintenance window to run first (for example, one week before production), the same set of approved patches is applied to staging instances first. This gives the team time to validate the patches in the staging environment before they are automatically applied to production during the production maintenance window. Using the same baseline ensures consistency between environments. Separate baselines (option B) could lead to different patches being applied in each environment. Manual patching does not scale, and a single maintenance window does not provide adequate testing time between environments.
</details>

---

### Question 43

A SysOps administrator has an AWS CloudFormation template that creates an EC2 instance with a `UserData` script that installs and configures a web application. The stack creation completes successfully, but the application is not running on the instance. What is the most likely cause?

**A)** CloudFormation marks the EC2 instance resource as `CREATE_COMPLETE` when the instance reaches the running state, regardless of whether the UserData script has finished or succeeded. The administrator should use a `cfn-signal` helper with a `CreationPolicy` to wait for the UserData script to complete.
**B)** The UserData script was not Base64-encoded in the CloudFormation template.
**C)** CloudFormation does not support UserData scripts; the administrator should use a configuration management tool.
**D)** The EC2 instance does not have permission to download the CloudFormation helper scripts.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** By default, CloudFormation considers an EC2 instance resource successfully created as soon as the instance enters the running state. It does not wait for the UserData script to complete or verify its success. To address this, administrators should use the `cfn-signal` helper script at the end of the UserData script in combination with a `CreationPolicy` attribute on the resource. The CreationPolicy tells CloudFormation to wait for a success signal before marking the resource as CREATE_COMPLETE. If the signal is not received within the specified timeout, the creation is marked as failed. CloudFormation automatically handles Base64 encoding when using the `Fn::Base64` intrinsic function, and UserData is fully supported.
</details>

---

### Question 44

A company uses Amazon EventBridge to route events from various AWS services to targets for processing. The SysOps administrator needs to configure a rule that triggers a Lambda function whenever an IAM user's access key is created. Which event pattern should be used?

**A)** An event pattern matching the `CreateAccessKey` API call from IAM via CloudTrail as the event source.
**B)** An event pattern matching EC2 instance state changes from the `aws.ec2` source.
**C)** A scheduled rule that runs every 5 minutes and checks IAM for newly created access keys.
**D)** An event pattern matching the `aws.iam` source with the `IAM Access Key Rotated` detail type.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** IAM API calls are recorded by AWS CloudTrail, and CloudTrail events can be matched by Amazon EventBridge rules. The event pattern should match the source `aws.iam` with the detail type `AWS API Call via CloudTrail` and the specific API call `CreateAccessKey` in the event detail. This allows the rule to trigger the Lambda function in near-real-time whenever an IAM access key is created. CloudTrail must be enabled in the account for these events to be captured. A scheduled polling approach (option C) introduces delays and unnecessary Lambda invocations. Option D uses a non-existent detail type.
</details>

---

### Question 45

A SysOps administrator is building a golden AMI pipeline. The pipeline must create an AMI from a base Amazon Linux 2 image, install security patches and required software, run validation tests, and distribute the AMI. The administrator wants a fully managed AWS solution. Which service should be used?

**A)** EC2 Image Builder with an image recipe containing build components for patching and software installation, test components for validation, and distribution settings for AMI distribution.
**B)** AWS CodePipeline with CodeBuild to launch an EC2 instance, configure it, create an AMI, and copy it across Regions.
**C)** AWS Systems Manager Automation with a custom runbook that launches an instance, runs commands, creates an AMI, and shares it.
**D)** An AWS Step Functions workflow that orchestrates Lambda functions to build, test, and distribute the AMI.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** EC2 Image Builder is the fully managed AWS service designed specifically for building, testing, and distributing golden AMIs (and container images). An image recipe defines the base image and build components (such as security patching and software installation). Test components run validation checks after the build phase. Distribution settings handle cross-Region AMI replication and cross-account sharing. Image Builder also supports scheduling pipelines to run automatically when new base images are available. While options B, C, and D could all technically achieve the goal, they require significant custom development and orchestration compared to the purpose-built Image Builder service.
</details>

---

### Question 46

A SysOps administrator receives an alert that an AWS CloudFormation stack creation failed with the error: `API: ec2:RunInstances You are not authorized to perform this operation. Encoded authorization failure message: ...`. What should the administrator do to troubleshoot?

**A)** Decode the encoded authorization failure message using the `aws sts decode-authorization-message` CLI command to determine the exact permission that was denied, then update the CloudFormation service role or the calling principal's IAM policy.
**B)** Add the `AdministratorAccess` managed policy to the CloudFormation service role.
**C)** Switch to a different Region where the EC2 instance type is available.
**D)** Increase the EC2 service quota for the number of running instances.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** When AWS returns an encoded authorization failure message, it contains detailed information about which permission was denied and under what conditions. The `aws sts decode-authorization-message` command decodes this message and reveals the exact API action, resource ARN, and conditions that caused the failure. This allows the administrator to add the precise missing permission to the CloudFormation service role (if one is used) or to the IAM principal that initiated the stack creation. Granting AdministratorAccess violates the principle of least privilege. Region availability and service quotas produce different error messages and are not related to authorization failures.
</details>

---

### Question 47

A SysOps administrator needs to ensure that all Amazon EC2 instances across the organization have specific tags (e.g., `CostCenter`, `Owner`, and `Environment`). Non-compliant instances should be automatically tagged with default values. Which solution achieves this?

**A)** Create an AWS Config rule `required-tags` to detect instances missing required tags, and configure automatic remediation using a custom Systems Manager Automation document that applies default tags to non-compliant instances.
**B)** Create an IAM policy that denies EC2 instance launches unless all required tags are present.
**C)** Use AWS Service Catalog to enforce tagging during provisioning and ignore instances launched outside Service Catalog.
**D)** Run a daily AWS Lambda function that scans all EC2 instances and applies default tags to those missing required tags.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** Using an AWS Config rule with automatic remediation is the most comprehensive approach. The `required-tags` managed Config rule evaluates whether resources have the specified tags. When an instance is found non-compliant, the automatic remediation triggers a Systems Manager Automation document that applies the default tag values. This approach is continuous, automatic, and covers instances regardless of how they were launched. IAM tag enforcement (option B) is a preventive control but does not remediate existing non-compliant instances. Service Catalog only covers instances provisioned through it. A daily Lambda function introduces delays between non-compliance and remediation compared to the event-driven Config approach.
</details>

---

### Question 48

A SysOps administrator manages a fleet of 200 Amazon EC2 instances using AWS Systems Manager. The administrator runs a command using Run Command to restart a service on all 200 instances, but wants to limit the blast radius in case the command causes issues. Which Run Command features should the administrator use? **(Choose TWO.)**

**A)** Set the error threshold to stop the command execution if more than 10% of instances report failures.
**B)** Set the concurrency rate to 20% so that only 40 instances execute the command at a time.
**C)** Use an S3 bucket to store the command output for later review.
**D)** Enable SNS notifications for command status updates.
**E)** Use a custom SSM document instead of the AWS-RunShellScript document.

<details>
<summary>Show Answer</summary>

**Correct Answer: A, B**

**Explanation:** Run Command provides rate control features specifically designed to limit blast radius. The concurrency setting (either a count or percentage) controls how many instances execute the command simultaneously — setting it to 20% means only 40 of the 200 instances run the command at any given time. The error threshold (also a count or percentage) defines when Run Command should stop sending the command to additional instances — setting it to 10% means execution stops if 20 or more instances fail. Together, these controls ensure that problems are detected early and the command does not propagate failures across the entire fleet. S3 output storage and SNS notifications are useful for monitoring but do not limit blast radius. The document type does not affect rate control.
</details>

---

### Question 49

A SysOps administrator has configured an AWS CloudFormation stack with a DeletionPolicy of `Retain` on an Amazon RDS database resource. The administrator deletes the CloudFormation stack. What happens to the RDS database?

**A)** The RDS database instance is preserved and continues to run independently after the stack is deleted. It is no longer managed by CloudFormation.
**B)** The RDS database is deleted along with all other stack resources, regardless of the DeletionPolicy.
**C)** The RDS database is stopped but not deleted, and a final snapshot is taken automatically.
**D)** The CloudFormation stack deletion fails because it cannot delete the RDS resource due to the Retain policy.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** The `DeletionPolicy: Retain` attribute instructs CloudFormation to preserve the resource when the stack is deleted. The RDS database instance continues to operate independently and is no longer managed by CloudFormation. This is commonly used to prevent accidental deletion of critical stateful resources like databases. The administrator would need to manage or delete the retained resource manually afterward. Other DeletionPolicy options include `Delete` (the default, which removes the resource) and `Snapshot` (which creates a final snapshot for supported resources like RDS and EBS volumes before deletion). Stack deletion succeeds; only the retained resources are skipped.
</details>

---

### Question 50

A SysOps administrator needs to run a Systems Manager Automation document across multiple AWS accounts and Regions in an AWS Organization. The automation should patch EC2 instances in all member accounts simultaneously. Which approach is most efficient?

**A)** Use the Systems Manager Automation `TargetLocations` parameter to specify multiple accounts and Regions, leveraging the management account or delegated administrator with appropriate cross-account IAM roles.
**B)** Create separate automation executions in each account and Region manually through the console.
**C)** Write an AWS Lambda function that assumes a role in each account and invokes the automation document via the API.
**D)** Use AWS CloudFormation StackSets to deploy a CloudFormation template that runs a Systems Manager automation in each target account.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** Systems Manager Automation supports multi-account and multi-Region execution natively. By using the `TargetLocations` parameter, an administrator can specify the target AWS account IDs (or organizational units) and Regions where the automation should execute. The management account or a delegated administrator account initiates the execution, and Systems Manager uses cross-account IAM roles (typically `AWS-SystemsManager-AutomationExecutionRole`) in the target accounts to perform the actions. This is the most efficient and scalable approach, eliminating the need for custom code or manual per-account execution. While Lambda (option C) and CloudFormation StackSets (option D) could work, they add unnecessary complexity compared to the built-in multi-account capability.
</details>

---
