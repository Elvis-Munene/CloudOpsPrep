# Domain 2: Reliability and Business Continuity - Part B

## Task 2.3: Backup and Restore Strategies

---

### Question 26

A company stores critical financial data in an Amazon S3 bucket. A compliance requirement mandates that objects must not be permanently deleted by any user, including the root account, for a minimum of 365 days. The operations team needs to implement a solution that prevents accidental or malicious deletions. What combination of S3 features should the SysOps administrator configure?

**A)** Enable S3 Versioning and configure an S3 Lifecycle policy to transition objects to S3 Glacier after 365 days
**B)** Enable S3 Versioning, enable MFA Delete, and apply an S3 Object Lock policy in Compliance mode with a retention period of 365 days
**C)** Enable S3 Versioning and configure cross-region replication to a backup bucket
**D)** Enable S3 Versioning and apply an S3 Object Lock policy in Governance mode with a retention period of 365 days

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** S3 Object Lock in Compliance mode ensures that no user, including the root account, can delete or overwrite an object version for the specified retention period. This is the strongest immutability guarantee S3 offers. Enabling MFA Delete adds an additional layer of protection by requiring multi-factor authentication to permanently delete object versions or change the versioning state. Governance mode (Option D) allows users with the `s3:BypassGovernanceRetention` permission to override the lock, which does not meet the requirement. Lifecycle policies and CRR do not prevent deletion of the original objects.
</details>

---

### Question 27

A SysOps administrator manages an Amazon RDS MySQL Multi-AZ instance that stores order transaction data. The business requires the ability to restore the database to any point within the last 30 days in case of data corruption. The default backup retention period is set to 7 days. What should the administrator do to meet this requirement?

**A)** Increase the automated backup retention period to 30 days in the RDS instance configuration
**B)** Create a manual RDS snapshot every day and retain snapshots for 30 days using a custom script
**C)** Enable RDS point-in-time recovery and set the retention to 30 days; no other changes needed since PITR is independent of backup retention
**D)** Configure AWS Backup with a backup plan that takes daily snapshots and retains them for 30 days

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** Amazon RDS automated backups support a retention period of up to 35 days. By increasing the retention period to 30 days, the administrator enables point-in-time restore (PITR) to any second within that 30-day window. PITR relies on the automated backup retention period setting — it is not independently configurable (ruling out Option C). While Option D with AWS Backup would create snapshots, it would not provide the granular second-by-second PITR capability that automated backups with transaction logs provide. Manual snapshots (Option B) would only allow restoring to the exact time each snapshot was taken, not to an arbitrary point in time.
</details>

---

### Question 28

A company runs a production workload across two AWS Regions: us-east-1 (primary) and eu-west-1 (DR). They use Amazon DynamoDB as the backend database. The business requires that the DR region have a near-real-time copy of the data with an RPO of less than 1 minute. Which solution meets this requirement with the LEAST operational overhead?

**A)** Enable DynamoDB point-in-time recovery (PITR) and restore the table to the DR region when needed
**B)** Configure DynamoDB global tables with replication to eu-west-1
**C)** Use AWS Backup to take hourly DynamoDB backups and copy them to eu-west-1
**D)** Create an AWS Lambda function triggered by DynamoDB Streams to write items to a DynamoDB table in eu-west-1

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** DynamoDB global tables provide fully managed, multi-region, multi-active replication with sub-second replication latency, easily meeting the RPO requirement of less than 1 minute. This is a native feature requiring minimal operational overhead — you simply add a replica region and DynamoDB handles the rest. PITR (Option A) only allows restoring within the same region and does not replicate data cross-region in near-real-time. AWS Backup hourly copies (Option C) would result in an RPO of up to 1 hour. A custom Lambda function (Option D) would work but introduces significant operational overhead compared to the fully managed global tables solution.
</details>

---

### Question 29

