# Amazon EventBridge - Deep Dive Questions (SOA-C03)

## Focused Exam Preparation: EventBridge

This file contains 20 scenario-based questions covering Amazon EventBridge topics critical for the AWS Certified SysOps Administrator - Associate (SOA-C03) exam.

**Topics Covered:**
- Event buses (default, custom, partner)
- Event rules and event patterns
- Schedule rules (cron and rate expressions)
- Multiple target types
- EventBridge Pipes
- Archive and replay
- Cross-account and cross-region delivery
- Schema registry
- EventBridge vs SNS vs SQS
- Auto-remediation and operational automation
- Input transformers
- Dead-letter queues
- Integration with AWS services

---

### Question 1

A SysOps administrator needs to set up automatic remediation when an EC2 instance is launched without encryption enabled on its EBS volumes. AWS Config has already been configured to detect non-compliant resources. Which solution provides the MOST automated remediation?

**A)** Create an Amazon EventBridge rule that matches AWS Config compliance change events and triggers a Lambda function to terminate non-compliant instances.
**B)** Create an Amazon EventBridge rule that matches EC2 RunInstances events from CloudTrail, filter for unencrypted volumes, and trigger an AWS Systems Manager Automation document to enable encryption.
**C)** Use AWS Config remediation actions to directly invoke an SSM Automation document when a non-compliant instance is detected.
**D)** Create an SNS topic subscribed to AWS Config notifications and use a Lambda function to remediate non-compliant instances.

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** AWS Config has native integration with SSM Automation for automatic remediation. When Config detects a non-compliant resource, it can directly invoke an SSM Automation document without requiring EventBridge or Lambda. This is the most automated and least complex solution. Option A involves EventBridge but terminates instances rather than fixing them. Option B is proactive but complex because it requires parsing CloudTrail events and cannot encrypt volumes after instance launch (encryption must be set at creation). Option D adds unnecessary SNS complexity when Config can remediate directly.

**Key Concept:** AWS Config automatic remediation is the preferred pattern for compliance-based fixes. EventBridge is better suited for event-driven workflows that don't fit Config's compliance model.

</details>

---

### Question 2

A company needs to send all EC2 instance state change events (running, stopped, terminated) from their production AWS account (Account A) to their central monitoring account (Account B) where a Lambda function processes these events. What is the MOST secure way to configure this?

**A)** In Account A, create an EventBridge rule that sends events to the default event bus in Account B. In Account B, configure the event bus policy to allow Account A to put events, then create a rule to trigger the Lambda function.
**B)** In Account A, create an EventBridge rule that triggers an SNS topic. Configure the SNS topic to allow cross-account access from Account B's Lambda function.
**C)** Configure AWS Organizations to automatically share all EventBridge events across accounts, then create a rule in Account B.
**D)** In Account A, create an EventBridge rule that writes events to an S3 bucket. In Account B, configure S3 event notifications to trigger the Lambda function.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** EventBridge supports native cross-account event delivery. The proper setup involves: (1) In Account A, create a rule with a target pointing to Account B's event bus ARN. (2) In Account B, add a resource-based policy to the event bus allowing Account A to put events using `events:PutEvents`. (3) In Account B, create a rule matching the events and targeting the Lambda function. This is secure, direct, and requires no intermediate services. Option B works but adds unnecessary SNS complexity. Option C is incorrect; Organizations doesn't automatically share EventBridge events. Option D is inefficient and introduces latency.

**Key Concept:** Cross-account EventBridge requires explicit bus policy permissions in the target account and proper rule configuration in both accounts.

</details>

---

### Question 3

A SysOps administrator needs to create an EventBridge rule that triggers only when an EC2 instance in the "Production" environment (tagged with Environment=Production) transitions to the "stopped" state. Which event pattern is correct?

**A)**
```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": {
    "state": ["stopped"],
    "tags": {
      "Environment": ["Production"]
    }
  }
}
```

