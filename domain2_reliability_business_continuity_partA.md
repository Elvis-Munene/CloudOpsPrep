# Domain 2: Reliability and Business Continuity - Part A

## Task 2.1: Implement Scalability and Elasticity
## Task 2.2: Implement Highly Available and Resilient Environments

---

### Question 1

A company runs a web application on EC2 instances behind an Application Load Balancer. The application experiences predictable traffic spikes every weekday between 9:00 AM and 11:00 AM. The SysOps administrator wants to ensure that enough instances are running before the traffic spike begins, while still scaling dynamically during unexpected surges. Which combination of Auto Scaling policies best meets this requirement?

**A)** Configure a scheduled scaling action to increase the desired capacity at 8:45 AM and a target tracking scaling policy to handle unexpected surges throughout the day.
**B)** Configure a step scaling policy with CloudWatch alarms set to trigger at 70% CPU utilization and a simple scaling policy as a backup.
**C)** Configure a target tracking scaling policy only, set to maintain 50% average CPU utilization across all instances.
**D)** Configure a scheduled scaling action to increase capacity at 8:45 AM and decrease it at 11:15 AM, with no additional dynamic policies.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** Scheduled scaling is ideal for predictable traffic patterns because it proactively adjusts capacity before the load arrives. Combining it with a target tracking policy ensures that unexpected surges beyond the predicted pattern are also handled dynamically. Option B lacks the proactive scaling needed for predictable spikes. Option C would react to the spike rather than prepare for it, causing potential latency. Option D handles only the predictable pattern and cannot respond to unexpected traffic increases.
</details>

---

### Question 2

A SysOps administrator has configured an Auto Scaling group with a target tracking scaling policy set to maintain average CPU utilization at 60%. After a scale-out event adds new instances, the administrator notices that another scale-out event is triggered almost immediately, even though the new instances have not finished initializing. What should the administrator do to prevent this behavior?

**A)** Increase the target value of the target tracking policy from 60% to 80%.
**B)** Increase the default cooldown period for the Auto Scaling group.
**C)** Replace the target tracking policy with a simple scaling policy.
**D)** Disable the Auto Scaling group's health checks until instances are fully initialized.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** The cooldown period prevents the Auto Scaling group from launching or terminating additional instances before previous scaling activities take effect. By increasing the cooldown period, newly launched instances have time to initialize and begin handling traffic, which allows the CPU metric to stabilize before another scaling decision is made. Increasing the target to 80% would reduce responsiveness to legitimate load. Replacing with a simple scaling policy does not inherently solve the initialization timing issue. Disabling health checks is unrelated to scaling policy triggers.
</details>

---

### Question 3

A company is deploying a containerized microservices application on Amazon ECS with Fargate. One of the services experiences highly variable request rates throughout the day. The team wants the service to scale based on the number of requests per target, as measured by the Application Load Balancer. Which approach should the SysOps administrator implement?

**A)** Create a CloudWatch alarm on the ALBRequestCountPerTarget metric and attach a step scaling policy to the ECS service.
**B)** Configure an Application Auto Scaling target tracking policy for the ECS service using the ALBRequestCountPerTarget predefined metric.
**C)** Configure EC2 Auto Scaling target tracking on the underlying Fargate instances using the RequestCount metric.
**D)** Create a scheduled scaling policy on the ECS service that adjusts the desired count every hour based on historical traffic patterns.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Amazon ECS services support Application Auto Scaling with target tracking policies. The ALBRequestCountPerTarget is a predefined metric specifically designed for scaling based on the number of requests each target receives from an ALB. This allows ECS to dynamically adjust task count to maintain the desired request rate per target. Option A would work but is more complex and less responsive than target tracking. Option C is invalid because Fargate manages the underlying infrastructure and does not expose EC2 instances for Auto Scaling. Option D does not address the highly variable nature of the traffic.
</details>

---

### Question 4

