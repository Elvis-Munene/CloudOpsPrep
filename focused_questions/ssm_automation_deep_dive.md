# AWS Systems Manager Automation - Deep Dive Questions (SOA-C03)

## 20 Scenario-Based Exam-Style Questions

---

### Question 1

A SysOps administrator needs to create a custom Systems Manager Automation runbook that performs the following tasks in sequence: stops an EC2 instance, creates an AMI, waits for the AMI to become available, and then restarts the instance. Which combination of automation actions should be used?

**A)** aws:executeAwsApi for StopInstances, aws:executeAwsApi for CreateImage, aws:executeAwsApi for StartInstances
**B)** aws:changeInstanceState (stop), aws:executeAwsApi (CreateImage), aws:sleep (300 seconds), aws:changeInstanceState (start)
**C)** aws:changeInstanceState (stop), aws:createImage, aws:waitForAwsResourceProperty, aws:changeInstanceState (start)
**D)** aws:runCommand (stop), aws:runCommand (create AMI), aws:approve, aws:runCommand (start)

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** The correct approach uses `aws:changeInstanceState` to stop the instance (which has built-in logic for instance state changes), `aws:createImage` as a specific action for creating AMIs, `aws:waitForAwsResourceProperty` to poll the AMI status until it becomes "available" (instead of a fixed sleep duration), and then `aws:changeInstanceState` to start the instance. Option A would work but requires more configuration and doesn't leverage the purpose-built `aws:createImage` action. Option B uses `aws:sleep` with a fixed duration, which is unreliable since AMI creation time varies - the AMI might not be ready in 300 seconds or you might wait longer than necessary. Option D incorrectly uses `aws:runCommand` which is for executing commands on instances via SSM Agent, not for AWS API operations, and `aws:approve` is for manual approval gates, not waiting for resource states.

</details>

---

### Question 2

A company has a Systems Manager Automation document that needs to execute a Python script to perform custom validation logic during the automation workflow. The script needs to interact with multiple AWS services including DynamoDB and SNS. Which automation action should be used?

**A)** aws:runCommand with AWS-RunShellScript document
**B)** aws:executeScript with runtime Python3.8 and the script content inline
**C)** aws:executeAwsApi to invoke a Lambda function that contains the Python script
**D)** aws:invokeLambdaFunction to call a pre-deployed Lambda function

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** The `aws:executeScript` action allows you to run scripts (Python or PowerShell) directly within the automation workflow without requiring an EC2 instance or Lambda function. You can specify the runtime (Python3.6, Python3.7, Python3.8, PowerShell Core 6.0), provide the script inline or via an attachment, and the script can use boto3 to interact with AWS services. Option A (`aws:runCommand`) requires a target EC2 instance with SSM Agent installed and is designed for executing commands on instances, not for standalone script execution in the automation workflow. Option C is unnecessarily complex and `aws:executeAwsApi` doesn't invoke Lambda functions - it calls AWS service APIs directly. Option D could work if you want to maintain the script as a separate Lambda function, but the question asks about executing a script during the automation, and `aws:executeScript` is the purpose-built action for this, providing simpler integration without deploying and managing a separate Lambda function.

</details>

---

### Question 3

A SysOps administrator is designing an automation runbook that must patch 200 EC2 instances across multiple Availability Zones. The automation should process a maximum of 10 instances at a time and should stop if more than 2 instances fail during patching. Which configuration should be used?

**A)** Set MaxConcurrency to 10 and MaxErrors to 2 in the automation execution parameters
**B)** Set Concurrency to 10% and ErrorThreshold to 1% in the runbook definition
**C)** Use aws:runCommand action with MaxConcurrency: "10" and MaxErrors: "2" in the action parameters
**D)** Set TargetParameterName with MaxConcurrency: "5%" and MaxErrors: "1%" in the automation document

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** When using multi-target automation with actions like `aws:runCommand`, you specify rate control settings directly in the action parameters using `MaxConcurrency` and `MaxErrors`. `MaxConcurrency: "10"` means 10 instances will be processed concurrently (you can use absolute numbers or percentages like "10%"). `MaxErrors: "2"` means the automation will stop if more than 2 targets fail. Option A refers to execution parameters which are used when starting an automation, but rate control for multi-target actions is specified in the document itself. Option B uses incorrect parameter names (should be MaxConcurrency and MaxErrors, not Concurrency and ErrorThreshold). Option D has incorrect values - setting MaxConcurrency to "5%" would only process 10 instances at a time if you have 200 instances (5% of 200 = 10), which matches the requirement, but MaxErrors: "1%" would mean stopping after 2 failures (1% of 200 = 2), which could work but the direct approach in Option C with absolute values is clearer and more standard for the specified requirements.

