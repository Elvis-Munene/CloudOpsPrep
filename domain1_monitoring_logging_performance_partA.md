# Domain 1: Monitoring, Logging, Analysis, Remediation, and Performance Optimization (Part A)

## Task 1.1: Implement Metrics, Alarms, and Filters
## Task 1.2: Identify and Remediate Issues

---

### Question 1

A SysOps administrator needs to monitor the memory utilization of a fleet of Amazon EC2 instances running a critical application. The default CloudWatch metrics do not show memory usage. What should the administrator do to collect memory utilization metrics?

**A)** Enable detailed monitoring on the EC2 instances to unlock memory metrics at 1-minute intervals.
**B)** Install and configure the unified CloudWatch agent on the EC2 instances to publish custom memory metrics.
**C)** Create a CloudWatch metric filter on VPC Flow Logs to extract memory utilization data.
**D)** Use AWS CloudTrail to capture memory utilization events from the EC2 instances.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Memory utilization is not a default CloudWatch metric for EC2 instances. Enabling detailed monitoring (Option A) only increases the frequency of existing default metrics to 1-minute intervals but does not add new metric types like memory. The unified CloudWatch agent must be installed on the EC2 instances to collect OS-level metrics such as memory and disk utilization. CloudTrail (Option D) logs API calls, not system performance metrics. VPC Flow Logs (Option C) capture network traffic metadata, not memory data.
</details>

---

### Question 2

A company uses Amazon CloudWatch Logs to centralize application logs from multiple microservices running on Amazon ECS. A developer needs to find the top 10 most frequently occurring error messages across all log groups in the past 24 hours. Which approach is MOST efficient?

**A)** Export the logs to Amazon S3 and run Amazon Athena queries against the exported data.
**B)** Create a CloudWatch metric filter for each error type and review the resulting custom metrics.
**C)** Use CloudWatch Logs Insights to run an interactive query across the relevant log groups.
**D)** Stream the logs to Amazon OpenSearch Service and build a Kibana dashboard.

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** CloudWatch Logs Insights is purpose-built for interactive, ad-hoc querying of log data in CloudWatch Logs. A query such as `filter @message like /ERROR/ | stats count(*) as errorCount by @message | sort errorCount desc | limit 10` would efficiently return the top 10 error messages. Exporting to S3 and querying with Athena (Option A) introduces delays because log export is not real-time. Creating individual metric filters (Option B) requires knowing each error type in advance and is not suitable for ad-hoc analysis. Streaming to OpenSearch (Option D) is a valid long-term solution but far too complex for a quick investigative query.
</details>

---

### Question 3

An operations team has configured a CloudWatch alarm to monitor the CPUUtilization of an Auto Scaling group. The alarm is currently in the INSUFFICIENT_DATA state. What is the MOST likely cause?

**A)** The Auto Scaling group has scaled in to zero instances, so no metric data points are being published.
**B)** The IAM role attached to the CloudWatch alarm does not have permission to read EC2 metrics.
**C)** The alarm threshold has been set to a value that the CPUUtilization metric can never reach.
**D)** The CloudWatch agent is not installed on the instances in the Auto Scaling group.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** A CloudWatch alarm enters the INSUFFICIENT_DATA state when it does not have enough data points to evaluate the alarm condition. If the Auto Scaling group has scaled to zero instances, no CPUUtilization data is being published, causing this state. CloudWatch alarms do not use IAM roles themselves to read metrics (Option B); they operate within the CloudWatch service. An unreachable threshold (Option C) would cause the alarm to remain in the OK state, not INSUFFICIENT_DATA. CPUUtilization is a default EC2 metric and does not require the CloudWatch agent (Option D).
</details>

---

### Question 4

A SysOps administrator needs to receive a single notification only when BOTH the CPUUtilization alarm AND the StatusCheckFailed alarm for an EC2 instance are in the ALARM state simultaneously. Which solution meets this requirement?

