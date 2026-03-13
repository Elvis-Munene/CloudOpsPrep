# Domain 3: Deployment, Provisioning, and Automation (Part A)

## Task 3.1: Provision and Maintain Cloud Resources

### 25 AWS SOA-C03 Practice Questions

---

### Question 1

A SysOps administrator needs to ensure that all EC2 instances launched in the organization use a hardened, pre-approved Amazon Machine Image that includes the latest security patches and monitoring agents. The image must be automatically rebuilt every week and distributed to three AWS regions. Which service should the administrator use?

**A)** AWS Systems Manager Patch Manager with a maintenance window to patch running instances weekly
**B)** EC2 Image Builder with a pipeline scheduled weekly and distribution settings for the three target regions
**C)** A CloudFormation StackSet that launches an EC2 instance, runs a user-data script, and creates an AMI in each region
**D)** AWS Lambda function triggered by EventBridge on a weekly schedule that calls the EC2 CreateImage API

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** EC2 Image Builder is purpose-built for creating, maintaining, and distributing AMIs. A pipeline in Image Builder can be scheduled to run on a recurring basis (e.g., weekly) and includes build and test phases defined by recipes and components. The distribution configuration allows the resulting AMI to be copied to multiple AWS regions automatically. Patch Manager patches running instances but does not produce new AMIs. A CloudFormation StackSet is designed for deploying infrastructure stacks, not for image-building workflows. A Lambda function could technically create AMIs, but it would require significant custom code and lacks the built-in testing, versioning, and distribution capabilities of Image Builder.
</details>

---

### Question 2

An administrator is building an EC2 Image Builder pipeline. The image recipe must install the CloudWatch agent, apply OS patches, and run a CIS benchmark test before the AMI is published. How should the administrator structure the Image Builder configuration?

**A)** Create a single component that installs CloudWatch agent, applies patches, and runs the CIS test in one build phase
**B)** Create separate build components for installing the CloudWatch agent and applying patches, and a separate test component for the CIS benchmark
**C)** Use a launch template with a user-data script for all three tasks and reference it from the image recipe
**D)** Create a single test component that performs all three tasks during the test phase

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** EC2 Image Builder recipes reference components, which are divided into build components and test components. Build components handle software installation and configuration (such as installing the CloudWatch agent and applying OS patches), while test components validate the image (such as running CIS benchmark checks). Separating them into distinct components follows best practices for modularity and reusability. Placing everything in one component reduces reusability and mixes build and test concerns. User-data scripts in launch templates are not how Image Builder recipes define their build steps. Placing installation tasks in a test component would be architecturally incorrect since test components should only validate, not modify the image.
</details>

---

### Question 3

A company uses AWS CloudFormation to manage infrastructure. A developer updates a stack and changes the `InstanceType` property of an `AWS::EC2::Instance` resource from `t3.medium` to `t3.large`. The instance uses an EBS-backed root volume and does not have the `DisableApiStop` attribute set. What happens during the stack update?

**A)** The instance is terminated and a new instance with a new instance ID is launched (replacement)
**B)** The instance is stopped, the instance type is changed, and the instance is restarted (some interruption)
**C)** The instance type is changed with no interruption to the running instance
**D)** The stack update fails because instance type changes are not supported by CloudFormation

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** According to the CloudFormation documentation for the `AWS::EC2::Instance` resource, changing the `InstanceType` property requires "some interruption." CloudFormation will stop the instance, modify the instance type, and restart it. The instance retains its instance ID and attached EBS volumes. This is different from a replacement update, which would create an entirely new resource. Replacement updates occur when properties such as `AvailabilityZone` or `ImageId` are changed. The operation is fully supported by CloudFormation and will not cause a failure.
</details>

---

### Question 4

A SysOps administrator has a CloudFormation stack that includes an RDS database instance containing critical production data. The administrator is concerned that if the stack is accidentally deleted, the database will be lost. What should the administrator do to protect the database?