</details>

---

### Question 4

An automation runbook is executing in Auto mode and has reached an `aws:approve` action. What happens next?

**A)** The automation pauses and waits indefinitely for manual approval before continuing
**B)** The automation fails immediately because aws:approve is only supported in Interactive mode
**C)** The automation automatically approves and continues to the next step without waiting
**D)** The automation pauses for 24 hours, then times out if no approval is received

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** The `aws:approve` action is only supported when the automation is executed in Interactive mode (formerly called Manual mode in older documentation). If you try to use `aws:approve` in Auto mode, the automation will fail. Interactive mode is specifically designed for workflows that require human intervention, such as approval gates. In Interactive mode, when the automation reaches an `aws:approve` action, it pauses and waits for a user to either approve (SendAutomationSignal with Approve) or reject (SendAutomationSignal with Reject) before continuing. Auto mode is for fully automated, unattended execution without manual intervention. If you need approval logic in Auto mode, you would need to use a different approach, such as integrating with an external approval system or using Step Functions.

</details>

---

### Question 5

A company needs to automate incident response. When an EC2 instance becomes unresponsive, an EventBridge rule should trigger a Systems Manager Automation that attempts to restart the instance. If the restart fails, the automation should create a snapshot of the instance's EBS volumes and terminate the instance. Which automation action enables this branching logic?

**A)** aws:branch with a choice based on the previous action's status
**B)** aws:waitForAwsResourceProperty with conditional next step
**C)** aws:executeScript with Python script that implements if-else logic
**D)** aws:approve with two different approval actions for success and failure paths

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** The `aws:branch` action provides conditional branching logic in Systems Manager Automation runbooks. It evaluates choices based on variables (including outputs from previous steps) and can branch to different steps based on the evaluation. You can check if a previous action succeeded or failed and direct the workflow accordingly. For example, after `aws:changeInstanceState` attempts to restart, you can use `aws:branch` to check if the restart succeeded (`{{restartInstance.Status}} equals "Success"`) and either end the automation or proceed to snapshot and terminate. Option B (`aws:waitForAwsResourceProperty`) is for polling resource properties until a condition is met, not for branching logic. Option C could technically implement the logic, but using `aws:executeScript` for control flow would be complex and not the intended design - automation documents have native branching capabilities. Option D is incorrect because `aws:approve` is for manual approval gates, not for automated conditional logic based on previous step outcomes.

</details>

---

### Question 6

A SysOps administrator needs to run a PowerShell script on 50 Windows EC2 instances using Systems Manager. The instances are already managed by Systems Manager and have the SSM Agent installed. The script needs to be executed once immediately. Which approach should be used?

**A)** Systems Manager Run Command with AWS-RunPowerShellScript document
**B)** Systems Manager Automation with aws:executeScript action
**C)** Systems Manager State Manager with a one-time association
**D)** Systems Manager Maintenance Window with a Run Command task

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** Run Command is the appropriate service for executing commands or scripts on managed EC2 instances on-demand. The AWS-RunPowerShellScript document is a pre-configured SSM document specifically designed for running PowerShell scripts on Windows instances. You target the 50 instances using instance IDs, tags, or resource groups, and Run Command executes the script immediately. Option B (Automation with `aws:executeScript`) is used for running scripts as part of an automation workflow on the Systems Manager service side, not for executing scripts on EC2 instances. Option C (State Manager) is for maintaining desired state configuration over time, ensuring compliance, and running scripts on a schedule - while it can run once, it's designed for ongoing configuration management. Option D (Maintenance Window) is for scheduling tasks during specific time windows, which is unnecessary overhead for immediate one-time execution. Run Command is the simplest and most direct solution for immediate, one-time script execution on instances.

</details>

---

### Question 7

An organization uses Systems Manager State Manager to ensure that all EC2 instances have the CloudWatch agent installed and configured. After initial deployment, several instances are showing as "Non-Compliant" in the association status. What could be the cause? (Choose TWO)