A SysOps administrator is configuring an Auto Scaling group that uses lifecycle hooks. When a new instance is launched, a configuration management tool must run a setup script that takes approximately 10 minutes. The administrator wants the instance to begin receiving traffic only after the script completes successfully. How should this be implemented?

**A)** Add a launch lifecycle hook with a heartbeat timeout of 15 minutes. Have the setup script call CompleteLifecycleAction with CONTINUE upon success.
**B)** Add a terminate lifecycle hook with a heartbeat timeout of 15 minutes. Have the setup script signal completion when finished.
**C)** Set the Auto Scaling group health check grace period to 10 minutes and rely on ELB health checks to detect readiness.
**D)** Configure the instance user data script to send a CloudWatch custom metric when initialization is complete, and use that metric to trigger a scaling policy.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** A launch lifecycle hook places the instance in a Pending:Wait state, preventing it from being registered with the load balancer and receiving traffic until the lifecycle action is completed. Setting the heartbeat timeout to 15 minutes gives the 10-minute script enough time to finish, and calling CompleteLifecycleAction with CONTINUE moves the instance into InService. Option B uses a terminate hook, which applies when instances are being removed, not launched. Option C would allow unhealthy traffic to reach the instance during the grace period. Option D does not prevent the instance from receiving traffic during initialization.
</details>

---

### Question 5

A company uses Amazon DynamoDB for its e-commerce product catalog. The table experiences steady read traffic of around 500 RCUs during normal hours but spikes unpredictably to 5,000 RCUs during flash sales that last 15-30 minutes. The current provisioned capacity is set to 1,000 RCUs with DynamoDB auto scaling enabled (target utilization 70%). During flash sales, customers report slow responses. What should the administrator do to resolve this?

**A)** Increase the provisioned RCUs to 5,000 permanently to handle peak traffic.
**B)** Switch the table to on-demand capacity mode.
**C)** Increase the DynamoDB auto scaling maximum capacity to 6,000 RCUs and decrease the target utilization to 50%.
**D)** Add a DynamoDB Accelerator (DAX) cluster in front of the table.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** On-demand capacity mode is ideal for workloads with unpredictable, spiky traffic patterns. It automatically accommodates up to double the previous peak traffic and scales instantly without throttling. DynamoDB auto scaling (Option C) relies on CloudWatch alarms and can take several minutes to react, which is too slow for sudden flash-sale spikes lasting only 15-30 minutes. Permanently provisioning 5,000 RCUs (Option A) is wasteful and expensive during normal hours. DAX (Option D) helps with read-heavy caching but does not address the provisioned throughput throttling issue for cache misses or first-time reads.
</details>

---

### Question 6

A SysOps administrator manages an application with a MySQL-compatible Amazon Aurora database. The application performs many read-heavy reporting queries that are degrading write performance on the primary instance. The administrator needs to offload read traffic without modifying the application connection strings stored in configuration files. What is the most operationally efficient solution?

**A)** Create an Aurora read replica and update the application to use the Aurora reader endpoint.
**B)** Create an Amazon RDS for MySQL read replica in a separate Region and point reporting queries to it.
**C)** Enable Amazon ElastiCache for Redis and cache frequently queried report data.
**D)** Migrate the database to Amazon Redshift for the reporting workload.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** Aurora automatically provides a reader endpoint that load-balances connections across all read replicas in the cluster. By adding one or more Aurora read replicas and pointing the reporting application to the reader endpoint, read traffic is offloaded from the primary instance. This is operationally efficient because Aurora handles replica creation, data synchronization, and load balancing natively. Option B introduces cross-Region replication complexity and higher latency. Option C requires application code changes to implement caching logic. Option D is a significant architectural change not warranted for this use case.
</details>

---

### Question 7

A company has a web application deployed across two Availability Zones behind an Application Load Balancer. The ALB has cross-zone load balancing enabled. AZ-A has 4 registered instances and AZ-B has 2 registered instances. What is the expected traffic distribution across the instances?