A SysOps administrator needs to implement a backup strategy for Amazon EBS volumes attached to a fleet of 200 EC2 instances. Snapshots must be taken every 12 hours and retained for 14 days. Old snapshots must be automatically deleted. What is the MOST operationally efficient approach?

**A)** Write a cron job on each EC2 instance that calls the AWS CLI to create and delete snapshots on a schedule
**B)** Create an Amazon Data Lifecycle Manager (DLM) policy targeting the EBS volumes with a 12-hour schedule and 14-day retention
**C)** Use AWS Systems Manager Maintenance Windows to run a snapshot script every 12 hours across all instances
**D)** Configure AWS Backup with a backup plan that runs every 12 hours and set a lifecycle rule to delete backups after 14 days

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Amazon Data Lifecycle Manager (DLM) is purpose-built for automating the creation, retention, and deletion of EBS snapshots and EBS-backed AMIs. You define a policy with a schedule (every 12 hours) and retention count or age-based rule (14 days), and DLM handles everything automatically. It targets volumes using tags, making it easy to manage 200 instances. While AWS Backup (Option D) could also work, DLM is the most operationally efficient native solution specifically designed for EBS snapshot lifecycle management. Cron jobs (Option A) are fragile and do not scale well. Systems Manager (Option C) adds unnecessary complexity for this specific use case.
</details>

---

### Question 30

A company is designing a disaster recovery strategy for a three-tier web application. The application currently runs in us-west-2. The business has an RTO of 4 hours and an RPO of 1 hour. The company wants to minimize costs while meeting these recovery objectives. Which DR strategy is MOST appropriate?

**A)** Multi-site active-active with full production infrastructure in a second Region
**B)** Warm standby with a scaled-down version of the production environment running in a second Region
**C)** Pilot light with core infrastructure (database replicas) in a second Region and the ability to rapidly provision compute resources
**D)** Backup and restore with regular backups stored in a second Region

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** A pilot light strategy maintains only the most critical core components (such as database replicas) running in the DR Region while other resources like web and application servers are pre-configured but not running. During a disaster, compute resources are quickly provisioned and scaled. With an RTO of 4 hours and RPO of 1 hour, pilot light is appropriate because the database replica ensures recent data availability (meeting RPO), and 4 hours provides sufficient time to launch and configure the remaining infrastructure. Backup and restore (Option D) may struggle to meet a 1-hour RPO if backups are taken less frequently. Multi-site (Option A) and warm standby (Option B) would meet the requirements but at significantly higher cost, violating the cost-minimization requirement.
</details>

---

### Question 31

A SysOps administrator has configured AWS Backup to protect resources across five AWS accounts in an organization. A security audit requires that backup copies stored in the central backup account cannot be deleted by anyone, including administrators, for a minimum of 1 year. What should the administrator configure?

**A)** Create an AWS Backup vault access policy that denies the `backup:DeleteRecoveryPoint` action for all principals
**B)** Enable AWS Backup Vault Lock in Compliance mode with a minimum retention period of 365 days on the vault in the central backup account
**C)** Enable AWS Backup Vault Lock in Governance mode with a minimum retention period of 365 days on the vault in the central backup account
**D)** Configure an SCP in AWS Organizations that denies the `backup:DeleteRecoveryPoint` action for all accounts

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** AWS Backup Vault Lock in Compliance mode provides WORM (Write Once Read Many) protection for recovery points. Once configured, the vault lock cannot be removed by anyone, including the root user, and recovery points cannot be deleted before the specified retention period expires. This is the correct choice for immutable backups required by audits and compliance mandates. Governance mode (Option C) allows users with specific IAM permissions to override the lock. A vault access policy (Option A) can be modified by administrators. An SCP (Option D) can also be modified by the management account administrator, and it does not provide the same level of immutability as Vault Lock in Compliance mode.
</details>

---

### Question 32

An application uses Amazon S3 to store critical log files. The company requires that all objects written to the bucket in us-east-1 be automatically replicated to a bucket in ap-southeast-1 for disaster recovery. The destination bucket must use a different storage class to reduce costs. Which steps must the administrator take? **(Select TWO.)**