**A)** Enable termination protection on the CloudFormation stack
**B)** Set the `DeletionPolicy` attribute to `Retain` on the RDS resource in the template
**C)** Create a snapshot of the RDS instance manually before any stack operations
**D)** Use a stack policy to deny `Update:Delete` actions on the RDS resource

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Setting `DeletionPolicy: Retain` on the RDS resource ensures that when the CloudFormation stack is deleted, the RDS instance is preserved and not deleted along with the stack. This is the most direct and reliable way to protect the database from accidental stack deletion. Termination protection prevents the stack itself from being deleted, but an administrator with sufficient permissions can disable it and then delete the stack. A manual snapshot provides a backup but does not prevent the database from being deleted. A stack policy protects against update operations, not stack deletion. For RDS resources, `DeletionPolicy: Snapshot` is also an option that creates a final snapshot before deletion, but `Retain` provides the strongest protection by keeping the resource intact.
</details>

---

### Question 5

An administrator notices that a security group rule in a CloudFormation-managed stack has been modified manually through the AWS Console. The administrator wants to identify all resources in the stack that have drifted from their template-defined configuration. Which approach should the administrator take?

**A)** Compare the current template with the original template using `aws cloudformation get-template`
**B)** Run CloudFormation drift detection on the stack and review the drift results
**C)** Delete and recreate the stack to restore the original configuration
**D)** Use AWS Config rules to evaluate compliance of the stack resources

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** CloudFormation drift detection compares the current state of stack resources against their expected configuration as defined in the template. Running drift detection on the stack will identify all resources whose actual configuration has diverged from what CloudFormation expects, including the manually modified security group. The results classify each resource as IN_SYNC, MODIFIED, or DELETED. Comparing templates with `get-template` only shows the template itself, not the actual state of deployed resources. Deleting and recreating the stack is disruptive and unnecessary for detection purposes. AWS Config can evaluate compliance against specific rules, but CloudFormation drift detection is the purpose-built feature for identifying configuration drift in stack resources.
</details>

---

### Question 6

A SysOps administrator is writing a CloudFormation template and needs to conditionally create a NAT Gateway only when the environment parameter is set to "Production." The NAT Gateway's Elastic IP also should only be created in this case. Which combination of CloudFormation features should the administrator use?

**A)** Use `Fn::If` in the resource properties and a `Condition` defined in the Conditions section based on the environment parameter
**B)** Use nested stacks and deploy the NAT Gateway stack only when the parameter is "Production"
**C)** Use `Fn::Select` to choose between creating or not creating the NAT Gateway based on an index
**D)** Use a `DependsOn` attribute with a conditional parameter to control resource creation order

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** CloudFormation Conditions allow you to define boolean logic based on parameter values, and you can associate a `Condition` key directly with a resource to control whether it is created. For example, you define a condition like `IsProduction: !Equals [!Ref EnvType, "Production"]` and then attach it to both the NAT Gateway and Elastic IP resources. The `Fn::If` intrinsic function can also be used within resource properties to conditionally set property values. Nested stacks could work but add unnecessary complexity for this use case. `Fn::Select` is used to pick an item from a list by index and is not designed for conditional resource creation. `DependsOn` controls creation order, not whether a resource is created at all.
</details>

---

### Question 7

A CloudFormation template contains the following snippet:

```yaml
Outputs:
  WebsiteURL:
    Value: !Join
      - ''
      - - 'http://'
        - !Ref ALBDnsName
        - '/app'
```

What does this output produce?

**A)** It joins the ALB DNS name with the string "/app" using a hyphen separator
**B)** It concatenates "http://", the value referenced by ALBDnsName, and "/app" into a single URL string with no separator
**C)** It selects the ALBDnsName from a list and appends "/app"
**D)** It fails because `Fn::Join` cannot be used with `Ref` inside the values list

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** The `Fn::Join` intrinsic function takes two arguments: a delimiter and a list of values to concatenate. In this case, the delimiter is an empty string (`''`), so the values are concatenated directly with no separator. The list includes the literal string "http://", the resolved value of `!Ref ALBDnsName`, and the literal string "/app". The result would be something like `http://my-alb-123456.us-east-1.elb.amazonaws.com/app`. `Fn::Join` fully supports using `Ref` and other intrinsic functions within its values list. This is a very common pattern for constructing URLs and ARNs in CloudFormation templates.
</details>

---

### Question 8

