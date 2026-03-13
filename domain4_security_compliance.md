# Domain 4: Security and Compliance (SOA-C03)

## AWS Certified SysOps Administrator - Associate Practice Questions

**Domain Weight:** 16% of exam

- **Task 4.1:** Implement and manage security and compliance policies
- **Task 4.2:** Implement data and infrastructure protection strategies

---

---
### Question 1

A company has an IAM user who has an IAM policy that explicitly allows `s3:PutObject` on a specific bucket. However, there is also a Service Control Policy (SCP) in AWS Organizations that does not include `s3:PutObject` in its allow list. The user reports being unable to upload objects to the bucket. What explains this behavior?

**A)** The IAM policy takes precedence over SCPs for IAM users within the account.
**B)** SCPs act as a permissions boundary; if the action is not allowed by the SCP, it is implicitly denied regardless of IAM policies.
**C)** The user needs to assume a role that has the SCP permission attached directly.
**D)** SCPs only affect the root user of the account and do not impact IAM users.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Service Control Policies (SCPs) in AWS Organizations act as a guardrail or permissions boundary for all principals in the member account, including IAM users and roles (but not the management account). Even if an IAM policy explicitly allows an action, the SCP must also permit that action for it to succeed. SCPs do not grant permissions themselves; they only restrict what is allowed. The effective permission is the intersection of what the SCP allows and what the IAM policy grants. In this case, since the SCP does not include `s3:PutObject` in its allow list, the action is implicitly denied at the organization level.
</details>

---
### Question 2

A SysOps administrator needs to verify whether a specific IAM policy grants the intended permissions to an IAM user before attaching it in production. Which AWS service should the administrator use to test this without affecting live resources?

**A)** AWS CloudTrail
**B)** IAM Access Analyzer
**C)** IAM Policy Simulator
**D)** AWS Trusted Advisor

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** The IAM Policy Simulator allows administrators to test and validate the effects of IAM policies without actually deploying them. You can simulate API calls against IAM policies attached to users, groups, or roles to see whether those calls would be allowed or denied. This is invaluable for debugging permission issues or verifying a new policy before production deployment. IAM Access Analyzer, by contrast, analyzes resource policies to identify resources shared with external entities. CloudTrail records API activity after it occurs rather than simulating future calls. Trusted Advisor provides best-practice recommendations but does not simulate IAM policy evaluations.
</details>

---
### Question 3

An organization wants to ensure that all AWS accounts in their organization enforce MFA for the root user. Which approach provides centralized compliance enforcement across all accounts?

**A)** Create an IAM policy in each account that requires MFA for all actions by the root user.
**B)** Use an SCP attached to the organization root that denies all actions unless `aws:MultiFactorAuthPresent` is true, with an exception for enabling MFA.
**C)** Enable AWS Config rules in each account to detect root user activity without MFA.
**D)** Use AWS Trusted Advisor to send alerts when root user MFA is not enabled.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** SCPs can enforce MFA requirements across all member accounts in an AWS Organization by using a condition key `aws:MultiFactorAuthPresent`. An SCP that denies all actions unless MFA is present effectively forces MFA usage. You must include exceptions for the actions needed to set up MFA in the first place, otherwise the root user could be locked out. While AWS Config rules (Option C) and Trusted Advisor (Option D) can detect that MFA is not enabled, they are detective controls and do not prevent actions. IAM policies cannot be applied to the root user in the same way as regular IAM users, making Option A impractical. SCPs provide the only preventative, centralized mechanism at the organization level.
</details>

---
### Question 4

A security engineer discovers that an S3 bucket has a bucket policy allowing public read access, but the IAM policy attached to the requesting user explicitly denies `s3:GetObject` on that bucket. What happens when the user tries to read an object from the bucket?

**A)** The request is allowed because the bucket policy grants public access.
**B)** The request is denied because an explicit deny in any policy always overrides an allow.
**C)** The request outcome depends on which policy was created most recently.
**D)** The request is allowed because resource-based policies take precedence over identity-based policies.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** AWS IAM policy evaluation follows a strict hierarchy: an explicit deny always takes precedence over any allow, regardless of where the deny or allow originates. This is the fundamental rule of IAM policy evaluation: explicit deny > explicit allow > implicit deny. Even though the S3 bucket policy grants public read access (an explicit allow), the explicit deny in the user's IAM policy overrides it. Resource-based policies and identity-based policies are evaluated together, and the deny wins. The order or timing of policy creation has no bearing on the evaluation logic. This principle ensures that administrators can always restrict access definitively using deny statements.
</details>

---
### Question 5

A company wants to detect when IAM access keys are shared with external AWS accounts. Which service should the SysOps administrator configure?

**A)** Amazon GuardDuty
**B)** IAM Access Analyzer
**C)** AWS CloudTrail
**D)** Amazon Inspector

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** IAM Access Analyzer helps identify resources in your account that are shared with external entities, including S3 buckets, IAM roles, KMS keys, Lambda functions, and SQS queues. It uses automated reasoning to analyze resource-based policies and generates findings when a resource is accessible from outside the zone of trust (your account or organization). While GuardDuty detects threats and suspicious activity patterns, it does not specifically analyze IAM policies for external sharing. CloudTrail logs API calls but requires manual analysis to identify sharing patterns. Amazon Inspector assesses EC2 instances and container images for vulnerabilities, not IAM policy configurations. Access Analyzer is purpose-built for this exact use case.
</details>

---
### Question 6

A SysOps administrator is configuring AWS CloudTrail for a multi-account environment using AWS Organizations. What is the recommended approach for centralizing trail logs?

**A)** Create an individual trail in each account and configure each to send logs to a central S3 bucket.
**B)** Create an organization trail from the management account, which automatically logs events for all member accounts.
**C)** Use AWS Config to aggregate CloudTrail logs from all accounts into a single dashboard.
**D)** Enable CloudTrail Lake in each account and use cross-account queries.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** AWS CloudTrail supports organization trails, which can be created from the management account and automatically apply to all member accounts in the organization. This is the simplest and most scalable approach for centralized logging. The organization trail logs management and (optionally) data events from all accounts into a single S3 bucket. Member accounts can see the trail but cannot modify or delete it, ensuring log integrity. While Option A would technically work, it requires manual configuration in every account and is not scalable. AWS Config aggregates configuration data, not CloudTrail logs directly. CloudTrail Lake is useful for querying but creating it in each account separately does not centralize data collection as efficiently as an organization trail.
</details>

