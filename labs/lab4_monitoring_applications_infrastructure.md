# Lab 4: Monitoring Applications and Infrastructure

## Estimated Time: 2 hours
## Objectives
- Master CloudWatch metrics, alarms, dashboards, Logs, and Logs Insights
- Configure the CloudWatch Agent for OS-level metrics and log collection
- Set up CloudTrail for API auditing
- Build event-driven architectures with EventBridge
- Implement alerting with SNS
- Understand Amazon Managed Service for Prometheus

## Prerequisites
- AWS account, AWS CLI configured
- EC2 instance with SSM Agent (Amazon Linux 2)

> **This lab covers your identified weak areas in depth. Take extra time here.**

---

## Part 1: CloudWatch Metrics Deep Dive

### Default EC2 Metrics (no agent required)

| Metric | Description | Notes |
|--------|-------------|-------|
| CPUUtilization | Percentage of CPU used | Most common exam metric |
| NetworkIn/NetworkOut | Bytes transferred | Does NOT include VPC internal traffic |
| DiskReadOps/DiskWriteOps | I/O operations | Instance store only (NOT EBS) |
| StatusCheckFailed | Instance + system status | 0 = OK, 1 = Failed |
| StatusCheckFailed_Instance | Instance-level check | OS issues |
| StatusCheckFailed_System | Hardware-level check | AWS infrastructure issues |

> **CRITICAL Exam Fact:** Memory utilization and disk space are NOT default CloudWatch metrics. You MUST install the CloudWatch Agent to collect them.

### Enable Detailed Monitoring

```bash
# Default: 5-minute intervals (free)
# Detailed: 1-minute intervals ($$$)
aws ec2 monitor-instances --instance-ids i-0123456789abcdef0

# Verify
aws ec2 describe-instances \
  --instance-ids i-0123456789abcdef0 \
  --query 'Reservations[0].Instances[0].Monitoring.State'
```

### Install and Configure CloudWatch Agent

**Step 1: Store agent configuration in Parameter Store**

```bash
aws ssm put-parameter \
  --name "AmazonCloudWatch-linux-config" \
  --type String \
  --value '{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "cwagent"
  },
  "metrics": {
    "namespace": "CustomMetrics/EC2",
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}",
      "AutoScalingGroupName": "${aws:AutoScalingGroupName}"
    },
    "metrics_collected": {
      "mem": {
        "measurement": ["mem_used_percent", "mem_available_percent"],
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": ["used_percent", "inodes_free"],
        "resources": ["/", "/tmp"],
        "metrics_collection_interval": 60
      },
      "swap": {
        "measurement": ["swap_used_percent"],
        "metrics_collection_interval": 60
      },
      "cpu": {
        "measurement": ["cpu_usage_idle", "cpu_usage_iowait"],
        "metrics_collection_interval": 60,
        "totalcpu": true
      },
      "netstat": {
        "measurement": ["tcp_established", "tcp_time_wait"],
        "metrics_collection_interval": 60
      }
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/messages",
            "log_group_name": "/ec2/system/messages",
            "log_stream_name": "{instance_id}",
            "retention_in_days": 30
          },
          {
            "file_path": "/var/log/secure",
            "log_group_name": "/ec2/system/secure",
            "log_stream_name": "{instance_id}",
            "retention_in_days": 30
          },
          {
            "file_path": "/var/log/httpd/access_log",
            "log_group_name": "/ec2/httpd/access",
            "log_stream_name": "{instance_id}",
            "retention_in_days": 14
          },
          {
            "file_path": "/var/log/httpd/error_log",
            "log_group_name": "/ec2/httpd/error",
            "log_stream_name": "{instance_id}",
            "retention_in_days": 14
          }
        ]
      }
    }
  }
}'
```

**Step 2: Install agent via SSM Run Command**

```bash
# Install the CloudWatch Agent package
aws ssm send-command \
  --document-name "AWS-ConfigureAWSPackage" \
  --targets "Key=tag:Environment,Values=Production" \
  --parameters '{
    "action": ["Install"],
    "name": ["AmazonCloudWatchAgent"]
  }'

# Configure the agent with the parameter
aws ssm send-command \
  --document-name "AmazonCloudWatch-ManageAgent" \
  --targets "Key=tag:Environment,Values=Production" \
  --parameters '{
    "action": ["configure"],
    "mode": ["ec2"],
    "optionalConfigurationSource": ["ssm"],
    "optionalConfigurationLocation": ["AmazonCloudWatch-linux-config"]
  }'
```

### Custom Metrics with put-metric-data

```bash
# Publish a custom metric
aws cloudwatch put-metric-data \
  --namespace "MyApp/Production" \
  --metric-name "ActiveConnections" \
  --value 42 \
  --unit Count \
  --dimensions "ServiceName=WebApp,Environment=Prod"

# High-resolution metric (1-second granularity)
aws cloudwatch put-metric-data \
  --namespace "MyApp/Production" \
  --metric-name "RequestLatency" \
  --value 150 \
  --unit Milliseconds \
  --storage-resolution 1
```

### Key Metric Concepts

| Concept | Description | Exam Relevance |
|---------|-------------|----------------|
| **Namespace** | Container for metrics (e.g., AWS/EC2, Custom/MyApp) | Custom metrics need custom namespace |
| **Dimension** | Name-value pair to filter (e.g., InstanceId, ASGName) | Up to 30 dimensions per metric |
| **Period** | Time length for each stat (e.g., 60s, 300s) | Default metrics: 5 min, Detailed: 1 min |
| **Statistics** | Aggregation: Average, Sum, Min, Max, SampleCount | Choose wisely for alarms |
| **Percentiles** | p50, p90, p95, p99 | Great for latency monitoring |
| **StorageResolution** | Standard (60s) or High-resolution (1s) | High-res costs more |