**A)** Create an Amazon EventBridge rule that matches both alarm state changes and triggers an SNS notification.
**B)** Create a CloudWatch composite alarm that combines both alarms using an AND rule, and attach an SNS action to the composite alarm.
**C)** Configure both alarms to send notifications to the same SNS topic so the team receives alerts from either alarm.
**D)** Use a CloudWatch Logs metric filter to correlate the two alarm states and trigger a notification.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** CloudWatch composite alarms allow you to combine multiple alarms using Boolean logic (AND, OR, NOT). By creating a composite alarm with an AND rule referencing both the CPUUtilization and StatusCheckFailed alarms, the composite alarm enters ALARM state only when both child alarms are simultaneously in ALARM. An EventBridge rule (Option A) would trigger on each alarm independently and cannot natively enforce a simultaneous AND condition without additional state management. Sending both alarms to the same SNS topic (Option C) would send separate notifications for each alarm, not a single combined notification. Metric filters (Option D) operate on log data, not alarm states.
</details>

---

### Question 5

A company wants to validate that their AWS CloudTrail log files have not been tampered with after delivery to the S3 bucket. Which feature should the SysOps administrator enable?

**A)** Enable S3 server-side encryption with AWS KMS (SSE-KMS) on the CloudTrail S3 bucket.
**B)** Enable CloudTrail log file integrity validation when creating or updating the trail.
**C)** Enable S3 Object Lock in governance mode on the CloudTrail S3 bucket.
**D)** Enable S3 versioning and MFA Delete on the CloudTrail S3 bucket.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** CloudTrail log file integrity validation creates a digitally signed digest file for each hour of log delivery. These digest files contain hash values for every log file delivered during that period, enabling you to detect whether log files have been modified, deleted, or remain unchanged after delivery. SSE-KMS (Option A) encrypts logs at rest but does not verify integrity. S3 Object Lock (Option C) prevents deletion or overwriting but does not provide cryptographic validation of file contents. S3 versioning with MFA Delete (Option D) protects against accidental deletion but does not validate that log file contents are unaltered.
</details>

---

### Question 6

A SysOps administrator is configuring the CloudWatch agent on an Amazon EC2 Linux instance. The administrator wants to collect custom application metrics using the StatsD protocol. Which section of the CloudWatch agent configuration file should be modified to enable StatsD metric collection?

**A)** The `logs` section with a `statsd` subsection specifying the UDP port.
**B)** The `metrics` section with a `metrics_collected.statsd` subsection specifying the service address.
**C)** The `traces` section with a `statsd` subsection specifying the collection interval.
**D)** The `metrics` section with a `metrics_collected.collectd` subsection specifying the StatsD plugin.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** In the CloudWatch agent configuration JSON file, StatsD is configured under `metrics.metrics_collected.statsd`. This subsection allows you to specify parameters such as `service_address` (the UDP port the agent listens on, default `:8125`), `metrics_collection_interval`, and `metrics_aggregation_interval`. The `logs` section (Option A) is for log collection, not metrics. The `traces` section (Option C) is for X-Ray trace collection. The `collectd` subsection (Option D) is a separate protocol from StatsD; while both are supported by the CloudWatch agent, they are configured independently under their own subsections.
</details>

---

### Question 7

An operations team needs to automatically restart an EC2 instance when its StatusCheckFailed_System alarm enters the ALARM state. The solution must require the LEAST operational overhead. What should the administrator configure?

