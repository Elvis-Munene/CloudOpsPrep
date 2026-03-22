# Amazon SNS - Deep Dive Questions (SOA-C03)

### Question 1
A company uses an SNS topic to distribute order notifications to multiple downstream systems. They recently experienced an issue where some orders were processed twice by their payment processing SQS queue. The company needs to ensure that each message is processed only once and in the exact order it was published. What changes should be made?

A) Enable content-based deduplication on the existing SNS Standard topic and configure the SQS queue to use FIFO mode
B) Convert the SNS Standard topic to an SNS FIFO topic with a .fifo suffix, subscribe an SQS FIFO queue with matching message group IDs
C) Enable server-side encryption on both SNS and SQS, and implement application-level deduplication logic
D) Configure the SNS topic delivery policy to include a deduplication ID and enable long polling on the SQS queue

<details>
<summary>Answer</summary>

**Correct Answer: B**

**Explanation:**
- SNS FIFO topics are specifically designed for strict message ordering and exactly-once delivery. They must end with the `.fifo` suffix
- SNS FIFO topics can only subscribe to SQS FIFO queues, not Standard queues
- Message group IDs must be used to maintain ordering within groups
- Content-based deduplication or explicit deduplication IDs prevent duplicate messages within the 5-minute deduplication interval

**Why other options are incorrect:**
- **A**: You cannot enable FIFO features on a Standard SNS topic. Standard topics don't support ordering or exactly-once delivery guarantees
- **C**: Encryption doesn't address ordering or deduplication requirements. While useful for security, it doesn't prevent duplicate processing
- **D**: SNS delivery policies control retry behavior, not deduplication. This approach doesn't solve the ordering or deduplication problem

**Key Concepts:**
- SNS FIFO topics support up to 300 messages per second (or 3,000 with batching)
- FIFO subscriptions are limited to SQS FIFO queues only
- Standard SNS topics provide best-effort ordering with at-least-once delivery
</details>

---

### Question 2
A CloudOps engineer needs to implement a fan-out pattern where messages from a single source are delivered to three different systems: an SQS queue for async processing, a Lambda function for real-time analytics, and an HTTP endpoint for third-party integration. Occasionally, the HTTP endpoint fails, and those messages must be captured for manual review. How should this be architected?

A) Create separate SNS topics for each subscriber type and use EventBridge to route messages based on content
B) Create a single SNS Standard topic with three subscriptions (SQS, Lambda, HTTP/S), and configure a dead-letter queue (DLQ) for the HTTP subscription
C) Use an SQS queue with three Lambda functions polling it, each forwarding to the respective destination
D) Create an SNS FIFO topic with three subscriptions and enable server-side encryption for failed message recovery

<details>
<summary>Answer</summary>

**Correct Answer: B**

**Explanation:**
- The SNS fan-out pattern is ideal for broadcasting a single message to multiple subscribers simultaneously
- Each subscription type (SQS, Lambda, HTTP/S) can be added to the same SNS topic
- SNS subscriptions support individual DLQs (Dead-Letter Queues) which capture messages that fail delivery after retry attempts
- For the HTTP subscription specifically, you can configure a DLQ (must be an SQS queue) to capture failed deliveries

**Why other options are incorrect:**
- **A**: Multiple SNS topics add unnecessary complexity. EventBridge is not needed for simple fan-out patterns; SNS handles this natively
- **C**: This defeats the purpose of fan-out. Polling an SQS queue with Lambda and then forwarding creates a complex, tightly-coupled architecture
- **D**: SNS FIFO topics only support SQS FIFO queue subscriptions, not Lambda or HTTP/S endpoints

**Key Concepts:**
- SNS Standard topics support: SQS (Standard/FIFO), Lambda, HTTP/S, Email, SMS, Mobile Push, Kinesis Data Firehose
- DLQs for SNS subscriptions must be SQS queues (Standard or FIFO)
- SNS automatically retries failed deliveries with exponential backoff
- Each subscription can have its own filter policy and DLQ configuration
</details>