> **Exam Tip:** Statistics matter! CPU alarm should use Average. Queue depth should use Sum. Latency SLA should use p99.

---

## Part 2: CloudWatch Alarms

### Create a Basic Alarm

```bash
# CPU alarm with SNS notification
aws cloudwatch put-metric-alarm \
  --alarm-name "HighCPU-WebServer" \
  --alarm-description "CPU exceeds 80% for 5 minutes" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --datapoints-to-alarm 2 \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:alerts" \
  --ok-actions "arn:aws:sns:us-east-1:123456789012:alerts" \
  --dimensions "Name=InstanceId,Value=i-0123456789abcdef0" \
  --treat-missing-data notBreaching
```

### Alarm States

| State | Meaning |
|-------|---------|
| **OK** | Metric is within the defined threshold |
| **ALARM** | Metric has breached the threshold |
| **INSUFFICIENT_DATA** | Not enough data to evaluate (new alarm, stopped instance, or wrong dimension) |

### Missing Data Treatment

| Setting | Behavior | Use Case |
|---------|----------|----------|
| **breaching** | Missing data = threshold breached | Instance should always be running |
| **notBreaching** | Missing data = within threshold | Batch jobs (OK if no data during off-hours) |
| **ignore** | Skip missing points, keep current state | Sporadic metrics |
| **missing** | Alarm goes to INSUFFICIENT_DATA | Default behavior |

> **Exam Tip:** If an alarm keeps going to INSUFFICIENT_DATA when instances are stopped, switch to `notBreaching` or `ignore`. If you WANT to know when an instance stops sending metrics, use `breaching`.

### Composite Alarms

```bash
# Create individual alarms first, then combine
aws cloudwatch put-composite-alarm \
  --alarm-name "HighResource-WebServer" \
  --alarm-rule 'ALARM("HighCPU-WebServer") AND ALARM("HighMemory-WebServer")' \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:critical-alerts" \
  --alarm-description "Both CPU AND memory are high"

# OR logic
aws cloudwatch put-composite-alarm \
  --alarm-name "AnyResource-WebServer" \
  --alarm-rule 'ALARM("HighCPU-WebServer") OR ALARM("HighMemory-WebServer")' \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:alerts"
```