An organization wants to deploy a standardized networking stack (VPC, subnets, route tables) across 12 AWS accounts in 3 regions. The deployment must be managed centrally from the management account. Which CloudFormation feature is best suited for this?

**A)** CloudFormation nested stacks deployed from the management account
**B)** CloudFormation StackSets with service-managed permissions targeting the organizational units
**C)** Individual CloudFormation stacks deployed via AWS CLI scripts in each account
**D)** AWS Service Catalog products shared across the organization

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** CloudFormation StackSets allow you to deploy CloudFormation stacks across multiple AWS accounts and regions from a single management template. With service-managed permissions (using AWS Organizations integration), StackSets can automatically deploy to all accounts within specified organizational units (OUs) without needing to manually configure IAM roles in each target account. This provides centralized management and ensures consistency across all 12 accounts and 3 regions. Nested stacks operate within a single account and region. Individual CLI scripts would be operationally complex and error-prone at this scale. AWS Service Catalog is useful for self-service provisioning but does not provide the same centralized push-deployment model that StackSets offer.
</details>

---

### Question 9

A SysOps administrator runs a CloudFormation stack update that modifies several resources. The update fails midway through when a new security group rule is rejected due to a duplicate rule. What is the default behavior of CloudFormation in this scenario?

**A)** The stack remains in the UPDATE_IN_PROGRESS state until the administrator manually intervenes
**B)** CloudFormation performs an automatic rollback to the previous known good state
**C)** The stack enters UPDATE_FAILED state and leaves resources in a partially updated configuration
**D)** CloudFormation skips the failed resource and continues updating the remaining resources

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** By default, CloudFormation automatically rolls back all changes when a stack update fails. This means any resources that were successfully modified during the update will be reverted to their previous configuration. The stack will transition through UPDATE_ROLLBACK_IN_PROGRESS and ultimately settle in UPDATE_ROLLBACK_COMPLETE status. This rollback behavior is a core safety feature of CloudFormation that ensures stacks do not remain in a partially updated, inconsistent state. Administrators can disable automatic rollback if they want to debug failures, but the default behavior prioritizes consistency. CloudFormation does not skip failed resources or leave stacks in a partially updated state by default.
</details>

---

### Question 10

Before applying a CloudFormation stack update that modifies critical production resources, an administrator wants to preview exactly which resources will be added, modified, or replaced. Which approach should the administrator use?

**A)** Run `aws cloudformation validate-template` to check for resource changes
**B)** Create a change set for the stack and review the proposed changes before executing it
**C)** Use `aws cloudformation describe-stack-resources` to compare current and desired states
**D)** Enable CloudTrail logging and perform the update, then review the logs if a rollback is needed

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** CloudFormation change sets allow an administrator to preview the impact of proposed changes to a stack before executing them. When you create a change set, CloudFormation compares the current stack template and parameters with the proposed changes and generates a detailed summary showing which resources will be added, modified (with the type of modification), or removed. The administrator can then review this summary and decide whether to execute or discard the change set. `validate-template` only checks template syntax and does not evaluate resource-level impacts. `describe-stack-resources` shows current resources but does not compare against proposed changes. Relying on CloudTrail after an update is reactive, not preventive.
</details>

---

### Question 11

A developer is using the AWS CDK with TypeScript. They define an S3 bucket using `new s3.Bucket(this, 'MyBucket')` without specifying any additional properties. What level of CDK construct is this, and what does CDK automatically configure?

**A)** L1 construct; it maps directly to the CloudFormation resource with no defaults applied
**B)** L2 construct; it applies sensible defaults such as encryption settings, lifecycle rules, and access policies
**C)** L3 construct; it creates an entire pattern including a CloudFront distribution in front of the bucket
**D)** L2 construct; it creates only the bucket with no additional configuration beyond what CloudFormation requires

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** The `s3.Bucket` class in the AWS CDK is an L2 (curated) construct. L2 constructs provide a higher-level abstraction over the raw CloudFormation resources (L1 constructs, which are prefixed with `Cfn`, e.g., `CfnBucket`). L2 constructs apply sensible defaults and offer convenient methods for common operations such as `bucket.grantRead(role)`. For an S3 bucket, CDK may apply defaults like blocking public access. L3 constructs (patterns) compose multiple resources into common architectures (e.g., `LambdaRestApi`). The key distinction is that L2 constructs provide intelligent defaults and a developer-friendly API while still representing a single primary resource.
</details>