**A)** Each instance in AZ-A receives 12.5% of traffic, and each instance in AZ-B receives 25% of traffic.
**B)** Each of the 6 instances receives approximately 16.67% of traffic.
**C)** AZ-A receives 50% and AZ-B receives 50% of traffic, distributed equally among instances in each AZ.
**D)** Traffic is distributed based on the health and capacity of each instance, with no predictable pattern.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** With cross-zone load balancing enabled, the load balancer distributes traffic evenly across all registered instances in all Availability Zones, regardless of how many instances are in each AZ. Since there are 6 total instances, each receives approximately 16.67% of the traffic. Option C describes the behavior when cross-zone load balancing is disabled, where each AZ receives 50% of traffic and then distributes it among its own instances. For ALBs, cross-zone load balancing is enabled by default and cannot be disabled at the ALB level. This ensures optimal utilization even when AZs have unequal numbers of instances.
</details>

---

### Question 8

A SysOps administrator configures Amazon Route 53 health checks for a web application running on EC2 instances in us-east-1 and eu-west-1. The administrator sets up a failover routing policy with the us-east-1 endpoint as the primary and eu-west-1 as the secondary. Users report that when us-east-1 becomes unavailable, it takes over 3 minutes before traffic fails over to eu-west-1. What should the administrator do to reduce the failover time?

**A)** Decrease the Route 53 health check request interval from 30 seconds to 10 seconds (fast health checks) and reduce the failure threshold from 3 to 1.
**B)** Change the routing policy from failover to latency-based routing with health checks.
**C)** Enable Cross-Region read replicas for the application database to improve availability.
**D)** Configure the health check to use a TCP check instead of an HTTP check.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** Route 53 health checks default to a 30-second interval with a failure threshold of 3, meaning it can take up to 90 seconds just for health checkers to determine an endpoint is unhealthy, plus additional time for DNS TTL propagation. By switching to fast health checks (10-second interval) and reducing the failure threshold to 1, the health check can detect failure in as little as 10 seconds. Combined with a low TTL on the DNS record, this significantly reduces overall failover time. Option B changes the routing strategy but does not inherently speed up failover detection. Options C and D do not address the health check detection speed.
</details>

---

### Question 9

A company runs a stateful web application on EC2 instances in an Auto Scaling group behind an Application Load Balancer. Users report that they are being logged out intermittently. Investigation reveals that when an instance is terminated during a scale-in event, active user sessions are lost. What should the administrator configure to minimize session disruption during scale-in events? **(Choose TWO.)**

**A)** Enable sticky sessions (session affinity) on the ALB target group.
**B)** Increase the deregistration delay (connection draining) timeout on the target group to allow in-flight requests to complete.
**C)** Store session data in Amazon ElastiCache for Redis instead of on the EC2 instances.
**D)** Configure the Auto Scaling group to use the OldestInstance termination policy.
**E)** Disable scale-in on the Auto Scaling group to prevent instance termination.

<details>
<summary>Show Answer</summary>

**Correct Answer: B, C**

**Explanation:** The root cause is that session data is stored locally on EC2 instances, so when an instance is terminated, its sessions are lost. The best long-term solution is to externalize session storage to ElastiCache for Redis (Option C), making sessions available to any instance. Increasing the deregistration delay (Option B) ensures that in-flight requests to a terminating instance can complete gracefully, preventing mid-request disruptions. Sticky sessions (Option A) would keep a user on the same instance but worsen the problem during scale-in since sessions are still lost when that instance terminates. Option D changes which instance is terminated but does not prevent session loss. Option E prevents scaling and is not a viable operational solution.
</details>

---

### Question 10

A SysOps administrator needs to deploy a highly available Amazon RDS for PostgreSQL database. The application requires automatic failover with minimal downtime and the ability to read from the standby during normal operations. Which configuration meets these requirements?