---
### Question 7

A company's IAM password policy requires passwords to be at least 14 characters, include uppercase and lowercase letters, numbers, and symbols, and prevents reuse of the last 12 passwords. An auditor also requests that passwords expire every 90 days. Where should the SysOps administrator configure these settings?

**A)** In each IAM user's security credentials settings
**B)** In the account-level IAM password policy
**C)** In an SCP applied to the organizational unit
**D)** In AWS IAM Identity Center (SSO) password settings

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** The IAM account password policy is configured at the AWS account level and applies to all IAM users within that account. It allows administrators to set requirements for minimum length, character types (uppercase, lowercase, numbers, symbols), password expiration period, and password reuse prevention. This is a single configuration point that enforces consistency across all IAM users. Individual user credentials settings (Option A) do not allow policy enforcement. SCPs (Option C) control API-level permissions, not password complexity requirements. IAM Identity Center (Option D) manages federated access and has its own separate password policies for the Identity Center directory, but it does not govern IAM user passwords. The account-level password policy is the correct and only mechanism for IAM user password requirements.
</details>

---
### Question 8

An application running on EC2 instances in Account A needs to access an S3 bucket in Account B. What is the recommended approach for granting cross-account access?

**A)** Create an IAM user in Account B with access keys and embed them in the application running in Account A.
**B)** Create an IAM role in Account B with the necessary S3 permissions and a trust policy allowing the EC2 instance role in Account A to assume it.
**C)** Add a bucket policy on the S3 bucket in Account B allowing the public IP addresses of the EC2 instances in Account A.
**D)** Create a VPC peering connection between Account A and Account B to share S3 access.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** The recommended approach for cross-account access is to use IAM roles with trust policies. In Account B, you create a role with the required S3 permissions and a trust policy that specifies the IAM role (or account) in Account A as a trusted entity. The application on the EC2 instance in Account A then calls `sts:AssumeRole` to obtain temporary credentials for the role in Account B. This avoids long-lived credentials and follows the principle of least privilege. Embedding access keys (Option A) is a security anti-pattern because keys can be leaked and are difficult to rotate. IP-based bucket policies (Option C) are fragile and do not authenticate the caller. VPC peering (Option D) facilitates network connectivity but does not grant IAM-level permissions to access S3.
</details>

---
### Question 9

A SysOps administrator needs to identify which AWS Trusted Advisor checks fall under the security category. Which of the following are security checks provided by Trusted Advisor? (Select TWO)

**A)** S3 bucket permissions that allow open access
**B)** EC2 instance CPU utilization optimization
**C)** Security groups with unrestricted access (0.0.0.0/0) on specific ports
**D)** DynamoDB read/write capacity optimization
**E)** CloudFront cache hit ratio analysis

<details>
<summary>Show Answer</summary>

**Correct Answer: A, C**

**Explanation:** AWS Trusted Advisor includes several security checks that are available even on the Basic and Developer support plans. These include checks for S3 bucket permissions that allow public or open access, and security groups with unrestricted access (0.0.0.0/0) on high-risk ports such as SSH (22) and RDP (3389). Other Trusted Advisor security checks include MFA on root account, IAM use, and exposed access keys. Options B and D relate to the performance and cost optimization categories, not security. Option E relates to performance optimization for CloudFront. Understanding the categories of Trusted Advisor checks (cost optimization, performance, security, fault tolerance, and service limits) is important for the exam.
</details>

---
### Question 10

A company uses AWS Organizations with multiple OUs. They want to prevent any account in the Development OU from launching EC2 instances larger than `t3.medium`. How should this be implemented?

**A)** Create an IAM policy in each development account that denies `ec2:RunInstances` for large instance types.
**B)** Attach an SCP to the Development OU that denies `ec2:RunInstances` unless the `ec2:InstanceType` condition key matches allowed types.
**C)** Use AWS Config rules to terminate instances that exceed the allowed size.
**D)** Create a Lambda function triggered by CloudTrail events to terminate oversized instances.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** SCPs attached to an OU affect all accounts within that OU and provide preventative enforcement. By attaching an SCP with a deny statement that uses the `ec2:InstanceType` condition key, you can restrict which instance types can be launched. The SCP would deny `ec2:RunInstances` when the instance type does not match the allowed list (e.g., `t3.micro`, `t3.small`, `t3.medium`). This is a proactive, preventative control that blocks the action before it occurs. Option A would work per-account but is not centralized and could be overridden by account administrators. Options C and D are reactive (detective and corrective) controls that allow the instance to launch first and then remediate, which is less secure and more costly than preventing the action outright. SCPs are the correct mechanism for organization-wide preventative guardrails.
</details>

---
### Question 11

A SysOps administrator wants to enforce that all IAM roles created in an account cannot exceed the permissions defined by a specific managed policy. Which IAM feature should be used?

**A)** Service-linked roles
**B)** Permission boundaries
**C)** Session policies
**D)** Resource-based policies

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Permission boundaries are an advanced IAM feature that sets the maximum permissions that an identity-based policy can grant to an IAM entity (user or role). When a permission boundary is attached, the effective permissions are the intersection of the identity-based policy and the permission boundary. Even if the identity-based policy allows an action, it will be denied if the permission boundary does not also allow it. This is particularly useful for delegating role creation to developers while ensuring they cannot escalate privileges beyond the boundary. Service-linked roles (Option A) are predefined by AWS services and cannot be customized. Session policies (Option C) limit permissions for a specific session when assuming a role but are not a persistent boundary. Resource-based policies (Option D) are attached to resources, not identities, and serve a different purpose.
</details>

---
### Question 12

A company requires that all API calls made in their AWS account be logged for audit purposes, including read-only management events and data events for S3 and Lambda. The administrator has enabled CloudTrail. What additional configuration is required?