---

### Question 3
A financial services application publishes transaction events to an SNS topic with 12 different microservices subscribed. However, each microservice should only receive events relevant to their function (e.g., fraud detection only needs high-value transactions). The solution must minimize code changes in the microservices. What is the MOST efficient approach?

A) Implement application-level filtering in each Lambda function to discard irrelevant messages
B) Create 12 separate SNS topics, one for each microservice type, and modify the publisher to send to multiple topics
C) Configure SNS subscription filter policies using message attributes to control which messages each subscriber receives
D) Use SQS message attributes with long polling to filter messages before Lambda processing

<details>
<summary>Answer</summary>

**Correct Answer: C**

**Explanation:**
- SNS subscription filter policies allow filtering at the SNS level based on message attributes
- Filters are evaluated before delivery, so subscribers never receive irrelevant messages (no wasted invocations)
- Filter policies use JSON to match message attributes (string, number, arrays, prefixes, etc.)
- No code changes needed in subscribers - filtering happens at the infrastructure level
- Publishers set message attributes like `{"transaction_type": "wire", "amount": 50000}` and each subscription filters based on its needs

**Why other options are incorrect:**
- **A**: This is inefficient - Lambda functions are invoked and charged for processing messages they'll discard. It also requires code changes
- **B**: Creates operational overhead with 12 topics. The publisher logic becomes complex and error-prone
- **D**: SQS doesn't natively filter messages before delivery to Lambda. This adds unnecessary SQS queues and complexity

**Key Concepts:**
- Filter policies can match on: exact match, anything-but, prefix, numeric ranges, exists checks, arrays (OR logic)
- Multiple conditions in a filter use AND logic
- Messages without matching attributes are not delivered to that subscription
- Filter policies can have up to 5 attribute names and 150 conditions total
</details>

---

### Question 4
A company's disaster recovery plan requires that SNS messages published in us-east-1 must be delivered to an SQS queue in an eu-west-1 AWS account owned by their European subsidiary. The subsidiary's security team requires that only authorized SNS topics can send messages to their queue. How should this be configured?

A) Configure SNS topic access policy in us-east-1 to allow cross-region delivery, and set SQS queue policy in eu-west-1 to allow sns.amazonaws.com principal
B) Create an SNS subscription in us-east-1 pointing to the eu-west-1 SQS queue ARN, and configure the SQS queue policy to allow the specific SNS topic ARN as a principal
C) Use SNS message delivery to an HTTP endpoint in eu-west-1 that forwards messages to the local SQS queue
D) Enable SNS global topics and configure cross-region replication with AWS PrivateLink

<details>
<summary>Answer</summary>

**Correct Answer: B**

**Explanation:**
- SNS supports cross-region and cross-account subscriptions to SQS queues
- The SQS queue policy must explicitly grant `SendMessage` permission to the SNS topic ARN
- The subscription is created in the SNS topic (us-east-1) with the SQS queue ARN (eu-west-1) as the endpoint
- Best practice: specify the exact SNS topic ARN in the SQS queue policy rather than the generic sns.amazonaws.com service principal

