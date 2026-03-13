# Domain 1: Monitoring, Logging, and Performance Optimization — Part B

## Task 1.3: Performance Optimization Strategies (Questions 26–50)

---

### Question 26

A SysOps Administrator notices that an application running on a fleet of C5 instances is consistently underperforming. CloudWatch metrics show that CPU utilization averages 15% across the fleet, and the application is memory-intensive. The team wants an automated, data-driven recommendation to right-size the instances. Which AWS service should the administrator use?

**A)** AWS Trusted Advisor
**B)** AWS Compute Optimizer
**C)** Amazon CloudWatch Anomaly Detection
**D)** AWS Cost Explorer right-sizing recommendations

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** AWS Compute Optimizer analyzes historical utilization metrics from CloudWatch and provides specific instance type recommendations based on workload patterns. It evaluates CPU, memory, network, and storage metrics to suggest optimal instance families and sizes. While Trusted Advisor and Cost Explorer offer some right-sizing guidance, Compute Optimizer uses machine learning to deliver more granular, performance-focused recommendations. It is purpose-built for identifying over-provisioned or under-provisioned compute resources.
</details>

---

### Question 27

A company stores large media files on Amazon S3 in the us-east-1 Region. Users in Asia-Pacific report slow upload speeds when transferring files that are several gigabytes in size. Which combination of S3 features should the administrator enable to maximize upload performance for these users?

**A)** Enable S3 Transfer Acceleration and use multipart uploads
**B)** Enable S3 Cross-Region Replication to an ap-southeast-1 bucket and use standard uploads
**C)** Enable S3 Versioning and use S3 Batch Operations for uploads
**D)** Enable S3 Intelligent-Tiering and use single PUT requests

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** S3 Transfer Acceleration uses Amazon CloudFront edge locations to accelerate uploads over long distances by routing traffic through the optimized AWS network backbone. When combined with multipart uploads, large files are split into smaller parts that can be uploaded in parallel, significantly improving throughput. Cross-Region Replication copies objects between buckets but does not accelerate the initial upload from end users. Multipart uploads are recommended by AWS for any file larger than 100 MB.
</details>

---

### Question 28

An application uses an Amazon RDS for MySQL Multi-AZ deployment. The database team reports that complex analytical queries are causing performance degradation for the primary transactional workload. The administrator needs to offload read traffic without modifying the application connection strings significantly. What should the administrator do?

**A)** Convert the Multi-AZ standby to a read replica
**B)** Create one or more read replicas and point analytical queries to the read replica endpoints
**C)** Enable Amazon RDS Proxy and route all traffic through it
**D)** Increase the RDS instance size to handle both workloads

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Amazon RDS read replicas allow you to offload read-heavy workloads such as analytical queries from the primary database instance. Each read replica has its own endpoint that analytical tools can be pointed to, leaving the primary instance free to handle transactional writes. The Multi-AZ standby instance cannot serve read traffic—it exists only for failover purposes. RDS Proxy helps manage database connections efficiently but does not inherently separate read and write workloads. Simply scaling up the instance delays the problem rather than solving the architectural concern.
</details>

---

### Question 29

A SysOps Administrator needs to select an EBS volume type for a production SQL Server database that requires sustained 40,000 IOPS with sub-millisecond latency. The volume must also provide durability of 99.999%. Which EBS volume type meets these requirements?

**A)** gp3 with provisioned IOPS set to 40,000
**B)** io1 with provisioned IOPS set to 40,000
**C)** io2 Block Express with provisioned IOPS set to 40,000
**D)** st1 with burst performance

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** io2 Block Express volumes deliver sub-millisecond latency and support up to 256,000 IOPS per volume. Critically, io2 volumes provide 99.999% durability compared to the 99.8%–99.9% durability of other EBS volume types including io1. While gp3 supports up to 16,000 IOPS (and can be pushed higher in some configurations), it does not meet the 40,000 IOPS requirement sustainably. The st1 volume type is a throughput-optimized HDD and is not suitable for IOPS-intensive database workloads. The io1 volume type can deliver 40,000 IOPS but only offers 99.8%–99.9% durability.
</details>

