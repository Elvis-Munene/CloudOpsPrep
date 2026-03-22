# Lab 7: High Availability and Auto Scaling

## Estimated Time: 1 hour 30 minutes

## Overview
This lab covers High Availability (HA) and Auto Scaling concepts critical for the AWS Certified SysOps Administrator - Associate (SOA-C03) exam, specifically Domain 2: Reliability and Business Continuity. You'll configure Auto Scaling Groups, Elastic Load Balancers, Route 53 health checks, and test resilience patterns.

## Prerequisites
- AWS CLI configured with appropriate credentials
- VPC with at least 2 public subnets in different Availability Zones
- Basic understanding of EC2, networking, and CloudWatch

## Learning Objectives
By the end of this lab, you will be able to:
- Create and configure Auto Scaling Groups with multiple scaling policies
- Deploy and configure Application Load Balancers with health checks
- Implement Route 53 health checks and failover routing
- Test and validate high availability patterns
- Understand lifecycle hooks and instance refresh strategies

---

## Part 1: Auto Scaling Groups

### Step 1.1: Create an IAM Role for EC2 Instances

First, create an IAM role that allows EC2 instances to interact with CloudWatch and Systems Manager.

```bash
# Create trust policy document
cat > /tmp/ec2-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the IAM role
aws iam create-role \
  --role-name CloudOpsLabEC2Role \
  --assume-role-policy-document file:///tmp/ec2-trust-policy.json

# Attach managed policies
aws iam attach-role-policy \
  --role-name CloudOpsLabEC2Role \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

aws iam attach-role-policy \
  --role-name CloudOpsLabEC2Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# Create instance profile
aws iam create-instance-profile \
  --instance-profile-name CloudOpsLabEC2Profile

# Add role to instance profile
aws iam add-role-to-instance-profile \
  --instance-profile-name CloudOpsLabEC2Profile \
  --role-name CloudOpsLabEC2Role

# Wait for the instance profile to be ready
sleep 10
```

### Step 1.2: Create a Security Group

```bash
# Get your VPC ID
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query "Vpcs[0].VpcId" --output text)

# Create security group
SG_ID=$(aws ec2 create-security-group \
  --group-name cloudops-lab7-asg-sg \
  --description "Security group for ASG instances" \
  --vpc-id $VPC_ID \
  --output text --query 'GroupId')

echo "Security Group ID: $SG_ID"

# Allow HTTP from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Allow SSH from anywhere (for testing only - restrict in production)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0

# Allow all traffic within the security group (for inter-instance communication)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol -1 \
  --source-group $SG_ID
```

### Step 1.3: Create a Launch Template with User Data

The launch template defines the instance configuration. The user data script installs a web server and creates a simple status page.

```bash
# Create user data script
cat > /tmp/user-data.sh <<'EOF'
#!/bin/bash
# Update system
yum update -y

# Install Apache web server and stress tool
yum install -y httpd stress

# Get instance metadata
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)
PRIVATE_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)

# Create a simple web page
cat > /var/www/html/index.html <<HTML
<!DOCTYPE html>
<html>
<head>
    <title>CloudOps Lab 7 - Auto Scaling</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; background: #f0f0f0; }
        .container { background: white; padding: 20px; border-radius: 10px; box-shadow: 0 2px 5px rgba(0,0,0,0.1); }
        .info { background: #e3f2fd; padding: 10px; margin: 10px 0; border-left: 4px solid #2196F3; }
        h1 { color: #1976D2; }
    </style>
    <meta http-equiv="refresh" content="30">
</head>
<body>
    <div class="container">
        <h1>Auto Scaling Instance Status</h1>
        <div class="info">
            <strong>Instance ID:</strong> $INSTANCE_ID<br>
            <strong>Availability Zone:</strong> $AZ<br>
            <strong>Private IP:</strong> $PRIVATE_IP<br>
            <strong>Timestamp:</strong> $(date)
        </div>
        <p>This page refreshes every 30 seconds to show the instance is healthy.</p>
        <h2>Test CPU Load</h2>
        <p>To trigger auto scaling, SSH to this instance and run:</p>
        <code>stress --cpu 2 --timeout 300s</code>
    </div>
</body>
</html>
HTML

# Create health check endpoint
cat > /var/www/html/health.html <<HTML
<!DOCTYPE html>
<html>
<head><title>Health Check</title></head>
<body>
    <h1>OK</h1>
    <p>Instance $INSTANCE_ID is healthy</p>
</body>
</html>
HTML

# Start and enable Apache
systemctl start httpd
systemctl enable httpd

# Install CloudWatch agent (optional but recommended for detailed metrics)
wget https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm
rpm -U ./amazon-cloudwatch-agent.rpm
EOF

# Get the latest Amazon Linux 2023 AMI
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023.*-x86_64" "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text)

echo "Using AMI: $AMI_ID"

# Create launch template
aws ec2 create-launch-template \
  --launch-template-name cloudops-lab7-lt \
  --version-description "v1-cloudops-lab7" \
  --launch-template-data "{
    \"ImageId\": \"$AMI_ID\",
    \"InstanceType\": \"t3.micro\",
    \"IamInstanceProfile\": {
      \"Name\": \"CloudOpsLabEC2Profile\"
    },
    \"SecurityGroupIds\": [\"$SG_ID\"],
    \"UserData\": \"$(base64 -w 0 /tmp/user-data.sh)\",
    \"TagSpecifications\": [
      {
        \"ResourceType\": \"instance\",
        \"Tags\": [
          {\"Key\": \"Name\", \"Value\": \"CloudOps-Lab7-ASG-Instance\"},
          {\"Key\": \"Environment\", \"Value\": \"Lab\"},
          {\"Key\": \"Lab\", \"Value\": \"7-HA-AutoScaling\"}
        ]
      }
    ],
    \"MetadataOptions\": {
      \"HttpTokens\": \"required\",
      \"HttpPutResponseHopLimit\": 1
    },
    \"Monitoring\": {
      \"Enabled\": true
    }
  }"

echo "Launch template created successfully"
```

**Exam Tip:** Launch templates are preferred over launch configurations because they support versioning, multiple instance types, and newer features like T2/T3 unlimited mode.

### Step 1.4: Get Subnet Information

```bash
# Get subnet IDs from your VPC (we need at least 2 in different AZs)
SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=default-for-az,Values=true" \
  --query "Subnets[0:2].SubnetId" \
  --output text | tr '\t' ',')

echo "Subnet IDs: $SUBNET_IDS"

# Store individual subnet IDs for later use
SUBNET_1=$(echo $SUBNET_IDS | cut -d',' -f1)
SUBNET_2=$(echo $SUBNET_IDS | cut -d',' -f2)

echo "Subnet 1: $SUBNET_1"
echo "Subnet 2: $SUBNET_2"
```

### Step 1.5: Create Auto Scaling Group