**A)** Create an Amazon EventBridge rule to match the alarm state change and invoke an AWS Lambda function that calls the EC2 RebootInstances API.
**B)** Configure the CloudWatch alarm action to directly perform the EC2 recover action on the instance.
**C)** Create an AWS Systems Manager Automation runbook that polls the alarm state and restarts the instance.
**D)** Configure the CloudWatch alarm action to send a message to an SQS queue, then use a Lambda function to process the queue and restart the instance.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** CloudWatch alarms support built-in EC2 actions including stop, terminate, reboot, and recover. For a system status check failure, the recover action is the most appropriate built-in action because it migrates the instance to new underlying hardware while retaining the instance ID, metadata, and EBS volumes. This requires no custom code or additional services, providing the least operational overhead. Using EventBridge with Lambda (Option A) or SQS with Lambda (Option D) adds unnecessary complexity. Polling from an SSM Automation runbook (Option C) is not event-driven and adds latency and overhead.
</details>

---

### Question 8

A SysOps administrator manages resources across three AWS accounts and needs to create a single CloudWatch dashboard that displays metrics from all three accounts. What must be configured to enable this?

**A)** Enable CloudWatch cross-account observability by configuring a monitoring account and source accounts using the CloudWatch cross-account sharing feature.
**B)** Replicate all CloudWatch metrics from the three accounts into a single account using Kinesis Data Firehose.
**C)** Create IAM roles in each account that trust the dashboard account and manually assume roles in the dashboard widgets.
**D)** Use AWS CloudTrail organization trails to aggregate metrics from all accounts into one dashboard.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** CloudWatch cross-account observability allows you to designate one account as the monitoring account and link other accounts as source accounts. Once configured, the monitoring account can view and create dashboards using metrics, logs, and traces from the source accounts without additional data movement. Replicating metrics via Kinesis Data Firehose (Option B) is complex and introduces cost and latency. While cross-account IAM roles (Option C) could theoretically work for API calls, CloudWatch dashboards natively support cross-account observability without manual role assumption. CloudTrail (Option D) logs API activity, not performance metrics.
</details>

---

### Question 9

A company runs a containerized application on Amazon EKS and wants to collect Prometheus-compatible metrics from its workloads and store them in a managed, scalable time-series database. Which AWS service should the SysOps administrator use?

**A)** Amazon CloudWatch with custom metrics published via the PutMetricData API.
**B)** Amazon Managed Service for Prometheus (AMP) with a Prometheus-compatible collector deployed in the EKS cluster.
**C)** Amazon Timestream with a custom Lambda function to ingest Prometheus metrics.
**D)** Amazon OpenSearch Service with a Prometheus remote write plugin.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Amazon Managed Service for Prometheus (AMP) is a fully managed, Prometheus-compatible monitoring service that can ingest metrics using the Prometheus remote write protocol. By deploying a Prometheus server or the AWS Distro for OpenTelemetry (ADOT) collector in the EKS cluster, metrics are scraped from workloads and forwarded to AMP. This provides native Prometheus compatibility with managed scalability. Publishing to CloudWatch via PutMetricData (Option A) loses the Prometheus query language (PromQL) capability and metric model. Amazon Timestream (Option C) is a general-purpose time-series database but does not natively support PromQL. OpenSearch (Option D) is primarily designed for search and log analytics.
</details>

---

### Question 10

An EventBridge rule is configured to invoke a Lambda function whenever an EC2 instance changes to the `terminated` state. However, the Lambda function is not being triggered. Which TWO steps should the administrator take to troubleshoot this issue? (Select TWO.)

**A)** Verify that the EventBridge rule event pattern correctly matches the `EC2 Instance State-change Notification` detail-type with `state: terminated`.
**B)** Check that the Lambda function's resource-based policy grants `events.amazonaws.com` permission to invoke the function.
**C)** Ensure that the EC2 instances have an IAM instance profile that allows publishing events to EventBridge.
**D)** Verify that CloudTrail is enabled in the Region, because EC2 state-change events depend on CloudTrail.
**E)** Confirm that the EventBridge rule is in the ENABLED state and is associated with the correct event bus.

<details>
<summary>Show Answer</summary>

**Correct Answers: A, B**