---

### Question 30

A company needs to migrate 50 TB of data from an on-premises NFS file server to Amazon S3 on a recurring weekly basis. The migration must be automated, bandwidth-throttled, and support data integrity verification. Which service should the administrator use?

**A)** AWS Snowball Edge
**B)** AWS DataSync
**C)** S3 multipart upload via the AWS CLI
**D)** AWS Storage Gateway in file mode

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** AWS DataSync is purpose-built for automated, recurring data transfers between on-premises storage and AWS services including Amazon S3, EFS, and FSx. It provides built-in data integrity verification, bandwidth throttling, and scheduling capabilities. Snowball Edge is designed for one-time bulk migrations or edge computing, not recurring transfers. While the AWS CLI with multipart uploads can transfer data to S3, it lacks the automation, scheduling, and integrity features of DataSync. Storage Gateway provides ongoing hybrid access but is not optimized for bulk migration tasks.
</details>

---

### Question 31

An application uses Amazon EFS for shared storage across multiple EC2 instances. The administrator notices that a significant portion of the files have not been accessed in over 90 days. The team wants to reduce storage costs without deleting any files or modifying the application. What should the administrator configure?

**A)** Enable EFS Lifecycle Management to transition files to the EFS Infrequent Access storage class
**B)** Create an S3 Lifecycle policy to move old files to S3 Glacier
**C)** Use AWS DataSync to move old files to Amazon S3 Standard-IA
**D)** Manually move infrequently accessed files to a separate EFS file system with lower throughput

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** EFS Lifecycle Management automatically moves files that have not been accessed for a configured period (7, 14, 30, 60, 90, 180, 270, or 365 days) to the EFS Infrequent Access (IA) storage class. This is transparent to the application—files in IA are still accessible through the same file system mount point with no code changes required. Moving files to S3 would require application changes to access them from a different storage service. The IA storage class can reduce storage costs by up to 92% compared to EFS Standard.
</details>

---

### Question 32

A SysOps Administrator is troubleshooting an EBS gp2 volume attached to an EC2 instance running a database. CloudWatch metrics show that the volume frequently reaches its IOPS limit, and the database experiences high latency during peak hours. The volume is 200 GiB. What is the most cost-effective solution?

**A)** Migrate to a gp3 volume and provision 6,000 IOPS
**B)** Migrate to an io1 volume with 10,000 provisioned IOPS
**C)** Increase the gp2 volume size to 2 TiB to get more baseline IOPS
**D)** Add a second gp2 volume and configure RAID 0

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** A 200 GiB gp2 volume provides a baseline of 600 IOPS (3 IOPS per GiB), which is easily saturated by database workloads. Migrating to gp3 is the most cost-effective approach because gp3 provides a baseline of 3,000 IOPS regardless of volume size, and allows provisioning additional IOPS independently of volume size at a lower cost than io1. Increasing the gp2 volume size to 2 TiB just to achieve 6,000 IOPS wastes storage capacity and money. Migrating to io1 is significantly more expensive than gp3 for this IOPS level. RAID 0 adds operational complexity unnecessarily.
</details>

---

### Question 33

A high-performance computing (HPC) application requires the lowest possible inter-node latency between 20 EC2 instances. The instances must communicate over a high-bandwidth, low-latency network. Which EC2 placement group strategy should the administrator use?

**A)** Spread placement group
**B)** Partition placement group
**C)** Cluster placement group
**D)** Default placement (no placement group)

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** A cluster placement group places instances close together within a single Availability Zone on the same underlying hardware rack, enabling the lowest possible network latency and highest throughput between instances. This is ideal for HPC workloads, tightly coupled node-to-node communication, and applications that benefit from enhanced networking. Spread placement groups distribute instances across distinct hardware to maximize fault isolation, which increases latency. Partition placement groups spread instances across logical partitions on separate racks, designed for distributed workloads like Hadoop or Kafka rather than low-latency HPC.
</details>