**A)** No additional configuration is needed; CloudTrail logs all events by default.
**B)** Data events for S3 and Lambda must be explicitly enabled in the trail configuration, as they are not logged by default.
**C)** The administrator must enable CloudTrail Insights to capture data events.
**D)** Read-only management events are not supported by CloudTrail and require a third-party tool.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** By default, CloudTrail logs management events (also called control plane operations) such as creating or deleting resources. However, data events (also called data plane operations), such as `s3:GetObject`, `s3:PutObject`, and Lambda function invocations, are not logged by default because they are high-volume and incur additional costs. These must be explicitly enabled in the trail configuration by specifying the S3 buckets and/or Lambda functions to monitor. Read-only management events are supported and enabled by default. CloudTrail Insights (Option C) detects unusual activity patterns in management events but is not related to enabling data event logging. Understanding the distinction between management events and data events is critical for the exam.
</details>

---
### Question 13

A developer needs to use an AWS service that automatically creates and manages an IAM role on the developer's behalf. The developer cannot modify or delete this role. What type of role is this?

**A)** A cross-account role
**B)** A service-linked role
**C)** A permission boundary role
**D)** A federated role

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** A service-linked role is a unique type of IAM role that is linked directly to an AWS service. The service defines all the permissions the role needs, and only that service can assume the role. Service-linked roles are predefined by AWS and cannot be modified by the user; they include all necessary permissions for the service to function. Examples include roles for Amazon ECS, AWS Config, and Amazon RDS. Some service-linked roles are created automatically when you use a feature, while others require you to create them but cannot be customized. Cross-account roles (Option A) are user-created roles for access between accounts. Permission boundary roles (Option C) are not a distinct role type. Federated roles (Option D) are used for identity federation but are user-configurable.
</details>

---
### Question 14

An organization uses AWS Config to monitor compliance across their infrastructure. They want to check that all EBS volumes are encrypted and all S3 buckets have versioning enabled using a single deployment mechanism. What is the most efficient approach?

**A)** Create individual AWS Config rules for each check manually in each account.
**B)** Deploy an AWS Config conformance pack that includes rules for EBS encryption and S3 versioning.
**C)** Use Amazon Inspector to assess EBS and S3 configurations.
**D)** Write a custom Lambda function that periodically scans all resources.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** AWS Config conformance packs are collections of Config rules and remediation actions that can be deployed as a single entity. AWS provides sample conformance packs for common compliance frameworks, and administrators can also create custom packs. A conformance pack can include rules like `encrypted-volumes` for EBS encryption and `s3-bucket-versioning-enabled` for S3 versioning. Conformance packs can be deployed across an organization using AWS Organizations integration, making them highly scalable. Individual rule creation (Option A) is tedious and error-prone at scale. Amazon Inspector (Option C) focuses on EC2 vulnerability assessments and network reachability, not resource configuration compliance. Custom Lambda functions (Option D) reinvent functionality already provided natively by AWS Config.
</details>

---
### Question 15

A SysOps administrator is investigating an IAM policy that uses the following condition: `"Condition": {"StringEquals": {"aws:RequestedRegion": "us-east-1"}}`. What does this condition enforce?

**A)** It only allows the action if the IAM user is physically located in the us-east-1 region.
**B)** It only allows the action if the API call targets the us-east-1 region.
**C)** It only allows the action if the resource was originally created in us-east-1.
**D)** It only allows the action if the request is routed through a VPC endpoint in us-east-1.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** The `aws:RequestedRegion` condition key refers to the AWS Region that the API call is targeting, not the caller's physical location or the resource's origin. When used in a policy with `StringEquals`, it restricts actions to only succeed when directed at the specified region. This is commonly used in SCPs or IAM policies to enforce regional restrictions, such as preventing resources from being created outside of approved regions. The user can be physically located anywhere in the world (Option A is incorrect). The condition evaluates each API call independently regardless of where the resource was originally created (Option C is incorrect). VPC endpoint routing (Option D) is unrelated to this condition key. This condition key is one of several global condition keys that can be used across all AWS services.
</details>

---
### Question 16

A company stores sensitive customer data in S3 and needs to encrypt it at rest using keys they fully control, including the ability to set key rotation schedules and define who can use the keys. Which KMS key type should they use?

**A)** AWS managed keys (aws/s3)
**B)** Customer managed keys (CMKs)
**C)** Customer provided keys (SSE-C)
**D)** S3-managed keys (SSE-S3)

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Customer managed keys (CMKs) in AWS KMS provide the highest level of control over encryption keys. With customer managed keys, you can define key policies that control who can administer and use the key, enable automatic key rotation (annually), create grants for temporary access, and audit key usage through CloudTrail. AWS managed keys (Option A) are created and managed by AWS on your behalf for specific services; you cannot control their key policies, rotation schedule, or grant access. SSE-C (Option C) requires customers to provide and manage the encryption key entirely outside of AWS, which means AWS does not store or manage the key at all, adding operational burden. SSE-S3 (Option D) uses keys fully managed by S3 with no customer visibility or control. For the balance of control and managed infrastructure, customer managed KMS keys are the recommended choice.
</details>

---
### Question 17

A SysOps administrator needs to encrypt data at rest in an application. The application encrypts large files (several GB each). How does AWS KMS handle the encryption of large data objects through envelope encryption?

**A)** KMS directly encrypts the entire large file using the CMK.
**B)** KMS generates a data encryption key (DEK); the DEK encrypts the data locally, and KMS encrypts the DEK with the CMK.
**C)** KMS splits the file into 4 KB chunks and encrypts each chunk individually with the CMK.
**D)** KMS compresses the file first, then encrypts the compressed version with the CMK.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** AWS KMS uses envelope encryption for encrypting data larger than 4 KB. KMS itself can only directly encrypt data up to 4 KB. For larger data, the process is: (1) Call `GenerateDataKey` to get a plaintext data encryption key (DEK) and an encrypted copy of that DEK. (2) Use the plaintext DEK locally to encrypt the large data file using a symmetric algorithm like AES-256. (3) Store the encrypted DEK alongside the encrypted data. (4) Discard the plaintext DEK from memory. To decrypt, you send the encrypted DEK to KMS, which decrypts it using the CMK, and then use the plaintext DEK to decrypt the data locally. This approach is more performant because it avoids sending large amounts of data to KMS over the network and reduces the load on the KMS service.
</details>