**Explanation:** The two most common causes for an EventBridge rule not triggering a Lambda function are an incorrect event pattern and missing permissions. The event pattern must correctly specify `"source": ["aws.ec2"]` and `"detail-type": ["EC2 Instance State-change Notification"]` with `"detail": {"state": ["terminated"]}` (Option A). Additionally, the Lambda function must have a resource-based policy allowing the EventBridge service principal to invoke it (Option B). EC2 instances do not need IAM permissions to publish state-change events (Option C); these are generated by the EC2 service itself. EC2 state-change notifications are native EventBridge events and do not depend on CloudTrail (Option D). Option E is a reasonable check but less likely to be the root cause compared to A and B, as rules are enabled by default and use the default event bus.
</details>

---

### Question 11

A SysOps administrator wants to create a CloudWatch Logs Insights query to find the average latency of API requests grouped by the HTTP method over the last 6 hours. The log entries contain fields `@timestamp`, `httpMethod`, and `latency`. Which query is correct?

**A)**
```
fields @timestamp, httpMethod, latency
| filter latency > 0
| stats avg(latency) as avgLatency by httpMethod
| sort avgLatency desc
```

**B)**
```
fields @timestamp, httpMethod, latency
| group by httpMethod
| select avg(latency) as avgLatency
| order by avgLatency desc
```

**C)**
```
SELECT avg(latency) AS avgLatency
FROM logs
GROUP BY httpMethod
ORDER BY avgLatency DESC
```

**D)**
```
fields @timestamp, httpMethod, latency
| filter latency > 0
| aggregate avg(latency) as avgLatency by httpMethod
| sort avgLatency desc
```

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** CloudWatch Logs Insights uses its own query syntax. The `stats` command is used for aggregation functions such as `avg()`, `sum()`, `count()`, `min()`, and `max()`, with `by` specifying the grouping field. The `sort` command orders results. Option B uses `group by` and `select`, which are not valid Logs Insights commands. Option C uses SQL syntax, which is not supported by Logs Insights. Option D uses `aggregate`, which is not a valid Logs Insights command; the correct command is `stats`.
</details>

---

### Question 12

A SysOps administrator needs to automatically remediate non-compliant security group rules that allow unrestricted SSH access (0.0.0.0/0 on port 22). The solution should use AWS Systems Manager. Which approach is MOST appropriate?

**A)** Create an SSM Automation runbook using the predefined `AWS-DisablePublicAccessForSecurityGroup` document, triggered by an EventBridge rule when AWS Config detects a non-compliant resource.
**B)** Create an SSM Run Command document that executes a shell script on each EC2 instance to modify its security group.
**C)** Use SSM Patch Manager to scan for and remediate insecure security group configurations.
**D)** Create an SSM State Manager association that periodically scans all security groups and removes offending rules using the AWS CLI.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** AWS Systems Manager Automation provides predefined runbooks such as `AWS-DisablePublicAccessForSecurityGroup` that can revoke insecure ingress rules. By combining an AWS Config rule (e.g., `restricted-ssh`) with an EventBridge rule that triggers the SSM Automation runbook, non-compliant security groups are automatically remediated. SSM Run Command (Option B) executes commands on managed instances, not on security group configurations. SSM Patch Manager (Option C) manages OS patching, not security group rules. While SSM State Manager (Option D) could theoretically run periodic checks, it is less event-driven and less efficient than the Config-EventBridge-Automation approach.
</details>

---

### Question 13

An administrator has configured a CloudWatch alarm with a period of 300 seconds and an evaluation period of 3 data points. The alarm uses the `Average` statistic and a threshold of 80% for CPUUtilization. In which scenario will the alarm transition to the ALARM state?