```bash
# Create Auto Scaling Group
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name cloudops-lab7-asg \
  --launch-template "LaunchTemplateName=cloudops-lab7-lt,Version=\$Latest" \
  --min-size 2 \
  --max-size 6 \
  --desired-capacity 2 \
  --default-cooldown 300 \
  --health-check-type ELB \
  --health-check-grace-period 300 \
  --vpc-zone-identifier "$SUBNET_IDS" \
  --tags "Key=Name,Value=CloudOps-Lab7-ASG-Instance,PropagateAtLaunch=true" \
         "Key=Environment,Value=Lab,PropagateAtLaunch=true" \
  --enabled-metrics GroupMinSize,GroupMaxSize,GroupDesiredCapacity,GroupInServiceInstances,GroupTotalInstances

echo "Auto Scaling Group created. Waiting for instances to launch..."
sleep 30

# Verify ASG was created
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names cloudops-lab7-asg \
  --query "AutoScalingGroups[0].[AutoScalingGroupName,MinSize,MaxSize,DesiredCapacity,Instances[*].InstanceId]" \
  --output table
```

**Exam Concepts:**
- **Min/Max/Desired Capacity:** Min = minimum instances, Max = maximum instances, Desired = target number
- **Health Check Grace Period:** Time ASG waits before checking instance health (allows time for bootstrapping)
- **Cooldown Period:** Time to wait between scaling activities to prevent rapid scaling
- **Health Check Type:** EC2 (instance status only) vs ELB (includes load balancer health checks)

### Step 1.6: Configure Instance Warmup and Termination Policies

```bash
# Update ASG with termination policies and default instance warmup
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name cloudops-lab7-asg \
  --default-instance-warmup 120 \
  --termination-policies "OldestLaunchTemplate" "OldestInstance"

# View current configuration
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names cloudops-lab7-asg \
  --query "AutoScalingGroups[0].[DefaultInstanceWarmup,TerminationPolicies]" \
  --output table
```

**Termination Policies Explained:**
- **Default:** Balanced across AZs, then oldest launch template/config
- **OldestInstance:** Terminate oldest instance first
- **NewestInstance:** Terminate newest instance first
- **OldestLaunchTemplate:** Instances from oldest launch template
- **AllocationStrategy:** For Spot instances
- **ClosestToNextInstanceHour:** Minimize wasted compute time

### Step 1.7: Configure Target Tracking Scaling Policy (CPU at 60%)

Target tracking automatically adjusts capacity to maintain a target metric value.

```bash
# Create target tracking scaling policy
cat > /tmp/target-tracking-policy.json <<EOF
{
  "PredefinedMetricSpecification": {
    "PredefinedMetricType": "ASGAverageCPUUtilization"
  },
  "TargetValue": 60.0
}
EOF

aws autoscaling put-scaling-policy \
  --auto-scaling-group-name cloudops-lab7-asg \
  --policy-name cpu-target-tracking-policy \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration file:///tmp/target-tracking-policy.json

echo "Target tracking policy created (CPU target: 60%)"
```

**Exam Tip:** Target tracking is the recommended scaling policy type. It automatically creates CloudWatch alarms and adjusts capacity to keep the metric close to the target value.

### Step 1.8: Configure Step Scaling Policy

Step scaling takes different actions based on the magnitude of the alarm breach.

```bash
# First, create CloudWatch alarms for step scaling
# High CPU alarm
aws cloudwatch put-metric-alarm \
  --alarm-name cloudops-lab7-high-cpu \
  --alarm-description "Trigger when CPU exceeds 70%" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 60 \
  --evaluation-periods 2 \
  --threshold 70 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=AutoScalingGroupName,Value=cloudops-lab7-asg

# Create step scaling policy for scale-out
cat > /tmp/step-scaling-out.json <<EOF
{
  "AdjustmentType": "PercentChangeInCapacity",
  "MetricAggregationType": "Average",
  "EstimatedInstanceWarmup": 120,
  "StepAdjustments": [
    {
      "MetricIntervalLowerBound": 0,
      "MetricIntervalUpperBound": 10,
      "ScalingAdjustment": 10
    },
    {
      "MetricIntervalLowerBound": 10,
      "MetricIntervalUpperBound": 20,
      "ScalingAdjustment": 20
    },
    {
      "MetricIntervalLowerBound": 20,
      "ScalingAdjustment": 30
    }
  ]
}
EOF

STEP_POLICY_ARN=$(aws autoscaling put-scaling-policy \
  --auto-scaling-group-name cloudops-lab7-asg \
  --policy-name step-scaling-out \
  --policy-type StepScaling \
  --adjustment-type PercentChangeInCapacity \
  --metric-aggregation-type Average \
  --step-adjustments file:///tmp/step-scaling-out.json \
  --output text --query 'PolicyARN')

echo "Step scaling policy ARN: $STEP_POLICY_ARN"

# Link the CloudWatch alarm to the step scaling policy
aws cloudwatch put-metric-alarm \
  --alarm-name cloudops-lab7-step-scale-out \
  --alarm-description "Step scaling when CPU is high" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 60 \
  --evaluation-periods 1 \
  --threshold 70 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=AutoScalingGroupName,Value=cloudops-lab7-asg \
  --alarm-actions $STEP_POLICY_ARN
```

**Step Scaling Explanation:**
- CPU 70-80% above threshold: Scale out by 10%
- CPU 80-90% above threshold: Scale out by 20%
- CPU >90% above threshold: Scale out by 30%

### Step 1.9: Configure Simple Scaling Policy

Simple scaling adds or removes a fixed number of instances when an alarm is breached.

```bash
# Create simple scaling policy
SIMPLE_POLICY_ARN=$(aws autoscaling put-scaling-policy \
  --auto-scaling-group-name cloudops-lab7-asg \
  --policy-name simple-scale-out \
  --policy-type SimpleScaling \
  --adjustment-type ChangeInCapacity \
  --scaling-adjustment 2 \
  --cooldown 300 \
  --output text --query 'PolicyARN')

echo "Simple scaling policy ARN: $SIMPLE_POLICY_ARN"

# Create CloudWatch alarm for simple scaling
aws cloudwatch put-metric-alarm \
  --alarm-name cloudops-lab7-simple-scale-alarm \
  --alarm-description "Simple scaling when CPU exceeds 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 120 \
  --evaluation-periods 2 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=AutoScalingGroupName,Value=cloudops-lab7-asg \
  --alarm-actions $SIMPLE_POLICY_ARN
```

**Exam Tip:** Simple scaling respects cooldown periods, while step scaling does not. For more responsive scaling, use step scaling or target tracking.

### Step 1.10: Configure Scheduled Scaling

Scheduled scaling adjusts capacity based on predictable load patterns.