---
### Question 18

A company wants to automatically rotate their database credentials stored securely in AWS. The credentials need to be retrieved programmatically by application servers. Which service is best suited for this requirement?

**A)** AWS Systems Manager Parameter Store with SecureString parameters
**B)** AWS Secrets Manager with automatic rotation enabled
**C)** AWS KMS with key rotation
**D)** Amazon S3 with server-side encryption

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** AWS Secrets Manager is purpose-built for managing, retrieving, and rotating secrets such as database credentials, API keys, and OAuth tokens. It has built-in integration with RDS, Redshift, and DocumentDB for automatic credential rotation using Lambda functions that Secrets Manager provisions on your behalf. You can configure rotation schedules (e.g., every 30 days) and Secrets Manager handles updating both the secret value and the database password. SSM Parameter Store (Option A) can store secrets as SecureString but does not natively support automatic rotation; you would need to build custom rotation logic. KMS key rotation (Option C) rotates encryption keys, not application credentials. S3 (Option D) is not designed for secret management. When the exam mentions automatic rotation of credentials, Secrets Manager is the correct answer.
</details>

---
### Question 19

A SysOps administrator needs to store configuration values that are non-sensitive (like feature flags) and some sensitive values (like database connection strings) for an application. Cost optimization is a priority. Which approach is recommended?

**A)** Store all values in AWS Secrets Manager.
**B)** Store non-sensitive values in SSM Parameter Store (Standard tier) and sensitive values in AWS Secrets Manager.
**C)** Store all values in S3 as encrypted objects.
**D)** Store all values in SSM Parameter Store as SecureString parameters.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** The most cost-effective approach is to use SSM Parameter Store Standard tier for non-sensitive configuration values (it is free for standard parameters) and AWS Secrets Manager for sensitive values that require automatic rotation. SSM Parameter Store Standard tier offers up to 10,000 parameters at no charge, making it ideal for feature flags and non-sensitive configuration. Secrets Manager charges per secret per month (~$0.40) plus per API call, so using it for all values (Option A) would be unnecessarily expensive for non-sensitive data. Storing everything in Secrets Manager or Parameter Store SecureString (Option D) works but is not cost-optimized when many values are non-sensitive. S3 (Option C) is not designed for application configuration management and lacks the integration with SDKs and parameter resolution that Parameter Store provides.
</details>

---
### Question 20

A company has enabled Amazon GuardDuty in their AWS account. GuardDuty generates a finding of type `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS`. What does this finding indicate?

**A)** An EC2 instance's security group allows unrestricted SSH access from the internet.
**B)** IAM credentials created for an EC2 instance through its instance profile are being used from an external IP address outside of AWS.
**C)** An IAM user's access keys have been publicly exposed on GitHub.
**D)** An S3 bucket is being accessed by an unauthorized IAM user.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** The `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS` GuardDuty finding indicates that credentials created exclusively for an EC2 instance through an instance metadata service are being used from an IP address outside of AWS. This strongly suggests that the instance credentials have been exfiltrated (stolen) and are being used by a malicious actor from outside the AWS network. This is a high-severity finding that requires immediate investigation. The typical attack vector involves compromising the instance (e.g., via SSRF) and stealing the temporary credentials from the instance metadata service. Remediation includes revoking the role's temporary credentials, investigating the instance for compromise, and enabling IMDSv2 to prevent future metadata theft. GuardDuty uses VPC Flow Logs, DNS logs, and CloudTrail event logs to detect this type of anomalous behavior.
</details>

---
### Question 21

A company wants a centralized view of their security posture across multiple AWS accounts, aggregating findings from GuardDuty, Inspector, IAM Access Analyzer, and Macie. Which service provides this capability?

**A)** AWS CloudTrail
**B)** AWS Config
**C)** AWS Security Hub
**D)** Amazon Detective

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** AWS Security Hub provides a comprehensive view of your security state across AWS accounts. It aggregates, organizes, and prioritizes security findings from multiple AWS services including GuardDuty, Inspector, IAM Access Analyzer, Macie, Firewall Manager, and third-party partner products. Security Hub uses a standardized format called AWS Security Finding Format (ASFF) to normalize findings from different sources. It also runs automated security checks against best-practice standards such as CIS AWS Foundations Benchmark, PCI DSS, and AWS Foundational Security Best Practices. CloudTrail (Option A) provides API logging, not finding aggregation. AWS Config (Option B) evaluates resource configurations but does not aggregate findings from multiple security services. Amazon Detective (Option D) is used for investigating findings but does not serve as the primary aggregation point.
</details>

---
### Question 22

A SysOps administrator enables AWS Security Hub and wants to assess their account against the CIS AWS Foundations Benchmark. After enabling the standard, several controls show as FAILED. Which of the following is a CIS Benchmark control that Security Hub checks? (Select TWO)

**A)** Ensure CloudTrail is enabled in all regions.
**B)** Ensure all EC2 instances run the latest Amazon Linux AMI.
**C)** Ensure the root account has MFA enabled.
**D)** Ensure all Lambda functions use Python 3.9 or later.
**E)** Ensure all DynamoDB tables use on-demand capacity mode.

<details>
<summary>Show Answer</summary>

**Correct Answer: A, C**

**Explanation:** The CIS (Center for Internet Security) AWS Foundations Benchmark is a set of security configuration best practices for AWS. It includes controls such as ensuring CloudTrail is enabled in all regions, ensuring the root account has hardware or virtual MFA enabled, ensuring IAM password policies meet complexity requirements, and ensuring access keys are rotated within 90 days. Security Hub automates these checks and reports findings. Option B (latest AMI) is not a CIS control because AMI choice is operational, not a security configuration benchmark. Options D and E relate to runtime versions and capacity modes, which are not part of the CIS AWS Foundations Benchmark. The CIS Benchmark focuses on identity and access management, logging, monitoring, and networking security configurations rather than application-level choices.
</details>