> **Exam Tip:** Composite alarms reduce alarm noise. Instead of getting 2 separate alerts, only alert when BOTH conditions are true. They also support suppression (don't alert on child alarms if composite is already alarming).

### EC2 Alarm Actions

```bash
# Auto-recover an instance (moves to new hardware)
aws cloudwatch put-metric-alarm \
  --alarm-name "AutoRecover-Instance" \
  --metric-name StatusCheckFailed_System \
  --namespace AWS/EC2 \
  --statistic Maximum \
  --period 60 \
  --evaluation-periods 2 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --dimensions "Name=InstanceId,Value=i-0123456789abcdef0" \
  --alarm-actions "arn:aws:automate:us-east-1:ec2:recover"

# Auto-stop idle dev instance
aws cloudwatch put-metric-alarm \
  --alarm-name "StopIdle-DevInstance" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 3600 \
  --evaluation-periods 2 \
  --threshold 5 \
  --comparison-operator LessThanThreshold \
  --dimensions "Name=InstanceId,Value=i-0123456789abcdef0" \
  --alarm-actions "arn:aws:automate:us-east-1:ec2:stop"
```

> **Exam Tip:** EC2 alarm actions: `recover` (StatusCheckFailed_System), `stop`, `terminate`, `reboot`. The recover action migrates the instance to new hardware with the same instance ID, IP, and EBS volumes.

### Metric Math in Alarms

```bash
# Alert when error rate exceeds 5%
aws cloudwatch put-metric-alarm \
  --alarm-name "HighErrorRate" \
  --metrics '[
    {"Id":"errors","MetricStat":{"Metric":{"Namespace":"AWS/ApplicationELB","MetricName":"HTTPCode_Target_5XX_Count","Dimensions":[{"Name":"LoadBalancer","Value":"app/my-alb/abc123"}]},"Period":300,"Stat":"Sum"},"ReturnData":false},
    {"Id":"requests","MetricStat":{"Metric":{"Namespace":"AWS/ApplicationELB","MetricName":"RequestCount","Dimensions":[{"Name":"LoadBalancer","Value":"app/my-alb/abc123"}]},"Period":300,"Stat":"Sum"},"ReturnData":false},
    {"Id":"error_rate","Expression":"(errors/requests)*100","Label":"Error Rate %","ReturnData":true}
  ]' \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:alerts"
```

### Anomaly Detection

```bash
# Create anomaly detection alarm (auto-learns normal patterns)
aws cloudwatch put-anomaly-detector \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0123456789abcdef0 \
  --stat Average

aws cloudwatch put-metric-alarm \
  --alarm-name "AnomalyCPU" \
  --metrics '[
    {"Id":"m1","MetricStat":{"Metric":{"Namespace":"AWS/EC2","MetricName":"CPUUtilization","Dimensions":[{"Name":"InstanceId","Value":"i-0123456789abcdef0"}]},"Period":300,"Stat":"Average"}},
    {"Id":"ad1","Expression":"ANOMALY_DETECTION_BAND(m1, 2)"}
  ]' \
  --threshold-metric-id "ad1" \
  --comparison-operator LessThanLowerOrGreaterThanUpperThreshold \
  --evaluation-periods 3 \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:alerts"
```

---

## Part 3: CloudWatch Dashboards

### Create a Dashboard

```bash
aws cloudwatch put-dashboard \
  --dashboard-name "Production-Overview" \
  --dashboard-body '{
  "widgets": [
    {
      "type": "metric",
      "x": 0, "y": 0, "width": 12, "height": 6,
      "properties": {
        "title": "EC2 CPU Utilization",
        "metrics": [
          ["AWS/EC2", "CPUUtilization", "AutoScalingGroupName", "prod-asg", {"stat": "Average"}]
        ],
        "period": 300,
        "region": "us-east-1",
        "view": "timeSeries"
      }
    },
    {
      "type": "metric",
      "x": 12, "y": 0, "width": 12, "height": 6,
      "properties": {
        "title": "ALB Request Count & Latency",
        "metrics": [
          ["AWS/ApplicationELB", "RequestCount", "LoadBalancer", "app/my-alb/abc123", {"stat": "Sum"}],
          [".", "TargetResponseTime", ".", ".", {"stat": "p99", "yAxis": "right"}]
        ],
        "period": 60,
        "region": "us-east-1"
      }
    },
    {
      "type": "alarm",
      "x": 0, "y": 6, "width": 24, "height": 3,
      "properties": {
        "title": "Alarm Status",
        "alarms": [
          "arn:aws:cloudwatch:us-east-1:123456789012:alarm:HighCPU-WebServer",
          "arn:aws:cloudwatch:us-east-1:123456789012:alarm:HighErrorRate"
        ]
      }
    },
    {
      "type": "log",
      "x": 0, "y": 9, "width": 24, "height": 6,
      "properties": {
        "title": "Recent Errors",
        "query": "SOURCE '\''/ec2/httpd/error'\'' | filter @message like /ERROR/\n| sort @timestamp desc\n| limit 20",
        "region": "us-east-1",
        "view": "table"
      }
    }
  ]
}'
```

### Dashboard Features

| Feature | Description |
|---------|-------------|
| **Cross-account** | Share dashboards across accounts using CloudWatch cross-account observability |
| **Cross-region** | Add widgets showing metrics from different regions |
| **Automatic dashboards** | AWS auto-generates dashboards per service (EC2, Lambda, etc.) |
| **Variables** | Dynamic dashboards with dropdown filters |
| **Sharing** | Share via link (public), SSO, or specific email |

---

## Part 4: CloudWatch Logs

### Create Log Group and Set Retention

```bash
# Create log group
aws logs create-log-group --log-group-name "/myapp/production"

# Set retention (important for cost control)
aws logs put-retention-policy \
  --log-group-name "/myapp/production" \
  --retention-in-days 30

# Available retention values: 1, 3, 5, 7, 14, 30, 60, 90, 120, 150, 180,
# 365, 400, 545, 731, 1096, 1827, 2192, 2557, 2922, 3288, 3653
# Or never expire (default)
```

### Metric Filters

```bash
# Count ERROR occurrences in application logs
aws logs put-metric-filter \
  --log-group-name "/myapp/production" \
  --filter-name "ErrorCount" \
  --filter-pattern "ERROR" \
  --metric-transformations \
    metricName=AppErrorCount,metricNamespace=MyApp/Errors,metricValue=1,defaultValue=0

# Count specific HTTP status codes
aws logs put-metric-filter \
  --log-group-name "/ec2/httpd/access" \
  --filter-name "5xxErrors" \
  --filter-pattern '[ip, id, user, timestamp, request, status_code=5*, size]' \
  --metric-transformations \
    metricName=Http5xxCount,metricNamespace=MyApp/HTTP,metricValue=1

# Extract a numeric value from logs
aws logs put-metric-filter \
  --log-group-name "/myapp/production" \
  --filter-name "ResponseTime" \
  --filter-pattern '[..., response_time]' \
  --metric-transformations \
    metricName=ResponseTime,metricNamespace=MyApp/Performance,metricValue='$response_time'

# Then create an alarm on the extracted metric
aws cloudwatch put-metric-alarm \
  --alarm-name "HighErrorRate-App" \
  --metric-name AppErrorCount \
  --namespace MyApp/Errors \
  --statistic Sum \
  --period 300 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:alerts"
```

> **Exam Tip:** Metric filters are applied RETROACTIVELY? NO! Metric filters only match log events that arrive AFTER the filter is created. They don't process existing log data.

### Subscription Filters

```bash
# Stream logs to Lambda for real-time processing
aws logs put-subscription-filter \
  --log-group-name "/myapp/production" \
  --filter-name "ErrorProcessor" \
  --filter-pattern "ERROR" \
  --destination-arn "arn:aws:lambda:us-east-1:123456789012:function:ProcessErrors"

# Stream to Kinesis Data Firehose -> S3 (for archiving)
aws logs put-subscription-filter \
  --log-group-name "/myapp/production" \
  --filter-name "ArchiveToS3" \
  --filter-pattern "" \
  --destination-arn "arn:aws:firehose:us-east-1:123456789012:deliverystream/logs-to-s3" \
  --role-arn "arn:aws:iam::123456789012:role/CWLToFirehoseRole"
```

> **Exam Tip:** Each log group can have up to 2 subscription filters. Destinations: Lambda, Kinesis Data Streams, Kinesis Data Firehose, or another account's destination.

### Export Logs to S3

```bash
# One-time export (for analysis with Athena)
aws logs create-export-task \
  --log-group-name "/myapp/production" \
  --from 1680307200000 \
  --to 1680393600000 \
  --destination "my-log-export-bucket" \
  --destination-prefix "exports/myapp"
```

> **Exam Tip:** Export to S3 is NOT real-time (can take up to 12 hours). For real-time delivery to S3, use a subscription filter to Kinesis Data Firehose.

### Log Classes

| Class | Features | Cost |
|-------|----------|------|
| **Standard** | Full features (metric filters, subscriptions, Logs Insights) | Higher |
| **Infrequent Access** | Ingestion, storage, Logs Insights only | 50% cheaper |

---

## Part 5: CloudWatch Logs Insights

### Query Syntax Tutorial

```sql
-- Find top 10 error messages in last 24 hours
filter @message like /ERROR/
| stats count(*) as errorCount by @message
| sort errorCount desc
| limit 10

-- Latency percentiles for API requests
filter @message like /API/
| stats avg(@duration) as avgLatency,
        pctl(@duration, 50) as p50,
        pctl(@duration, 95) as p95,
        pctl(@duration, 99) as p99
| limit 1

-- Time-series: requests per 5-minute bucket
stats count(*) as requests by bin(5m)
| sort @timestamp asc

-- Parse structured log messages
parse @message "* - - [*] \"* * *\" * *" as ip, timestamp, method, path, protocol, status, size
| filter status >= 500
| stats count(*) as errors by path
| sort errors desc

-- Top 10 source IPs generating errors
parse @message "* - - [*] \"* * *\" * *" as ip, timestamp, method, path, protocol, status, size
| filter status >= 400
| stats count(*) as errorCount by ip
| sort errorCount desc
| limit 10

-- Find slowest requests
filter @duration > 5000
| sort @duration desc
| limit 20
| display @timestamp, @duration, @message

-- Unique error types over time
filter @message like /Exception/
| stats count_distinct(@message) as uniqueErrors by bin(1h)

-- Cross-log-group query (specify multiple log groups in the UI)
filter @logStream like /web/
| stats count(*) by @logStream
```

### Run from CLI

```bash
# Start a query
QUERY_ID=$(aws logs start-query \
  --log-group-names "/myapp/production" "/ec2/httpd/error" \
  --start-time $(date -d '24 hours ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'filter @message like /ERROR/ | stats count(*) by bin(1h)' \
  --query 'queryId' --output text)

# Get results (may need to wait a moment)
aws logs get-query-results --query-id $QUERY_ID
```

> **Exam Tip:** Logs Insights is for ad-hoc, interactive queries. Metric filters are for continuous monitoring. If the question asks about "investigate" or "troubleshoot" -> Logs Insights. If it asks about "ongoing monitoring" or "alarm" -> metric filter.

---

## Part 6: Amazon EventBridge

### Default Bus Events (AWS Service Events)

```bash
# Rule: detect EC2 state changes
aws events put-rule \
  --name "ec2-state-change" \
  --event-pattern '{
    "source": ["aws.ec2"],
    "detail-type": ["EC2 Instance State-change Notification"],
    "detail": {
      "state": ["stopped", "terminated"]
    }
  }' \
  --description "Detect when EC2 instances stop or terminate"

# Rule: detect console sign-in (especially root)
aws events put-rule \
  --name "root-login-alert" \
  --event-pattern '{
    "source": ["aws.signin"],
    "detail-type": ["AWS Console Sign In via CloudTrail"],
    "detail": {
      "userIdentity": {
        "type": ["Root"]
      }
    }
  }'

# Rule: detect security group changes
aws events put-rule \
  --name "sg-change-alert" \
  --event-pattern '{
    "source": ["aws.ec2"],
    "detail-type": ["AWS API Call via CloudTrail"],
    "detail": {
      "eventSource": ["ec2.amazonaws.com"],
      "eventName": ["AuthorizeSecurityGroupIngress", "RevokeSecurityGroupIngress"]
    }
  }'
```

### Add Targets

```bash
# SNS target
aws events put-targets \
  --rule "root-login-alert" \
  --targets '[{
    "Id": "sns-alert",
    "Arn": "arn:aws:sns:us-east-1:123456789012:security-alerts"
  }]'

# Lambda target with input transformer
aws events put-targets \
  --rule "ec2-state-change" \
  --targets '[{
    "Id": "process-event",
    "Arn": "arn:aws:lambda:us-east-1:123456789012:function:ProcessEC2Event",
    "InputTransformer": {
      "InputPathsMap": {
        "instance": "$.detail.instance-id",
        "state": "$.detail.state",
        "time": "$.time"
      },
      "InputTemplate": "\"Instance <instance> changed to <state> at <time>\""
    }
  }]'
```

### Custom Events

```bash
# Send custom event
aws events put-events --entries '[{
  "Source": "myapp.orders",
  "DetailType": "Order Placed",
  "Detail": "{\"orderId\": \"12345\", \"amount\": 99.99, \"customer\": \"john@example.com\"}",
  "EventBusName": "default"
}]'
```

### Schedule Rules

```bash
# Run every 5 minutes
aws events put-rule \
  --name "health-check" \
  --schedule-expression "rate(5 minutes)"

# Run every weekday at 8 AM UTC
aws events put-rule \
  --name "morning-report" \
  --schedule-expression "cron(0 8 ? * MON-FRI *)"
```

### Archive and Replay

```bash
# Archive events for replay
aws events create-archive \
  --archive-name "ec2-events-archive" \
  --event-source-arn "arn:aws:events:us-east-1:123456789012:event-bus/default" \
  --event-pattern '{"source": ["aws.ec2"]}' \
  --retention-days 90

# Replay events from archive
aws events start-replay \
  --replay-name "replay-ec2-events" \
  --event-source-arn "arn:aws:events:us-east-1:123456789012:event-bus/default" \
  --destination '{"Arn": "arn:aws:events:us-east-1:123456789012:event-bus/replay-bus"}' \
  --event-start-time "2024-01-01T00:00:00Z" \
  --event-end-time "2024-01-02T00:00:00Z"
```

### EventBridge vs SNS vs SQS

| Feature | EventBridge | SNS | SQS |
|---------|------------|-----|-----|
| **Pattern** | Event routing with filtering | Pub/Sub fan-out | Message queue |
| **Filtering** | Complex event patterns (nested JSON) | Message attribute filtering | No filtering |
| **Targets** | 30+ AWS services directly | Protocol-based (HTTP, Lambda, SQS, Email) | Consumer pulls messages |
| **Use case** | Event-driven automation | Notifications, fan-out | Decoupling, buffering |
| **Dead letter** | DLQ per target | DLQ per subscription | DLQ per queue |
| **Scheduling** | Built-in (cron/rate) | No | No |

> **Exam Tip:** EventBridge is the "glue" for event-driven architectures. SNS is for notifications and fan-out. SQS is for queuing and decoupling.

---

## Part 7: Amazon SNS

### Create Topic and Subscriptions

```bash
# Create Standard topic
TOPIC_ARN=$(aws sns create-topic --name "app-alerts" --query 'TopicArn' --output text)

# Create FIFO topic (strict ordering, deduplication)
aws sns create-topic --name "order-events.fifo" \
  --attributes FifoTopic=true,ContentBasedDeduplication=true

# Subscribe email
aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol email \
  --notification-endpoint admin@example.com

# Subscribe Lambda
aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol lambda \
  --notification-endpoint "arn:aws:lambda:us-east-1:123456789012:function:ProcessAlert"

# Subscribe SQS (fan-out pattern)
aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol sqs \
  --notification-endpoint "arn:aws:sqs:us-east-1:123456789012:alert-queue"
```

### Message Filtering

```bash
# Only receive critical alerts
aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol email \
  --notification-endpoint oncall@example.com \
  --attributes '{
    "FilterPolicy": "{\"severity\": [\"CRITICAL\"], \"environment\": [\"production\"]}"
  }'

# Publish with attributes
aws sns publish \
  --topic-arn $TOPIC_ARN \
  --message "Database connection pool exhausted" \
  --message-attributes '{
    "severity": {"DataType": "String", "StringValue": "CRITICAL"},
    "environment": {"DataType": "String", "StringValue": "production"},
    "service": {"DataType": "String", "StringValue": "database"}
  }'
```

### SNS + SQS Fan-out Pattern

```
CloudWatch Alarm -> SNS Topic -> SQS Queue 1 (email notifications)
                              -> SQS Queue 2 (auto-remediation Lambda)
                              -> SQS Queue 3 (logging/archiving)
```

> **Exam Tip:** SNS fan-out to multiple SQS queues is a common pattern for decoupling. Each queue processes the message independently. FIFO topics pair with FIFO queues for ordering guarantees.

---

## Part 8: AWS CloudTrail

### Create a Multi-Region Trail

```bash
# Create trail
aws cloudtrail create-trail \
  --name "org-audit-trail" \
  --s3-bucket-name "cloudtrail-logs-123456789012" \
  --is-multi-region-trail \
  --enable-log-file-validation \
  --include-global-service-events \
  --kms-key-id "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123"

# Start logging
aws cloudtrail start-logging --name "org-audit-trail"

# Add CloudWatch Logs integration
aws cloudtrail update-trail \
  --name "org-audit-trail" \
  --cloud-watch-logs-log-group-arn "arn:aws:logs:us-east-1:123456789012:log-group:CloudTrail/Logs:*" \
  --cloud-watch-logs-role-arn "arn:aws:iam::123456789012:role/CloudTrailToCloudWatch"
```

### Event Types

| Event Type | What It Captures | Cost | Examples |
|------------|-----------------|------|----------|
| **Management events** | Control plane API calls | Free (1 copy) | CreateBucket, RunInstances, CreateUser |
| **Data events** | Data plane operations | Per-event charge | S3 GetObject/PutObject, Lambda Invoke |
| **Insights events** | Unusual API activity patterns | Per-event charge | Spike in RunInstances calls |

```bash
# Enable data events for S3
aws cloudtrail put-event-selectors \
  --trail-name "org-audit-trail" \
  --advanced-event-selectors '[{
    "Name": "S3DataEvents",
    "FieldSelectors": [
      {"Field": "eventCategory", "Equals": ["Data"]},
      {"Field": "resources.type", "Equals": ["AWS::S3::Object"]}
    ]
  }]'

# Enable CloudTrail Insights
aws cloudtrail put-insight-selectors \
  --trail-name "org-audit-trail" \
  --insight-selectors '[{"InsightType": "ApiCallRateInsight"}, {"InsightType": "ApiErrorRateInsight"}]'
```

### Log File Integrity Validation

```bash
# Validate log files haven't been tampered with
aws cloudtrail validate-logs \
  --trail-arn "arn:aws:cloudtrail:us-east-1:123456789012:trail/org-audit-trail" \
  --start-time "2024-01-01T00:00:00Z" \
  --end-time "2024-01-02T00:00:00Z"
```

> **Exam Tip:** Log file integrity validation uses SHA-256 digest files. If someone tampers with log files in S3, validation will detect it. This is critical for compliance (PCI, HIPAA).

### CloudTrail + CloudWatch Logs Pipeline

```
CloudTrail -> S3 (storage) + CloudWatch Logs (real-time analysis)
                              -> Metric Filter (count specific API calls)
                                -> CloudWatch Alarm
                                  -> SNS notification
```

Example: Alert on root user API calls

```bash
# Metric filter for root user activity
aws logs put-metric-filter \
  --log-group-name "CloudTrail/Logs" \
  --filter-name "RootAccountUsage" \
  --filter-pattern '{ $.userIdentity.type = "Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != "AwsServiceEvent" }' \
  --metric-transformations \
    metricName=RootAccountUsageCount,metricNamespace=CloudTrailMetrics,metricValue=1

# Alarm on the metric
aws cloudwatch put-metric-alarm \
  --alarm-name "RootUserActivity" \
  --metric-name RootAccountUsageCount \
  --namespace CloudTrailMetrics \
  --statistic Sum \
  --period 300 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:security-alerts"
```

### CloudTrail Lake

```bash
# Create an event data store
aws cloudtrail create-event-data-store \
  --name "audit-lake" \
  --retention-period 365

# Query with SQL
aws cloudtrail start-query \
  --query-statement "SELECT eventName, COUNT(*) as cnt FROM audit-lake WHERE eventTime > '2024-01-01' AND userIdentity.type = 'Root' GROUP BY eventName ORDER BY cnt DESC"
```

### When to Use Which

| Need | Service |
|------|---------|
| Who called an API? | CloudTrail |
| Is my CPU high? | CloudWatch Metrics |
| What's in my application logs? | CloudWatch Logs |
| Was network traffic allowed/denied? | VPC Flow Logs |
| What changed in my resource configuration? | AWS Config |

---

## Part 9: Amazon Managed Service for Prometheus (AMP)

### Create a Workspace

```bash
aws amp create-workspace \
  --alias "production-metrics" \
  --query 'workspaceId' --output text
```

### When to Use AMP vs CloudWatch

| Factor | AMP | CloudWatch |
|--------|-----|------------|
| **Workload type** | Containers (EKS, ECS) | EC2, Lambda, all AWS services |
| **Query language** | PromQL | CloudWatch Insights, metric math |
| **Ecosystem** | Prometheus/Grafana ecosystem | AWS native dashboards |
| **Existing setup** | Teams already using Prometheus | Greenfield or AWS-native |
| **Cost model** | Per metric sample ingested + stored + queried | Per metric, alarm, dashboard, log |
| **Visualization** | Amazon Managed Grafana | CloudWatch Dashboards |

### PromQL Examples

```promql
# CPU usage by container
rate(container_cpu_usage_seconds_total{namespace="production"}[5m])

# Memory usage percentage
container_memory_usage_bytes / container_spec_memory_limit_bytes * 100

# Request rate
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
```

### Integration Architecture

```
EKS/ECS -> Prometheus remote_write -> AMP Workspace -> Amazon Managed Grafana
                                                     -> PromQL alerts -> SNS
```

> **Exam Tip:** AMP is for organizations that want to use Prometheus for container monitoring without managing Prometheus servers. CloudWatch Container Insights is the AWS-native alternative. If the question mentions "existing Prometheus setup" or "PromQL", the answer is AMP.

---

## Part 10: Review Questions

### Question 1

A SysOps administrator notices that CloudWatch alarms for EC2 instances keep transitioning to INSUFFICIENT_DATA state at night when instances are stopped for cost savings. The team doesn't want to be notified during this time. What is the BEST solution?

**A)** Delete the alarms at night and recreate them in the morning
**B)** Change the alarm's missing data treatment to `notBreaching`
**C)** Disable the alarm actions at night using `disable-alarm-actions`
**D)** Set a longer evaluation period to cover the overnight gap

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Setting `treat-missing-data` to `notBreaching` means missing data points are treated as within the threshold (OK). The alarm stays in OK state when instances stop sending metrics. `disable-alarm-actions` (C) prevents notifications but the alarm still flips to INSUFFICIENT_DATA which is confusing. Deleting and recreating (A) is manual and error-prone. Longer evaluation periods (D) just delay the state change.
</details>