---

### Question 12

A SysOps administrator wants to see what CloudFormation template the AWS CDK will generate from their CDK application before deploying it. They also want to compare the generated template against the currently deployed stack. Which CDK CLI commands should they use?

**A)** `cdk synth` to generate the template and `cdk diff` to compare against the deployed stack
**B)** `cdk deploy --dry-run` to preview and `cdk status` to compare
**C)** `cdk build` to compile the template and `cdk plan` to compare against the deployed stack
**D)** `cdk template` to generate the output and `cdk compare` to show differences

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** The `cdk synth` command synthesizes the CDK application and outputs the generated CloudFormation template to the console (and to the `cdk.out` directory). This allows the administrator to review exactly what CloudFormation resources and configurations will be created. The `cdk diff` command compares the current CDK application's synthesized template against the currently deployed CloudFormation stack and shows the differences, similar to a `git diff`. These are the two standard CDK CLI commands for previewing changes. Commands like `cdk build`, `cdk plan`, `cdk template`, and `cdk compare` do not exist in the CDK CLI. The `cdk deploy` command performs the actual deployment and does not have a `--dry-run` flag.
</details>

---

### Question 13

A company is planning a new VPC and needs to allocate a /16 CIDR block. Within this VPC, they need to create 4 subnets of equal size across 2 Availability Zones (one public and one private subnet per AZ). What is the appropriate subnet CIDR prefix length?

**A)** /18, providing 16,382 usable IP addresses per subnet
**B)** /20, providing 4,091 usable IP addresses per subnet
**C)** /24, providing 251 usable IP addresses per subnet
**D)** /17, providing 32,766 usable IP addresses per subnet

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** A /16 CIDR block contains 65,536 IP addresses. To divide this equally into 4 subnets, each subnet needs 65,536 / 4 = 16,384 addresses, which corresponds to a /18 prefix length (2^(32-18) = 16,384). AWS reserves 5 IP addresses in each subnet (network address, VPC router, DNS server, future use, and broadcast), leaving 16,379 usable addresses per subnet. A /18 is the largest equal division that fits exactly 4 subnets into a /16 VPC. Using /20 would create smaller subnets and waste address space, while /17 subnets are too large to fit 4 into a /16 block.
</details>

---

### Question 14

An organization with a shared services account and multiple workload accounts wants to share a set of private subnets with the workload accounts so they can launch resources into the shared VPC subnets. The organization uses AWS Organizations. Which service enables this?

**A)** VPC Peering between the shared services account and each workload account
**B)** AWS Resource Access Manager (RAM) to share the subnets with the workload accounts
**C)** AWS Transit Gateway with route table associations for each workload account
**D)** CloudFormation StackSets to deploy identical subnets in each workload account

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** AWS Resource Access Manager (RAM) allows you to share AWS resources, including VPC subnets, across accounts within an AWS Organization. When subnets are shared via RAM, workload accounts can launch their own resources (EC2 instances, RDS databases, Lambda functions, etc.) directly into the shared subnets without needing to create their own VPC or subnets. The resources remain owned by the workload account, but they reside in the shared VPC's network. VPC Peering connects two VPCs but does not allow launching resources into another account's subnets. Transit Gateway provides network connectivity routing but does not enable cross-account resource placement within subnets. StackSets would deploy separate, independent subnets rather than sharing existing ones.
</details>

---

### Question 15

A company is deploying a new version of their web application running on an Auto Scaling group behind an Application Load Balancer. They want to route 10% of traffic to the new version initially, monitor error rates for 10 minutes, and then gradually shift all traffic if no issues are detected. Which deployment strategy does this describe?