---

### Question 34

A company runs a web application with an Amazon RDS PostgreSQL database. The database team suspects that certain SQL queries are causing performance issues but lacks visibility into query-level metrics. Which AWS feature provides the most detailed insight into database query performance?

**A)** Amazon CloudWatch Enhanced Monitoring for RDS
**B)** Amazon RDS Performance Insights
**C)** AWS X-Ray tracing
**D)** Amazon CloudWatch Logs with slow query log enabled

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Amazon RDS Performance Insights provides a dashboard that visualizes database load and allows the team to identify which SQL queries, users, hosts, or wait events are contributing the most to database load. It uses the DB load metric to show the number of active sessions over time and breaks down the load by top SQL statements. Enhanced Monitoring provides OS-level metrics like CPU, memory, and file system usage but does not analyze individual query performance. CloudWatch Logs with slow query logs can capture slow queries but lack the interactive visualization and wait-event analysis that Performance Insights provides.
</details>

---

### Question 35

A SysOps Administrator is designing shared storage for a Linux-based application that requires POSIX-compliant file access, scales automatically, and is accessible from multiple Availability Zones. The workload is bursty with occasional high-throughput requirements. Which storage solution is the best fit?

**A)** Amazon FSx for Windows File Server
**B)** Amazon EFS with bursting throughput mode
**C)** Amazon S3 with S3 Select
**D)** Amazon EBS Multi-Attach with io2 volumes

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Amazon EFS is a fully managed, POSIX-compliant file system that supports concurrent access from multiple EC2 instances across multiple Availability Zones. EFS with bursting throughput mode automatically scales throughput based on the file system size and accumulates burst credits during idle periods, making it ideal for bursty workloads. FSx for Windows File Server uses the SMB protocol and is designed for Windows workloads, not POSIX-compliant Linux access. EBS Multi-Attach is limited to instances within a single Availability Zone and requires a cluster-aware file system. S3 is object storage and does not provide POSIX file system semantics.
</details>

---

### Question 36

An application processes large datasets and requires a high-performance parallel file system with sub-millisecond latency that integrates natively with Amazon S3. Which AWS storage service should the administrator deploy?

**A)** Amazon EFS with provisioned throughput
**B)** Amazon FSx for Lustre
**C)** Amazon FSx for NetApp ONTAP
**D)** Amazon FSx for Windows File Server

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Amazon FSx for Lustre is a high-performance parallel file system designed for compute-intensive workloads such as machine learning, HPC, and media processing. It provides sub-millisecond latencies and can deliver hundreds of gigabytes per second of throughput. FSx for Lustre natively integrates with Amazon S3, allowing it to transparently present S3 objects as files and lazily load data on first access. EFS provides reasonable throughput but does not offer the parallel file system architecture or sub-millisecond latency that Lustre provides. FSx for NetApp ONTAP supports multi-protocol access but is not optimized for parallel HPC workloads.
</details>

---

### Question 37

A company has an application that writes millions of small objects per second to an Amazon S3 bucket. The application uses sequential key names (e.g., timestamps) as prefixes. Users report intermittent HTTP 503 Slow Down errors. What should the administrator do to improve S3 write performance?

**A)** Enable S3 Transfer Acceleration on the bucket
**B)** Randomize the object key name prefixes to distribute requests across S3 partitions
**C)** Increase the number of S3 buckets and distribute writes evenly
**D)** Enable S3 Versioning to improve write throughput

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Amazon S3 automatically partitions bucket data based on key name prefixes. When sequential key names are used, all writes may be directed to the same partition, causing throttling and HTTP 503 Slow Down errors. By randomizing the prefix—such as adding a hash to the beginning of the key name—requests are distributed more evenly across S3 partitions, dramatically improving request rates. While S3 now supports 5,500 GET/HEAD and 3,500 PUT/POST/DELETE requests per second per prefix, randomizing prefixes ensures the workload is spread across many prefixes. Transfer Acceleration helps with geographic distance, not partition-level throttling. Creating multiple buckets adds unnecessary operational complexity.
</details>