```bash
# Scale up during business hours (9 AM UTC)
aws autoscaling put-scheduled-action \
  --auto-scaling-group-name cloudops-lab7-asg \
  --scheduled-action-name scale-up-business-hours \
  --recurrence "0 9 * * MON-FRI" \
  --min-size 3 \
  --max-size 8 \
  --desired-capacity 4

# Scale down after business hours (6 PM UTC)
aws autoscaling put-scheduled-action \
  --auto-scaling-group-name cloudops-lab7-asg \
  --scheduled-action-name scale-down-off-hours \
  --recurrence "0 18 * * MON-FRI" \
  --min-size 2 \
  --max-size 6 \
  --desired-capacity 2

# View scheduled actions
aws autoscaling describe-scheduled-actions \
  --auto-scaling-group-name cloudops-lab7-asg \
  --output table
```

**Cron Format:** `minute hour day-of-month month day-of-week`

**Exam Tip:** Scheduled scaling is perfect for predictable patterns. It overrides min/max/desired values at the scheduled time.

### Step 1.11: Configure Predictive Scaling

Predictive scaling uses machine learning to forecast capacity needs based on historical patterns.

```bash
# Create predictive scaling policy
cat > /tmp/predictive-scaling-policy.json <<EOF
{
  "MetricSpecifications": [
    {
      "TargetValue": 50.0,
      "PredefinedMetricPairSpecification": {
        "PredefinedMetricType": "ASGCPUUtilization"
      }
    }
  ],
  "Mode": "ForecastAndScale",
  "SchedulingBufferTime": 600
}
EOF

aws autoscaling put-scaling-policy \
  --auto-scaling-group-name cloudops-lab7-asg \
  --policy-name predictive-scaling-policy \
  --policy-type PredictiveScaling \
  --predictive-scaling-configuration file:///tmp/predictive-scaling-policy.json

echo "Predictive scaling policy created"
```

**Predictive Scaling Modes:**
- **ForecastOnly:** Generate forecasts but don't scale
- **ForecastAndScale:** Generate forecasts and scale proactively

**Exam Tip:** Predictive scaling requires at least 24 hours of historical data. It's best for workloads with recurring patterns.

### Step 1.12: Configure Lifecycle Hooks

Lifecycle hooks allow you to perform custom actions when instances launch or terminate.

```bash
# Create SNS topic for lifecycle notifications (optional)
TOPIC_ARN=$(aws sns create-topic \
  --name cloudops-lab7-lifecycle-notifications \
  --output text --query 'TopicArn')

echo "SNS Topic ARN: $TOPIC_ARN"

# Create lifecycle hook for instance launch
aws autoscaling put-lifecycle-hook \
  --lifecycle-hook-name instance-launching-hook \
  --auto-scaling-group-name cloudops-lab7-asg \
  --lifecycle-transition autoscaling:EC2_INSTANCE_LAUNCHING \
  --default-result CONTINUE \
  --heartbeat-timeout 300 \
  --notification-target-arn $TOPIC_ARN

# Create lifecycle hook for instance termination
aws autoscaling put-lifecycle-hook \
  --lifecycle-hook-name instance-terminating-hook \
  --auto-scaling-group-name cloudops-lab7-asg \
  --lifecycle-transition autoscaling:EC2_INSTANCE_TERMINATING \
  --default-result CONTINUE \
  --heartbeat-timeout 300 \
  --notification-target-arn $TOPIC_ARN

# View lifecycle hooks
aws autoscaling describe-lifecycle-hooks \
  --auto-scaling-group-name cloudops-lab7-asg \
  --output table
```

**Lifecycle States:**
1. **Pending** → **Pending:Wait** (lifecycle hook) → **Pending:Proceed** → **InService**
2. **Terminating** → **Terminating:Wait** (lifecycle hook) → **Terminating:Proceed** → **Terminated**

**Use Cases:**
- **Launching:** Install software, register with service discovery, warm up caches
- **Terminating:** Drain connections, backup logs, deregister from service discovery

**Exam Tip:** If your custom action completes before the timeout, use `complete-lifecycle-action` to proceed immediately. Otherwise, the instance waits for the heartbeat timeout.

### Step 1.13: Configure Instance Refresh for Rolling Updates

Instance refresh performs a rolling update of all instances in the ASG.

```bash
# Start an instance refresh
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name cloudops-lab7-asg \
  --preferences '{
    "MinHealthyPercentage": 80,
    "InstanceWarmup": 120,
    "CheckpointPercentages": [50, 100],
    "CheckpointDelay": 300
  }'

# Monitor instance refresh progress
aws autoscaling describe-instance-refreshes \
  --auto-scaling-group-name cloudops-lab7-asg \
  --output table
```

**Instance Refresh Parameters:**
- **MinHealthyPercentage:** Minimum % of instances that must remain healthy during refresh
- **InstanceWarmup:** Time to wait after instance replacement before moving to next
- **CheckpointPercentages:** Pause at these percentages for validation
- **CheckpointDelay:** Seconds to wait at each checkpoint

**Exam Tip:** Instance refresh is the recommended way to update instances after changing a launch template. It's better than manually terminating instances.

---

## Part 2: Elastic Load Balancing

### Step 2.1: Create Application Load Balancer

```bash
# Create ALB
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name cloudops-lab7-alb \
  --subnets $SUBNET_1 $SUBNET_2 \
  --security-groups $SG_ID \
  --scheme internet-facing \
  --type application \
  --ip-address-type ipv4 \
  --tags Key=Name,Value=CloudOps-Lab7-ALB Key=Lab,Value=7 \
  --output text --query 'LoadBalancers[0].LoadBalancerArn')

echo "ALB ARN: $ALB_ARN"

# Get ALB DNS name
ALB_DNS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].DNSName' \
  --output text)

echo "ALB DNS Name: $ALB_DNS"
echo "Access your application at: http://$ALB_DNS"
```

### Step 2.2: Create Target Group with Health Check Configuration

```bash
# Create target group
TG_ARN=$(aws elbv2 create-target-group \
  --name cloudops-lab7-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id $VPC_ID \
  --health-check-enabled \
  --health-check-protocol HTTP \
  --health-check-path /health.html \
  --health-check-interval-seconds 30 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --matcher HttpCode=200 \
  --output text --query 'TargetGroups[0].TargetGroupArn')

echo "Target Group ARN: $TG_ARN"
```

**Health Check Parameters Explained:**
- **health-check-path:** URL path to check (should return 200 OK when healthy)
- **health-check-interval-seconds:** Time between health checks (5-300 seconds)
- **health-check-timeout-seconds:** Time to wait for response (2-120 seconds)
- **healthy-threshold-count:** Consecutive successful checks to mark healthy
- **unhealthy-threshold-count:** Consecutive failed checks to mark unhealthy

**Exam Formula:**
- Time to mark unhealthy = interval × unhealthy-threshold (30s × 3 = 90s)
- Time to mark healthy = interval × healthy-threshold (30s × 2 = 60s)

### Step 2.3: Configure Connection Draining (Deregistration Delay)