---
### Question 23

A company uses AWS KMS with a customer managed key. The key policy grants the root account full access, and a specific IAM user has `kms:Encrypt` permission in their IAM policy but is not listed in the key policy. Can the IAM user encrypt data with this key?

**A)** No, the user must be explicitly listed in the key policy to use the key.
**B)** Yes, because the key policy grants the root account access, which enables IAM policies in that account to grant KMS permissions.
**C)** No, IAM policies cannot grant KMS permissions under any circumstances.
**D)** Yes, but only if the user also has a grant on the key.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** KMS key policies work differently from most AWS resource policies. By default, when you create a key policy statement that grants the root account principal (`arn:aws:iam::ACCOUNT-ID:root`) access to the key, it does not give the root user full access directly. Instead, it enables the account to use IAM policies to delegate KMS permissions to IAM users and roles within the account. This is a crucial KMS concept: the key policy must allow IAM policy-based access for IAM policies to be effective. Without this root account statement in the key policy, only principals explicitly named in the key policy itself could access the key, regardless of their IAM policies. In this scenario, since the key policy includes the root account and the user's IAM policy grants `kms:Encrypt`, the user can encrypt data. This two-layer authorization (key policy + IAM policy) is unique to KMS.
</details>

---
### Question 24

A SysOps administrator needs to issue SSL/TLS certificates for a web application behind an Application Load Balancer. The certificates should auto-renew without manual intervention. Which service and validation method should be used?

**A)** AWS Certificate Manager (ACM) with email validation
**B)** AWS Certificate Manager (ACM) with DNS validation
**C)** Import a third-party certificate into ACM
**D)** Use IAM Server Certificates

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** AWS Certificate Manager (ACM) can issue free public SSL/TLS certificates for use with integrated AWS services like ALB, CloudFront, and API Gateway. ACM supports two validation methods: DNS validation and email validation. DNS validation requires adding a CNAME record to your domain's DNS configuration, and once set up, ACM can automatically renew the certificate before expiration without any manual action. Email validation requires responding to emails sent to domain-registered contacts, and renewal requires responding to another email, making it a manual process. For automated renewal (which the question specifies), DNS validation is the correct choice. Imported third-party certificates (Option C) cannot be auto-renewed by ACM. IAM Server Certificates (Option D) are a legacy approach that does not provide automatic issuance or renewal.
</details>

---
### Question 25

An application stores API keys for a third-party service. The keys need to be rotated every 60 days, and the application must always retrieve the current valid key without downtime. How should this be implemented using AWS Secrets Manager?

**A)** Store two versions of the secret manually and update the application to switch between them.
**B)** Enable automatic rotation with a rotation interval of 60 days and a custom Lambda rotation function that implements a single-user rotation strategy.
**C)** Use Secrets Manager versioning stages (AWSCURRENT and AWSPREVIOUS) without a rotation function.
**D)** Store the key in Parameter Store and use a CloudWatch Events rule to trigger rotation.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** AWS Secrets Manager supports automatic rotation by invoking a Lambda rotation function on a defined schedule. For third-party API keys, you need a custom Lambda function because Secrets Manager only provides built-in rotation functions for RDS, Redshift, and DocumentDB. The rotation function implements the rotation steps: create a new key with the third-party service, store it in Secrets Manager, test it, and finalize the rotation. Secrets Manager uses versioning stages (AWSCURRENT and AWSPENDING) to manage the transition seamlessly. The application always retrieves the AWSCURRENT version, ensuring zero downtime. A rotation interval of 60 days is configured in the rotation settings. Option A requires manual effort and coordination. Option C provides versioning but no actual rotation mechanism. Option D uses Parameter Store, which lacks native rotation capabilities.
</details>

---
### Question 26

A company has enabled Amazon Inspector on their EC2 instances. They want to assess both network reachability vulnerabilities and software vulnerabilities on the host operating system. What is required for each type of assessment?

**A)** Network assessments require the Inspector agent; host assessments do not.
**B)** Both network and host assessments require the Inspector agent (SSM Agent) installed on the instance.
**C)** Network assessments analyze VPC configuration and do not require an agent; host assessments require the SSM Agent on the instance.
**D)** Neither assessment type requires an agent; Inspector uses VPC Flow Logs for both.

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** Amazon Inspector (v2) performs two types of assessments. Network reachability assessments analyze your VPC configuration, including security groups, NACLs, route tables, and internet gateways, to identify EC2 instances that are reachable from the internet on specific ports. This does not require an agent because it analyzes network configuration data. Host assessments (software vulnerability assessments) scan the operating system for known CVEs (Common Vulnerabilities and Exposures) and require the AWS Systems Manager (SSM) Agent to be installed and running on the instance. The SSM Agent collects software inventory from the instance, which Inspector then analyzes against vulnerability databases. Without the SSM Agent, Inspector cannot perform host-level vulnerability assessments. Understanding this distinction between agentless network assessment and agent-based host assessment is important for the exam.
</details>

---
### Question 27

A SysOps administrator configures an AWS Config rule `s3-bucket-server-side-encryption-enabled` and enables automatic remediation using an SSM Automation document that enables default encryption. After a new unencrypted bucket is created, the Config rule shows NON_COMPLIANT but remediation does not execute. What is the most likely cause?

**A)** The Config rule requires manual approval before remediation executes.
**B)** The IAM role used by the remediation action does not have sufficient permissions to modify S3 bucket encryption settings.
**C)** AWS Config rules cannot trigger automatic remediation.
**D)** The SSM Automation document must be in the same region as the S3 bucket.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** AWS Config supports automatic remediation through SSM Automation documents. When a resource is found NON_COMPLIANT, Config can automatically invoke the specified SSM Automation document to remediate the issue. However, the remediation action requires an IAM role (the automation assume role) with sufficient permissions to perform the corrective action. If this role lacks permissions to call `s3:PutEncryptionConfiguration`, the remediation will fail silently or with an error. This is one of the most common issues with Config auto-remediation. Option A is incorrect because automatic remediation does not require manual approval (though you can also configure manual remediation that requires approval). Option C is incorrect because Config does support auto-remediation. Option D is not a requirement because the SSM document executes within the same region as the Config rule evaluation. Always verify the IAM role permissions when troubleshooting remediation failures.
</details>