---

### Question 2

A company needs to create a CloudWatch alarm that triggers ONLY when BOTH CPU utilization exceeds 80% AND memory utilization exceeds 75% on the same instance. What should they use?

**A)** A single alarm with metric math combining both metrics
**B)** Two separate alarms sending to the same SNS topic
**C)** A composite alarm that requires both child alarms to be in ALARM state
**D)** An EventBridge rule matching both CloudWatch alarm state changes

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** Composite alarms combine the state of multiple alarms using AND/OR logic. Create two child alarms (CPU and Memory), then a composite alarm with rule `ALARM("HighCPU") AND ALARM("HighMemory")`. Metric math (A) can't combine metrics from different namespaces (CW Agent custom namespace for memory vs AWS/EC2 for CPU) in this way. Two separate alarms (B) would trigger on EITHER condition. EventBridge (D) adds unnecessary complexity.
</details>

---

### Question 3

An application writes JSON logs to CloudWatch Logs. The team needs to continuously track the number of orders placed per minute and alert if it drops below 10. What is the correct approach?

**A)** Use CloudWatch Logs Insights to run a scheduled query
**B)** Create a metric filter to extract and count order events, then create an alarm on the extracted metric
**C)** Export logs to S3 and query with Athena
**D)** Use CloudWatch Contributor Insights

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Metric filters continuously process log events as they arrive and generate CloudWatch metrics. You can then create an alarm on the resulting metric to alert when orders drop below threshold. Logs Insights (A) is for ad-hoc interactive queries, not continuous monitoring (and can't natively schedule alerts). Athena (C) is not real-time. Contributor Insights (D) identifies top contributors but isn't for counting specific events.
</details>

---

### Question 4

A SysOps administrator receives a CloudWatch alarm notification for high 5xx errors on an ALB. They need to quickly investigate which backend instances are returning errors and what error messages are being logged. What is the FASTEST approach?

**A)** SSH into each instance and check the Apache error logs
**B)** Use CloudWatch Logs Insights to query the ALB access logs and application error logs
**C)** Enable ALB access logging to S3 and query with Athena
**D)** Check the CloudWatch metrics for each target group

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** CloudWatch Logs Insights provides interactive, fast querying across multiple log groups. You can query both ALB access logs and application logs simultaneously to correlate errors. SSH (A) is slow and doesn't scale. S3 + Athena (C) has data freshness lag. CloudWatch metrics (D) show aggregate counts but not the specific error messages or affected requests.
</details>