**A)** Enable Multi-AZ deployment for the RDS instance, which provides automatic failover and allows read traffic on the standby.
**B)** Create an RDS read replica in a different AZ and configure the application to read from the replica endpoint.
**C)** Deploy Amazon Aurora PostgreSQL with one or more Aurora Replicas in different AZs.
**D)** Deploy two separate RDS for PostgreSQL instances in different AZs with application-level failover logic.

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** Amazon Aurora PostgreSQL supports up to 15 Aurora Replicas that can serve read traffic and also act as failover targets. Failover is automatic and typically completes within 30 seconds. Standard RDS Multi-AZ (Option A) provides automatic failover but the standby instance cannot serve read traffic. An RDS read replica (Option B) can serve reads but does not provide automatic failover to the replica by default, and promotion is a manual process. Option D requires custom application logic for failover and adds significant operational overhead. Aurora best satisfies both requirements simultaneously.
</details>

---

### Question 11

A media company uses Amazon CloudFront to distribute video content globally. Users in Asia-Pacific report higher latency and buffering compared to users in North America. The origin is an S3 bucket in us-east-1. What should the SysOps administrator do to improve performance for Asia-Pacific users?

**A)** Create an S3 bucket in an Asia-Pacific Region and configure CloudFront with an origin group that includes both buckets.
**B)** Increase the CloudFront cache TTL for video content and ensure the cache policy maximizes the cache hit ratio.
**C)** Disable CloudFront and have Asia-Pacific users connect directly to the S3 bucket using S3 Transfer Acceleration.
**D)** Add more CloudFront edge locations by requesting a price class upgrade to include all edge locations.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Higher latency for Asia-Pacific users likely indicates cache misses at edge locations, forcing requests back to the us-east-1 origin. By increasing the TTL and optimizing the cache policy (e.g., minimizing query string and header forwarding), more requests are served from the edge cache, reducing origin fetches and latency. Option A adds complexity and CloudFront origin groups are designed for failover, not geographic routing. Option C removes the CDN benefit entirely. Option D is unlikely the issue since CloudFront's default price class already includes Asia-Pacific edge locations, and CloudFront automatically routes users to the nearest edge location.
</details>

---

### Question 12

A SysOps administrator is configuring an Auto Scaling group with a launch template. The team wants to use a mix of instance types (m5.large, m5.xlarge, and c5.large) across multiple Availability Zones to optimize cost and availability. Which Auto Scaling group configuration supports this requirement?

**A)** Create three separate Auto Scaling groups, one for each instance type, and coordinate them with a common CloudWatch alarm.
**B)** Configure the Auto Scaling group with a mixed instances policy that specifies multiple instance types and uses a capacity-optimized allocation strategy.
**C)** Specify all three instance types in a single launch template and let the Auto Scaling group select randomly.
**D)** Use a launch configuration with the m5.large instance type and manually override during scale-out events.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Auto Scaling groups support mixed instances policies that allow specifying multiple instance types and purchase options (On-Demand and Spot) within a single group. The capacity-optimized allocation strategy launches instances in the Spot pools with the most available capacity, reducing the chance of interruption while maintaining cost savings. A single launch template cannot specify multiple instance types for random selection (Option C). Launch configurations (Option D) are legacy and do not support mixed instance types. Multiple Auto Scaling groups (Option A) add unnecessary operational complexity.
</details>

---

### Question 13

A company uses Amazon ElastiCache for Redis as a session store for its web application. The application requires high availability with automatic failover. The Redis cluster runs in a single Availability Zone. After an AZ outage, the application experienced downtime because the cache was unavailable. What should the administrator do to prevent this in the future?