```bash
# Set deregistration delay to 60 seconds
aws elbv2 modify-target-group-attributes \
  --target-group-arn $TG_ARN \
  --attributes \
    Key=deregistration_delay.timeout_seconds,Value=60 \
    Key=deregistration_delay.connection_termination.enabled,Value=true

echo "Deregistration delay set to 60 seconds"
```

**Exam Tip:** Deregistration delay (connection draining) allows in-flight requests to complete before removing an instance. Range: 0-3600 seconds. Default: 300 seconds.

### Step 2.4: Configure Sticky Sessions

```bash
# Enable sticky sessions (session affinity)
aws elbv2 modify-target-group-attributes \
  --target-group-arn $TG_ARN \
  --attributes \
    Key=stickiness.enabled,Value=true \
    Key=stickiness.type,Value=lb_cookie \
    Key=stickiness.lb_cookie.duration_seconds,Value=3600

echo "Sticky sessions enabled (1 hour duration)"
```

**Stickiness Types:**
- **lb_cookie:** ALB-generated cookie (AWSALB)
- **app_cookie:** Application-generated cookie
- **duration_seconds:** Cookie lifetime (1 second - 7 days)

**Exam Tip:** Sticky sessions can lead to uneven load distribution. Use only when necessary (e.g., stateful applications without shared session storage).

### Step 2.5: Enable Cross-Zone Load Balancing

```bash
# Enable cross-zone load balancing
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn $ALB_ARN \
  --attributes Key=load_balancing.cross_zone.enabled,Value=true

echo "Cross-zone load balancing enabled"
```

**Cross-Zone Load Balancing:**
- **Enabled:** Distributes traffic evenly across all registered instances in all AZs
- **Disabled:** Distributes traffic evenly only within each AZ

**Exam Tip:** For ALB, cross-zone load balancing is always enabled and cannot be disabled. For NLB, it's disabled by default and may incur inter-AZ data transfer charges when enabled.

### Step 2.6: Enable ALB Access Logs to S3

```bash
# Create S3 bucket for ALB logs
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION=$(aws configure get region)
BUCKET_NAME="cloudops-lab7-alb-logs-${ACCOUNT_ID}"

aws s3 mb s3://$BUCKET_NAME --region $REGION

# Get the ELB account ID for your region (this is AWS's account for ALB logging)
# For us-east-1, the ELB account ID is 127311923021
# For other regions, see: https://docs.aws.amazon.com/elasticloadbalancing/latest/application/enable-access-logging.html
case $REGION in
  us-east-1) ELB_ACCOUNT=127311923021 ;;
  us-east-2) ELB_ACCOUNT=033677994240 ;;
  us-west-1) ELB_ACCOUNT=027434742980 ;;
  us-west-2) ELB_ACCOUNT=797873946194 ;;
  eu-west-1) ELB_ACCOUNT=156460612806 ;;
  eu-central-1) ELB_ACCOUNT=054676820928 ;;
  ap-southeast-1) ELB_ACCOUNT=114774131450 ;;
  ap-northeast-1) ELB_ACCOUNT=582318560864 ;;
  *) ELB_ACCOUNT=127311923021 ;;  # Default to us-east-1
esac

# Create bucket policy
cat > /tmp/alb-logs-bucket-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::${ELB_ACCOUNT}:root"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::${BUCKET_NAME}/*"
    }
  ]
}
EOF

aws s3api put-bucket-policy \
  --bucket $BUCKET_NAME \
  --policy file:///tmp/alb-logs-bucket-policy.json

# Enable access logs on ALB
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn $ALB_ARN \
  --attributes \
    Key=access_logs.s3.enabled,Value=true \
    Key=access_logs.s3.bucket,Value=$BUCKET_NAME \
    Key=access_logs.s3.prefix,Value=alb-logs

echo "ALB access logs enabled. Logs will be stored in s3://$BUCKET_NAME/alb-logs/"
```

**Exam Tip:** ALB access logs are stored in S3 and contain detailed information about requests. They're useful for security analysis, troubleshooting, and compliance.

### Step 2.7: Create Listener and Link to Target Group

```bash
# Create HTTP listener
LISTENER_ARN=$(aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN \
  --output text --query 'Listeners[0].ListenerArn')

echo "Listener ARN: $LISTENER_ARN"
```

### Step 2.8: Attach Auto Scaling Group to Target Group

```bash
# Attach ASG to target group
aws autoscaling attach-load-balancer-target-groups \
  --auto-scaling-group-name cloudops-lab7-asg \
  --target-group-arns $TG_ARN

echo "ASG attached to target group. Waiting for instances to register..."
sleep 30

# Verify targets are registered
aws elbv2 describe-target-health \
  --target-group-arn $TG_ARN \
  --output table
```

### Step 2.9: NLB vs ALB vs Gateway Load Balancer Comparison

| Feature | ALB (Layer 7) | NLB (Layer 4) | Gateway Load Balancer |
|---------|---------------|---------------|----------------------|
| **Protocol** | HTTP, HTTPS, gRPC | TCP, UDP, TLS | IP protocol |
| **Use Case** | Web applications, microservices | High-performance, low-latency apps | Virtual appliances (firewalls, IDS/IPS) |
| **Routing** | Path, host, header, query string | Connection-based | Flow-based |
| **Target Types** | Instance, IP, Lambda | Instance, IP, ALB | Instance, IP |
| **Preservation** | X-Forwarded-For header | Source IP preserved | Source/dest IP preserved |
| **Health Checks** | HTTP/HTTPS | TCP, HTTP, HTTPS | TCP, HTTP, HTTPS |
| **Sticky Sessions** | Yes (cookie-based) | Yes (flow hash) | N/A |
| **SSL/TLS** | Termination & passthrough | Passthrough (TLS listener) | N/A |
| **WebSocket** | Yes | Yes | No |
| **Static IP** | No (DNS only) | Yes (Elastic IP per AZ) | Yes |
| **Cross-Zone LB** | Always enabled | Optional (extra charges) | Required |
| **Pricing** | Per hour + LCU | Per hour + NLCU | Per hour + GLCU |

**Exam Scenarios:**
- **Use ALB when:** HTTP/HTTPS routing, microservices, Lambda targets, path-based routing
- **Use NLB when:** Ultra-low latency required, static IP needed, millions of requests/sec, non-HTTP protocols
- **Use GWLB when:** Deploying 3rd-party network appliances (Palo Alto, Fortinet, Check Point)

---

## Part 3: Route 53 Health Checks & Failover

### Step 3.1: Create Health Checks

```bash
# Health check for ALB endpoint
HEALTH_CHECK_ID=$(aws route53 create-health-check \
  --caller-reference $(date +%s) \
  --health-check-config \
    Type=HTTPS,ResourcePath=/health.html,FullyQualifiedDomainName=$ALB_DNS,Port=443,RequestInterval=30,FailureThreshold=3 \
  --query 'HealthCheck.Id' \
  --output text)

echo "Health Check ID: $HEALTH_CHECK_ID"

# Add tags to health check
aws route53 change-tags-for-resource \
  --resource-type healthcheck \
  --resource-id $HEALTH_CHECK_ID \
  --add-tags Key=Name,Value=CloudOps-Lab7-ALB-HealthCheck

# View health check status
aws route53 get-health-check-status \
  --health-check-id $HEALTH_CHECK_ID \
  --output table
```