**A)** Blue/green deployment
**B)** Rolling deployment
**C)** Canary deployment
**D)** All-at-once deployment

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** A canary deployment involves routing a small percentage of traffic to the new version first (in this case, 10%) while the majority of traffic continues to the existing version. The new version is monitored for a defined period, and if the metrics are healthy, traffic is gradually shifted until 100% goes to the new version. This approach minimizes risk by exposing only a small subset of users to potential issues. Blue/green deployments maintain two full environments and switch traffic all at once. Rolling deployments replace instances in batches without fine-grained traffic percentage control. All-at-once deployments update every instance simultaneously, offering no gradual rollout or monitoring period.
</details>

---

### Question 16

A SysOps administrator is performing a blue/green deployment for an application running on EC2 instances behind an Application Load Balancer. The green environment has been fully tested and is ready. What is the recommended way to cut over traffic from the blue to the green environment?

**A)** Update the DNS CNAME record to point to the green environment's load balancer
**B)** Stop all instances in the blue environment so the Auto Scaling group launches new instances with the green AMI
**C)** Modify the Auto Scaling group's launch template to use the new AMI and perform an instance refresh
**D)** Swap the target groups on the ALB listener from the blue target group to the green target group

<details>
<summary>Show Answer</summary>

**Correct Answer: D**

**Explanation:** The most efficient and fastest way to perform a blue/green cutover with an ALB is to swap the target groups associated with the ALB listener rule. The blue environment's target group is replaced with the green environment's target group, immediately shifting traffic to the new version. This approach provides instant cutover and easy rollback by simply swapping back. DNS-based cutover (option A) works but introduces TTL delays that make the transition slower and rollback less immediate. Stopping instances (option B) is disruptive and not a proper blue/green strategy. Modifying the launch template and performing an instance refresh (option C) is a rolling deployment approach, not blue/green, as it replaces instances in the same environment rather than switching between two separate environments.
</details>

---

### Question 17

A SysOps administrator is managing Terraform state for a production environment. Multiple team members need to collaborate on the same Terraform configuration. What is the recommended approach for managing Terraform state in this scenario?

**A)** Store the state file in a shared Git repository so all team members can pull the latest state
**B)** Use a remote backend such as Amazon S3 with DynamoDB for state locking
**C)** Have each team member maintain their own local state file and merge changes manually
**D)** Disable state management and use `terraform import` to reconcile resources before each apply

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** For team collaboration, Terraform state should be stored in a remote backend. Amazon S3 is a commonly used backend that provides durable, versioned storage for the state file. DynamoDB is used alongside S3 to provide state locking, which prevents concurrent operations that could corrupt the state file. When a team member runs `terraform plan` or `terraform apply`, Terraform acquires a lock in DynamoDB, preventing others from making simultaneous changes. Storing state in Git is strongly discouraged because state files can contain sensitive information and Git does not provide locking mechanisms. Individual local state files would lead to conflicts and drift. Disabling state management would break Terraform's ability to track and manage resources.
</details>

---

### Question 18

A developer runs `terraform plan` and sees that 2 resources will be added, 1 will be changed, and 1 will be destroyed. What should the developer do before applying these changes to a production environment?

**A)** Run `terraform apply -auto-approve` immediately to minimize the window of inconsistency
**B)** Review the plan output carefully, verify the resource being destroyed is expected, and then run `terraform apply`
**C)** Run `terraform refresh` to ensure the state is current, then run `terraform apply` without reviewing the plan
**D)** Run `terraform destroy` first to clean up and then run `terraform apply` to recreate everything

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** The `terraform plan` output is a critical review step before making changes to infrastructure. The developer should carefully review which resources are being added, changed, and especially destroyed to ensure the changes are intentional and expected. A resource being destroyed could indicate an unintended configuration change that might cause data loss or service disruption. Only after confirming the plan matches expectations should the developer run `terraform apply`. Using `-auto-approve` bypasses the confirmation prompt and is inappropriate for production changes. Running `terraform destroy` would remove all managed resources, which is extremely destructive. While `terraform refresh` updates state, it does not replace the need for careful plan review.
</details>

---

### Question 19

A team is using Git for their infrastructure-as-code repository. A developer has created a feature branch to add a new CloudFormation template for a microservice. The feature is complete and tested. What is the standard workflow to integrate this change into the main branch?