**A)** Enable Redis cluster mode and distribute shards across multiple AZs.
**B)** Enable Multi-AZ with automatic failover on the ElastiCache Redis replication group, placing replicas in different AZs.
**C)** Take hourly snapshots of the Redis cluster and restore to a different AZ if a failure occurs.
**D)** Replace ElastiCache with DynamoDB for session storage since DynamoDB is inherently multi-AZ.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Enabling Multi-AZ with automatic failover on an ElastiCache Redis replication group creates read replicas in different Availability Zones. If the primary node fails or the AZ experiences an outage, ElastiCache automatically promotes a replica in a healthy AZ to primary, typically within seconds. Option A enables cluster mode for sharding but does not automatically ensure multi-AZ failover without also configuring replicas. Option C requires manual intervention and results in data loss between snapshots. Option D would work but is a significant architectural change when simply enabling Multi-AZ on the existing Redis setup solves the problem directly.
</details>

---

### Question 14

A SysOps administrator notices that an Auto Scaling group is launching and terminating instances in rapid succession, a behavior known as "thrashing." The group uses a simple scaling policy with a CloudWatch alarm that triggers at 75% CPU utilization. The alarm evaluates every 1 minute with a single data point. What is the most likely cause and resolution?

**A)** The CloudWatch alarm evaluation period is too short, causing premature triggers. Increase the evaluation period to 5 minutes and set the datapoints to alarm to 3 out of 5.
**B)** The instance type is too small for the workload. Upgrade to a larger instance type.
**C)** The Auto Scaling group has too many Availability Zones. Reduce to a single AZ.
**D)** Simple scaling policies do not support CloudWatch alarms. Switch to a target tracking policy.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** Thrashing occurs when scaling actions are triggered too aggressively, often due to alarm configurations that react to brief CPU spikes rather than sustained load. With a 1-minute evaluation and a single data point required, even momentary CPU spikes trigger scaling. By increasing the evaluation period and requiring multiple data points before alarming (e.g., 3 out of 5 periods), the alarm only triggers on sustained high utilization, preventing unnecessary scaling actions. Additionally, ensuring an appropriate cooldown period would further stabilize the behavior. Option B may help but does not address the root cause of alarm sensitivity. Option C would reduce availability. Option D is incorrect because simple scaling policies do work with CloudWatch alarms.
</details>

---

### Question 15

A financial services company requires their Amazon RDS for Oracle database to survive an entire AWS Region failure. The database must be accessible in a secondary Region with minimal data loss. Which solution meets this requirement?

**A)** Configure RDS Multi-AZ deployment in the primary Region with automated backups enabled.
**B)** Create a cross-Region RDS read replica and promote it manually in the event of a Region failure.
**C)** Use AWS Database Migration Service to continuously replicate data to an RDS instance in a secondary Region.
**D)** Take automated RDS snapshots and copy them to a secondary Region using a Lambda function.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Cross-Region read replicas for RDS provide asynchronous replication to a secondary Region. In the event of a Region failure, the read replica can be manually promoted to a standalone database instance, providing access with minimal data loss (limited to the replication lag). Multi-AZ (Option A) protects against AZ failures but not Region-level failures. AWS DMS (Option C) could work but introduces additional complexity and operational overhead compared to native read replicas. Snapshot copying (Option D) results in potentially significant data loss since snapshots are point-in-time and the copy process adds delay. Cross-Region read replicas provide near-real-time replication and are the recommended approach.
</details>

---

### Question 16

A SysOps administrator is deploying a highly available web application using an Application Load Balancer. The ALB health check is configured with a health check path of `/health`, an interval of 30 seconds, a healthy threshold of 2, and an unhealthy threshold of 3. An instance starts returning HTTP 500 errors on the `/health` endpoint. What is the minimum time before the ALB marks the instance as unhealthy and stops sending traffic to it?

**A)** 30 seconds
**B)** 60 seconds
**C)** 90 seconds
**D)** 120 seconds

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** The ALB must receive 3 consecutive failed health checks (the unhealthy threshold) before marking the instance as unhealthy. With a health check interval of 30 seconds, the minimum time is 3 x 30 = 90 seconds. After the first failed check, the ALB waits 30 seconds for the second check, and another 30 seconds for the third. Only after the third consecutive failure does the ALB mark the target as unhealthy and stop routing new requests to it. Existing connections may continue based on the deregistration delay setting. Understanding these timing calculations is critical for SysOps administrators when designing health check configurations that balance quick detection with avoiding false positives.
</details>