**A)** Enable versioning on both the source and destination buckets
**B)** Create a cross-region replication (CRR) rule on the source bucket, specifying the destination bucket and optionally overriding the storage class
**C)** Enable versioning on the source bucket only; the destination bucket does not require versioning
**D)** Enable S3 Transfer Acceleration on both buckets to speed up replication
**E)** Create an S3 Lifecycle policy on the source bucket to copy objects to the destination bucket

<details>
<summary>Show Answer</summary>

**Correct Answer: A, B**

**Explanation:** S3 cross-region replication (CRR) requires versioning to be enabled on both the source and destination buckets — this is a mandatory prerequisite (Option A, not Option C). The administrator then creates a CRR rule on the source bucket specifying the destination bucket in the target Region, and can optionally override the storage class at the destination to use a cheaper tier like S3 Standard-IA or S3 Glacier (Option B). S3 Transfer Acceleration (Option D) is unrelated to replication — it accelerates uploads to S3. Lifecycle policies (Option E) manage transitions between storage classes and expiration within the same bucket; they do not replicate objects across buckets.
</details>

---

### Question 33

A company runs an Amazon FSx for Windows File Server file system that stores shared departmental documents. The team needs to ensure that backups are taken daily and retained for 90 days. They also need the ability to restore individual files from backups. What is the recommended approach?

**A)** Configure automatic daily backups in Amazon FSx with a 90-day retention period and use the FSx console to restore individual files from backups
**B)** Use AWS DataSync to copy files to S3 daily and configure an S3 Lifecycle policy to delete objects after 90 days
**C)** Enable Windows Volume Shadow Copy Service (VSS) on the file system and configure a 90-day retention policy
**D)** Use AWS Backup to create a backup plan with daily frequency and 90-day retention; restore files by mounting the backup as a new file system

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** Amazon FSx for Windows File Server supports automatic daily backups natively, with configurable retention periods of up to 90 days (and longer via AWS Backup). These backups are incremental, cost-effective, and fully managed. Individual files can be restored from FSx backups by restoring the backup to a new file system or by using Windows Previous Versions functionality if shadow copies are also enabled. Option D with AWS Backup is also viable and would work, but Option A is the most straightforward recommended approach using FSx's built-in backup capabilities. DataSync (Option B) copies data but does not provide the same integrated backup and restore experience. VSS (Option C) provides point-in-time snapshots but has limited retention capabilities compared to FSx backups.
</details>

---

### Question 34

A SysOps administrator needs to copy an encrypted Amazon EBS snapshot from us-east-1 to eu-central-1 for disaster recovery purposes. The snapshot is encrypted with a customer-managed AWS KMS key. What must the administrator do to successfully copy the snapshot?

**A)** Copy the snapshot to eu-central-1 and specify a KMS key in eu-central-1 for re-encryption; ensure the IAM role has permissions to use both KMS keys
**B)** First decrypt the snapshot in us-east-1, then copy the unencrypted snapshot to eu-central-1, and re-encrypt it there
**C)** Share the KMS key from us-east-1 with eu-central-1 since KMS keys are global resources
**D)** Copy the snapshot to eu-central-1; the same KMS key will automatically be used since it is a customer-managed key

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** When copying an encrypted EBS snapshot across Regions, you must specify a KMS key in the destination Region because KMS keys are regional resources, not global (ruling out Options C and D). The copy operation decrypts the snapshot using the source Region's KMS key and re-encrypts it with the specified destination Region's KMS key. The IAM principal performing the copy must have permissions to use the KMS key in both Regions — `kms:Decrypt` in the source Region and `kms:CreateGrant` and `kms:Encrypt` in the destination Region. You cannot have an unencrypted copy of a snapshot that was originally encrypted (ruling out Option B), as EBS enforces encryption throughout the copy process.
</details>

---

### Question 35