**B)**
```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": {
    "state": ["stopped"]
  },
  "resources": {
    "tags": {
      "Environment": ["Production"]
    }
  }
}
```

**C)**
```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": {
    "state": ["stopped"]
  }
}
```

**D)**
```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": {
    "state": ["stopped"]
  },
  "tags": {
    "Environment": ["Production"]
  }
}
```

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** This is a tricky question. EC2 Instance State-change Notification events do NOT include instance tags in their event payload. The event only contains instance-id, state, and other core metadata. To filter based on tags, you would need to either: (1) Use the Lambda function to look up tags via the EC2 API, or (2) Create separate rules for tagged instances using resource-based filtering if supported, or (3) Use EventBridge Pipes with enrichment. Options A, B, and D attempt to filter on tags that aren't present in the event structure. Option C is the correct event pattern for stopped instances; additional tag filtering must happen downstream.

**Key Concept:** Understand what data is actually present in AWS service events. Not all resource attributes are included in EventBridge events. Check AWS documentation for exact event schemas.

</details>

---

### Question 4

A company wants to automatically take a snapshot of an RDS database every day at 2:00 AM UTC and every 6 hours throughout the day. Which EventBridge schedule expressions should be used?

**A)** Create one rule with schedule: `cron(0 2 * * ? *)` and `rate(6 hours)`
**B)** Create two separate rules: one with `cron(0 2 * * ? *)` and another with `rate(6 hours)`
**C)** Create one rule with schedule: `rate(6 hours)` and use input transformer to skip execution except at 2:00 AM
**D)** Create one rule with schedule: `cron(0 2,8,14,20 * * ? *)`

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** EventBridge schedule rules can only have ONE schedule expression per rule. To implement both a specific daily time and a recurring interval, you must create two separate rules. Option A is syntactically invalid; you cannot specify two schedule expressions in one rule. Option C is inefficient and complex. Option D uses a cron expression for 2 AM, 8 AM, 2 PM, and 8 PM, which satisfies the 6-hour interval but is more rigid than a rate expression. However, it doesn't exactly meet the requirement of "every 6 hours throughout the day" if that means continuous 6-hour cycles. Option B is the cleanest and most flexible approach with one rule for the exact daily time and another for the recurring interval.

**Key Concept:** Each EventBridge rule supports one schedule expression. Use `rate()` for intervals and `cron()` for specific times. Multiple schedules require multiple rules.

</details>

---

### Question 5

A SysOps administrator has configured an EventBridge rule to trigger a Lambda function when an AWS Health event indicates an issue with EC2 instances. The rule is active, but the Lambda function is not being invoked during a known health event. What is the MOST likely cause?

**A)** The EventBridge rule is on a custom event bus, but AWS Health events are only published to the default event bus.
**B)** The Lambda function's execution role does not have permissions to be invoked by EventBridge.
**C)** AWS Health events are account-specific and require AWS Business or Enterprise Support to receive events via EventBridge.
**D)** The event pattern does not match because AWS Health events use a different source name than specified in the rule.

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** AWS Health events are delivered to EventBridge, but account-specific events (not just service-wide events) require AWS Business, Enterprise On-Ramp, or Enterprise Support plans. Without the appropriate support plan, only public AWS Health events affecting AWS services globally are available, not events specific to your account's resources. Option A is incorrect; AWS Health events do publish to the default event bus, which is where most AWS service events appear. Option B is unlikely; EventBridge automatically adds the necessary resource-based policy to Lambda when you create the rule via the console or properly via API/CloudFormation. Option D could be checked, but the more common issue is the support plan limitation.

**Key Concept:** Account-specific AWS Health events require Business or Enterprise Support. This is a common exam gotcha.

</details>

---

### Question 6

A company uses EventBridge to route events from multiple sources to various targets. They need to ensure that if a Lambda target fails to process an event, the event is retained for later analysis. What should the SysOps administrator configure?