**A)** The instances do not have the SSM Agent installed or it is not running
**B)** The instances are not in the same AWS Region as the State Manager association
**C)** The IAM instance profile attached to the instances lacks necessary permissions for the association
**D)** The association is configured with a schedule expression and hasn't run yet on those instances
**E)** State Manager does not support installing the CloudWatch agent

<details>
<summary>Show Answer</summary>

**Correct Answers: A, C**

**Explanation:** State Manager associations require managed instances (instances with SSM Agent installed, running, and properly configured). If SSM Agent is not installed or not running, the instance cannot receive and execute the association, resulting in a "Non-Compliant" status (Option A is correct). Additionally, the IAM instance profile must have permissions to execute the association, which includes permissions to read SSM documents, report status, and perform the actions defined in the document (such as downloading from S3 if the CloudWatch agent installer is stored there). Insufficient permissions will cause the association to fail (Option C is correct). Option B is incorrect because State Manager associations are Region-specific and instances must be in the same Region, but this would prevent the association from targeting the instances at all, not show them as Non-Compliant. Option D is incorrect because State Manager evaluates compliance based on the last execution, and newly targeted instances should be evaluated on the next scheduled run or immediately if manually executed. Option E is incorrect because State Manager fully supports installing and configuring the CloudWatch agent using documents like AWS-ConfigureAWSPackage.

</details>

---

### Question 8

A company stores database connection strings in Systems Manager Parameter Store using SecureString parameters with the default AWS-managed KMS key. A Lambda function needs to retrieve these parameters. What IAM permissions does the Lambda function's execution role require? (Choose TWO)

**A)** ssm:GetParameter or ssm:GetParameters
**B)** kms:Decrypt on the AWS-managed KMS key (aws/ssm)
**C)** ssm:DescribeParameters
**D)** kms:CreateGrant on the AWS-managed KMS key
**E)** secretsmanager:GetSecretValue

<details>
<summary>Show Answer</summary>

**Correct Answers: A, B**

**Explanation:** To retrieve SecureString parameters from Parameter Store, the Lambda function needs `ssm:GetParameter` (for single parameter) or `ssm:GetParameters` (for multiple parameters) permission to read the parameter values (Option A is correct). SecureString parameters are encrypted using KMS, so the function also needs `kms:Decrypt` permission on the KMS key used for encryption. When using the AWS-managed key (aws/ssm), you must explicitly grant `kms:Decrypt` permission (Option B is correct). Option C (`ssm:DescribeParameters`) only allows listing parameter metadata (names, types, descriptions) without retrieving the actual values, so it's not sufficient. Option D (`kms:CreateGrant`) is not required for decryption - it's used for granting other principals the ability to use the key. Option E is incorrect because this is about Parameter Store, not Secrets Manager - while they serve similar purposes, they are different services with different APIs and permissions.

</details>

---

### Question 9

A SysOps administrator creates a Systems Manager parameter with the name `/production/database/connection-string`. The administrator also creates parameters named `/production/database/username` and `/production/database/password`. An application needs to retrieve all parameters under the `/production/database/` path with a single API call. Which API call should be used?

**A)** GetParameter with the parameter name `/production/database/*`
**B)** GetParametersByPath with the path `/production/database/` and Recursive set to true
**C)** GetParameters with a comma-separated list of all parameter names
**D)** DescribeParameters with a filter for parameters beginning with `/production/database/`

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** The `GetParametersByPath` API is specifically designed for retrieving all parameters under a parameter hierarchy. By specifying the path `/production/database/` and setting `Recursive: true`, the API returns all parameters at that level and all nested levels. This is the most efficient way to retrieve multiple related parameters organized in a hierarchy. Option A is incorrect because `GetParameter` retrieves a single parameter by exact name and does not support wildcards. Option C (`GetParameters`) can retrieve multiple parameters, but requires you to specify each parameter name explicitly (up to 10 parameters per call), which is less efficient than using the hierarchy feature. Option D (`DescribeParameters`) only returns parameter metadata (name, type, last modified date, etc.) without the actual parameter values, so it cannot be used to retrieve the connection string, username, and password values. Parameter hierarchies with `GetParametersByPath` are a best practice for organizing related configuration values.