---
### Question 28

A security audit reveals that an S3 bucket has both a bucket policy that allows `s3:GetObject` for a specific IAM role and an S3 Block Public Access setting enabled at the account level. A user assumes the role and attempts to access an object. What is the outcome?

**A)** Access is denied because Block Public Access overrides all bucket policies.
**B)** Access is allowed because Block Public Access only blocks public (anonymous) access, not authenticated access through IAM roles.
**C)** Access depends on whether the bucket was created before or after Block Public Access was enabled.
**D)** Access is denied because Block Public Access prevents all cross-principal access.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** S3 Block Public Access is designed to prevent public (anonymous) access to S3 buckets and objects. It blocks bucket policies and ACLs that grant public access, meaning access to unauthenticated users or to any authenticated AWS user. However, it does not prevent access granted to specific IAM principals (users, roles, or accounts) through bucket policies. When a bucket policy grants `s3:GetObject` to a specific IAM role ARN, this is a targeted, authenticated access grant, not public access. Therefore, Block Public Access does not interfere with it. The role can still access the objects as long as the IAM policy attached to the role also allows the action (for same-account access, either the bucket policy or IAM policy granting access is sufficient). Block Public Access specifically targets policies that use wildcards or `*` for principals that would allow public or unauthenticated access.
</details>

---
### Question 29

A company wants to detect when an EC2 instance is performing DNS queries to a known command-and-control server. Which AWS service would detect this threat?

**A)** AWS Config
**B)** Amazon Inspector
**C)** Amazon GuardDuty
**D)** AWS Security Hub

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** Amazon GuardDuty is a threat detection service that continuously monitors for malicious activity and unauthorized behavior. It analyzes three primary data sources: VPC Flow Logs, DNS logs, and CloudTrail event logs. GuardDuty maintains threat intelligence feeds that include known command-and-control (C2) server domains and IP addresses. When an EC2 instance makes DNS queries to a known C2 domain, GuardDuty generates a finding such as `Backdoor:EC2/C&CActivity.DNS`. This is a high-severity finding indicating that the instance may be compromised. AWS Config (Option A) monitors resource configurations, not network traffic. Amazon Inspector (Option B) identifies software vulnerabilities, not active threats. Security Hub (Option D) aggregates findings but does not perform threat detection itself. GuardDuty is the service specifically designed for detecting active threats through analysis of network and account activity.
</details>

---
### Question 30

A SysOps administrator needs to enable automatic rotation for a customer managed KMS key. What is true about KMS automatic key rotation?

**A)** Automatic rotation creates a new key ID each year and requires updating all references to the key.
**B)** Automatic rotation generates new key material annually while retaining the same key ID and ARN; KMS automatically uses the new material for encrypt operations and retains old material for decrypt operations.
**C)** Automatic rotation can be configured for any interval between 30 and 365 days.
**D)** Automatic rotation is enabled by default for all customer managed keys.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** When automatic key rotation is enabled for a customer managed KMS key, AWS KMS generates new cryptographic material for the key every year (365 days). Crucially, the key ID, key ARN, and alias remain unchanged. KMS retains all previous versions of the key material indefinitely, so data encrypted with any previous version can still be decrypted transparently. New encrypt operations use the latest key material, while decrypt operations automatically use the appropriate version. This means no application changes are required when rotation occurs. Automatic rotation is not enabled by default for customer managed keys (Option D is incorrect); it must be explicitly enabled. The rotation period for automatic rotation is fixed at approximately 365 days and cannot be customized to shorter intervals (Option C is incorrect). AWS managed keys are rotated automatically every year by AWS, but customers cannot control this.
</details>

---
### Question 31

A SysOps administrator needs to grant temporary access to a KMS key for an AWS service to encrypt data on behalf of a user, without modifying the key policy. Which KMS feature should be used?

**A)** Key aliases
**B)** Key grants
**C)** Key policies
**D)** IAM policy conditions

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** KMS grants are a mechanism for providing temporary, programmatic access to KMS keys without modifying the key policy or IAM policies. A grant specifies a grantee principal, the KMS key, and the allowed operations (e.g., Encrypt, Decrypt, GenerateDataKey). Grants are commonly used by AWS services that need to encrypt or decrypt data on your behalf. For example, when you create an encrypted EBS volume, EBS uses a grant to call KMS for encryption operations. Grants can be retired or revoked when access is no longer needed, making them ideal for temporary or delegated access. Key aliases (Option A) are friendly names for keys and do not control access. Key policies (Option C) would work but require modification, which the question explicitly excludes. IAM policy conditions (Option D) control access through IAM policies, not through KMS-specific temporary delegation.
</details>

---
### Question 32

A company uses federated access with an external identity provider (IdP) to allow corporate employees to access AWS. They use SAML 2.0 federation. What happens during the authentication process?

**A)** The user authenticates directly with AWS IAM using their corporate credentials.
**B)** The user authenticates with the corporate IdP, receives a SAML assertion, and exchanges it with AWS STS for temporary security credentials.
**C)** The user's corporate credentials are stored in AWS Secrets Manager and validated by IAM during login.
**D)** The IdP sends the user's password to AWS IAM, which validates it against the corporate directory.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** SAML 2.0 federation allows users to authenticate with an external identity provider (such as Active Directory Federation Services or Okta) and then access AWS using temporary credentials. The process works as follows: (1) The user authenticates with the corporate IdP using their corporate credentials. (2) The IdP validates the credentials and issues a SAML assertion containing attributes about the user, including which IAM role they should assume. (3) The user's browser or application sends this SAML assertion to AWS STS (AssumeRoleWithSAML). (4) STS validates the assertion and returns temporary security credentials. At no point are corporate passwords sent to or stored in AWS (Options A, C, and D are incorrect). This approach follows the principle of keeping credentials with the identity provider and only exchanging tokens with AWS. Temporary credentials have a limited lifetime and do not require IAM users to be created.
</details>