---

### Question 5

Which CloudTrail event type should be enabled to detect who deleted objects from an S3 bucket?

**A)** Management events
**B)** Data events for S3
**C)** Insights events
**D)** Network activity events

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** S3 object-level operations (GetObject, PutObject, DeleteObject) are data events, not management events. Management events (A) capture bucket-level operations like CreateBucket and DeleteBucket, but NOT object-level operations. Insights events (C) detect unusual patterns, not individual operations. Data events must be explicitly enabled and incur additional costs.
</details>

---

### Question 6

A company wants to be alerted within 1 minute whenever the root user signs into the AWS console. What is the MOST reliable architecture?

**A)** CloudTrail -> S3 -> Lambda trigger -> SNS
**B)** CloudTrail -> CloudWatch Logs -> Metric Filter -> Alarm -> SNS
**C)** EventBridge rule matching console sign-in events -> SNS
**D)** AWS Config rule -> SNS

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** EventBridge receives AWS console sign-in events in near real-time directly from CloudTrail. An event pattern matching `"source": ["aws.signin"]` with `"userIdentity.type": ["Root"]` will trigger immediately. CloudTrail -> CloudWatch Logs (B) works but has a delivery delay of up to 15 minutes. CloudTrail -> S3 (A) is even slower. AWS Config (D) doesn't monitor sign-in events.
</details>