**Example SQS Queue Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "*"},
    "Action": "SQS:SendMessage",
    "Resource": "arn:aws:sqs:eu-west-1:ACCOUNT-B:queue-name",
    "Condition": {
      "ArnEquals": {
        "aws:SourceArn": "arn:aws:sns:us-east-1:ACCOUNT-A:topic-name"
      }
    }
  }]
}
```

**Why other options are incorrect:**
- **A**: Using the generic sns.amazonaws.com principal is overly permissive and doesn't meet the security requirement for "authorized topics only"
- **C**: Adding an HTTP endpoint is unnecessary complexity and introduces a potential failure point
- **D**: SNS doesn't have "global topics" or cross-region replication features. PrivateLink is not applicable here

**Key Concepts:**
- Cross-account subscriptions require proper IAM permissions on both sides
- SQS queue policies control who can send messages to the queue
- SNS automatically handles cross-region message delivery
- Data transfer charges apply for cross-region SNS to SQS delivery
</details>

---

### Question 5
An e-commerce platform publishes order events to SNS, which fans out to multiple SQS queues. During a recent traffic spike, they noticed that some messages were delivered to SQS queues but subsequently disappeared before processing. CloudWatch metrics show no processing errors in Lambda functions. What is the MOST likely cause and solution?

A) Messages exceeded the SQS visibility timeout; increase the visibility timeout to match Lambda execution time
B) Messages exceeded the SQS retention period; increase retention period from default 4 days to 14 days
C) SNS message delivery failed and messages were sent to the DLQ; configure proper DLQ monitoring
D) Messages exceeded the maximum message size of 256 KB; use S3 for large payloads with SNS Extended Client Library

<details>
<summary>Answer</summary>

**Correct Answer: A**

**Explanation:**
- The visibility timeout determines how long a message is hidden from other consumers after being received
- If Lambda processing time exceeds the visibility timeout, the message becomes visible again and may be processed by another Lambda instance
- If the original Lambda completes and tries to delete an already-reprocessed message, it may appear that messages "disappeared"
- During traffic spikes, Lambda cold starts can increase processing time, exacerbating this issue

**Solution:**
- Set SQS visibility timeout to at least 6 times the Lambda function timeout (recommended best practice)
- For a Lambda with 30-second timeout, set visibility timeout to 180 seconds (3 minutes) or more
- Configure Lambda reserved concurrency to prevent excessive parallel executions during spikes

**Why other options are incorrect:**
- **B**: The default retention period is 4 days, but messages disappeared quickly (during processing), not after days. This isn't a retention issue
- **C**: CloudWatch shows no errors, suggesting successful delivery. DLQ would only capture messages that failed SNS-to-SQS delivery after retries
- **D**: The scenario doesn't mention message size issues, and messages were delivered successfully (just disappeared during processing)

**Key Concepts:**
- Default visibility timeout: 30 seconds (configurable from 0 seconds to 12 hours)
- SQS automatically deletes messages only after explicit DeleteMessage API call
- Lambda polls SQS and automatically handles visibility timeout and deletion for successful executions
- Use CloudWatch metric `ApproximateAgeOfOldestMessage` to detect visibility timeout issues
</details>

---

### Question 6
A CloudOps team needs to send critical system alerts via SNS to multiple channels: email for on-call engineers, SMS for executives, and Slack via Lambda. They require that all messages are encrypted at rest and in transit. The SNS topic receives messages containing sensitive customer data. What is the correct encryption configuration?

A) Enable SNS server-side encryption (SSE) using AWS KMS Customer Managed Key (CMK), ensure subscribers have kms:Decrypt permissions, enable HTTPS-only delivery for HTTP/S endpoints
B) Enable SNS SSE with AWS managed key (aws/sns), configure TLS 1.2 for all subscriptions, and encrypt message content at application level before publishing
C) Use AWS Certificate Manager (ACM) to encrypt SNS messages, enable encryption in transit for all protocols including SMS and Email
D) Configure SNS topic policy to enforce encryption in transit, use AWS managed KMS key for at-rest encryption, disable unencrypted subscriptions

<details>
<summary>Answer</summary>

**Correct Answer: A**

**Explanation:**
- SNS Server-Side Encryption (SSE) encrypts messages at rest using AWS KMS
- Customer Managed Keys (CMK) provide better control, auditing, and rotation policies compared to AWS managed keys
- Subscribers (Lambda, SQS, HTTP/S) need `kms:Decrypt` and `kms:GenerateDataKey` permissions in the KMS key policy
- HTTPS-only delivery can be enforced for HTTP/S endpoints via topic policy with `aws:SecureTransport` condition

**KMS Key Policy Example:**
```json
{
  "Effect": "Allow",
  "Principal": {
    "Service": "lambda.amazonaws.com"
  },
  "Action": [
    "kms:Decrypt",
    "kms:GenerateDataKey"
  ],
  "Resource": "*"
}
```

**Why other options are incorrect:**
- **B**: AWS managed key (aws/sns) provides less control and auditability. Application-level encryption is redundant if SSE is properly configured
- **C**: ACM is for TLS/SSL certificates, not message encryption. SMS and Email cannot be encrypted end-to-end by AWS (they leave AWS infrastructure)
- **D**: This is partially correct but less specific than A. "Disabling unencrypted subscriptions" is not a standard SNS feature

**Key Concepts:**
- SNS SSE encrypts message body and message attributes
- SMS and Email are encrypted in transit to AWS boundaries but not beyond
- Mobile push notifications use provider-specific encryption (APNS, FCM, etc.)
- Lambda and SQS subscriptions automatically handle KMS decryption if permissions are correct
</details>

---

### Question 7
A SaaS company publishes product updates to an SNS topic subscribed by thousands of customer accounts via SQS queues in their own AWS accounts. Customers report intermittent message delivery failures. CloudWatch Logs show "Access Denied" errors for some customer queues. What is the most scalable solution?

A) Share a pre-configured SQS queue policy template with customers that grants sns:SendMessage permission to the SNS topic ARN
B) Create IAM roles in each customer account and assume roles from the SNS topic account to deliver messages
C) Switch to SNS HTTP/S subscriptions pointing to customer-owned API endpoints instead of direct SQS delivery
D) Use AWS Organizations to create SCPs that automatically grant SNS permissions to all member account SQS queues

<details>
<summary>Answer</summary>

**Correct Answer: A**

**Explanation:**
- For cross-account SNS to SQS subscriptions, the SQS queue policy in the customer account must allow the SNS topic to send messages
- Providing a template SQS queue policy is the standard, scalable approach for SaaS providers
- Customers apply the policy to their queues, granting specific permissions to the SNS topic ARN
- This follows the principle of explicit permission grants for cross-account access

**Template Policy Example:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "*"},
    "Action": "SQS:SendMessage",
    "Resource": "arn:aws:sqs:REGION:CUSTOMER-ACCOUNT:customer-queue-name",
    "Condition": {
      "ArnEquals": {
        "aws:SourceArn": "arn:aws:sns:REGION:SAAS-ACCOUNT:product-updates"
      }
    }
  }]
}
```