A company wants to implement a multi-site active-active disaster recovery architecture for their e-commerce platform across us-east-1 and us-west-2. The application uses Amazon Aurora MySQL as the database. Which Aurora feature should the administrator use to support active-active database operations across both Regions?

**A)** Aurora cross-region read replicas with manual promotion during failover
**B)** Aurora Global Database with write forwarding enabled
**C)** Aurora Multi-AZ deployment spanning two Regions
**D)** Aurora Serverless v2 with automatic cross-region scaling

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Aurora Global Database spans multiple Regions with a primary cluster for read-write operations and secondary clusters in other Regions for reads. With write forwarding enabled, applications in the secondary Region can submit write operations locally, which are automatically forwarded to the primary Region. This effectively supports an active-active architecture where both Regions can handle read and write traffic. Cross-region read replicas (Option A) only support reads and require manual promotion, which does not support active-active. Aurora Multi-AZ (Option C) operates within a single Region, not across Regions. Aurora Serverless v2 (Option D) does not natively provide cross-region capabilities.
</details>

---

### Question 36

A SysOps administrator is tasked with implementing a backup strategy using AWS Backup for resources across multiple AWS accounts in an AWS Organization. Backups from all accounts must be centrally stored in a dedicated backup account. What should the administrator configure?

**A)** Enable cross-account backup in the AWS Backup settings of the management account and create a backup policy in AWS Organizations that targets the central backup account vault
**B)** In each member account, configure AWS Backup to copy backups to an S3 bucket in the central backup account
**C)** Share the backup vault from the central account to all member accounts using AWS Resource Access Manager (RAM)
**D)** Create identical backup plans in each member account and use AWS Lambda to copy recovery points to the central account

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** AWS Backup supports cross-account backup management through integration with AWS Organizations. The administrator enables cross-account backup in the AWS Backup settings from the management account, then creates backup policies at the organizational level that specify backup rules including a copy action targeting a backup vault in the central backup account. This provides a centralized, automated, and policy-driven approach to cross-account backup management. Option B is incorrect because AWS Backup does not store backups in S3 buckets in this manner. Option C is not how cross-account backup vaults work. Option D would work but is unnecessarily complex and operationally burdensome compared to the native AWS Backup Organizations integration.
</details>

---

### Question 37

A company uses Amazon S3 for storing application data in the us-east-1 Region. For compliance purposes, a copy of all objects must also exist in a bucket in the same Region but in a different AWS account owned by the compliance department. What is the MOST efficient way to achieve this?

**A)** Configure S3 cross-region replication (CRR) between the two accounts
**B)** Configure S3 same-region replication (SRR) from the source bucket to the compliance account's bucket
**C)** Set up an AWS Lambda function triggered by S3 event notifications to copy objects to the compliance account's bucket
**D)** Use AWS DataSync to continuously sync objects between the two buckets

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** S3 same-region replication (SRR) is designed for exactly this use case — replicating objects to another bucket in the same Region, including across different AWS accounts. SRR is fully managed, supports automatic and asynchronous replication, and can be configured with proper IAM roles and bucket policies to enable cross-account access. CRR (Option A) is for cross-region scenarios and would be incorrect since both buckets are in us-east-1. Lambda (Option C) would work but adds unnecessary operational overhead. DataSync (Option D) is designed for data transfer tasks and is not the most efficient solution for continuous S3-to-S3 replication within the same Region.
</details>

---

### Question 38

An organization has an RTO of 15 minutes and an RPO of near-zero for their mission-critical order processing application. The application runs on EC2 instances behind an Application Load Balancer with an Amazon Aurora database. Which disaster recovery strategy should the SysOps administrator implement?

**A)** Backup and restore with frequent automated backups stored in a secondary Region
**B)** Pilot light with an Aurora cross-region read replica and pre-configured AMIs in the secondary Region
**C)** Warm standby with a scaled-down environment in the secondary Region and Aurora Global Database
**D)** Multi-site active-active with full production capacity in two Regions using Aurora Global Database and Route 53 health checks