**A)** Push directly to the main branch from the feature branch using `git push origin main`
**B)** Create a pull request from the feature branch to the main branch for code review, then merge after approval
**C)** Delete the main branch and rename the feature branch to main
**D)** Use `git rebase main` on the feature branch and force-push to main without review

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** The standard Git workflow for integrating changes involves creating a pull request (also called a merge request) from the feature branch to the main branch. This allows team members to review the code changes, leave comments, and approve or request modifications before the code is merged. Pull requests provide an audit trail and ensure code quality through peer review, which is especially important for infrastructure-as-code changes that affect production environments. Pushing directly to main bypasses review safeguards. Deleting and renaming branches is destructive and would lose the main branch's history. Force-pushing to main is dangerous, can overwrite others' work, and also bypasses the review process.
</details>

---

### Question 20

A SysOps administrator manages a CloudFormation stack with rollback triggers configured to monitor a CloudWatch alarm on the application's HTTP 5xx error rate. The administrator initiates a stack update. During the update, the alarm transitions to the ALARM state. What happens? **(Select TWO.)**

**A)** CloudFormation pauses the update and waits for administrator approval to continue
**B)** CloudFormation automatically initiates a rollback of the stack update
**C)** The stack transitions to UPDATE_ROLLBACK_IN_PROGRESS state
**D)** CloudFormation ignores the alarm and completes the update, then rolls back afterward
**E)** CloudFormation sends an SNS notification but continues the update

<details>
<summary>Show Answer</summary>

**Correct Answers: B, C**

**Explanation:** CloudFormation rollback triggers allow you to specify CloudWatch alarms that are monitored during stack creation or update operations, and during a configurable monitoring period after the operation completes. If any of the specified alarms enter the ALARM state during the update or monitoring period, CloudFormation automatically initiates a rollback. The stack transitions to UPDATE_ROLLBACK_IN_PROGRESS state as CloudFormation reverts all resources to their previous configuration. CloudFormation does not pause for approval, does not ignore triggered alarms, and does not simply send notifications. This feature provides an automated safety net that connects application-level health metrics to infrastructure deployment decisions.
</details>

---

### Question 21

A SysOps administrator needs to deploy a serverless application stack that includes an API Gateway, Lambda functions, DynamoDB tables, and associated IAM roles. Using the AWS CDK, which approach provides the fastest development with the least boilerplate code?

**A)** Use L1 constructs (Cfn resources) for each resource to maintain full control over every property
**B)** Use L3 constructs (patterns) such as `LambdaRestApi` that compose multiple resources together
**C)** Write raw CloudFormation JSON and import it into CDK using `CfnInclude`
**D)** Use L2 constructs for each resource and manually wire all IAM permissions and integrations

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** L3 constructs (patterns) in the AWS CDK represent opinionated, higher-level abstractions that compose multiple AWS resources into common architectural patterns. For example, `LambdaRestApi` automatically creates an API Gateway REST API integrated with a Lambda function, including the necessary permissions and configurations. This dramatically reduces boilerplate code compared to defining each resource individually. L1 constructs are direct CloudFormation mappings with no defaults or convenience methods, requiring the most code. Importing raw CloudFormation defeats the purpose of using CDK. L2 constructs are a good middle ground but still require manual integration between resources, which L3 patterns handle automatically.
</details>

---

### Question 22

An administrator has a VPC with CIDR block 10.0.0.0/16 and needs to create a subnet for a small management tier that will host no more than 10 EC2 instances. Which subnet CIDR block is the most appropriate to conserve IP address space?

**A)** 10.0.1.0/24 (256 addresses, 251 usable)
**B)** 10.0.1.0/28 (16 addresses, 11 usable)
**C)** 10.0.1.0/27 (32 addresses, 27 usable)
**D)** 10.0.0.0/20 (4,096 addresses, 4,091 usable)

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** To conserve IP address space, you should choose the smallest subnet that accommodates the required number of hosts. A /28 subnet provides 16 IP addresses, and after AWS reserves 5 addresses (first four and last one in the block), 11 usable addresses remain. This is sufficient for 10 EC2 instances while being the smallest option that meets the requirement. A /28 is also the smallest subnet AWS allows in a VPC. A /27 with 27 usable addresses would work but wastes address space. A /24 with 251 usable addresses is far more than needed. A /20 with 4,091 usable addresses is vastly oversized for just 10 instances.
</details>