**A)** A single 5-minute period shows an average CPU of 95%.
**B)** Three consecutive 5-minute periods each show an average CPU above 80%.
**C)** At least one out of three consecutive 5-minute periods shows an average CPU above 80%.
**D)** The cumulative average CPU over a 15-minute window exceeds 80%.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** With a period of 300 seconds (5 minutes) and an evaluation period of 3 data points, the alarm evaluates 3 consecutive 5-minute data points. By default (when "datapoints to alarm" equals the evaluation period), ALL 3 data points must breach the threshold for the alarm to transition to ALARM. A single breaching period (Option A) is not enough. Option C describes an "M out of N" configuration where datapoints to alarm is set to 1 out of 3, which is not the default. Option D is incorrect because each period is evaluated independently, not as a cumulative average over the full window.
</details>

---

### Question 14

A SysOps administrator needs to forward CloudWatch Logs from a production account to a centralized logging account in near real-time. Which solution provides the LOWEST latency?

**A)** Schedule hourly exports of the log group to an S3 bucket in the centralized account using the `CreateExportTask` API.
**B)** Create a CloudWatch Logs subscription filter that streams logs to a Kinesis Data Stream or Kinesis Data Firehose in the centralized account.
**C)** Use AWS DataSync to replicate log data from the production account's CloudWatch Logs to the centralized account.
**D)** Configure the CloudWatch agent on all instances to send logs directly to both the local and centralized accounts simultaneously.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** CloudWatch Logs subscription filters enable near real-time streaming of log data to destinations such as Kinesis Data Streams, Kinesis Data Firehose, or Lambda. Cross-account delivery via Kinesis provides the lowest latency for centralizing logs. The `CreateExportTask` API (Option A) is a batch operation with significant delay and cannot run more frequently than every few hours for a given log group. AWS DataSync (Option C) is designed for file-based data transfer, not CloudWatch Logs streaming. The CloudWatch agent (Option D) cannot natively write to log groups in a different account without complex cross-account role assumption and is not the recommended approach.
</details>

---

### Question 15

A company uses Amazon EventBridge to route events from a custom application to multiple targets including Lambda, Step Functions, and SQS. The administrator notices that events are reaching the Lambda target but NOT the SQS queue. What is the MOST likely cause?

**A)** EventBridge rules can only have a single target; the SQS target is being ignored.
**B)** The SQS queue policy does not grant `sqs:SendMessage` permission to the `events.amazonaws.com` service principal.
**C)** The Lambda function is consuming the event and preventing it from reaching the SQS queue.
**D)** SQS is not a supported target for Amazon EventBridge rules.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** EventBridge rules can have up to five targets, and each target receives the event independently. If events reach the Lambda target but not SQS, the most likely cause is a missing permission on the SQS queue. The SQS queue's resource-based policy must include an `sqs:SendMessage` permission for the `events.amazonaws.com` service principal. EventBridge supports multiple targets per rule (Option A is incorrect). Targets receive events independently and do not block each other (Option C is incorrect). SQS is a fully supported EventBridge target (Option D is incorrect).
</details>

---

### Question 16

A SysOps administrator wants to detect unusual patterns in application request counts without manually setting static thresholds. Which CloudWatch feature should be used?

**A)** CloudWatch metric math expressions to calculate standard deviations.
**B)** CloudWatch anomaly detection, which uses machine learning to create a model of expected metric behavior.
**C)** CloudWatch Logs Insights pattern analysis to group similar log entries.
**D)** CloudWatch Synthetics canaries to simulate traffic and compare against actual request counts.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** CloudWatch anomaly detection uses machine learning algorithms to analyze historical metric data and create a band of expected values that accounts for trends and seasonal patterns. You can create alarms based on this anomaly detection model, which will trigger when the metric value falls outside the expected band. Metric math (Option A) can calculate statistics but requires manual threshold configuration and does not learn patterns automatically. Logs Insights pattern analysis (Option C) groups log entries by similarity but does not perform time-series anomaly detection on metrics. Synthetics canaries (Option D) monitor endpoint availability and latency, not request count anomalies.
</details>

---

### Question 17

An administrator needs to create a custom SSM Automation runbook that performs the following steps in order: (1) Stop an EC2 instance, (2) Modify the instance type, (3) Start the instance. Which SSM Automation action types should be used for these steps?