---
### Question 33

A SysOps administrator is configuring an SCP hierarchy. The organization root has an SCP allowing all actions. The parent OU has an SCP that allows only EC2 and S3 actions. A child OU inherits from the parent and has an additional SCP that allows EC2, S3, and RDS actions. Can an account in the child OU perform RDS actions?

**A)** Yes, because the child OU's SCP explicitly allows RDS actions.
**B)** No, because the effective permissions are the intersection of all SCPs in the hierarchy; the parent OU does not allow RDS.
**C)** Yes, because child OU SCPs override parent OU SCPs.
**D)** No, because only the organization root SCP determines effective permissions.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** SCP inheritance follows an intersection model, meaning the effective permissions for an account are the intersection of all SCPs applied at every level of the hierarchy, from the organization root down through each OU to the account. Even if the child OU's SCP allows RDS, the parent OU's SCP does not include RDS in its allow list. Since SCPs at each level must all permit the action, and the parent OU only allows EC2 and S3, RDS is effectively denied for all accounts in the child OU. SCPs do not override each other (Option C is incorrect); they are cumulative in a restrictive manner. The organization root SCP is part of the chain but is not the sole determinant (Option D is incorrect). This intersection behavior is critical to understand: each level of the OU hierarchy acts as an additional filter, and the most restrictive combination prevails.
</details>

---
### Question 34

A company needs to encrypt data in transit between their CloudFront distribution and the origin (an ALB). They also need to encrypt traffic between clients and CloudFront. Which is the correct configuration?

**A)** Use ACM certificates in us-east-1 for CloudFront and in the ALB's region for the ALB. Configure the CloudFront origin protocol policy to HTTPS only.
**B)** Use a single ACM certificate attached to CloudFront; the ALB does not need a certificate for origin communication.
**C)** Use self-signed certificates on both CloudFront and the ALB; ACM is not required.
**D)** Enable S3 Transfer Acceleration to encrypt all traffic between CloudFront and the ALB.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** To encrypt data in transit end-to-end, you need TLS certificates at both CloudFront and the ALB. CloudFront requires that its certificate be provisioned in the us-east-1 (N. Virginia) region, which is a specific ACM requirement for CloudFront distributions. The ALB certificate must be provisioned in ACM in the same region as the ALB. On CloudFront, you configure the viewer protocol policy to redirect HTTP to HTTPS (client to CloudFront) and set the origin protocol policy to HTTPS Only (CloudFront to ALB). This ensures encryption both between the client and CloudFront and between CloudFront and the origin. Option B is incorrect because without a certificate on the ALB, CloudFront cannot establish HTTPS connections to the origin. Self-signed certificates (Option C) are not recommended and would cause trust issues. S3 Transfer Acceleration (Option D) is for S3 uploads, not CloudFront-to-ALB communication.
</details>

---
### Question 35

A GuardDuty finding of type `Recon:EC2/PortProbeUnprotectedPort` is generated for an EC2 instance. What does this indicate and what is the recommended first remediation step?

**A)** The instance is scanning other instances' ports. Terminate the instance immediately.
**B)** An external actor is probing an unprotected port on the instance. Review the instance's security group and restrict access to only necessary ports and IP ranges.
**C)** The instance has a malware infection. Isolate the instance and perform a forensic analysis.
**D)** The instance is part of a DDoS attack. Enable AWS Shield Advanced.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** The `Recon:EC2/PortProbeUnprotectedPort` finding indicates that a port on the EC2 instance is unprotected (e.g., open to the internet via a security group) and is being probed by an external actor. This is a reconnaissance activity finding, meaning an attacker is scanning for open ports that could be exploited. The recommended first remediation step is to review the instance's security group rules and restrict inbound access to only the necessary ports and trusted IP ranges. You should also check if the port needs to be publicly accessible or if it can be moved behind a load balancer or restricted to a VPN. This finding does not necessarily indicate that the instance is compromised (Options A and C assume compromise), and it is not a DDoS situation (Option D). However, if the probed port is a sensitive service like SSH or RDP, urgency is higher because successful exploitation could lead to compromise.
</details>

---
### Question 36

A SysOps administrator wants to enforce that all newly created S3 buckets in an account must have default encryption enabled using a specific customer managed KMS key. Which combination of AWS services achieves this with preventative enforcement? (Select TWO)

**A)** AWS Config rule to detect non-compliant buckets after creation.
**B)** An SCP that denies `s3:CreateBucket` unless the `s3:x-amz-server-side-encryption-aws-kms-key-id` condition matches the required key ARN.
**C)** An IAM policy with a condition that denies `s3:PutObject` unless server-side encryption with the specified KMS key is used.
**D)** Amazon Macie to classify data in unencrypted buckets.
**E)** AWS CloudFormation hooks that validate encryption configuration before stack deployment.

<details>
<summary>Show Answer</summary>

**Correct Answer: B, C**

**Explanation:** To achieve preventative enforcement, you need controls that block non-compliant actions before they succeed. An SCP (Option B) can deny bucket creation unless the required encryption configuration is specified, preventing any account in the organization from creating unencrypted buckets. Additionally, an IAM policy or bucket policy with conditions (Option C) can deny `s3:PutObject` requests that do not specify the required KMS key for server-side encryption, ensuring that even if a bucket is created, objects cannot be uploaded without proper encryption. Together, these provide defense in depth. AWS Config (Option A) is a detective control that identifies non-compliance after the fact rather than preventing it. Amazon Macie (Option D) classifies sensitive data but does not enforce encryption. CloudFormation hooks (Option E) only apply to resources created through CloudFormation, not direct API/console actions.
</details>

---
### Question 37

A company has AWS Config enabled and wants to ensure compliance with PCI DSS requirements across their organization. They need automated checks and a compliance score. How should they implement this?