</details>

---

### Question 10

An organization wants to implement automatic expiration for temporary API keys stored in Parameter Store. The keys should be automatically deleted after 30 days. How can this be achieved?

**A)** Set the parameter TTL (Time To Live) attribute to 2592000 seconds when creating the parameter
**B)** Create a parameter policy with type "Expiration" and specify the expiration date
**C)** Use Systems Manager Automation to delete parameters based on their LastModifiedDate
**D)** Enable parameter versioning and set the retention period to 30 days

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Parameter Store supports parameter policies, which include an "Expiration" policy type. You can attach a policy to a parameter that specifies an expiration date, and when that date is reached, Parameter Store can trigger an EventBridge event, allowing you to take action (such as deleting the parameter, notifying administrators, or triggering a rotation workflow). The policy JSON looks like: `{"Type": "Expiration", "Version": "1.0", "Attributes": {"Timestamp": "2026-04-22T00:00:00.000Z"}}`. Parameter policies also support "ExpirationNotification" to warn before expiration and "NoChangeNotification" to alert if a parameter hasn't been updated within a specified time. Option A is incorrect because Parameter Store parameters do not have a TTL attribute. Option C would work as a custom solution but requires building and maintaining automation, whereas parameter policies are a native feature. Option D is incorrect because parameter versioning controls how many previous versions are retained, not automatic deletion based on time.

</details>

---

### Question 11

A SysOps administrator needs to allow developers to access EC2 instances without opening SSH port 22 in security groups and without distributing SSH keys. Which Systems Manager capability should be used?

**A)** Systems Manager Run Command with AWS-RunShellScript
**B)** Systems Manager Session Manager with IAM authentication
**C)** Systems Manager Patch Manager with remote access enabled
**D)** Systems Manager Automation with aws:executeScript action

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Session Manager provides secure, auditable instance access without requiring open inbound ports, bastion hosts, or SSH keys. Users authenticate using their IAM credentials, and sessions are established through SSM Agent over HTTPS (port 443 outbound). Session Manager provides full shell access and supports logging session activity to S3 and CloudWatch Logs. You can also control which instances users can access and what commands they can run using IAM policies. Option A (Run Command) allows executing specific commands on instances but doesn't provide interactive shell access - it's for one-off command execution, not persistent sessions. Option C (Patch Manager) is for managing OS patches and does not provide remote access capabilities. Option D (Automation with `aws:executeScript`) runs scripts as part of automation workflows on the Systems Manager service side, not on EC2 instances. Session Manager is specifically designed for secure, auditable, interactive access to instances without SSH/RDP.

</details>

---

### Question 12

An organization uses Session Manager for instance access. The security team requires that all session activity be logged and stored for 7 years. Session logs should not be modifiable after creation. Which configuration meets these requirements?

**A)** Enable Session Manager logging to CloudWatch Logs with a retention period of 7 years
**B)** Enable Session Manager logging to an S3 bucket with Object Lock enabled in compliance mode and a retention period of 7 years
**C)** Enable Session Manager logging to CloudWatch Logs and use a CloudWatch Logs subscription to stream logs to S3
**D)** Enable AWS CloudTrail to log all Session Manager API calls and store CloudTrail logs in S3

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Session Manager can send session logs (complete session transcripts) to S3 buckets. To meet the immutability requirement, you use S3 Object Lock in compliance mode, which prevents objects from being deleted or overwritten for a specified retention period, even by the root account. This ensures logs cannot be modified after creation, meeting regulatory and security requirements. The retention period can be set to 7 years. Option A stores logs in CloudWatch Logs with the correct retention period but doesn't provide immutability - CloudWatch Logs can be deleted or modified by users with appropriate permissions. Option C adds complexity without solving the immutability requirement. Option D is incorrect because CloudTrail logs API calls (such as starting and terminating sessions) but does not capture the actual session activity (commands typed, output generated) - CloudTrail logs *metadata* about sessions, not the session content itself. Session Manager's S3 logging captures the full session transcript, and Object Lock ensures immutability.

</details>

---

### Question 13

A SysOps administrator is configuring Systems Manager Inventory to collect metadata from EC2 instances. The administrator wants to collect information about installed applications, AWS components, network configuration, and Windows updates. How should this be configured?