<details>
<summary>Show Answer</summary>

**Correct Answer: D**

**Explanation:** With an RTO of 15 minutes and near-zero RPO, only a multi-site active-active architecture can reliably meet these aggressive recovery objectives. Aurora Global Database provides replication with typically less than 1 second of lag, achieving near-zero RPO. A full production environment in the second Region with Route 53 health checks and automatic failover ensures the 15-minute RTO is met since traffic is simply rerouted — no infrastructure provisioning is needed. Warm standby (Option C) might approach this but requires scaling up during failover, risking the 15-minute RTO. Pilot light (Option B) and backup and restore (Option A) have RTOs measured in hours, not minutes, due to infrastructure provisioning time.
</details>

---

### Question 39

A SysOps administrator needs to ensure that EBS snapshots created by Amazon Data Lifecycle Manager (DLM) are automatically copied to a secondary Region for disaster recovery. The snapshots in the secondary Region should be retained for 30 days. How should this be configured?

**A)** Create a DLM policy with a cross-region copy rule specifying the target Region and a 30-day retention period for the copied snapshots
**B)** Create a second DLM policy in the target Region that imports snapshots from the source Region
**C)** Use AWS Backup instead of DLM, as DLM does not support cross-region snapshot copies
**D)** Configure an EventBridge rule to detect DLM snapshot creation events and trigger a Lambda function to copy snapshots to the target Region

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** Amazon Data Lifecycle Manager (DLM) natively supports cross-region snapshot copy rules as part of a snapshot lifecycle policy. When creating or editing a DLM policy, you can add a cross-region copy rule that specifies the target Region and an independent retention period for the copied snapshots. In this case, the administrator sets the target Region and a 30-day retention for the copies. Option B is incorrect because DLM does not have an import mechanism. Option C is incorrect because DLM does support cross-region copies. Option D would work but adds unnecessary complexity when DLM provides this capability natively.
</details>

---

### Question 40

A company has an Amazon RDS PostgreSQL database in us-east-1 that serves a globally distributed application. For disaster recovery, the team needs a database copy in eu-west-1 that can be promoted to a standalone instance within minutes if the primary Region becomes unavailable. What should the administrator configure?

**A)** Enable automated backups and configure AWS Backup to copy RDS snapshots to eu-west-1 hourly
**B)** Create a cross-region read replica of the RDS instance in eu-west-1
**C)** Export RDS snapshots to S3 and replicate them to eu-west-1 using cross-region replication
**D)** Use AWS Database Migration Service (DMS) to continuously replicate data from us-east-1 to a new RDS instance in eu-west-1

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** An RDS cross-region read replica maintains a near-real-time copy of the primary database in another Region using asynchronous replication. In a disaster scenario, the read replica can be promoted to a standalone read-write instance within minutes, meeting the rapid failover requirement. Option A with hourly backup copies would result in an RPO of up to 1 hour and restoring from a snapshot takes much longer than promoting a replica. Option C with S3 export is a slow, manual process not suitable for rapid disaster recovery. Option D with DMS would work but adds operational complexity compared to the native cross-region read replica feature.
</details>

---

### Question 41

A SysOps administrator discovers that DynamoDB point-in-time recovery (PITR) was accidentally disabled on a critical table two weeks ago. The table currently has on-demand backups from one month ago. A developer accidentally deleted several items from the table earlier today. What is the BEST course of action to recover the deleted items?

**A)** Re-enable PITR and restore the table to a point before the deletion occurred today
**B)** Restore the on-demand backup from one month ago to a new table and manually reconcile the missing items with the current table
**C)** Use DynamoDB Streams to replay the deleted items back into the table
**D)** Contact AWS Support to recover the deleted items from internal backups

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Since PITR was disabled two weeks ago, re-enabling it now (Option A) would only allow restoring from the point of re-enablement forward — it cannot recover historical data from before it was enabled. The only available backup is the on-demand backup from one month ago. The administrator should restore this backup to a new table and then compare and reconcile the missing items with the production table. This is not ideal as the backup is one month old, but it is the best available option. DynamoDB Streams (Option C) only retains data for 24 hours and replaying streams does not un-delete items in a straightforward manner. AWS Support (Option D) cannot recover data from internal backups for DynamoDB.
</details>