**A)** Create custom Config rules for each PCI DSS requirement manually.
**B)** Enable the PCI DSS conformance pack in AWS Config, which provides pre-built rules mapped to PCI DSS controls.
**C)** Use AWS Artifact to download PCI DSS compliance reports and manually verify each control.
**D)** Enable Amazon Inspector with PCI DSS assessment templates.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** AWS Config provides conformance packs, which are collections of Config rules and remediation actions mapped to specific compliance frameworks. AWS offers a PCI DSS conformance pack that includes pre-built managed rules aligned with PCI DSS controls. When deployed, these rules automatically evaluate your resources against PCI DSS requirements and provide a compliance score showing the percentage of rules that are compliant. This eliminates the need to manually create and map individual rules (Option A). AWS Artifact (Option C) provides AWS's own compliance reports and certifications but does not assess your resources. Amazon Inspector (Option D) assesses EC2 instances for vulnerabilities but does not provide a PCI DSS-specific assessment framework across all resource types. Security Hub also offers a PCI DSS standard that complements Config conformance packs by aggregating findings from multiple sources.
</details>

---
### Question 38

A SysOps administrator is troubleshooting why an IAM user cannot perform `dynamodb:PutItem` on a specific table. The user's IAM policy allows `dynamodb:*` on all resources. The account has no SCPs (it's not part of an Organization). There are no resource-based policies on the DynamoDB table. However, the user has a permission boundary that only allows `dynamodb:GetItem` and `dynamodb:Query`. Why is the action denied?

**A)** DynamoDB does not support IAM policy-based access control for write operations.
**B)** The permission boundary restricts the effective permissions to only `dynamodb:GetItem` and `dynamodb:Query`; `PutItem` is not within the boundary.
**C)** The `dynamodb:*` wildcard does not include `dynamodb:PutItem` in the same account.
**D)** The user needs to have a trust policy that allows DynamoDB write actions.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Permission boundaries define the maximum permissions that an IAM entity can have. The effective permissions for a user are the intersection of their identity-based policies and their permission boundary. In this case, the identity-based policy allows `dynamodb:*` (all DynamoDB actions), but the permission boundary only allows `dynamodb:GetItem` and `dynamodb:Query`. The intersection of these two means the user can only perform `GetItem` and `Query` operations. `PutItem` is allowed by the identity-based policy but not by the permission boundary, so it is effectively denied. Permission boundaries are a critical concept for limiting privilege escalation in delegated administration scenarios. Option A is incorrect because DynamoDB fully supports IAM policy-based access control. Option C is incorrect because `dynamodb:*` includes all DynamoDB actions. Option D is incorrect because trust policies are for roles, not users.
</details>

---
### Question 39

A company wants to classify and protect sensitive data stored in their S3 buckets, specifically detecting personally identifiable information (PII) like social security numbers and credit card numbers. Which AWS service is designed for this purpose?

**A)** Amazon GuardDuty
**B)** Amazon Macie
**C)** AWS Config
**D)** Amazon Inspector

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Amazon Macie is a fully managed data security and data privacy service that uses machine learning and pattern matching to discover and protect sensitive data stored in Amazon S3. Macie automatically discovers sensitive data such as PII (names, addresses, social security numbers), financial data (credit card numbers), and credentials. It creates an inventory of your S3 buckets with details about their encryption status, public accessibility, and whether they contain sensitive data. Macie generates findings that can be integrated with Security Hub and EventBridge for automated alerting and remediation. GuardDuty (Option A) detects threats and anomalous behavior but does not classify data content. AWS Config (Option C) evaluates resource configurations, not data content. Amazon Inspector (Option D) assesses instances and container images for vulnerabilities, not data classification. Macie is the purpose-built service for data classification and sensitive data discovery.
</details>

---
### Question 40

A SysOps administrator needs to ensure that an IAM role used by a Lambda function can only be assumed by the Lambda service and no other principals. Additionally, the role should only have permissions to write to a specific DynamoDB table and read from a specific S3 bucket. Which combination of configurations correctly achieves this? (Select TWO)

**A)** A trust policy on the IAM role that specifies `"Service": "lambda.amazonaws.com"` as the only trusted principal.
**B)** A trust policy that allows `"Principal": "*"` with a condition key limiting to Lambda invocations.
**C)** An identity-based policy attached to the role granting `dynamodb:PutItem` on the specific table ARN and `s3:GetObject` on the specific bucket ARN with a prefix.
**D)** A resource-based policy on DynamoDB that allows the Lambda function's ARN to write data.
**E)** An SCP that only allows Lambda to assume IAM roles.

<details>
<summary>Show Answer</summary>

**Correct Answer: A, C**

**Explanation:** To ensure that only the Lambda service can assume the role, the trust policy (Option A) must specify `"Service": "lambda.amazonaws.com"` as the trusted principal. This means only the Lambda service can call `sts:AssumeRole` for this role. Using `"Principal": "*"` (Option B) is overly permissive and a security risk even with conditions. For the permissions, an identity-based policy (Option C) attached to the role should grant the minimum required permissions: `dynamodb:PutItem` scoped to the specific table ARN and `s3:GetObject` scoped to the specific bucket and prefix. This follows the principle of least privilege. DynamoDB does not support resource-based policies (Option D is incorrect). An SCP (Option E) restricts all principals in an account and would be far too broad for this use case. The combination of a properly scoped trust policy and a least-privilege identity-based policy is the standard pattern for Lambda execution roles.
</details>

---

**Study Tips for Domain 4:**
- Memorize the IAM policy evaluation logic: explicit deny > explicit allow > implicit deny (default)
- Understand the difference between SCPs (preventative guardrails) and AWS Config rules (detective controls)
- Know when to use Secrets Manager (rotation needed) vs. Parameter Store (simple configuration)
- Remember that KMS key policies are the primary access control mechanism for KMS keys
- Security Hub aggregates findings; GuardDuty detects threats; Inspector finds vulnerabilities; Macie classifies data
- Permission boundaries limit the maximum permissions of an IAM entity (intersection model)
- ACM certificates for CloudFront must be in us-east-1
- Envelope encryption: KMS encrypts the data key, the data key encrypts the data