**Why other options are incorrect:**
- **B**: SNS cannot assume IAM roles for message delivery. Cross-account delivery requires resource-based policies (SQS queue policy), not role assumption
- **C**: HTTP/S endpoints add operational complexity (customers need to maintain endpoints, handle retries, etc.). Direct SQS is more reliable
- **D**: SCPs are permission boundaries, not permission grants. They restrict, not allow. Also, not all customers may be in the same AWS Organization

**Key Concepts:**
- Cross-account SNS requires resource-based policies on the target resource
- Use condition keys like `aws:SourceArn` to prevent unauthorized topics from sending messages
- SNS subscription confirmation is required even with proper policies
- Monitor failed deliveries with CloudWatch metric `NumberOfNotificationsFailed`
</details>

---

### Question 8
A development team wants to test SNS message processing locally before deploying to production. They need to receive SNS notifications during development without exposing their local development servers to the internet. Which subscription protocol should they use?

A) HTTP endpoint with ngrok tunnel to expose localhost to the internet
B) Email or Email-JSON protocol to receive messages in their development inbox
C) SQS queue subscription and poll the queue from their local development environment
D) SMS protocol to receive messages on their mobile devices during development

<details>
<summary>Answer</summary>

**Correct Answer: C**

**Explanation:**
- SQS queues can be subscribed to SNS topics and polled from anywhere with proper credentials
- Developers can use AWS CLI, SDKs, or tools like LocalStack to poll SQS from their local machines
- This approach doesn't require exposing local servers to the internet
- SQS queues persist messages (default 4 days) so developers won't miss messages during breaks