**Health Check Types:**
- **HTTP/HTTPS:** Checks for 200/300 status code
- **TCP:** Checks if connection succeeds
- **Calculated:** Combines multiple health checks with AND/OR logic
- **CloudWatch Alarm:** Based on CloudWatch alarm state

**Exam Tip:** Route 53 health checkers are located globally. At least 18% of checkers must report healthy for the health check to pass (for RequestInterval=30, this means 3+ out of ~15 checkers).

### Step 3.2: Create Calculated Health Check

Calculated health checks allow you to create complex health check logic.

```bash
# For demo purposes, create a secondary health check
HEALTH_CHECK_ID_2=$(aws route53 create-health-check \
  --caller-reference $(date +%s)-secondary \
  --health-check-config \
    Type=TCP,FullyQualifiedDomainName=$ALB_DNS,Port=80,RequestInterval=30,FailureThreshold=3 \
  --query 'HealthCheck.Id' \
  --output text)

# Create calculated health check (requires both checks to pass)
CALCULATED_HC_ID=$(aws route53 create-health-check \
  --caller-reference $(date +%s)-calculated \
  --health-check-config \
    Type=CALCULATED,ChildHealthChecks=$HEALTH_CHECK_ID,$HEALTH_CHECK_ID_2,HealthThreshold=2 \
  --query 'HealthCheck.Id' \
  --output text)

echo "Calculated Health Check ID: $CALCULATED_HC_ID"
```

**Calculated Health Check Logic:**
- **HealthThreshold:** Number of child health checks that must pass
- If HealthThreshold=2 and you have 2 children, this is an AND condition
- If HealthThreshold=1 and you have 2 children, this is an OR condition

### Step 3.3: Configure Failover Routing Policy

**Note:** This requires a registered domain in Route 53. The following shows the structure:

```bash
# Example: Create failover records (requires hosted zone)
# This is a demonstration of the command structure

# Primary record (points to ALB)
cat > /tmp/primary-record.json <<EOF
{
  "Changes": [
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "SetIdentifier": "Primary",
        "Failover": "PRIMARY",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "$ALB_DNS",
          "EvaluateTargetHealth": true
        },
        "HealthCheckId": "$HEALTH_CHECK_ID"
      }
    }
  ]
}
EOF

# Secondary record (points to backup)
cat > /tmp/secondary-record.json <<EOF
{
  "Changes": [
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "SetIdentifier": "Secondary",
        "Failover": "SECONDARY",
        "ResourceRecords": [
          {
            "Value": "192.0.2.1"
          }
        ],
        "TTL": 60
      }
    }
  ]
}
EOF

# If you have a hosted zone, apply these changes:
# HOSTED_ZONE_ID="YOUR_HOSTED_ZONE_ID"
# aws route53 change-resource-record-sets --hosted-zone-id $HOSTED_ZONE_ID --change-batch file:///tmp/primary-record.json
# aws route53 change-resource-record-sets --hosted-zone-id $HOSTED_ZONE_ID --change-batch file:///tmp/secondary-record.json

echo "Failover routing configured (example shown - requires domain)"
```

### Step 3.4: Active-Passive vs Active-Active Patterns

**Active-Passive Failover:**
- Primary resource serves all traffic
- Secondary resource is on standby
- Uses Route 53 failover routing policy
- Lower cost but underutilized standby

**Active-Active Failover:**
- All resources serve traffic simultaneously
- Uses weighted, latency-based, or geolocation routing
- Better resource utilization
- Higher cost but better performance

**Exam Tip:**
- Active-Passive = Failover routing policy
- Active-Active = Weighted, latency, geolocation, or geoproximity routing policies

---

## Part 4: Testing Resilience

### Step 4.1: Test Auto Scaling Group Replaces Failed Instances

```bash
# Get current instances in ASG
INSTANCE_IDS=$(aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names cloudops-lab7-asg \
  --query "AutoScalingGroups[0].Instances[*].InstanceId" \
  --output text)

echo "Current instances: $INSTANCE_IDS"

# Terminate one instance to simulate failure
FIRST_INSTANCE=$(echo $INSTANCE_IDS | awk '{print $1}')
echo "Terminating instance: $FIRST_INSTANCE"

aws ec2 terminate-instances --instance-ids $FIRST_INSTANCE

echo "Instance terminated. Waiting for ASG to replace it..."
sleep 30

# Watch ASG launch a replacement
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names cloudops-lab7-asg \
  --query "AutoScalingGroups[0].[DesiredCapacity,Instances[*].[InstanceId,LifecycleState,HealthStatus]]" \
  --output table

echo "ASG should launch a new instance to maintain desired capacity"
```

### Step 4.2: Test Load Balancer Health Checks

```bash
# Get a target instance ID
TARGET_INSTANCE=$(aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names cloudops-lab7-asg \
  --query "AutoScalingGroups[0].Instances[0].InstanceId" \
  --output text)

echo "Testing instance: $TARGET_INSTANCE"

# SSH to the instance and stop Apache to simulate application failure
# (You'll need to do this manually with your key pair)
echo "To test health checks, SSH to the instance and run:"
echo "  sudo systemctl stop httpd"
echo ""
echo "SSH command (replace KEY_FILE with your key):"
INSTANCE_IP=$(aws ec2 describe-instances \
  --instance-ids $TARGET_INSTANCE \
  --query "Reservations[0].Instances[0].PublicIpAddress" \
  --output text)
echo "  ssh -i KEY_FILE ec2-user@$INSTANCE_IP"
echo ""

echo "After stopping Apache, watch the target health:"
echo "  aws elbv2 describe-target-health --target-group-arn $TG_ARN --output table"
echo ""
echo "The instance should become unhealthy after ~90 seconds (interval × unhealthy-threshold = 30s × 3)"
echo "The ASG will then terminate the unhealthy instance and launch a replacement"
```

### Step 4.3: Test CPU-Based Auto Scaling

```bash
# Get an instance to stress test
TEST_INSTANCE=$(aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names cloudops-lab7-asg \
  --query "AutoScalingGroups[0].Instances[0].InstanceId" \
  --output text)

TEST_INSTANCE_IP=$(aws ec2 describe-instances \
  --instance-ids $TEST_INSTANCE \
  --query "Reservations[0].Instances[0].PublicIpAddress" \
  --output text)

echo "To test auto scaling, SSH to instance $TEST_INSTANCE:"
echo "  ssh -i KEY_FILE ec2-user@$TEST_INSTANCE_IP"
echo ""
echo "Then run this command to generate CPU load:"
echo "  stress --cpu 2 --timeout 600s"
echo ""
echo "Monitor scaling activity:"
echo "  aws autoscaling describe-scaling-activities --auto-scaling-group-name cloudops-lab7-asg --max-records 10 --output table"
echo ""
echo "Watch CloudWatch metrics:"
echo "  aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name CPUUtilization --dimensions Name=AutoScalingGroupName,Value=cloudops-lab7-asg --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 60 --statistics Average"
```