**A)** Enable EventBridge event archiving with a retention period, and create a replay to reprocess failed events.
**B)** Configure a dead-letter queue (DLQ) on the EventBridge rule target configuration, specifying an SQS queue.
**C)** Configure a dead-letter queue on the Lambda function itself to capture failed invocations.
**D)** Enable EventBridge Pipes with a DLQ for failed event processing.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** EventBridge rules support per-target dead-letter queues. When configuring a target (like Lambda) on an EventBridge rule, you can specify an SQS queue as the DLQ. If EventBridge cannot deliver the event to the target (after retries), it sends the event to the DLQ. This is distinct from the Lambda function's own DLQ (Option C), which captures events that the function processed but failed during execution. Option A (archiving and replay) is for reprocessing events at scale, not for capturing failed deliveries. Option D mentions Pipes, which have their own DLQ configuration, but the question is about standard EventBridge rules.

**Key Concept:** EventBridge rules support target-level DLQs (SQS) for failed event delivery. This is separate from Lambda's asynchronous invocation DLQ.

</details>

---

### Question 7

A development team wants to replay all S3 object creation events from the past 7 days in a test environment to validate a new Lambda function. The events were originally processed in production. What must be configured to enable this?

**A)** Enable S3 event notifications to republish events from the past 7 days to EventBridge.
**B)** Create an EventBridge archive on the rule that processes S3 events, ensuring it has at least 7 days retention. Then create a replay targeting the test environment's event bus.
**C)** Use AWS CloudTrail Lake to query S3 API events and manually republish them to EventBridge.
**D)** Configure S3 to send events to an SQS queue with message retention of 7 days, then replay from the queue.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** EventBridge archive and replay is specifically designed for this use case. You create an archive on an event bus with a specified retention period (up to indefinite). The archive captures all events matching its event pattern. To replay, you specify the time range and target event bus. You can replay to the same bus or a different bus (like one in a test environment). Option A is incorrect; S3 event notifications are real-time and cannot republish historical events. Option C is possible but complex and not the intended solution. Option D doesn't help with replay; SQS would only retain unconsumed messages, and events already processed would not be in the queue.

**Key Concept:** EventBridge archive and replay enables event sourcing patterns and testing. Archives must be created before events occur.

</details>

---

### Question 8

An organization wants to centrally collect custom application events from 50 AWS accounts. Each account runs applications that publish custom events to their local EventBridge bus. What is the MOST scalable architecture?

**A)** Create a custom event bus in a central monitoring account. In each of the 50 accounts, create EventBridge rules that target the central account's custom bus. Configure the central bus policy to allow all 50 accounts to put events.
**B)** In each of the 50 accounts, configure EventBridge rules to send events to an SNS topic in the central account. Subscribe an SQS queue in the central account to process events.
**C)** Use AWS Organizations to automatically aggregate all custom events to the organization's management account event bus.
**D)** In each of the 50 accounts, configure EventBridge rules to write events to a central S3 bucket, then use S3 event notifications to trigger processing.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** EventBridge is designed for this cross-account event aggregation pattern. Creating a custom event bus in a central account and configuring cross-account rules is scalable and native to EventBridge. The resource-based policy on the central bus can use wildcards or conditions for the organization to avoid listing all 50 accounts individually. Option B works but adds unnecessary SNS/SQS complexity. Option C is incorrect; Organizations doesn't auto-aggregate EventBridge events. Option D is inefficient and introduces latency and cost.

**Key Concept:** Custom event buses with cross-account rules are the standard pattern for centralized event collection in multi-account environments.

</details>

---

### Question 9

A company receives events from a SaaS partner via Amazon EventBridge. The events arrive on a partner event bus. The SysOps administrator needs to route these events to multiple targets: a Lambda function, an SQS queue, and a Step Functions state machine. What is the correct approach?