---

### Question 23

A company is deploying a CloudFormation stack that uses nested stacks. The parent template references three child templates stored in S3. During deployment, one of the child stacks fails. What is the behavior of CloudFormation? **(Select TWO.)**

**A)** Only the failed child stack is rolled back; the other child stacks remain deployed
**B)** The parent stack and all child stacks are rolled back
**C)** The parent stack enters the ROLLBACK_IN_PROGRESS state
**D)** CloudFormation retries the failed child stack three times before rolling back
**E)** The failed child stack is skipped and the parent stack completes with a warning

<details>
<summary>Show Answer</summary>

**Correct Answers: B, C**

**Explanation:** In CloudFormation nested stacks, the child stacks are resources of the parent stack. If any child stack fails during creation, CloudFormation treats it as a resource creation failure in the parent stack. This triggers a rollback of the entire parent stack, which in turn rolls back all child stacks (including those that had successfully deployed). The parent stack transitions to ROLLBACK_IN_PROGRESS state during this process. CloudFormation does not selectively roll back only the failed child stack, does not retry failed stacks, and does not skip failures. The all-or-nothing behavior ensures consistency across the entire nested stack hierarchy.
</details>

---

### Question 24

A SysOps administrator needs to use the `Fn::Select` intrinsic function in a CloudFormation template to choose a specific Availability Zone from a list. The template should select the second AZ from the list returned by `Fn::GetAZs`. Which is the correct syntax?

**A)** `!Select [2, !GetAZs '']`
**B)** `!Select [1, !GetAZs '']`
**C)** `!Select ['second', !GetAZs '']`
**D)** `!Select [1, !Ref 'AWS::AvailabilityZones']`

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** The `Fn::Select` intrinsic function retrieves a single object from a list of objects by index. The index is zero-based, so to select the second item, you use index 1. `!GetAZs ''` (with an empty string) returns the list of Availability Zones for the current region. Therefore, `!Select [1, !GetAZs '']` correctly selects the second Availability Zone. Option A uses index 2, which would select the third AZ. Option C uses a string key, but `Fn::Select` requires a numeric index. Option D references `AWS::AvailabilityZones`, which is not a valid pseudo-parameter in CloudFormation; the correct way to get AZs is through `Fn::GetAZs`.
</details>

---

### Question 25

A company wants to implement a rolling deployment strategy for their application running on an Auto Scaling group with 10 instances. They want to update instances in batches of 2, ensuring at least 8 instances are always serving traffic during the deployment. A health check grace period of 5 minutes should apply to each batch. Which AWS service and configuration achieves this? **(Select THREE.)**

**A)** Configure the Auto Scaling group's instance refresh feature with `MinHealthyPercentage` set to 80%
**B)** Use AWS CodeDeploy with a deployment configuration specifying `MinimumHealthyHosts` of 8
**C)** Set the Auto Scaling group's update policy in CloudFormation with `MaxBatchSize: 2` and `MinInstancesInService: 8`
**D)** Use Elastic Beanstalk rolling update with batch size of 2
**E)** Set `PauseTime` to PT5M in the CloudFormation `AutoScalingRollingUpdate` policy
**F)** Configure the ALB health check interval to 5 minutes

<details>
<summary>Show Answer</summary>

**Correct Answers: C, E, B**

**Explanation:** There are multiple valid ways to achieve rolling deployments on AWS. Using CloudFormation's `UpdatePolicy` attribute with `AutoScalingRollingUpdate`, you can set `MaxBatchSize: 2` to update 2 instances at a time, `MinInstancesInService: 8` to ensure 8 instances always remain running, and `PauseTime: PT5M` to wait 5 minutes between batches for health checks to stabilize (options C and E). Alternatively, AWS CodeDeploy supports rolling deployments with `MinimumHealthyHosts` configuration to ensure a minimum number of healthy instances during deployment (option B). Auto Scaling instance refresh with `MinHealthyPercentage: 80%` would also achieve a similar result for 10 instances. Elastic Beanstalk's rolling updates could work but were not specified as the platform. The ALB health check interval is not the same as a deployment health grace period.
</details>

---