---

### Question 38

A SysOps Administrator manages a fleet of EC2 instances that run a latency-sensitive trading application. The application stores temporary data on instance store volumes. After a routine stop and start of an instance, the team discovers all temporary data is lost. How should the administrator address this issue going forward?

**A)** Use EBS volumes instead of instance store volumes for temporary data that must survive stop/start cycles
**B)** Enable instance store encryption to persist data across stop/start events
**C)** Create an AMI before stopping the instance to preserve instance store data
**D)** Configure instance store volumes with RAID 1 for data persistence

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** Instance store volumes provide temporary block-level storage that is physically attached to the host machine. Data on instance store volumes is lost when the instance is stopped, terminated, or moved to a different host. If the application requires temporary data to survive stop/start cycles, the administrator should use EBS volumes, which persist independently of the instance lifecycle. Creating an AMI does not preserve instance store data—only EBS-backed volumes are captured in the AMI snapshot. Neither encryption nor RAID configurations change the ephemeral nature of instance store volumes. Instance store is appropriate only when data loss on stop/terminate is acceptable.
</details>

---

### Question 39

A company wants to use Amazon RDS Proxy to improve the performance of a serverless application that uses AWS Lambda to connect to an Amazon RDS MySQL database. Which performance benefits does RDS Proxy provide in this scenario? **(Select TWO.)**

**A)** RDS Proxy caches frequently accessed query results to reduce database load
**B)** RDS Proxy multiplexes database connections, reducing the overhead of establishing new connections from Lambda
**C)** RDS Proxy automatically scales the underlying RDS instance based on traffic
**D)** RDS Proxy pools and shares database connections, preventing connection exhaustion from Lambda's concurrent invocations
**E)** RDS Proxy automatically creates read replicas when read traffic increases

<details>
<summary>Show Answer</summary>

**Correct Answers: B, D**

**Explanation:** Amazon RDS Proxy sits between the application and the database, pooling and sharing database connections. This is especially valuable for serverless applications using Lambda because each Lambda invocation can open a new database connection, and with high concurrency this can exhaust the database's connection limit. RDS Proxy multiplexes many application connections over a smaller number of database connections, reducing connection overhead and improving connection establishment time. RDS Proxy does not cache query results, does not auto-scale the RDS instance, and does not create read replicas. It focuses exclusively on connection management and improved availability during failovers.
</details>

---

### Question 40

A SysOps Administrator needs to configure enhanced networking on an EC2 instance to achieve higher packets-per-second (PPS) performance and lower latency. The instance type supports the Elastic Network Adapter (ENA). Which steps must the administrator take to enable enhanced networking?

**A)** Attach an Elastic IP address and enable jumbo frames on the instance
**B)** Ensure the ENA driver is installed on the instance and the ENA support attribute is enabled on the instance
**C)** Place the instance in a placement group and attach an additional elastic network interface
**D)** Enable SR-IOV in the AWS Management Console and reboot the instance

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Enhanced networking on EC2 uses single root I/O virtualization (SR-IOV) to provide higher PPS performance, lower latency, and lower jitter compared to traditional virtualized networking. For instances that support the Elastic Network Adapter (ENA), the administrator must ensure the ENA driver is installed within the operating system and that the ENA support attribute is enabled on the instance. Most current-generation instance types and Amazon Linux AMIs have ENA enabled by default. Placement groups can help with inter-instance latency but are not required for enhanced networking. Elastic IP addresses and jumbo frames are unrelated to enabling enhanced networking.
</details>

---

### Question 41

A company stores compliance documents in Amazon S3. Documents are accessed frequently for the first 30 days, occasionally for the next 90 days, and rarely after that but must be retained for 7 years. Which S3 Lifecycle configuration provides the most cost-effective storage strategy?