---

### Question 17

A company operates a global application and needs to route users to the nearest healthy Region. The application runs in us-east-1, eu-west-1, and ap-southeast-1 behind Application Load Balancers in each Region. The administrator must ensure that if a Region becomes unhealthy, traffic is automatically redirected to the next closest healthy Region. Which Route 53 configuration achieves this?

**A)** Configure weighted routing with equal weights for all three Regions and associate health checks with each record.
**B)** Configure latency-based routing records for each Region and associate Route 53 health checks with each record.
**C)** Configure failover routing with us-east-1 as the primary and eu-west-1 as the secondary.
**D)** Configure geolocation routing records for each continent and associate health checks with each record.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Latency-based routing directs users to the Region with the lowest network latency, which effectively routes them to the nearest Region. When health checks are associated with each record, Route 53 automatically removes unhealthy endpoints from DNS responses and routes users to the next-best (lowest latency) healthy Region. Weighted routing (Option A) distributes traffic proportionally rather than by proximity. Failover routing (Option C) only supports one primary and one secondary, not three Regions. Geolocation routing (Option D) routes based on the user's geographic location, but it requires explicit continent/country mappings and does not automatically determine the closest Region based on network latency.
</details>

---

### Question 18

A SysOps administrator manages an Amazon EKS cluster running a microservices application. One of the services has a Horizontal Pod Autoscaler (HPA) configured but the cluster frequently runs out of compute capacity when new pods need to be scheduled. What should the administrator implement to ensure the cluster can accommodate the additional pods?

**A)** Increase the maximum pod count in the HPA configuration so more pods can be scheduled on existing nodes.
**B)** Configure the Kubernetes Cluster Autoscaler to automatically add or remove EC2 nodes based on pending pod resource requests.
**C)** Manually add more EC2 instances to the EKS managed node group based on a daily schedule.
**D)** Switch all workloads to Fargate profiles to eliminate the need for node management.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** The Kubernetes Cluster Autoscaler integrates with AWS Auto Scaling groups to automatically adjust the number of nodes in the cluster. When the HPA scales up pods but there is insufficient node capacity to schedule them, the Cluster Autoscaler detects pending pods and adds nodes to the cluster. Increasing the HPA maximum (Option A) would schedule more pods but does not solve the underlying capacity problem. Manual scaling (Option C) is not responsive to dynamic demand. While Fargate (Option D) eliminates node management, migrating all workloads to Fargate may not be practical due to compatibility limitations, cost differences, or specific requirements like GPU support or daemonsets.
</details>

---

### Question 19

A company uses Amazon EFS (Elastic File System) for shared storage accessed by EC2 instances across two Availability Zones. The administrator needs to ensure the file system remains accessible even if one AZ experiences an outage. Which statement about EFS availability is correct?

**A)** EFS stores data in a single AZ by default and requires manual replication to a second AZ for redundancy.
**B)** EFS with the Standard storage class automatically replicates data across multiple AZs within a Region, so no additional configuration is needed for multi-AZ availability.
**C)** EFS requires mount targets in at least three AZs to be considered highly available.
**D)** EFS must be paired with Amazon S3 cross-Region replication to achieve multi-AZ durability.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Amazon EFS with the Standard storage class automatically stores data redundantly across multiple Availability Zones within a Region. This provides built-in high availability and durability without any additional configuration. To access EFS from EC2 instances, mount targets should be created in each AZ where instances reside, but the data itself is already replicated across AZs. Option A describes the EFS One Zone storage class, not the Standard class. Option C is incorrect because two AZs are sufficient; there is no three-AZ minimum requirement. Option D conflates S3 and EFS capabilities; EFS handles its own multi-AZ replication natively.
</details>