**A)** Create a Systems Manager Automation document that queries each type of information and stores it in a DynamoDB table
**B)** Create a State Manager association using the AWS-GatherSoftwareInventory document and specify the inventory types to collect
**C)** Install the CloudWatch agent on instances and configure custom metrics for inventory data
**D)** Use Systems Manager Run Command to execute AWS-RunInventory document on a schedule

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Systems Manager Inventory uses State Manager associations with the AWS-GatherSoftwareInventory document to collect metadata from managed instances. You create an association targeting the desired instances, specify which inventory types to collect (AWS components, applications, network configuration, Windows updates, instance information, etc.), and set a schedule for inventory collection. The collected data is stored in Systems Manager and can be queried, used for compliance reporting, or synced to S3 for analysis with tools like Athena. Option A would be building a custom solution that duplicates native functionality - Systems Manager Inventory is purpose-built for this use case. Option C (CloudWatch agent) is for collecting logs and metrics, not system inventory metadata like installed applications and Windows update status. Option D is incorrect because there is no "AWS-RunInventory" document - inventory collection is configured through State Manager associations with AWS-GatherSoftwareInventory, not through Run Command.

</details>

---

### Question 14

A company has defined a patch baseline in Systems Manager Patch Manager that approves only critical and important security updates. An administrator runs a Scan operation on a fleet of instances and finds that many instances are marked as "Non-Compliant." What does this mean?

**A)** The instances have failed to install patches that are approved by the patch baseline
**B)** The instances have missing patches that are approved by the patch baseline but have not yet been installed
**C)** The SSM Agent on the instances is outdated and needs to be updated
**D)** The instances are not associated with any patch group

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** In Patch Manager, a Scan operation evaluates instances against the patch baseline to determine which approved patches are missing. If missing patches are found, the instance is marked as "Non-Compliant," but no patches are installed during a Scan. Compliance status indicates that patches approved by the baseline are available but not yet installed on the instance. To remediate, you would run an Install operation (during a maintenance window or using Run Command with AWS-RunPatchBaseline) to install the missing patches. Option A is close but incorrect wording - "failed to install" implies an installation was attempted and failed, but Scan operations don't install anything. Option C is incorrect because SSM Agent version is separate from patch compliance - while an outdated agent could cause issues, Non-Compliant specifically refers to missing patches relative to the baseline. Option D is incorrect because instances don't need to be in a patch group to be scanned or assessed for compliance - patch groups are used to apply different baselines to different instance groups.

</details>

---

### Question 15

An organization wants to automate patching of EC2 instances during a 2-hour maintenance window every Sunday at 2 AM. If patching takes longer than 2 hours, any remaining instances should be patched during the next maintenance window. Which combination of services should be used?

**A)** EventBridge scheduled rule to trigger Systems Manager Automation every Sunday at 2 AM with a 2-hour timeout
**B)** Systems Manager Maintenance Window configured with a schedule, duration of 2 hours, and a cutoff time to stop scheduling new tasks 30 minutes before the window ends
**C)** State Manager association with a cron expression for Sunday 2 AM and a 2-hour execution timeout
**D)** Lambda function scheduled by EventBridge that calls the Patch Manager API with a 2-hour timeout

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Systems Manager Maintenance Windows are specifically designed for scheduling operational tasks like patching during defined time periods. You configure the schedule (e.g., `cron(0 2 ? * SUN *)`), duration (2 hours), and a cutoff time before the window ends to stop scheduling new tasks (e.g., 30 minutes before end). Tasks registered with the maintenance window (such as Run Command tasks executing AWS-RunPatchBaseline) will execute during the window. If the window closes before all instances are patched, those instances will be patched during the next window. Maintenance Windows also provide detailed execution history and integrate with SNS for notifications. Option A (EventBridge with Automation) could work but lacks the sophisticated scheduling features of Maintenance Windows, such as cutoff times and the ability to automatically handle tasks that span multiple windows. Option C (State Manager) is for maintaining desired state configuration, not for scheduled operational tasks during specific time windows. Option D is a custom solution that requires building and maintaining Lambda functions instead of using native capabilities.

</details>

---

### Question 16

A SysOps administrator needs to distribute a custom monitoring agent package to 500 EC2 instances across multiple AWS accounts. The package should be versioned, and instances should be able to install specific versions. Which Systems Manager feature should be used?