---

### Question 42

A company requires that all Amazon EBS snapshots across their AWS Organization be encrypted. Some development accounts have been creating unencrypted snapshots. What should the SysOps administrator implement to enforce encryption going forward?

**A)** Enable EBS encryption by default in each AWS account and Region, and create an SCP that denies the `ec2:CreateSnapshot` action when encryption is not specified
**B)** Enable EBS encryption by default in each AWS account and Region; this ensures all new snapshots are automatically encrypted
**C)** Create an AWS Config rule to detect unencrypted snapshots and an SCP to deny snapshot creation without encryption
**D)** Use AWS Backup exclusively for all snapshots, as AWS Backup encrypts all backups by default

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** When EBS encryption by default is enabled for an AWS account in a specific Region, all new EBS volumes and snapshots created in that Region are automatically encrypted using the default KMS key (or a specified customer-managed key). This is the simplest and most effective way to enforce encryption for all new snapshots. The setting must be enabled per account and per Region. Option A adds an unnecessary SCP since enabling encryption by default already handles this. Option C is reactive (detecting after creation) rather than preventive. Option D is impractical as it would require changing all existing automation and workflows to use AWS Backup exclusively.
</details>

---

### Question 43

A SysOps administrator is designing a backup plan in AWS Backup for a production environment containing EC2 instances, RDS databases, EFS file systems, and DynamoDB tables. The plan requires daily backups at 2:00 AM UTC with 35-day retention and weekly backups with 1-year retention. How should the administrator structure this in AWS Backup?

**A)** Create two separate backup plans — one for daily and one for weekly — and assign the same resources to both
**B)** Create a single backup plan with two backup rules (one for daily, one for weekly), each with the appropriate schedule and retention, and assign resources using tags
**C)** Create a single backup plan with one rule and use lifecycle policies within the rule to handle the different retention periods
**D)** Create individual backup plans for each resource type since AWS Backup requires separate plans per service

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** AWS Backup allows a single backup plan to contain multiple backup rules, each with its own schedule, retention period, and lifecycle settings. The administrator should create one backup plan with a daily rule (every day at 2:00 AM UTC, 35-day retention) and a weekly rule (once per week, 1-year retention). Resources are assigned to the plan using resource tags or resource ARNs. This is the most organized and manageable approach. Option A would work but is unnecessarily complex. Option C is incorrect because a single rule cannot have two different retention periods. Option D is incorrect because AWS Backup supports multiple resource types within a single backup plan.
</details>

---

### Question 44

A company operates a web application with an Amazon Aurora MySQL cluster in us-west-2. They need to maintain a disaster recovery capability in us-east-1. The current RPO requirement is 30 seconds. During a failover event, the application in us-east-1 should be able to start writing to the database within 1 minute. Which solution meets these requirements?

**A)** Configure Aurora cross-region read replicas in us-east-1 with manual promotion during failover
**B)** Configure an Aurora Global Database with a secondary cluster in us-east-1 and use the managed planned failover or detach-and-promote process
**C)** Take Aurora clones every minute and restore them in us-east-1
**D)** Use AWS DMS for continuous replication from us-west-2 to a separate Aurora cluster in us-east-1

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Aurora Global Database uses dedicated replication infrastructure that provides cross-region replication with typical latency of under 1 second, meeting the 30-second RPO requirement. The managed failover process for Aurora Global Database can promote a secondary cluster in about 1 minute, meeting the RTO requirement. Cross-region read replicas (Option A) use standard MySQL binlog replication which has higher latency and manual promotion takes longer. Cloning (Option C) is not a cross-region feature and cannot run every minute. DMS (Option D) introduces additional latency and complexity compared to the native Aurora Global Database replication.
</details>