**A)** Store in S3 Standard, transition to S3 Standard-IA after 30 days, transition to S3 Glacier Deep Archive after 120 days
**B)** Store in S3 Intelligent-Tiering for the entire 7-year period
**C)** Store in S3 Standard, transition to S3 One Zone-IA after 30 days, transition to S3 Glacier after 120 days
**D)** Store in S3 Standard and manually move objects to S3 Glacier after 120 days

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** An S3 Lifecycle policy that transitions objects from S3 Standard to S3 Standard-IA after 30 days and then to S3 Glacier Deep Archive after 120 days aligns storage costs with the access pattern. Standard-IA is appropriate for infrequently accessed data that still needs millisecond retrieval. Glacier Deep Archive offers the lowest storage cost for long-term archival with retrieval times of up to 12 hours, suitable for rarely accessed compliance data. S3 One Zone-IA stores data in a single Availability Zone and is not suitable for compliance documents that require high durability across multiple AZs. S3 Intelligent-Tiering adds per-object monitoring fees that may be unnecessary when the access pattern is well understood and predictable.
</details>

---

### Question 42

A SysOps Administrator is evaluating EBS volume performance for an I/O-intensive application. CloudWatch metrics show the volume's throughput is at its maximum but IOPS utilization is low. The current volume is a 500 GiB gp3 with default settings. What should the administrator do to improve throughput?

**A)** Increase the provisioned IOPS on the gp3 volume
**B)** Increase the provisioned throughput on the gp3 volume beyond the 125 MiB/s default
**C)** Migrate to a gp2 volume of the same size
**D)** Enable EBS-optimized mode on the EC2 instance

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Amazon EBS gp3 volumes provide a default baseline of 3,000 IOPS and 125 MiB/s throughput, and these can be independently scaled. Since the CloudWatch metrics indicate that throughput is maxed out while IOPS utilization is low, the issue is throughput-bound, not IOPS-bound. The administrator should increase the provisioned throughput on the gp3 volume (up to 1,000 MiB/s) without needing to change IOPS. Increasing IOPS would not resolve a throughput bottleneck. Migrating to gp2 would tie throughput to volume size and remove the ability to independently tune performance. Most current-generation instances are EBS-optimized by default.
</details>

---

### Question 43

A company needs a shared file system for a Windows-based application running on Amazon EC2. The file system must support SMB protocol, Active Directory integration, and provide consistent sub-millisecond latency for file operations. Which AWS service should the administrator deploy?

**A)** Amazon EFS with a mount target in each Availability Zone
**B)** Amazon FSx for Windows File Server
**C)** Amazon FSx for Lustre
**D)** Amazon S3 with AWS Storage Gateway File Gateway

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Amazon FSx for Windows File Server provides a fully managed, native Windows file system built on Windows Server. It supports the SMB protocol, integrates with Microsoft Active Directory, and delivers consistent sub-millisecond latency for file operations. It supports Windows features such as DFS namespaces, shadow copies, and access control lists. Amazon EFS uses the NFS protocol, which is designed for Linux workloads, not Windows SMB access. FSx for Lustre is a parallel file system for HPC and does not support SMB or Active Directory. Storage Gateway File Gateway introduces additional latency due to its caching architecture.
</details>

---

### Question 44

A SysOps Administrator is configuring a distributed Apache Kafka cluster on Amazon EC2. The cluster requires fault isolation so that a single hardware failure does not affect more than one broker. The cluster has 7 brokers spread across 3 Availability Zones. Which placement group strategy should the administrator use?

**A)** Cluster placement group
**B)** Spread placement group
**C)** Partition placement group
**D)** No placement group is needed

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** A partition placement group divides instances into logical partitions, each placed on separate underlying hardware racks. This is ideal for distributed and replicated workloads like Apache Kafka, HDFS, and Cassandra, where you want to minimize the impact of correlated hardware failures while running more than 7 instances per AZ. A spread placement group limits you to 7 instances per Availability Zone per group, which could become a constraint as the cluster grows. While 7 brokers fit within the spread group limit, partition placement groups are specifically designed for large distributed workloads and provide the rack-awareness information that Kafka can leverage for replica placement. Cluster placement groups prioritize low latency over fault isolation.
</details>