**A)** Use `aws:executeScript` for all three steps with Python boto3 code.
**B)** Use `aws:changeInstanceState` to stop and start the instance, and `aws:executeAwsApi` to call the ModifyInstanceAttribute API.
**C)** Use `aws:runCommand` for all three steps, executing AWS CLI commands on the instance.
**D)** Use `aws:invokeLambdaFunction` for all three steps, delegating each action to a separate Lambda function.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** SSM Automation provides specific action types optimized for common operations. The `aws:changeInstanceState` action is designed to stop, start, or terminate EC2 instances and includes built-in polling to wait for the desired state. The `aws:executeAwsApi` action can call any AWS API, making it ideal for calling `ModifyInstanceAttribute` to change the instance type. While `aws:executeScript` (Option A) could work, it adds unnecessary complexity. `aws:runCommand` (Option C) executes commands on the instance, which is not possible when the instance is stopped. Using Lambda for each step (Option D) adds unnecessary overhead and complexity.
</details>

---

### Question 18

A SysOps administrator is configuring the CloudWatch agent on Amazon EC2 Linux instances and wants to collect both StatsD custom metrics and collectd system metrics. Which statement about this configuration is correct?

**A)** The CloudWatch agent can collect StatsD OR collectd metrics, but not both simultaneously on the same instance.
**B)** Both StatsD and collectd can be configured under the `metrics.metrics_collected` section, each with their own subsection, and both run simultaneously.
**C)** StatsD metrics are collected in the `metrics` section and collectd metrics are collected in the `logs` section of the agent configuration.
**D)** collectd is a built-in feature of the CloudWatch agent and does not require a separate collectd daemon on the host.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** The CloudWatch agent configuration file supports both `statsd` and `collectd` as separate subsections under `metrics.metrics_collected`. Both protocols can run simultaneously on the same instance, each listening on its own network port. The CloudWatch agent acts as a receiver for both protocols. Option A is incorrect because both can coexist. Option C is incorrect because both are metric protocols configured in the `metrics` section. Option D is incorrect because the CloudWatch agent relies on an external collectd daemon to send metrics to it; collectd must be separately installed and configured to use the network plugin to send data to the CloudWatch agent's listener.
</details>

---

### Question 19

An EC2 instance in a production environment becomes unresponsive periodically. The operations team suspects memory pressure but needs automated remediation. Which solution uses AWS Systems Manager to automatically resolve this with MINIMAL custom code?

**A)** Create a CloudWatch alarm on a custom memory metric and use an EventBridge rule to trigger the `AWS-RestartEC2Instance` SSM Automation runbook.
**B)** Deploy a cron job on the instance that monitors memory and restarts the instance using the AWS CLI.
**C)** Create a CloudWatch alarm that sends an SNS notification to the operations team for manual investigation.
**D)** Use SSM Patch Manager to schedule weekly instance reboots during a maintenance window.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** The `AWS-RestartEC2Instance` is a predefined SSM Automation runbook that stops and starts an EC2 instance. By creating a CloudWatch alarm on the custom memory metric (published via the CloudWatch agent) and using an EventBridge rule to trigger this runbook when the alarm enters ALARM state, the team achieves automated remediation with minimal custom code. A cron job (Option B) requires custom scripting and does not integrate with AWS monitoring services. An SNS notification (Option C) requires manual intervention and is not automated remediation. SSM Patch Manager (Option D) is for OS patching, and scheduled weekly reboots do not address the root cause of memory pressure events.
</details>

---

### Question 20

A SysOps administrator needs to monitor a custom application metric that is published every 10 seconds. The metric should be available for alarming with a 10-second resolution. Which configuration is required when publishing the metric?