**A)** Systems Manager Parameter Store to store the agent binary and State Manager to deploy it
**B)** Systems Manager Distributor to create a package with versioning and deploy it using associations or Run Command
**C)** S3 bucket with versioning enabled and Systems Manager Automation to download and install the agent
**D)** AWS Systems Manager Compliance with a custom compliance type for the monitoring agent

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Systems Manager Distributor is specifically designed for packaging and distributing software to managed instances. You create a package (a .zip file containing the software and installation scripts), upload it to Distributor, and Distributor handles versioning. You can then deploy packages using State Manager associations (for automated, scheduled installation) or Run Command (for one-time installation). Distributor supports installing specific package versions, uninstalling packages, and updating to new versions. Distributor packages can be shared across AWS accounts using AWS Resource Access Manager (RAM), making it ideal for multi-account deployments. Option A (Parameter Store) has size limits (standard parameters are limited to 4 KB, advanced parameters to 8 KB) and is not designed for distributing large binaries. Option C (S3 with Automation) would work but requires building custom automation documents and doesn't provide the versioning, package management, and deployment capabilities of Distributor. Option D (Compliance) is for evaluating compliance against desired state, not for software distribution.

</details>

---

### Question 17

A company wants to implement automated remediation for security group rule violations. When an EventBridge rule detects that a security group has been modified to allow SSH (port 22) from 0.0.0.0/0, an automation should remove the offending rule. Which combination provides this solution?

**A)** EventBridge rule for AWS API Call via CloudTrail (AuthorizeSecurityGroupIngress) → SNS topic → Lambda function to remove the rule
**B)** EventBridge rule for AWS API Call via CloudTrail (AuthorizeSecurityGroupIngress) → Systems Manager Automation document with aws:executeAwsApi to revoke the rule
**C)** AWS Config rule (restricted-ssh) with automatic remediation → Systems Manager Automation document with aws:executeAwsApi to revoke the rule
**D)** Systems Manager Compliance with a custom compliance type → EventBridge rule → Lambda function

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** AWS Config has a managed rule called `restricted-ssh` (or you can use `vpc-sg-open-only-to-authorized-ports`) that checks if security groups allow unrestricted SSH access (0.0.0.0/0 on port 22). When a violation is detected, Config can trigger automatic remediation using a Systems Manager Automation document. You would create an automation document that uses the `aws:executeAwsApi` action to call the EC2 `RevokeSecurityGroupIngress` API to remove the offending rule. This is the native, fully-managed approach without custom code. Option A would work but requires writing and maintaining Lambda functions, whereas Config with SSM Automation is code-free. Option B is close but monitoring CloudTrail events directly is less reliable than using Config rules - Config evaluates the actual configuration state, handles rule changes that might have occurred outside of API calls, and provides compliance tracking. Option D is incorrect because Systems Manager Compliance is for custom compliance evaluations on managed instances, not for evaluating AWS resource configurations like security groups.

</details>

---

### Question 18

A SysOps administrator needs to create a Systems Manager Automation document that executes tasks across multiple AWS accounts. The automation should update security group rules in the central security account based on findings from member accounts. How should this be configured?

**A)** Create a cross-account IAM role in each member account that the automation execution role can assume, and use aws:executeAwsApi with the AssumeRole parameter
**B)** Configure Systems Manager Resource Data Sync to aggregate data from member accounts, then execute the automation in the security account
**C)** Use AWS Organizations service control policies (SCPs) to allow the automation to access resources in member accounts
**D)** Deploy the automation document to each member account and execute it independently in each account

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** For cross-account automation, the standard approach is to create an IAM role in the target account (member account) with the necessary permissions, establish a trust relationship with the source account (where the automation runs), and have the automation assume that role to perform actions. In Systems Manager Automation documents, you can use `aws:executeAwsApi` or other actions with the appropriate cross-account role. The automation execution role in the central account needs `sts:AssumeRole` permission, and the cross-account role must trust the central account's automation execution role. Option B (Resource Data Sync) is used to aggregate inventory, compliance, and OpsData from multiple accounts into a central S3 bucket for analysis, not for executing cross-account actions. Option C (SCPs) are preventive controls that restrict what actions can be performed in member accounts - they don't grant permissions or enable cross-account access. Option D distributes the automation execution instead of centralizing it, which doesn't meet the requirement of updating security groups in a central account based on findings from member accounts.