**Development Workflow:**
1. Create SNS topic and SQS queue in development AWS account
2. Subscribe SQS queue to SNS topic
3. Use `aws sqs receive-message` or SDK to poll from local environment
4. Process messages with local code
5. Delete messages after successful processing

**Why other options are incorrect:**
- **A**: While ngrok works, it exposes local services to the internet (security risk) and requires running additional tools. Not ideal for development
- **B**: Email protocols are for human consumption, not programmatic processing. Parsing emails is cumbersome and not reliable for testing
- **D**: SMS is expensive, has rate limits, and is designed for human alerts, not automated testing

**Key Concepts:**
- SQS is the bridge between cloud services and local development
- Consider using LocalStack for completely offline SNS/SQS development
- For CI/CD testing, use dedicated test topics/queues separate from production
- SNS supports subscription confirmation tokens - handle these in tests
</details>

---

### Question 9
A media streaming company uses SNS to notify multiple downstream systems when new content is uploaded. The message payload includes metadata (title, category, duration) and a large thumbnail image (500 KB). Messages are failing with "Payload too large" errors. What is the BEST solution that maintains the fan-out pattern?

A) Compress the thumbnail image before including it in the SNS message payload to fit within the 256 KB limit
B) Upload the thumbnail to S3, include only the S3 object URL in the SNS message, and have subscribers retrieve the image from S3
C) Split the message into multiple smaller SNS messages and send them sequentially
D) Use SNS Extended Client Library to automatically store large payloads in S3 (similar to SQS Extended Client)

<details>
<summary>Answer</summary>

**Correct Answer: B**

**Explanation:**
- SNS has a maximum message size of 256 KB (262,144 bytes)
- Best practice for large payloads: store data in S3 and send S3 reference in SNS message
- This pattern keeps messages small, fast, and cost-effective
- Subscribers can retrieve the full data from S3 using the object URL/key
- Provides additional benefits: object versioning, lifecycle policies, access control

**Implementation:**
1. Upload thumbnail to S3: `s3://media-bucket/thumbnails/video-123.jpg`
2. Publish SNS message: `{"title": "Video Title", "s3Key": "thumbnails/video-123.jpg"}`
3. Subscribers receive message and fetch from S3 using SDK
4. Optional: use S3 presigned URLs for temporary access

**Why other options are incorrect:**
- **A**: Even compressed, 500 KB is unlikely to fit in 256 KB. Compression adds complexity and may still fail
- **C**: Multiple messages break atomicity and require complex reassembly logic. Ordering is not guaranteed in SNS Standard topics
- **D**: SNS does NOT have an Extended Client Library. This exists only for SQS Extended Client Library (common confusion point)

**Key Concepts:**
- SNS message size limit: 256 KB (includes message body, attributes, and protocol overhead)
- SQS Extended Client Library allows messages up to 2 GB via S3
- For SNS, always use the S3 reference pattern for large data
- Consider S3 Event Notifications as an alternative if the workflow starts with S3 uploads
</details>

---

### Question 10
An IoT platform publishes sensor data to an SNS topic with multiple Lambda function subscribers performing different analytics. During peak hours (1000 messages/second), some Lambda functions are throttled and messages are lost. The architecture must handle bursts and ensure no message loss. What is the MOST cost-effective solution?

A) Increase Lambda reserved concurrency for all subscriber functions to handle peak load
B) Place an SQS queue between SNS and each Lambda function, configure Lambda to poll SQS with appropriate batch size and reserved concurrency
C) Convert to SNS FIFO topic to throttle incoming message rate and prevent Lambda throttling
D) Implement API Gateway in front of Lambda functions to buffer requests and rate limit

<details>
<summary>Answer</summary>

**Correct Answer: B**

**Explanation:**
- The SNS + SQS + Lambda pattern is a best practice for high-volume, bursty workloads
- SQS acts as a buffer, absorbing traffic spikes and preventing Lambda throttling
- Lambda polls SQS at a controlled rate based on configured reserved concurrency
- If Lambda is throttled, messages remain in SQS and are retried automatically
- This pattern prevents message loss and provides natural backpressure