---

### Question 7

A metric filter is created on a log group that already contains 30 days of historical log data. Will the metric filter process the existing 30 days of data?

**A)** Yes, metric filters process all existing and new data
**B)** No, metric filters only process log events that arrive AFTER the filter is created
**C)** Yes, but only if you manually trigger a backfill
**D)** It depends on the log group's retention policy

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Metric filters are NOT retroactive. They only match log events published to the log group after the filter is created. If you need to analyze historical data, use CloudWatch Logs Insights or export to S3 and use Athena.
</details>

---

### Question 8

A company runs containerized workloads on Amazon EKS. Their DevOps team is experienced with Prometheus and wants to monitor all clusters with PromQL queries and Grafana dashboards. What is the LEAST operational overhead solution?

**A)** Deploy self-managed Prometheus servers on EC2 instances
**B)** Use Amazon Managed Service for Prometheus (AMP) with Amazon Managed Grafana
**C)** Use CloudWatch Container Insights
**D)** Deploy Prometheus on EKS using Helm charts

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** AMP provides a fully managed Prometheus-compatible monitoring service. Combined with Amazon Managed Grafana, the team can use PromQL and Grafana without managing any servers. Self-managed Prometheus on EC2 (A) or EKS (D) requires patching, scaling, and storage management. CloudWatch Container Insights (C) doesn't support PromQL or native Grafana integration.
</details>