### Step 4.4: Simulate Availability Zone Failure

```bash
# Get instances in each AZ
echo "Instances by Availability Zone:"
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names cloudops-lab7-asg \
  --query "AutoScalingGroups[0].Instances[*].[InstanceId,AvailabilityZone,HealthStatus]" \
  --output table

# Get first AZ
FIRST_AZ=$(aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names cloudops-lab7-asg \
  --query "AutoScalingGroups[0].Instances[0].AvailabilityZone" \
  --output text)

# Get all instances in first AZ
INSTANCES_IN_AZ=$(aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names cloudops-lab7-asg \
  --query "AutoScalingGroups[0].Instances[?AvailabilityZone=='$FIRST_AZ'].InstanceId" \
  --output text)

echo "Simulating $FIRST_AZ failure by stopping instances: $INSTANCES_IN_AZ"
echo ""
echo "To simulate AZ failure, run:"
echo "  aws ec2 stop-instances --instance-ids $INSTANCES_IN_AZ"
echo ""
echo "The ALB will detect unhealthy targets and route traffic to the remaining AZ"
echo "The ASG will launch replacement instances (distributed across AZs)"
echo ""
echo "Monitor target health:"
echo "  aws elbv2 describe-target-health --target-group-arn $TG_ARN --output table"
```

### Step 4.5: Verify Load Balancer Routing

```bash
# Make multiple requests to see different instances responding
echo "Testing load distribution across instances:"
echo "Making 10 requests to $ALB_DNS..."
echo ""

for i in {1..10}; do
  curl -s http://$ALB_DNS | grep "Instance ID:" | sed 's/<[^>]*>//g'
  sleep 1
done

echo ""
echo "You should see requests distributed across different instance IDs"
```

---

## Part 5: Review Questions

### Question 1
You have an Auto Scaling group with a target tracking policy maintaining 60% CPU utilization. The ASG has min=2, max=10, desired=4. Currently, CPU is at 75%. What happens next?

<details>
<summary>Click to reveal answer</summary>

**Answer:** The ASG will scale out by launching additional instances.

**Explanation:** Target tracking policies continuously adjust capacity to keep the metric close to the target value. When CPU exceeds 60%, CloudWatch alarms trigger scale-out actions. The ASG will launch instances incrementally until CPU returns to ~60% or max capacity (10) is reached.

**Key Points:**
- Target tracking automatically creates CloudWatch alarms
- It uses a proportional scaling algorithm
- Scale-out is aggressive, scale-in is conservative (prevents flapping)
- Min/max values act as guardrails

**Exam Reference:** SOA-C03 Domain 2.2 - Implement scaling policies
</details>

---

### Question 2
An ALB health check is configured with interval=30s, timeout=5s, healthy-threshold=2, unhealthy-threshold=3. How long until a healthy instance is marked unhealthy after it fails health checks?

<details>
<summary>Click to reveal answer</summary>

**Answer:** 90 seconds

**Explanation:**
- Time to mark unhealthy = interval × unhealthy-threshold
- 30 seconds × 3 = 90 seconds

**Timeline:**
- t=0s: Instance is healthy
- t=30s: First failed health check (1/3)
- t=60s: Second failed health check (2/3)
- t=90s: Third failed health check (3/3) → Instance marked unhealthy

**Note:** The timeout value (5s) determines how long to wait for a response but doesn't affect the calculation of when an instance is marked unhealthy.

**Exam Reference:** SOA-C03 Domain 2.1 - Configure health checks
</details>

---

### Question 3
What is the difference between the health check grace period and the default cooldown period in Auto Scaling?

<details>
<summary>Click to reveal answer</summary>

**Answer:**

**Health Check Grace Period (default: 300s):**
- Time ASG waits before checking the health of a newly launched instance
- Allows instance time to boot up, run user data scripts, and pass initial health checks
- During grace period, failing health checks don't trigger termination
- Applies to: EC2 health checks and ELB health checks

**Default Cooldown Period (default: 300s):**
- Time to wait between scaling activities
- Prevents rapid scaling (flapping)
- Applies only to simple scaling policies (NOT target tracking or step scaling)
- Allows metrics to stabilize before evaluating scaling again

**Key Difference:** Grace period protects new instances from termination; cooldown prevents rapid scaling actions.

**Exam Reference:** SOA-C03 Domain 2.2 - Configure Auto Scaling parameters
</details>

---

### Question 4
You need to perform maintenance on instances in an ASG without triggering a scale-in event. What should you do?

<details>
<summary>Click to reveal answer</summary>

**Answer:** Use the Standby state.

**Commands:**
```bash
# Put instance in standby
aws autoscaling enter-standby \
  --instance-ids i-1234567890abcdef0 \
  --auto-scaling-group-name my-asg \
  --should-decrement-desired-capacity

# After maintenance, return to service
aws autoscaling exit-standby \
  --instance-ids i-1234567890abcdef0 \
  --auto-scaling-group-name my-asg
```

**Explanation:**
- Standby state temporarily removes instance from ASG without terminating it
- Instance stops receiving traffic from load balancer
- If `--should-decrement-desired-capacity` is used, ASG launches a replacement
- Instance continues running and can be returned to service later
- Perfect for: patching, troubleshooting, or debugging

**Alternatives:**
- **Suspend processes:** Suspends all ASG activities (Launch, Terminate, HealthCheck, etc.)
- **Detach instance:** Permanently removes instance from ASG

**Exam Reference:** SOA-C03 Domain 1.2 - Perform instance management tasks
</details>

---

### Question 5
Your application requires instances to drain connections for 120 seconds before termination. The ALB deregistration delay is set to 60 seconds. What's the issue and how do you fix it?

<details>
<summary>Click to reveal answer</summary>

**Answer:** The deregistration delay is too short. Increase it to at least 120 seconds.

**Command:**
```bash
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --attributes Key=deregistration_delay.timeout_seconds,Value=120
```

**Explanation:**
- Deregistration delay (connection draining) determines how long the load balancer waits for in-flight requests to complete before deregistering an instance
- If set too short, active connections are forcibly closed, leading to errors
- Range: 0-3600 seconds (0 = immediate, 3600 = 1 hour)
- Default: 300 seconds (5 minutes)

**Timeline when instance is terminated:**
1. ASG initiates termination
2. ALB marks instance as "draining"
3. ALB stops sending new requests to instance
4. ALB waits `deregistration_delay` seconds for active requests to complete
5. ALB deregisters instance
6. ASG terminates instance

**Best Practice:** Set deregistration delay to match or exceed your longest request timeout.

**Exam Reference:** SOA-C03 Domain 2.1 - Configure load balancing
</details>