**A)** Partner event buses cannot have rules; the events must be forwarded to the default event bus first, then create rules on the default bus.
**B)** Create a single EventBridge rule on the partner event bus with all three targets configured in the rule.
**C)** Create three separate EventBridge rules on the partner event bus, one for each target.
**D)** Partner events must be sent to an SNS topic, which then fans out to the three targets.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Partner event buses function like custom event buses and fully support EventBridge rules. A single EventBridge rule can have up to 5 targets, so configuring one rule with three targets is efficient and correct. Option A is incorrect; partner event buses support rules directly. Option C would work but is less efficient than using one rule with multiple targets. Option D adds unnecessary SNS complexity; EventBridge natively supports multiple targets.

**Key Concept:** EventBridge rules support up to 5 targets per rule. Partner event buses support rules just like custom and default event buses.

</details>

---

### Question 10

A SysOps administrator needs to transform the input sent to a Lambda function target from an EventBridge rule. The event contains an EC2 instance ID in the detail object, but the Lambda function expects a JSON payload with a specific structure including a custom message. Which EventBridge feature should be used?

**A)** Event pattern filtering to reshape the event structure before matching.
**B)** Input transformer to define a custom input template with variables from the event.
**C)** EventBridge Pipes with transformation functions to modify the event payload.
**D)** Lambda function environment variables to map the event structure at runtime.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** EventBridge input transformers allow you to customize the JSON sent to a target. You define an input path to extract values from the event (e.g., `"instance": "$.detail.instance-id"`) and an input template to construct the desired JSON structure (e.g., `{"instanceId": <instance>, "message": "Custom message"}`). Option A is incorrect; event patterns are for matching events, not transforming them. Option C refers to EventBridge Pipes, which do support transformations, but the question describes a standard EventBridge rule scenario. Option D works but pushes complexity to the Lambda function; input transformers handle this at the EventBridge level, which is cleaner.

**Key Concept:** Input transformers in EventBridge rules enable customization of the payload sent to targets without modifying the target code.

</details>

---

### Question 11

A company's compliance policy requires all security-related events from AWS GuardDuty to be processed by three different systems: a SIEM tool (via Kinesis Data Firehose), a Lambda function for auto-remediation, and an S3 bucket for audit logging. What is the MOST efficient EventBridge configuration?

**A)** Create three separate EventBridge rules, each matching GuardDuty findings and targeting one system.
**B)** Create one EventBridge rule matching GuardDuty findings with all three targets: Kinesis Data Firehose, Lambda, and an EventBridge API destination configured to write to S3.
**C)** Create one EventBridge rule matching GuardDuty findings with Kinesis Data Firehose and Lambda as targets, and enable EventBridge archive to write to S3.
**D)** Create one EventBridge rule matching GuardDuty findings with Lambda as the only target, then have Lambda fan out to the other systems.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** A single EventBridge rule can target up to 5 different services. For this scenario, configure one rule with three targets: Kinesis Data Firehose (to SIEM), Lambda (auto-remediation), and an EventBridge API destination or use EventBridge's ability to target S3 via a Firehose delivery stream configured for S3. Actually, EventBridge doesn't directly target S3, so you'd use a second Kinesis Data Firehose stream to S3, or use Lambda to write to S3, or put events in CloudWatch Logs with S3 export. The key principle is using one rule with multiple targets. Option A is less efficient (three rules vs. one). Option C doesn't send events to S3 for audit logging as required. Option D pushes fan-out logic to Lambda unnecessarily.

**Correction for Best Practice:** Given that EventBridge doesn't natively target S3, the most practical answer is B if we assume the S3 write is done via Kinesis Firehose or another supported target type. If that's not possible, then A (three rules) or D (Lambda fan-out) would be acceptable, with A being more maintainable.

**Key Concept:** Use a single EventBridge rule with multiple targets when processing the same event across multiple systems. This reduces rule count and simplifies management.

</details>

---

### Question 12

A SysOps administrator is designing an event-driven architecture for processing order events. The system should guarantee exactly-once processing and maintain strict ordering of events per customer. Which combination of AWS services is MOST appropriate?