---

### Question 45

A SysOps Administrator observes that an RDS for PostgreSQL instance has high CPU utilization and the database is experiencing connection timeouts during peak hours. Performance Insights shows that the top wait event is `Client:ClientRead`, and there are over 500 active connections. What is the most effective remediation?

**A)** Increase the RDS instance size to an instance type with more vCPUs
**B)** Enable Multi-AZ deployment to distribute the connection load
**C)** Deploy Amazon RDS Proxy to manage and pool database connections
**D)** Modify the `max_connections` parameter in the RDS parameter group

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** The `Client:ClientRead` wait event indicates the database is waiting for data from the client, which often occurs when there are too many idle or slow connections consuming resources. Deploying RDS Proxy pools and reuses database connections, dramatically reducing the number of active connections to the database. This frees up CPU and memory resources that were being consumed by connection management overhead. Simply increasing `max_connections` would allow more connections but worsen the resource contention. Scaling up the instance addresses the symptom rather than the root cause. Multi-AZ standby instances do not serve traffic and cannot distribute connection load.
</details>

---

### Question 46

A company wants to optimize its EC2 fleet costs while maintaining performance. The administrator has been asked to use resource tags to track instance performance alongside costs. Which approach allows the administrator to correlate performance metrics with cost data using resource tags?

**A)** Enable AWS Compute Optimizer and use cost allocation tags in AWS Cost Explorer to filter recommendations by tagged resource groups
**B)** Create CloudWatch custom metrics with tag dimensions and use AWS Budgets for cost tracking
**C)** Use AWS Systems Manager inventory to collect tag and performance data and export to Amazon QuickSight
**D)** Enable detailed monitoring in CloudWatch and activate user-defined cost allocation tags in the Billing console, then use Cost Explorer to analyze cost and usage by tag

<details>
<summary>Show Answer</summary>

**Correct Answer: D**

**Explanation:** To correlate performance metrics with cost data using resource tags, the administrator should enable detailed monitoring in CloudWatch for granular performance metrics and activate user-defined cost allocation tags in the AWS Billing console. Once activated, Cost Explorer can filter and group cost and usage data by these tags, allowing the team to see spending alongside resource utilization patterns. Compute Optimizer provides right-sizing recommendations but does not directly correlate tagged costs with performance metrics in a single view. While Systems Manager and QuickSight could build a custom solution, this requires significantly more effort than using the built-in Cost Explorer tag filtering capability.
</details>

---

### Question 47

A SysOps Administrator needs to configure shared storage for a multi-protocol environment where both Linux (NFS) and Windows (SMB) clients need to access the same data simultaneously. The solution must support automatic tiering between SSD and capacity pool storage. Which AWS storage service meets these requirements?

**A)** Amazon EFS with both NFS and SMB mount targets
**B)** Amazon FSx for NetApp ONTAP
**C)** Amazon FSx for Windows File Server with NFS enabled
**D)** Amazon FSx for Lustre with SMB compatibility mode

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Amazon FSx for NetApp ONTAP is the only AWS managed file storage service that supports simultaneous multi-protocol access via both NFS and SMB on the same data set. It also provides automatic storage tiering between high-performance SSD storage and cost-effective capacity pool storage based on access patterns. Amazon EFS supports only NFS and does not offer SMB access. FSx for Windows File Server supports only SMB and does not natively serve NFS clients. FSx for Lustre is a parallel file system for HPC workloads and does not support SMB. FSx for NetApp ONTAP also offers features like snapshots, clones, and data replication.
</details>

---

### Question 48

An application team reports that their Amazon EBS io1 volume is not delivering the provisioned 20,000 IOPS. The volume is 200 GiB and attached to an m5.large instance. What is the most likely cause of the performance issue? **(Select TWO.)**