---

### Question 6
When should you use NLB instead of ALB?

<details>
<summary>Click to reveal answer</summary>

**Answer:** Use NLB when you need:

1. **Static IP addresses:** NLB supports Elastic IPs (one per AZ)
2. **Ultra-low latency:** NLB operates at Layer 4, has lower latency than ALB
3. **High throughput:** Millions of requests per second
4. **Non-HTTP protocols:** TCP, UDP, TLS
5. **Source IP preservation:** NLB preserves client source IP without X-Forwarded-For
6. **Long-lived connections:** WebSocket, database connections, gaming

**ALB is better for:**
- HTTP/HTTPS applications
- Path-based routing (/api, /images)
- Host-based routing (api.example.com, www.example.com)
- Lambda targets
- Microservices architectures
- Content-based routing (headers, query strings)

**Decision Tree:**
- Need Layer 7 routing/features? → ALB
- Need static IPs? → NLB
- Need ultra-low latency? → NLB
- HTTP/HTTPS only? → ALB
- TCP/UDP? → NLB

**Exam Reference:** SOA-C03 Domain 2.1 - Select appropriate load balancer type
</details>

---

### Question 7
You have an ASG with scheduled scaling that sets min=5, desired=10 at 9 AM daily. At 8:30 AM, a target tracking policy scales the ASG to 15 instances due to high load. What happens at 9 AM?

<details>
<summary>Click to reveal answer</summary>

**Answer:** At 9 AM, the ASG will scale down to 10 instances (desired capacity).

**Explanation:**
- Scheduled actions override the current min, max, and desired capacity values
- At 9 AM, the scheduled action executes: min=5, desired=10
- Even though there are 15 instances running, ASG will terminate 5 to meet desired=10
- After 9 AM, target tracking can still scale between the new min (5) and max values

**Scheduled Scaling Priority:**
- Scheduled actions always execute at the specified time
- They don't consider current load or other scaling policies
- After scheduled action executes, dynamic policies (target tracking, step scaling) resume normal operation within the new min/max boundaries

**Best Practice:**
- Use scheduled scaling for predictable patterns (business hours)
- Ensure scheduled min/max values are appropriate for typical load at that time
- Combine with dynamic scaling for handling unexpected spikes

**Exam Reference:** SOA-C03 Domain 2.2 - Implement scheduled scaling
</details>

---

### Question 8
What is the difference between cross-zone load balancing for ALB and NLB?

<details>
<summary>Click to reveal answer</summary>

**Answer:**

**ALB (Application Load Balancer):**
- Cross-zone load balancing is **always enabled**
- Cannot be disabled
- No additional charges for inter-AZ data transfer
- Distributes traffic evenly across all registered targets in all enabled AZs

**NLB (Network Load Balancer):**
- Cross-zone load balancing is **disabled by default**
- Can be enabled at load balancer creation or later
- When enabled, **inter-AZ data transfer charges apply**
- When disabled, traffic is distributed only among targets in the same AZ as the NLB node

**Example Scenario:**
- ALB with 2 AZs: AZ-A has 4 instances, AZ-B has 2 instances
  - Each instance receives ~16.67% of traffic (1/6)

- NLB with cross-zone disabled:
  - 50% of traffic goes to AZ-A → split among 4 instances (12.5% each)
  - 50% of traffic goes to AZ-B → split among 2 instances (25% each)
  - Result: Uneven distribution

**Exam Tip:** Remember the defaults:
- ALB: Cross-zone always ON, no charges
- NLB: Cross-zone OFF by default, charges apply when enabled

**Exam Reference:** SOA-C03 Domain 2.1 - Configure load balancer attributes
</details>

---

### Question 9
You have a lifecycle hook with a timeout of 300 seconds. Your custom action completes in 60 seconds. How do you make the instance proceed immediately without waiting for the full timeout?

<details>
<summary>Click to reveal answer</summary>

**Answer:** Use the `complete-lifecycle-action` command.

**Command:**
```bash
aws autoscaling complete-lifecycle-action \
  --lifecycle-action-result CONTINUE \
  --lifecycle-hook-name my-hook \
  --auto-scaling-group-name my-asg \
  --lifecycle-action-token $TOKEN
```

**Explanation:**
- Lifecycle hooks pause instance launch/termination for custom actions
- Default behavior: Instance waits for the full heartbeat timeout
- `complete-lifecycle-action` signals early completion
- `--lifecycle-action-result` can be:
  - **CONTINUE:** Proceed with launch/termination
  - **ABANDON:** Cancel the action (instance is terminated)

**Getting the Token:**
- Token is provided in SNS/SQS notification from lifecycle hook
- Or describe the ASG's lifecycle activities:
```bash
aws autoscaling describe-lifecycle-hooks \
  --auto-scaling-group-name my-asg
```

**Alternative - Extend Timeout:**
```bash
aws autoscaling record-lifecycle-action-heartbeat \
  --lifecycle-action-token $TOKEN \
  --lifecycle-hook-name my-hook \
  --auto-scaling-group-name my-asg
```
This extends the timeout by another heartbeat period.

**Use Cases:**
- **Continue early:** Software installation completed
- **Abandon:** Validation failed, instance shouldn't be added

**Exam Reference:** SOA-C03 Domain 2.2 - Implement lifecycle hooks
</details>

---

### Question 10
Route 53 health checks are failing for your ALB even though the application is responding correctly. What could be the issue?

<details>
<summary>Click to reveal answer</summary>

**Answer:** Security group rules may be blocking Route 53 health checker IP ranges.

**Common Issues:**

1. **Security Group Blocking Route 53:**
   - Route 53 health checkers come from specific IP ranges
   - If ALB security group doesn't allow these IPs, health checks fail
   - Solution: Allow the Route 53 health checker IP ranges

2. **Wrong Protocol/Port:**
   - Health check configured for HTTPS but ALB only has HTTP listener
   - Solution: Match health check protocol to ALB listener

3. **Invalid Health Check Path:**
   - Path doesn't exist or returns non-200 status
   - Solution: Ensure /health or configured path returns 200 OK

4. **String Matching Misconfigured:**
   - Optional string match doesn't appear in response
   - Solution: Remove string match or fix response content

5. **Request Interval Too Aggressive:**
   - Threshold set too strict for actual performance
   - Solution: Increase FailureThreshold or RequestInterval

**Route 53 Health Checker IP Ranges:**
```bash
# Route 53 health checks come from these ranges (as of 2025):
# - Use Route 53 IP address ranges published by AWS
# - Or allow all HTTPS/HTTP from 0.0.0.0/0 for health checks

# Better approach: Use ALB alias records with EvaluateTargetHealth
# This uses internal health checks, avoiding external checker issues
```

**Best Practice for ALB:**
```json
{
  "AliasTarget": {
    "HostedZoneId": "Z35SXDOTRQ7X7K",
    "DNSName": "my-alb-123456.us-east-1.elb.amazonaws.com",
    "EvaluateTargetHealth": true
  }
}
```