**A)** EventBridge with Lambda targets using reserved concurrency set to 1.
**B)** Amazon SNS FIFO topic with SQS FIFO queue subscribers, bypassing EventBridge.
**C)** EventBridge with SQS standard queue target, processed by Lambda.
**D)** EventBridge with Kinesis Data Streams target using partition keys, processed by Lambda.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** This question tests understanding of when NOT to use EventBridge. EventBridge does not guarantee exactly-once delivery or strict message ordering. For exactly-once processing with ordering guarantees, use SNS FIFO topics and SQS FIFO queues. The FIFO queue provides exactly-once processing (using deduplication) and maintains message order within a message group (using message group IDs based on customer ID). Option A doesn't guarantee exactly-once delivery. Option C uses a standard queue, which doesn't guarantee ordering. Option D could maintain ordering per partition, but EventBridge's at-least-once delivery semantics mean you might get duplicates.

**Key Concept:** EventBridge is at-least-once delivery and does not guarantee ordering. Use SNS FIFO/SQS FIFO for exactly-once and ordered processing.

</details>

---

### Question 13

A company has enabled EventBridge schema registry with schema discovery on their default event bus. A developer wants to use a discovered schema to generate code bindings for a Lambda function. The schema is not appearing in the registry. What is the MOST likely cause?

**A)** Schema discovery only works on custom event buses, not the default event bus.
**B)** Schema discovery requires at least 5 events matching the same structure to appear in the registry.
**C)** The event source is a custom application, and schema discovery only works for AWS service events.
**D)** Schema discovery has a delay and may take several hours to discover and register schemas after events are published.

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** EventBridge schema discovery automatically detects and registers schemas for events from AWS services (like EC2, S3, etc.) and SaaS partner integrations. Custom application events (events you publish via PutEvents) are not automatically discovered. For custom events, you must manually create and upload the schema to the registry. Option A is incorrect; schema discovery works on default and custom event buses. Option B is incorrect; there's no minimum event threshold. Option D is misleading; while there can be some delay, it's typically minutes, not hours, and the primary issue here is that custom events aren't auto-discovered.

**Key Concept:** Schema discovery is automatic for AWS service and partner events, but manual for custom application events.

</details>

---

### Question 14

A SysOps administrator needs to create an EventBridge rule that triggers when an EC2 instance is terminated in ANY region within the account. What is the correct configuration?

**A)** Create a rule in each region with the same event pattern and targets, as EventBridge rules are region-specific.
**B)** Create a rule in the us-east-1 region with a global event pattern that specifies "region": ["*"].
**C)** Create a cross-region event bus in one region and configure it to receive events from all other regions.
**D)** Use AWS CloudTrail Lake to query cross-region EC2 termination events and publish to EventBridge.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** EventBridge is a regional service. Events are published to the event bus in the region where the AWS service action occurred. To capture EC2 terminations from all regions, you must create an EventBridge rule in each region. Option B is incorrect; there's no global event bus or wildcard region matching. Option C is close to a pattern you might implement (cross-region event forwarding), but EventBridge doesn't have a "cross-region event bus" feature out-of-the-box; you'd need to create rules in each region that target a central region's event bus. Option D is overly complex.

**Best Practice:** For true multi-region event capture, create rules in each region that forward events to a central event bus in one region for unified processing.

**Key Concept:** EventBridge is regional. Multi-region monitoring requires rules in each region or cross-region forwarding.

</details>

---

### Question 15

A company wants to implement an auto-remediation workflow where non-compliant S3 buckets (missing encryption) trigger an EventBridge rule that starts an SSM Automation document. The automation should pass the bucket name to the document. How should the input to SSM Automation be configured?

**A)** Use the EventBridge input transformer to extract the bucket name from the event and construct the SSM input parameters.
**B)** Configure SSM Automation to automatically parse the full EventBridge event and extract the bucket name.
**C)** Use EventBridge Pipes to transform the event before sending to SSM Automation.
**D)** Store the bucket name in a DynamoDB table and have the SSM Automation document query the table.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** When targeting SSM Automation from an EventBridge rule, you need to provide the automation document's input parameters. Use the input transformer to extract values from the event (e.g., bucket name from `$.detail.requestParameters.bucketName` if triggered by a CloudTrail event) and construct the JSON structure expected by the automation document's parameters. Option B is incorrect; SSM Automation doesn't automatically parse EventBridge events. Option C could work but is more complex than needed for this use case. Option D is inefficient and adds unnecessary state management.