</details>

---

### Question 19

A SysOps administrator is troubleshooting failed Systems Manager Automation executions. Where can the administrator find detailed execution logs showing step-by-step execution, input/output variables, and failure reasons?

**A)** CloudWatch Logs under the /aws/ssm/automation log group
**B)** The Automation execution history in the Systems Manager console, including step details and outputs
**C)** AWS CloudTrail logs filtered for Systems Manager API calls
**D)** Systems Manager OpsCenter as OpsItems with execution details

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Systems Manager Automation provides detailed execution history directly in the Systems Manager console. For each automation execution, you can view the overall status, start and end times, and drill down into individual steps to see their status, inputs, outputs, and failure messages. This includes the exact error message if a step fails, making it the primary location for troubleshooting automation issues. Option A is incorrect because automation executions do not automatically send logs to CloudWatch Logs (though you can configure output logging for specific actions like `aws:executeScript`). Option C (CloudTrail) logs API calls related to starting, stopping, and managing automations but does not contain the detailed step execution information and variable outputs. Option D (OpsCenter) can be integrated with Systems Manager to create OpsItems for automation executions (especially failed executions), but the detailed step-by-step execution logs are in the Automation execution history, not OpsCenter - OpsItems would reference or link to the execution history.

</details>

---

### Question 20

A company uses Systems Manager OpsCenter to manage operational issues. When a CloudWatch alarm enters the ALARM state, an OpsItem should be automatically created and assigned to the on-call engineer. If the alarm remains in ALARM state for 30 minutes, a high-priority incident ticket should be created in the company's ticketing system. How should this be implemented?

**A)** CloudWatch alarm → EventBridge rule → create OpsItem with default runbook → automation creates ticket after 30 minutes
**B)** CloudWatch alarm → SNS topic → Lambda function to create OpsItem and schedule ticket creation
**C)** CloudWatch alarm → EventBridge rule → Systems Manager Automation with aws:sleep action (30 minutes) → aws:executeAwsApi to create ticket
**D)** CloudWatch alarm → create OpsItem with EventBridge integration → EventBridge rule monitors OpsItem status → Lambda function creates ticket after 30 minutes if not resolved

<details>
<summary>Show Answer</summary>

**Correct Answer: D**

**Explanation:** Systems Manager OpsCenter integrates with EventBridge to automatically create OpsItems from various sources including CloudWatch alarms. You configure the CloudWatch alarm to send state change notifications, which EventBridge captures to create an OpsItem. OpsItems themselves emit state change events to EventBridge (e.g., when created, updated, or resolved). You can create an EventBridge rule that monitors OpsItem events - if an OpsItem related to this alarm remains open (not resolved) for 30 minutes, trigger a Lambda function to create a ticket in your external ticketing system. This approach uses native integrations and separates concerns. Option A is close but "default runbook with automation creates ticket after 30 minutes" is vague and doesn't clearly define the monitoring mechanism. Option B (SNS → Lambda) works but is less elegant than using OpsCenter's native EventBridge integration, which provides better operational visibility and tracking. Option C has a significant flaw: using `aws:sleep` for 30 minutes wastes automation execution time and resources - automation executions have limits and shouldn't be used as schedulers for long delays. EventBridge scheduled rules or Step Functions would be better for time-based workflows.

</details>

---

## Summary

These questions cover:
- Automation runbook actions (aws:executeScript, aws:runCommand, aws:executeAwsApi, aws:approve, aws:sleep, aws:branch, aws:changeInstanceState, aws:createImage, aws:waitForAwsResourceProperty)
- Execution modes (Auto vs Interactive)
- Rate control (MaxConcurrency, MaxErrors)
- Service differentiation (Run Command vs Automation vs State Manager)
- Maintenance Windows
- Parameter Store (types, hierarchies, parameter policies, encryption)
- Session Manager (security, logging, IAM)
- Inventory and Compliance
- Patch Manager (scan vs install, baselines, patch groups)
- Distributor
- EventBridge integration for auto-remediation
- Cross-account automation
- OpsCenter and OpsItems

These scenario-based questions reflect the depth and complexity expected in the AWS Certified SysOps Administrator - Associate (SOA-C03) exam.