**Architecture Flow:**
```
SNS Topic (fan-out)
  ├─> SQS Queue 1 -> Lambda Function 1 (analytics type 1)
  ├─> SQS Queue 2 -> Lambda Function 2 (analytics type 2)
  └─> SQS Queue 3 -> Lambda Function 3 (analytics type 3)
```

**Configuration:**
- Set Lambda batch size (1-10 messages) to optimize processing
- Configure reserved concurrency to control throughput
- Set SQS visibility timeout to 6x Lambda timeout
- Use DLQs on SQS queues for failed processing

**Why other options are incorrect:**
- **A**: Increasing reserved concurrency helps but doesn't prevent throttling at extreme bursts. Also increases cost without buffering
- **C**: SNS FIFO topics have much lower throughput (300 msg/s, or 3000 with batching) and only support SQS FIFO subscriptions. This reduces capacity
- **D**: API Gateway is for HTTP APIs, not SNS subscriptions. This fundamentally changes the architecture unnecessarily

**Key Concepts:**
- SNS + SQS + Lambda is the recommended pattern for reliable, scalable event processing
- Lambda scales automatically but has account/regional concurrency limits
- SQS provides buffering, retries, and DLQ for resilience
- SNS Standard topics can handle thousands of messages per second
</details>

---

### Question 11
A financial application requires audit logging of all SNS message publications, including who published what message and when. The solution must be tamper-proof and allow historical querying for compliance. How should this be implemented?

A) Enable SNS delivery status logging to CloudWatch Logs for all subscription protocols
B) Configure SNS to publish message metadata to Amazon CloudTrail, and enable CloudTrail log file integrity validation
C) Enable AWS Config to track SNS configuration changes and message publications
D) Use SNS message attributes to include audit metadata and store messages in DynamoDB via Lambda subscriber

<details>
<summary>Answer</summary>

**Correct Answer: B**

**Explanation:**
- CloudTrail logs SNS API calls including `Publish` actions with details: who (identity), when (timestamp), what (topic ARN, message ID)
- CloudTrail log file integrity validation uses digital signatures to make logs tamper-evident
- CloudTrail logs can be stored in S3 with object lock for immutability
- CloudTrail Insights and Athena enable querying historical API activity

**CloudTrail captures:**
- Principal (IAM user/role) who published
- Source IP address
- Timestamp
- Topic ARN
- Message ID (not message content for privacy)
- Request parameters

**Why other options are incorrect:**
- **A**: Delivery status logging tracks message delivery to subscribers (success/failure), not who published the message. It doesn't provide tamper-proof audit trails
- **C**: AWS Config tracks configuration changes (topic creation, policy updates) but doesn't log individual message publications
- **D**: This requires custom implementation, is not tamper-proof, and adds operational complexity. Message attributes don't capture the publisher's identity automatically

**Key Concepts:**
- CloudTrail is the standard for API audit logging in AWS
- Enable log file integrity validation for compliance requirements
- SNS doesn't store message content; only delivery metadata is logged
- For message content auditing, implement application-level logging to immutable storage
</details>

---

### Question 12
A company uses SNS to send order confirmations via email to customers. Customers in certain countries report not receiving emails while others receive them successfully. The SNS delivery status shows successful delivery to Amazon SES. What is the MOST likely cause?

A) SNS topic is not configured for international email delivery
B) Email protocol subscription requires confirmation from each recipient, which may be blocked by spam filters or not completed
C) SES is in sandbox mode and can only send to verified email addresses
D) SNS message attributes don't include proper locale settings for international delivery

<details>
<summary>Answer</summary>

**Correct Answer: C**

**Explanation:**
- Amazon SES starts in sandbox mode with restrictions: can only send to verified email addresses and domains
- Production use requires requesting SES to move out of sandbox mode via AWS Support
- SNS uses SES for email delivery (Email and Email-JSON protocols)
- If SES is in sandbox mode, emails to unverified addresses fail at the SES level, but SNS may report "successful handoff"