---

### Question 45

A SysOps administrator needs to protect an Amazon FSx for Lustre file system used for high-performance computing workloads. The file system is linked to an S3 bucket as its data repository. What backup strategy should the administrator implement?

**A)** Rely solely on the linked S3 bucket as the backup since all data is automatically synchronized back to S3
**B)** Configure automatic backups for the FSx for Lustre file system and ensure that modified data is exported to S3 using `hsm_archive` or auto-export before any planned maintenance
**C)** Use AWS Backup to create daily backups of the FSx for Lustre file system
**D)** Take manual snapshots of the underlying EBS volumes attached to the FSx file system

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** FSx for Lustre supports automatic daily backups, which capture the state of the file system. However, since FSx for Lustre is a high-performance scratch or persistent file system linked to S3, data written to the file system is not automatically written back to S3 unless auto-export is configured or files are explicitly archived using the `hsm_archive` command. The administrator should configure automatic backups and ensure a process exists to export modified data back to S3. Option A is incorrect because data is not automatically synced back to S3 without explicit configuration. Option C is valid for persistent deployment types but may not be sufficient alone. Option D is incorrect because FSx does not expose underlying EBS volumes for direct snapshot management.
</details>

---

### Question 46

An organization wants to implement a pilot light disaster recovery strategy for their application running in us-east-1. The application uses an Auto Scaling group of EC2 instances, an Application Load Balancer, and an Amazon RDS MySQL database. Which components should be running in the DR Region (us-west-2) at all times? **(Select TWO.)**

**A)** A fully scaled Auto Scaling group of EC2 instances matching the production capacity
**B)** An RDS cross-region read replica of the production database
**C)** An Application Load Balancer with registered targets
**D)** Pre-configured AMIs and launch templates stored in us-west-2
**E)** A scaled-down Auto Scaling group running a minimum number of EC2 instances

<details>
<summary>Show Answer</summary>

**Correct Answer: B, D**

**Explanation:** In a pilot light DR strategy, only the most critical core components are kept running in the DR Region. The database replica (Option B) is the core component that must be running to maintain data synchronization and minimize RPO. Pre-configured AMIs and launch templates (Option D) should be maintained in the DR Region so that compute resources can be rapidly provisioned when needed. A fully scaled Auto Scaling group (Option A) describes a multi-site active-active strategy. An ALB with registered targets (Option C) is not needed until failover occurs. A scaled-down Auto Scaling group (Option E) describes a warm standby strategy, not pilot light. The key distinction of pilot light is that compute resources are not running — only the data layer is active.
</details>

---

### Question 47

A SysOps administrator manages an S3 bucket with versioning enabled. Users have reported that storage costs are increasing significantly due to the accumulation of noncurrent object versions. The administrator needs to reduce costs while maintaining the ability to recover accidentally deleted objects for up to 30 days. What should the administrator do?

**A)** Disable S3 versioning to stop creating noncurrent versions
**B)** Create an S3 Lifecycle policy to transition noncurrent versions to S3 Glacier Instant Retrieval after 1 day and permanently delete them after 30 days
**C)** Enable MFA Delete to prevent accidental version accumulation
**D)** Create an S3 Lifecycle policy to permanently delete all noncurrent versions after 1 day

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** An S3 Lifecycle policy can target noncurrent object versions specifically. By transitioning noncurrent versions to a cheaper storage class like S3 Glacier Instant Retrieval after 1 day and expiring (permanently deleting) them after 30 days, the administrator reduces storage costs while maintaining a 30-day recovery window. Disabling versioning (Option A) would prevent future versioning but would not address existing noncurrent versions and would eliminate the ability to recover deleted objects. MFA Delete (Option C) controls who can delete versions but does not reduce storage costs. Deleting noncurrent versions after 1 day (Option D) would not meet the 30-day recovery requirement.
</details>

---

### Question 48