**Key Concept:** Input transformers are essential when EventBridge events don't match the input format expected by targets like SSM Automation.

</details>

---

### Question 16

A SysOps administrator needs to set up an EventBridge rule that triggers a Step Functions state machine only during business hours (9 AM to 5 PM EST, Monday-Friday). What is the BEST approach?

**A)** Create a schedule rule using cron: `cron(0 9 ? * MON-FRI *)` that enables the event rule, and another cron that disables it at 5 PM.
**B)** Use a Lambda function in the event rule's target that checks the current time and conditionally starts the Step Functions execution.
**C)** Create a schedule rule using cron: `cron(0 9-17 ? * MON-FRI *)` that triggers the Step Functions state machine.
**D)** Use EventBridge Pipes with filtering to check the current time before invoking Step Functions.

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** EventBridge schedule rules trigger at specific times but don't inherently limit event-based rules to time windows. If the requirement is to respond to events (not scheduled triggers) only during business hours, the logic must be in the target. Use a Lambda function that checks the current time and conditionally starts Step Functions. Option A is complex and involves enabling/disabling rules, which is not an intended use pattern. Option C creates scheduled executions at each hour from 9 AM to 5 PM, which is different from responding to events during those hours. Option D overcomplicates; Pipes are for stream/queue processing with transformation.

**Note:** If the question meant "schedule Step Functions execution during business hours," then Option C would be close but would execute every hour. For daily execution at 9 AM, use `cron(0 9 ? * MON-FRI *)`.

**Key Concept:** EventBridge rules don't have time-based conditions for event matching. Time-based logic must be implemented in the target (Lambda) or use schedule rules for time-triggered actions.

</details>

---

### Question 17

An application publishes custom events to a custom EventBridge event bus. The SysOps administrator needs to ensure that all events are retained for 30 days for compliance purposes, but processing rules may change over time. What should be configured?

**A)** Create an EventBridge archive with a 30-day retention period and an event pattern matching all events on the custom bus.
**B)** Create an EventBridge rule that sends all events to an S3 bucket with a 30-day lifecycle policy.
**C)** Enable CloudTrail logging for EventBridge API calls to capture all events for 30 days.
**D)** Create an EventBridge rule that sends all events to CloudWatch Logs with a 30-day retention period.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** EventBridge archives are purpose-built for retaining events with configurable retention periods (1 to indefinite). An archive captures events matching its pattern (use `{}` or omit pattern to match all events) and stores them for the specified retention. This allows future replay if processing rules change. Option B works but is less integrated; you'd need a rule and S3 configuration. Option C is incorrect; CloudTrail logs API calls (like PutEvents) but not the event payloads themselves. Option D works for retention but CloudWatch Logs is not designed for event replay like EventBridge archives.

**Key Concept:** EventBridge archives are the native solution for event retention and replay. They're designed for event sourcing patterns.

</details>

---

### Question 18

A company uses EventBridge Pipes to connect an SQS queue to an EventBridge event bus with enrichment and filtering. The pipe is processing messages, but some messages are not appearing on the target event bus. What is the MOST likely cause?

**A)** The pipe's filter criteria are rejecting messages that don't match the filter pattern.
**B)** The SQS queue has a message retention period shorter than the pipe's processing time.
**C)** EventBridge Pipes do not support SQS as a source; only Kinesis and DynamoDB Streams are supported.
**D)** The target event bus has a resource policy that rejects events from the pipe.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** EventBridge Pipes support optional filtering. If configured, only messages matching the filter criteria are sent to the enrichment step and target. Messages that don't match are filtered out. This is a common reason for missing messages. Option B is unlikely unless there's significant backlog and processing delay. Option C is incorrect; EventBridge Pipes support SQS, Kinesis Data Streams, DynamoDB Streams, Amazon MQ, and Managed Streaming for Apache Kafka as sources. Option D is less likely; if the pipe is processing some messages successfully, the policy is probably correct.