**SES Sandbox Limitations:**
- Maximum 200 emails per 24-hour period
- Maximum 1 email per second
- Can only send TO verified addresses
- Can only send FROM verified addresses/domains

**Solution:**
1. Request production access for SES in the AWS Console
2. Verify domain with DNS records (better than individual addresses)
3. Implement bounce and complaint handling
4. Monitor sender reputation

**Why other options are incorrect:**
- **A**: SNS doesn't have country-specific email delivery restrictions. Email delivery is handled by SES, which is global
- **B**: Subscription confirmation is for subscribing endpoints to SNS topics, not for individual email recipients. Once subscribed, all emails go to that endpoint
- **D**: Locale settings don't affect email deliverability. This is not an SNS or SES concept

**Key Concepts:**
- SNS Email protocol uses SES backend
- Check SES sending limits and sandbox status in SES console
- For production, move out of sandbox and verify domains
- Monitor SES bounces and complaints to maintain reputation
</details>

---

### Question 13
A microservices architecture uses SNS for event-driven communication. Service A publishes events, and Services B, C, and D subscribe. Service D requires additional context data that's not needed by Services B and C. Adding this data to all messages would increase costs and payload size. What is the BEST approach?

A) Publish two separate messages to different SNS topics: one for B/C and one for D
B) Use SNS message attributes to include the additional data, and configure Service D to read attributes while B/C ignore them
C) Publish the base message to SNS, and have Service D make a synchronous API call to Service A for additional context
D) Use SNS message filtering to send different message content to different subscribers

<details>
<summary>Answer</summary>

**Correct Answer: B**

**Explanation:**
- SNS message attributes are separate from the message body and can be used for metadata or additional context
- Message attributes support types: String, Number, Binary, String.Array
- Each message can have up to 10 message attributes
- Subscribers can choose to read attributes or ignore them - no impact on those who don't need the data
- Message attributes are included in the message size limit (256 KB total)

**Example:**
```json
Message Body: {"eventType": "ORDER_PLACED", "orderId": "12345"}
Message Attributes: {
  "customerSegment": "premium",
  "additionalContext": "complex data for Service D"
}
```

**Service D Lambda reads attributes:**
```python
message_attributes = event['Records'][0]['Sns']['MessageAttributes']
additional_context = message_attributes.get('additionalContext', {}).get('Value')
```

**Why other options are incorrect:**
- **A**: Maintaining two topics adds operational complexity. Service A must publish to both topics, and if requirements change, more topics are needed
- **C**: Synchronous calls to Service A create tight coupling and defeat the purpose of event-driven architecture. Also adds latency
- **D**: SNS message filtering controls which subscribers receive messages based on attributes, not different message content to different subscribers

**Key Concepts:**
- Message attributes are perfect for optional/contextual data
- Filter policies can use message attributes for subscription filtering
- Message attributes are available in all subscription protocols
- Consider cost: attributes count toward message size and API calls
</details>

---

### Question 14
A CloudOps engineer is troubleshooting an issue where Lambda functions subscribed to an SNS topic are not being invoked. The SNS topic shows successful publishes in CloudWatch metrics, and the Lambda function executes successfully when tested manually. What are the possible causes? (Choose TWO)

A) The Lambda function's resource policy doesn't grant SNS topic permission to invoke the function
B) The SNS topic's access policy doesn't allow publishing messages to Lambda functions
C) The Lambda function's execution role doesn't have permissions to receive SNS notifications
D) The SNS subscription is in "Pending Confirmation" status and hasn't been confirmed
E) The SNS topic is encrypted with KMS, but Lambda doesn't have decrypt permissions

<details>
<summary>Answer</summary>

**Correct Answers: A and D**

**Explanation:**

**A is correct:**
- Lambda resource-based policies control who can invoke the function
- When subscribing Lambda to SNS, the subscription process normally auto-adds this permission
- If manually created or if permissions were removed, SNS cannot invoke Lambda
- Check with: `aws lambda get-policy --function-name MyFunction`