---

### Question 9

An EventBridge rule is configured to trigger a Lambda function when an EC2 instance is terminated. The Lambda function occasionally fails. How should the administrator ensure no events are lost?

**A)** Configure a retry policy and dead-letter queue on the EventBridge target
**B)** Enable CloudTrail logging for EventBridge
**C)** Create a second EventBridge rule as backup
**D)** Archive all events in EventBridge

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** EventBridge targets support retry policies (max retries, max age) and dead-letter queues (SQS) for failed deliveries. Failed events go to the DLQ for later processing. CloudTrail (B) logs API calls but doesn't prevent event loss. A duplicate rule (C) doesn't help if Lambda fails for both. Archiving (D) is for replay purposes, not failure handling.
</details>

---

### Question 10

A SysOps administrator needs to export CloudWatch Logs from the last 7 days to S3 for long-term archiving. What should they be aware of?

**A)** Export is real-time and data appears in S3 within seconds
**B)** Export can take up to 12 hours to complete and only one export task per log group can run at a time
**C)** Export requires the log group to have Infrequent Access log class
**D)** Export automatically sets up ongoing synchronization

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** The `CreateExportTask` API creates a one-time export that can take up to 12 hours. Only one export task per log group can run concurrently. It does NOT set up ongoing sync (D). For real-time delivery to S3, use a subscription filter to Kinesis Data Firehose. No special log class is required (C). It is definitely not real-time (A).
</details>

---

### Question 11