**A)** Publish the metric using `PutMetricData` with the `StorageResolution` parameter set to 1 (high-resolution).
**B)** Publish the metric using `PutMetricData` with detailed monitoring enabled in the API call.
**C)** Publish the metric using the CloudWatch agent with the `metrics_collection_interval` set to 10 seconds.
**D)** Enable high-resolution mode on the CloudWatch alarm; metric resolution does not need to change.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** To store metrics at a resolution finer than 60 seconds (the default), you must set the `StorageResolution` parameter to 1 when calling the `PutMetricData` API. This enables high-resolution custom metrics that can store data at 1-second granularity and allows alarms to evaluate at periods as short as 10 seconds. Detailed monitoring (Option B) is an EC2-specific feature and not an API parameter for custom metrics. The CloudWatch agent (Option C) can collect at high frequency, but when publishing custom metrics via the API directly, `StorageResolution` must be explicitly set. Option D is incorrect because alarm evaluation period depends on the metric's storage resolution; a standard-resolution metric cannot be alarmed at 10-second intervals.
</details>

---

### Question 21

A SysOps administrator needs to create a CloudWatch metric filter that extracts the response time value from log entries formatted as: `[INFO] Request completed - responseTime=245ms status=200`. The administrator wants to create a metric called `ResponseTime` in the `CustomApp` namespace. Which metric filter pattern correctly extracts the response time?

**A)** `[level, ..., responseTime=*ms, status]` with a metric value of `$responseTime`
**B)** `{ $.responseTime = * }` with a metric value of `$.responseTime`
**C)** `[level, info, request, completed, dash, responseTimeLabel, statusLabel] ` with a metric value of `$responseTimeLabel`
**D)** `"responseTime"` with a metric value of `1` to count occurrences

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** CloudWatch Logs metric filter patterns support space-delimited filter syntax using square brackets. The pattern `[level, ..., responseTime=*ms, status]` uses `...` as a wildcard to skip intermediate fields and matches the `responseTime` field by its prefix pattern. The extracted value can be referenced as `$responseTime` in the metric value configuration. Option B uses JSON filter syntax (`$.field`), but the log format is not JSON. Option C attempts to match each space-delimited token but the field naming does not correctly extract the numeric value. Option D simply counts occurrences and does not extract the actual response time value, losing the metric's usefulness for latency monitoring.
</details>

---

### Question 22

An organization has an EventBridge event bus that receives custom events from multiple applications. A new rule was created to match events with `"source": "app.orders"` and `"detail-type": "OrderPlaced"`, but the rule is not matching any events. The administrator confirms that the application is publishing events. Which TWO actions should the administrator take to diagnose the issue? (Select TWO.)

**A)** Check the event bus specified in the rule matches the event bus to which the application is publishing events.
**B)** Verify that the event pattern in the rule uses exact case-sensitive matching for the `source` and `detail-type` fields.
**C)** Enable CloudTrail logging because EventBridge rules only process events that are also logged in CloudTrail.
**D)** Increase the EventBridge throughput quota because the events may be throttled.
**E)** Restart the EventBridge service in the AWS console to clear any cached rule configurations.

<details>
<summary>Show Answer</summary>

**Correct Answers: A, B**

**Explanation:** The two most common causes for EventBridge rules not matching events are incorrect event bus routing and event pattern mismatches. If the application publishes to a custom event bus but the rule is on the default event bus (or vice versa), no events will match (Option A). EventBridge event patterns are case-sensitive, so `"OrderPlaced"` will not match `"orderPlaced"` or `"ORDERPLACED"` (Option B). CloudTrail (Option C) is not required for EventBridge rules to function; EventBridge processes events independently. EventBridge quotas (Option D) are generous and throttling would generate errors, not silent failures. EventBridge is a managed serverless service that cannot be restarted (Option E).
</details>

---

### Question 23

A company wants to receive an SNS notification whenever an IAM policy is modified in their AWS account. Which combination of services should the administrator configure?