---

### Question 20

A SysOps administrator is implementing connection draining for an application behind a Network Load Balancer. The application has long-lived TCP connections that can last up to 10 minutes. When instances are deregistered from the target group, active connections are being dropped immediately. What should the administrator configure?

**A)** Enable sticky sessions on the NLB target group to maintain connections during deregistration.
**B)** Increase the deregistration delay (connection draining timeout) on the target group to 600 seconds.
**C)** Switch from the NLB to an ALB, which supports connection draining by default.
**D)** Configure the Auto Scaling group termination policy to use NewestInstance to avoid terminating instances with active connections.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** The deregistration delay (also called connection draining) setting on the target group specifies how long the load balancer waits before completing the deregistration of a target that has in-flight requests. The default value is 300 seconds. Since the application has connections lasting up to 10 minutes (600 seconds), the deregistration delay should be set to 600 seconds to allow existing connections to complete gracefully. Sticky sessions (Option A) keep users on the same target but do not prevent connection drops during deregistration. ALBs (Option C) also support deregistration delay, but switching load balancer type is unnecessary. Termination policies (Option D) determine which instance is terminated but do not affect connection draining behavior.
</details>

---

### Question 21

A SysOps administrator manages an Auto Scaling group with a target tracking scaling policy set to maintain the average CPU utilization at 50%. The group has a minimum of 2 instances, a desired capacity of 4, and a maximum of 10 instances. During a deployment, one instance fails health checks and is terminated. What happens next?

**A)** The Auto Scaling group launches a new instance to replace the unhealthy one and maintain the desired capacity of 4.
**B)** The desired capacity is reduced to 3 and the target tracking policy evaluates whether to scale out.
**C)** The Auto Scaling group waits for the cooldown period before launching a replacement instance.
**D)** The administrator must manually adjust the desired capacity to trigger a new instance launch.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** When an Auto Scaling group detects an unhealthy instance, it terminates that instance and automatically launches a replacement to maintain the desired capacity. This is a core function of Auto Scaling and operates independently of scaling policies. The desired capacity remains at 4, so the group immediately launches a new instance. Health check replacements do not observe the scaling policy cooldown period (Option C). The desired capacity is not reduced (Option B) because the replacement is to maintain the existing desired count, not a scaling decision. No manual intervention is required (Option D) because Auto Scaling groups inherently maintain the desired capacity.
</details>

---

### Question 22

A company is designing a disaster recovery architecture for a critical application. The application uses Amazon Aurora MySQL in us-east-1. The RPO requirement is less than 1 second and the RTO requirement is less than 1 minute for a Region-level failure. Which Aurora feature meets these requirements?

**A)** Aurora cross-Region read replicas with automated promotion.
**B)** Aurora Global Database with managed planned failover.
**C)** Aurora automated backups with cross-Region backup replication.
**D)** Aurora Serverless v2 with multi-Region write capabilities.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Aurora Global Database uses dedicated replication infrastructure that provides cross-Region replication with typical lag of less than 1 second, meeting the RPO requirement. In the event of a Region-level failure, an Aurora Global Database secondary Region can be promoted to full read/write capability, typically within less than 1 minute, meeting the RTO requirement. Cross-Region read replicas (Option A) have higher replication lag and promotion takes longer. Automated backups (Option C) would result in minutes to hours of data loss and lengthy restoration times. Aurora Serverless v2 (Option D) does not offer multi-Region write capabilities as described.
</details>

---

### Question 23

A company runs a three-tier web application. The SysOps administrator must ensure the architecture can tolerate the failure of an entire Availability Zone. The application tier runs on EC2 instances in an Auto Scaling group, the database tier uses Amazon RDS for MySQL, and the caching tier uses Amazon ElastiCache for Memcached. Which components require configuration changes to achieve AZ fault tolerance? **(Choose THREE.)**