What is the primary difference between CloudWatch Logs subscription filters and CloudWatch Logs export tasks?

**A)** Subscription filters are more expensive
**B)** Export tasks deliver in real-time; subscription filters have delays
**C)** Subscription filters deliver matching logs in near real-time to targets; export tasks are one-time batch operations
**D)** Export tasks support more destination types

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** Subscription filters continuously process log events as they arrive and deliver matching events to Lambda, Kinesis, or Firehose in near real-time. Export tasks are one-time batch exports to S3 only. The relationship is reversed from option B.
</details>

---

### Question 12

A CloudWatch alarm has `EvaluationPeriods: 3` and `DatapointsToAlarm: 2` with a period of 300 seconds. What does this mean?

**A)** The alarm triggers if the metric breaches the threshold for 3 consecutive periods
**B)** The alarm triggers if the metric breaches the threshold in at least 2 out of the last 3 evaluation periods
**C)** The alarm evaluates every 2 minutes over a 3-minute window
**D)** The alarm triggers after 2 seconds with a 3-period cooldown

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** `EvaluationPeriods: 3` means CloudWatch looks at the last 3 periods (3 x 300s = 15 minutes). `DatapointsToAlarm: 2` means at least 2 of those 3 data points must breach the threshold to trigger ALARM. This is called "M out of N" alarming and helps reduce false positives from brief spikes.
</details>

---

### Question 13

An organization has AWS accounts in multiple regions. They want a single CloudWatch dashboard that displays metrics from all regions. How can they achieve this?

**A)** This is not possible; CloudWatch dashboards are region-specific
**B)** Create cross-region widgets in the dashboard by specifying the region property for each metric
**C)** Enable CloudWatch cross-account observability
**D)** Aggregate all metrics to a central region using Kinesis

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** CloudWatch dashboards support cross-region widgets. Each metric widget can specify a `region` property to pull metrics from any region. The dashboard itself is in one region but displays data from multiple regions. Cross-account observability (C) is for multiple accounts, not multiple regions. Kinesis aggregation (D) is unnecessary complexity.
</details>

---

### Question 14

A CloudTrail trail is configured with log file integrity validation. An auditor asks the team to prove that the log files have not been tampered with. What should they use?

**A)** Check the S3 bucket access logs
**B)** Run `aws cloudtrail validate-logs` to verify digest file integrity
**C)** Enable S3 Object Lock on the bucket
**D)** Check CloudTrail Insights for anomalies

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** `validate-logs` checks the SHA-256 digest files that CloudTrail creates alongside log files. Each digest file contains hashes of the log files delivered in the previous period. If any log file is modified or deleted, validation will report it. S3 access logs (A) show who accessed files but don't prove integrity. Object Lock (C) prevents modification but doesn't prove historical integrity. Insights (D) detect unusual API patterns, not file tampering.
</details>

---

### Question 15

A SysOps administrator wants to create an EventBridge rule that matches EC2 instance state changes but ONLY for instances tagged with `Environment=Production`. How should the event pattern be structured?

**A)** Add a tag filter in the event pattern detail
**B)** This is not possible; EventBridge cannot filter by tags
**C)** Use an input transformer to check tags
**D)** Route to a Lambda function that checks the tag and conditionally processes the event

<details>
<summary>Show Answer</summary>

**Correct Answer: D**

**Explanation:** EC2 state change notification events do NOT include instance tags in the event payload. EventBridge can only filter on fields present in the event. The workaround is to route all EC2 state changes to a Lambda function that calls `describe-instances` to check tags and then processes or discards the event accordingly. Input transformers (C) transform the event but cannot add data that isn't in the original event.
</details>

---

## Cleanup Steps

```bash
# Delete CloudWatch alarms
aws cloudwatch delete-alarms --alarm-names \
  "HighCPU-WebServer" "HighErrorRate" "AutoRecover-Instance" \
  "StopIdle-DevInstance" "AnomalyCPU" "HighResource-WebServer" \
  "HighErrorRate-App" "RootUserActivity"

# Delete CloudWatch dashboard
aws cloudwatch delete-dashboards --dashboard-names "Production-Overview"

# Delete metric filters
aws logs delete-metric-filter --log-group-name "/myapp/production" --filter-name "ErrorCount"
aws logs delete-metric-filter --log-group-name "/ec2/httpd/access" --filter-name "5xxErrors"

# Delete log groups
aws logs delete-log-group --log-group-name "/myapp/production"
aws logs delete-log-group --log-group-name "/ec2/httpd/access"
aws logs delete-log-group --log-group-name "/ec2/httpd/error"

# Delete SNS topic and subscriptions
aws sns delete-topic --topic-arn $TOPIC_ARN

# Delete EventBridge rules (remove targets first)
for RULE in ec2-state-change root-login-alert sg-change-alert health-check morning-report; do
  TARGETS=$(aws events list-targets-by-rule --rule $RULE --query 'Targets[*].Id' --output text 2>/dev/null)
  if [ -n "$TARGETS" ]; then
    aws events remove-targets --rule $RULE --ids $TARGETS
  fi
  aws events delete-rule --name $RULE 2>/dev/null
done

# Delete CloudTrail trail
aws cloudtrail delete-trail --name "org-audit-trail"

# Delete AMP workspace
aws amp delete-workspace --workspace-id WORKSPACE_ID

# Delete CloudWatch Agent SSM parameter
aws ssm delete-parameter --name "AmazonCloudWatch-linux-config"
```

> **Important:** Some resources (like log groups with data) will incur storage charges until deleted. Always verify cleanup is complete.