**A)** The io1 volume size is too small to support 20,000 IOPS given the 50:1 IOPS-to-GiB ratio
**B)** The m5.large instance type has an EBS throughput limit that is lower than what 20,000 IOPS requires
**C)** The io1 volume needs to be converted to io2 to achieve 20,000 IOPS
**D)** The application is performing sequential reads instead of random I/O operations
**E)** The EBS-optimized bandwidth of the m5.large instance is insufficient to saturate the provisioned IOPS

<details>
<summary>Show Answer</summary>

**Correct Answers: A, E**

**Explanation:** io1 volumes support a maximum ratio of 50 IOPS per GiB of volume size. A 200 GiB volume can support a maximum of 10,000 IOPS (200 x 50), not the provisioned 20,000 IOPS—the volume would need to be at least 400 GiB. Additionally, the m5.large instance type has limited EBS-optimized bandwidth (up to 4,750 Mbps), which may not be sufficient to drive 20,000 IOPS depending on the I/O size. Both the volume size constraint and the instance-level EBS bandwidth limit can prevent the volume from delivering its provisioned performance. The administrator should increase the volume size and consider a larger instance type with higher EBS bandwidth.
</details>

---

### Question 49

A company is migrating a large on-premises data lake to Amazon S3. The total dataset is 200 TB and must be transferred within two weeks. The corporate internet connection is 1 Gbps. After accounting for overhead and existing traffic, only 500 Mbps is available for the migration. Which approach ensures the data is transferred within the deadline?

**A)** Use AWS DataSync over the existing internet connection with bandwidth throttling set to 500 Mbps
**B)** Use S3 Transfer Acceleration with multipart uploads over the existing internet connection
**C)** Order multiple AWS Snowball Edge Storage Optimized devices and ship them to AWS
**D)** Set up an AWS Direct Connect connection and use DataSync to transfer the data

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** At 500 Mbps, the maximum data transfer in two weeks is approximately 75 TB (500 Mbps x 14 days x 86,400 seconds / 8 bits per byte), which is far short of the 200 TB requirement. AWS Snowball Edge Storage Optimized devices each hold up to 80 TB of usable storage, so ordering three devices allows the full 200 TB to be transferred within the timeframe, as data is copied locally and then shipped to AWS. Transfer Acceleration improves upload speed from distant locations but cannot overcome the physical bandwidth limit. Setting up a new Direct Connect connection typically takes weeks to months to provision, making it impractical for a two-week deadline. DataSync over the existing connection would also be limited by the available 500 Mbps bandwidth.
</details>

---

### Question 50

A SysOps Administrator needs to improve the performance of an application that runs on EC2 instances behind an Application Load Balancer. The application accesses an RDS database and stores session data. During load testing, the administrator identifies the following bottlenecks: database read latency is high, Lambda-based microservices are exhausting database connections, and EC2 instances in the same cluster placement group occasionally fail to launch due to insufficient capacity. Which combination of actions addresses all three issues? **(Select THREE.)**

**A)** Create RDS read replicas and configure the application to direct read queries to the replica endpoints
**B)** Deploy Amazon ElastiCache in front of the RDS database to cache all write operations
**C)** Deploy Amazon RDS Proxy to pool and manage connections from Lambda functions
**D)** Use a spread placement group instead of a cluster placement group to avoid capacity errors
**E)** Use a partition placement group or launch instances across multiple Availability Zones to reduce capacity constraints
**F)** Increase the RDS instance to the largest available instance class

<details>
<summary>Show Answer</summary>

**Correct Answers: A, C, E**

**Explanation:** Creating RDS read replicas offloads read traffic from the primary instance, directly reducing read latency for the application. Deploying RDS Proxy addresses the Lambda connection exhaustion issue by pooling and reusing database connections efficiently. For the cluster placement group capacity issue, using a partition placement group or distributing instances across multiple Availability Zones reduces the risk of insufficient capacity errors that occur when trying to place many instances on the same rack. ElastiCache caches read operations, not writes, making option B incorrect. A spread placement group limits instances to 7 per AZ per group, which may be too restrictive. Simply scaling up the RDS instance does not address the connection exhaustion or placement group issues.
</details>

---