**A)** AWS CloudTrail logging to CloudWatch Logs, with a CloudWatch metric filter matching IAM policy change API calls, and a CloudWatch alarm that notifies via SNS.
**B)** AWS Config rule for IAM policy changes with a direct SNS notification action.
**C)** Enable IAM Access Analyzer and configure it to send SNS notifications for policy modifications.
**D)** Enable VPC Flow Logs and create a metric filter to detect IAM-related API traffic.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** CloudTrail records all IAM API calls. By sending CloudTrail logs to CloudWatch Logs and creating a metric filter that matches API calls like `CreatePolicy`, `DeletePolicy`, `AttachRolePolicy`, `PutRolePolicy`, etc., you can create a CloudWatch alarm that triggers an SNS notification when these events occur. An alternative would be an EventBridge rule matching CloudTrail events, but among the given options, Option A is correct. AWS Config (Option B) tracks configuration compliance but does not directly send SNS notifications for individual API changes without additional configuration. IAM Access Analyzer (Option C) identifies overly permissive policies but does not monitor for policy modification events. VPC Flow Logs (Option D) capture network traffic and have no visibility into IAM operations.
</details>

---

### Question 24

A SysOps administrator is configuring CloudWatch alarms for a fleet of EC2 instances behind an Application Load Balancer. The alarm should trigger when the average response time exceeds 2 seconds AND the request count is above 1,000 per minute, to avoid false positives during low-traffic periods. What is the BEST way to implement this?

**A)** Create a single CloudWatch alarm using metric math to calculate `IF(RequestCount > 1000, TargetResponseTime, 0)` and set the threshold at 2.
**B)** Create two separate CloudWatch alarms and configure a composite alarm with an AND condition.
**C)** Create a single alarm on TargetResponseTime with the threshold set to 2 seconds and set the alarm evaluation to only occur during business hours.
**D)** Create a Lambda function that periodically queries both metrics and publishes a combined custom metric, then alarm on that metric.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** CloudWatch composite alarms are designed to combine multiple alarm conditions using Boolean logic. By creating one alarm for TargetResponseTime exceeding 2 seconds and another for RequestCount exceeding 1,000, a composite alarm with an AND condition will only trigger when both conditions are simultaneously met. This cleanly avoids false positives during low-traffic periods. Metric math (Option A) could work but the expression is less intuitive and harder to maintain. Business-hours-only evaluation (Option C) is not a built-in CloudWatch alarm feature and does not address the request count condition. A Lambda function (Option D) adds unnecessary complexity and operational overhead compared to the native composite alarm feature.
</details>

---

### Question 25

A SysOps administrator is troubleshooting an SSM Automation runbook that is failing at a step that calls `aws:executeAwsApi` to describe EC2 instances. The runbook is invoked by an EventBridge rule. The error message indicates "AccessDeniedException". What is the MOST likely cause?

**A)** The EventBridge rule does not have a resource-based policy allowing it to call SSM Automation.
**B)** The SSM Automation assume role (the role specified in the runbook execution) does not have the required `ec2:DescribeInstances` permission.
**C)** The EC2 instances do not have the SSM Agent installed, preventing the API call.
**D)** The CloudWatch alarm that triggers the EventBridge rule does not have permission to pass the IAM role to SSM.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** When an SSM Automation runbook executes, it uses an IAM role (the assume role) that provides the permissions for all actions performed by the runbook. If the `aws:executeAwsApi` step is calling `ec2:DescribeInstances` and receiving an AccessDeniedException, the automation's assume role lacks the `ec2:DescribeInstances` permission. The EventBridge rule's ability to start the automation (Option A) is a separate concern; since the runbook started and reached the failing step, EventBridge permissions are not the issue. The SSM Agent (Option C) is required for Run Command and Session Manager, not for Automation API calls. The CloudWatch alarm (Option D) does not directly pass roles to SSM; the EventBridge rule's target configuration specifies the IAM role for automation execution.
</details>

---