**Key Concept:** EventBridge Pipes provide filtering, enrichment (via Lambda or API destinations), and transformation between sources and targets. Check filter criteria if messages are missing.

</details>

---

### Question 19

A SysOps administrator needs to monitor the number of failed invocations for an EventBridge rule's Lambda target. Which CloudWatch metric should be used?

**A)** EventBridge metric: `FailedInvocations` with dimensions for the rule name and target.
**B)** Lambda metric: `Errors` for the specific Lambda function.
**C)** EventBridge metric: `TriggeredRules` minus `Invocations` to calculate failures.
**D)** CloudWatch Logs Insights query on the EventBridge log group to count failed deliveries.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** EventBridge publishes `FailedInvocations` metric to CloudWatch when it cannot successfully deliver an event to a target. This metric has dimensions including `RuleName`, so you can track failures per rule. Option B tracks Lambda function errors during execution, which is different from EventBridge failing to invoke Lambda. Option C is incorrect; those metrics don't directly calculate delivery failures. Option D is not a standard approach; EventBridge doesn't have a default log group for delivery tracking (though you can enable CloudTrail for API calls).

**Key Concept:** Monitor EventBridge rule health using the `FailedInvocations` metric. This tracks delivery failures, not target execution failures.

</details>

---

### Question 20

A company needs to process events from their custom EventBridge event bus in a specific order based on a customer ID, with guaranteed exactly-once processing. The current architecture uses EventBridge rules triggering Lambda directly, but duplicate processing is occurring. What architectural change should the SysOps administrator make?

**A)** Add an SQS FIFO queue as a target for the EventBridge rule, using customer ID as the message group ID, and process messages from the queue with Lambda.
**B)** Configure Lambda reserved concurrency to 1 to ensure serial processing and prevent duplicates.
**C)** Use EventBridge Pipes with a DynamoDB table for deduplication tracking before invoking Lambda.
**D)** Enable EventBridge archive with replay to reprocess only unique events.

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** EventBridge provides at-least-once delivery, which can result in duplicates. To achieve exactly-once processing with ordering, insert an SQS FIFO queue between EventBridge and Lambda. Configure the EventBridge rule to target the FIFO queue, using an input transformer to set the message group ID based on customer ID (to maintain ordering per customer). Enable content-based deduplication or use message deduplication ID to prevent duplicates. Lambda then processes messages from the FIFO queue. Option B doesn't prevent duplicates, only serializes processing. Option C is complex and not a standard pattern. Option D doesn't solve the duplicate processing issue.

**Key Concept:** For exactly-once processing and ordering with EventBridge, use an SQS FIFO queue as an intermediary. EventBridge alone does not guarantee exactly-once delivery.

</details>

---

## Summary

These 20 questions cover the critical EventBridge concepts for the SOA-C03 exam:
- Event bus types and cross-account delivery
- Event patterns and filtering limitations
- Schedule expressions (cron vs rate)
- Multiple targets and input transformers
- Dead-letter queues for reliability
- Archive and replay for event sourcing
- EventBridge Pipes for stream processing
- Schema registry for development
- When to use EventBridge vs SNS/SQS
- Integration with AWS services (Health, GuardDuty, Config, Systems Manager)
- Common pitfalls and limitations

**Study Tips:**
1. Understand EventBridge event structure and what data is actually present in events
2. Know the difference between EventBridge delivery guarantees (at-least-once) vs SQS FIFO (exactly-once)
3. Practice writing event patterns and understanding the JSON matching syntax
4. Memorize cron expression syntax for schedule rules
5. Understand cross-account event delivery setup (requires resource policy on target bus)
6. Know when EventBridge is the right choice vs SNS, SQS, or direct integrations