With `EvaluateTargetHealth: true`, Route 53 uses the ALB's internal health checks instead of external health checker IPs.

**Exam Reference:** SOA-C03 Domain 2.3 - Troubleshoot network connectivity issues
</details>

---

## Cleanup Steps

To avoid ongoing charges, delete all resources created in this lab:

```bash
# Step 1: Detach ASG from target group and delete scaling policies
aws autoscaling detach-load-balancer-target-groups \
  --auto-scaling-group-name cloudops-lab7-asg \
  --target-group-arns $TG_ARN

# Delete all scaling policies
for policy in $(aws autoscaling describe-policies \
  --auto-scaling-group-name cloudops-lab7-asg \
  --query "ScalingPolicies[*].PolicyName" --output text); do
  aws autoscaling delete-policy \
    --auto-scaling-group-name cloudops-lab7-asg \
    --policy-name $policy
done

# Delete scheduled actions
for action in $(aws autoscaling describe-scheduled-actions \
  --auto-scaling-group-name cloudops-lab7-asg \
  --query "ScheduledUpdateGroupActions[*].ScheduledActionName" --output text); do
  aws autoscaling delete-scheduled-action \
    --auto-scaling-group-name cloudops-lab7-asg \
    --scheduled-action-name $action
done

# Delete lifecycle hooks
aws autoscaling delete-lifecycle-hook \
  --lifecycle-hook-name instance-launching-hook \
  --auto-scaling-group-name cloudops-lab7-asg

aws autoscaling delete-lifecycle-hook \
  --lifecycle-hook-name instance-terminating-hook \
  --auto-scaling-group-name cloudops-lab7-asg

# Step 2: Delete Auto Scaling Group (sets desired/min/max to 0, then deletes)
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name cloudops-lab7-asg \
  --min-size 0 \
  --max-size 0 \
  --desired-capacity 0

# Wait for instances to terminate
echo "Waiting for ASG instances to terminate..."
sleep 60

aws autoscaling delete-auto-scaling-group \
  --auto-scaling-group-name cloudops-lab7-asg \
  --force-delete

# Step 3: Delete Load Balancer and Target Group
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN

# Wait for ALB to delete before deleting target group
echo "Waiting for ALB to delete..."
sleep 60

aws elbv2 delete-target-group --target-group-arn $TG_ARN

# Step 4: Delete CloudWatch Alarms
for alarm in $(aws cloudwatch describe-alarms \
  --alarm-name-prefix cloudops-lab7 \
  --query "MetricAlarms[*].AlarmName" --output text); do
  aws cloudwatch delete-alarms --alarm-names $alarm
done

# Step 5: Delete Route 53 Health Checks
aws route53 delete-health-check --health-check-id $HEALTH_CHECK_ID
aws route53 delete-health-check --health-check-id $HEALTH_CHECK_ID_2
aws route53 delete-health-check --health-check-id $CALCULATED_HC_ID

# Step 6: Delete SNS Topic
aws sns delete-topic --topic-arn $TOPIC_ARN

# Step 7: Delete Launch Template
aws ec2 delete-launch-template --launch-template-name cloudops-lab7-lt

# Step 8: Delete Security Group
aws ec2 delete-security-group --group-id $SG_ID

# Step 9: Delete S3 Bucket (ALB logs)
aws s3 rb s3://$BUCKET_NAME --force

# Step 10: Delete IAM Resources
aws iam remove-role-from-instance-profile \
  --instance-profile-name CloudOpsLabEC2Profile \
  --role-name CloudOpsLabEC2Role

aws iam delete-instance-profile \
  --instance-profile-name CloudOpsLabEC2Profile

aws iam detach-role-policy \
  --role-name CloudOpsLabEC2Role \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

aws iam detach-role-policy \
  --role-name CloudOpsLabEC2Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam delete-role --role-name CloudOpsLabEC2Role

echo "Cleanup complete! All Lab 7 resources have been deleted."
```

---

## Additional Exam Tips

### High Availability Principles
1. **Design for failure:** Assume everything fails
2. **Use multiple AZs:** Distribute across at least 2 AZs
3. **Implement health checks:** At multiple layers (ELB, ASG, Route 53)
4. **Automate recovery:** Use Auto Scaling, health checks, and alarms
5. **Test failure scenarios:** Regularly test failover and recovery

### Auto Scaling Best Practices
- Use target tracking for most use cases (simplest and most effective)
- Set appropriate health check grace periods (match your bootstrap time)
- Enable detailed monitoring for faster scaling decisions
- Use lifecycle hooks for graceful instance startup/shutdown
- Implement proper termination policies
- Use instance refresh for rolling updates (better than manual termination)

### Load Balancing Best Practices
- Use ALB for HTTP/HTTPS workloads (better routing, lower cost)
- Use NLB for ultra-low latency or static IPs
- Configure appropriate health check parameters
- Enable access logs for troubleshooting
- Use connection draining (deregistration delay)
- Implement cross-zone load balancing for even distribution

### Route 53 Best Practices
- Use alias records for AWS resources (free queries, better integration)
- Implement health checks for critical endpoints
- Use calculated health checks for complex logic
- Set appropriate TTL values (lower = faster failover, higher = less queries)
- Test failover scenarios regularly

### Common Exam Scenarios
1. **Scenario:** Application needs to scale based on custom metrics (e.g., queue depth)
   - **Solution:** Create custom CloudWatch metric, use target tracking

2. **Scenario:** Need to drain connections before instance termination
   - **Solution:** Configure deregistration delay on target group

3. **Scenario:** Instances failing health checks immediately after launch
   - **Solution:** Increase health check grace period

4. **Scenario:** Need to update software on all ASG instances
   - **Solution:** Update launch template, start instance refresh

5. **Scenario:** Uneven load distribution across AZs
   - **Solution:** Enable cross-zone load balancing

---

## Summary

In this lab, you learned how to:
- Create and configure Auto Scaling Groups with multiple scaling policies
- Implement target tracking, step scaling, simple scaling, scheduled scaling, and predictive scaling
- Configure lifecycle hooks for custom actions during instance launch/termination
- Create and configure Application Load Balancers with health checks
- Configure connection draining, sticky sessions, and cross-zone load balancing
- Set up Route 53 health checks and failover routing
- Test resilience by simulating failures
- Understand the differences between ALB, NLB, and Gateway Load Balancers

These skills are essential for the AWS Certified SysOps Administrator - Associate (SOA-C03) exam, particularly Domain 2: Reliability and Business Continuity (28% of exam).

**Key Takeaways:**
- High availability requires redundancy across multiple AZs
- Auto Scaling automatically maintains application availability and capacity
- Load balancers distribute traffic and detect unhealthy instances
- Route 53 health checks provide DNS-level failover
- Testing failure scenarios is critical to validate resilience
- Monitoring and metrics drive scaling decisions

Good luck with your exam preparation!