**Required Lambda Resource Policy:**
```json
{
  "Effect": "Allow",
  "Principal": {
    "Service": "sns.amazonaws.com"
  },
  "Action": "lambda:InvokeFunction",
  "Resource": "arn:aws:lambda:region:account:function:MyFunction",
  "Condition": {
    "ArnLike": {
      "AWS:SourceArn": "arn:aws:sns:region:account:MyTopic"
    }
  }
}
```

**D is correct:**
- Lambda subscriptions to SNS can be in "Pending Confirmation" status
- While most AWS service-to-service subscriptions (Lambda, SQS) auto-confirm, sometimes manual confirmation is required
- Check subscription status in SNS console or CLI: `aws sns list-subscriptions-by-topic`
- Unconfirmed subscriptions don't receive messages

**Why other options are incorrect:**
- **B**: SNS topic access policies control who can publish/subscribe to the topic, not whether Lambda receives messages. Default policy allows publishing
- **C**: Lambda execution role is for what Lambda can do (access other AWS resources), not for receiving SNS invocations. Resource policy controls invocation
- **E**: This is partially plausible, but the question states "Lambda executes successfully when tested manually," implying KMS permissions are correct. Also, KMS decrypt permission would be in Lambda execution role, not resource policy

**Key Concepts:**
- Lambda resource-based policy = who can invoke Lambda
- Lambda execution role = what Lambda can do
- Always confirm subscriptions (check status)
- Use CloudWatch Logs for Lambda to see if it's being invoked
</details>

---

### Question 15
A global company needs to send mobile push notifications to millions of users across iOS, Android, and web platforms. They want to use SNS for message delivery but need to send different message formats for each platform (APNS for iOS, FCM for Android, web push). How should this be architected?

A) Create separate SNS topics for each platform (iOS, Android, Web) and publish platform-specific messages to each
B) Use SNS mobile push with platform application endpoints, and publish a single message with platform-specific message overrides using JSON message format
C) Publish to a single SNS Standard topic and use message attributes to specify platform type, then filter subscriptions by platform
D) Use SNS to trigger Lambda functions that translate and forward messages to platform-specific notification services

<details>
<summary>Answer</summary>

**Correct Answer: B**

**Explanation:**
- SNS Mobile Push supports multiple platforms: APNS (iOS), FCM (Android), ADM (Amazon), Baidu, WNS (Windows), MPNS
- Platform applications represent each platform's push notification service
- Device tokens/endpoints are registered with SNS for each user's device
- Publish once to a topic or directly to endpoints using JSON message format with platform-specific overrides

**JSON Message Format with Overrides:**
```json
{
  "default": "Generic message for unsupported platforms",
  "APNS": "{\"aps\":{\"alert\":\"iOS specific message\",\"badge\":1}}",
  "APNS_SANDBOX": "{\"aps\":{\"alert\":\"iOS dev message\"}}",
  "GCM": "{\"data\":{\"message\":\"Android FCM message\"}}",
  "FCM": "{\"notification\":{\"title\":\"Android\",\"body\":\"FCM message\"}}"
}
```

**Architecture:**
1. Create platform applications in SNS for APNS, FCM, etc.
2. Register device tokens as endpoints
3. Optionally create topic and subscribe endpoints
4. Publish with platform-specific message overrides
5. SNS routes to appropriate platform service

**Why other options are incorrect:**
- **A**: Multiple topics create operational overhead. You'd need to maintain device subscriptions across multiple topics and publish multiple times
- **C**: Message filtering doesn't help with different message formats. Subscriptions would still receive the same message body
- **D**: Lambda adds unnecessary complexity, latency, and cost. SNS natively supports multi-platform push notifications

**Key Concepts:**
- SNS Mobile Push abstracts platform-specific push notification services
- One publish call can deliver to multiple platforms with different formats
- Endpoints represent individual devices
- Topics can have multiple platform endpoints subscribed
- Monitor delivery with CloudWatch metrics: `NumberOfNotificationsFailed`
</details>