A company runs a critical application with an RDS Oracle database that must comply with a regulatory requirement to retain database backups for 7 years. Automated RDS backups only support a maximum retention of 35 days. How should the SysOps administrator meet the 7-year retention requirement?

**A)** Configure AWS Backup with a backup plan that creates RDS snapshots and sets the lifecycle to move backups to cold storage after 35 days with a total retention of 7 years
**B)** Use a scheduled Lambda function to create manual RDS snapshots and tag them with retention metadata, with a cleanup function to delete snapshots older than 7 years
**C)** Increase the RDS automated backup retention period beyond 35 days by contacting AWS Support
**D)** Export RDS snapshots to S3 using native RDS export and apply an S3 Lifecycle policy with 7-year retention

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** AWS Backup supports long-term retention for RDS snapshots and can transition recovery points to cold storage after a specified period to reduce costs. By creating a backup plan with a lifecycle rule that moves backups to cold storage after 35 days and sets total retention to 7 years (2,555 days), the administrator meets the compliance requirement in a fully managed way. Option B would work but requires custom code and ongoing maintenance. Option C is incorrect because the 35-day maximum for automated backups cannot be extended. Option D is feasible but adds complexity and does not preserve the ability to restore directly to an RDS instance from cold storage as seamlessly as AWS Backup.
</details>

---

### Question 49

A SysOps administrator is implementing cross-region backup for an AWS environment. The administrator creates a backup plan in AWS Backup that includes a cross-region copy rule to copy recovery points from us-east-1 to eu-west-1. After several days, the administrator notices that cross-region copies for some RDS snapshots are failing. What is the MOST likely cause?

**A)** The backup vault in eu-west-1 does not have a KMS key configured for encrypting the copied recovery points
**B)** The RDS instances are using an unencrypted database, and cross-region copies require encryption
**C)** The IAM role used by AWS Backup does not have permissions to use the KMS key in the destination Region for re-encryption
**D)** Cross-region backup copies are not supported for RDS resources in AWS Backup

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** When copying encrypted RDS snapshots across Regions using AWS Backup, the recovery points must be re-encrypted with a KMS key in the destination Region. If the IAM service role used by AWS Backup does not have the necessary KMS permissions (such as `kms:CreateGrant`, `kms:Decrypt`, and `kms:GenerateDataKey`) in the destination Region, the cross-region copy will fail. This is one of the most common causes of cross-region copy failures. Option A is less likely because AWS Backup vaults have a default encryption key. Option B is incorrect because cross-region copies work for both encrypted and unencrypted snapshots. Option D is incorrect because AWS Backup fully supports cross-region copies for RDS.
</details>

---

### Question 50

A company is evaluating disaster recovery strategies for their workloads and needs to understand the trade-offs between different approaches. The CTO asks the SysOps administrator to map each scenario to the appropriate DR strategy. Which of the following correctly matches the DR strategy to its characteristics?

**A)** Backup and restore: RPO of hours, RTO of hours, lowest cost; all infrastructure is provisioned from backups during recovery
**B)** Pilot light: RPO of near-zero, RTO of minutes; full production environment runs in both Regions at all times
**C)** Warm standby: RPO of hours, RTO of days; only backups are maintained in the secondary Region
**D)** Multi-site active-active: RPO of minutes, RTO of hours; a scaled-down environment runs in the secondary Region

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** Backup and restore is the lowest-cost DR strategy where backups (snapshots, AMIs, database backups) are stored in a secondary Region. During a disaster, all infrastructure is provisioned from these backups, resulting in an RPO and RTO measured in hours. Option B incorrectly describes pilot light — pilot light keeps only core infrastructure (like database replicas) running, not a full production environment, and has an RTO of tens of minutes to hours, not minutes. Option C describes backup and restore characteristics, not warm standby — warm standby runs a scaled-down but functional environment with RPO of seconds to minutes and RTO of minutes. Option D describes warm standby characteristics, not multi-site — multi-site active-active has near-zero RPO and RTO measured in seconds to minutes.
</details>

---