**A)** The Auto Scaling group must be configured to span at least two Availability Zones with a minimum capacity that can handle the full load in a single AZ.
**B)** The RDS for MySQL instance must have Multi-AZ enabled.
**C)** The ElastiCache for Memcached cluster must be configured with nodes in multiple AZs using the AZ-aware node placement feature.
**D)** The Application Load Balancer must be configured to use a single AZ to avoid routing traffic to the failed AZ.
**E)** The Auto Scaling group must use a single AZ to ensure consistent performance.

<details>
<summary>Show Answer</summary>

**Correct Answer: A, B, C**

**Explanation:** To tolerate an AZ failure, each tier must be distributed across multiple AZs. The Auto Scaling group (Option A) should span at least two AZs with enough minimum capacity to handle the workload if one AZ fails. RDS Multi-AZ (Option B) maintains a synchronous standby replica in a different AZ with automatic failover. ElastiCache for Memcached (Option C) supports spreading nodes across multiple AZs, so losing one AZ only affects a portion of the cached data. Option D is incorrect because ALBs should span multiple AZs and automatically stop routing to unhealthy targets. Option E would eliminate AZ fault tolerance entirely. All three tiers need multi-AZ configuration to achieve comprehensive fault tolerance.
</details>

---

### Question 24

A SysOps administrator has configured an Amazon CloudFront distribution with an Application Load Balancer as the origin. The ALB serves a dynamic web application. The administrator wants to reduce the load on the origin while ensuring that users always see up-to-date content for API responses that change every 60 seconds. Which caching strategy should be implemented?

**A)** Set the CloudFront minimum TTL to 0 and configure the origin to send `Cache-Control: max-age=60` headers on API responses.
**B)** Set the CloudFront maximum TTL to 3600 seconds and override all origin cache headers.
**C)** Disable caching entirely for dynamic content and use CloudFront only for static assets.
**D)** Configure CloudFront to cache all content for 24 hours and use Lambda@Edge to invalidate stale content.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** By setting the CloudFront minimum TTL to 0 and having the origin send `Cache-Control: max-age=60`, CloudFront will cache API responses for 60 seconds, matching the content update interval. This reduces origin load by serving cached responses for the majority of requests while ensuring content is never more than 60 seconds stale. Option B would cache content for up to an hour regardless of origin headers, serving stale content. Option C misses the opportunity to reduce load on the origin for content that only changes every 60 seconds. Option D is overly complex and caching for 24 hours would serve stale API responses. The origin-controlled caching approach gives the application precise control over cache behavior.
</details>

---

### Question 25

A company runs a production workload with an Auto Scaling group that has instances in three Availability Zones: AZ-A (3 instances), AZ-B (3 instances), and AZ-C (2 instances). The desired capacity is 8. A scale-in event needs to reduce the desired capacity to 6. Using the default termination policy, which instances will Auto Scaling terminate?

**A)** One instance from AZ-A and one from AZ-B, selecting the instances with the oldest launch template version in each AZ.
**B)** Two instances from AZ-C because it has the fewest instances and terminating those minimizes impact.
**C)** One instance from AZ-A and one from AZ-B, selecting the instances closest to the next billing hour in each AZ.
**D)** Two random instances from any AZ to maintain even distribution.

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** The default Auto Scaling termination policy first identifies the AZ(s) with the most instances to maintain balance across AZs. AZ-A and AZ-B each have 3 instances (tied for the most), so one instance will be terminated from each. Within each AZ, the default policy first selects instances with the oldest launch configuration or launch template. If there is still a tie, it selects the instance closest to the next billing hour to maximize instance usage. After termination, the distribution becomes AZ-A: 2, AZ-B: 2, AZ-C: 2, achieving even balance. The policy does not terminate from AZ-C (Option B) because it already has the fewest instances, and the goal is to maintain AZ balance.
</details>

---
