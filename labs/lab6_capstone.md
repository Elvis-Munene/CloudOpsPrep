# Lab 6: Capstone - Building a Fully Managed, Monitored, and Automated AWS Environment

## Estimated Time: 2 hours
## Objectives
- Deploy a production-ready web application stack using CloudFormation
- Configure comprehensive monitoring and alerting
- Enforce compliance with AWS Config
- Automate backups with AWS Backup and DLM
- Implement security best practices
- Validate everything works end-to-end

## Scenario

You are a SysOps administrator at a company launching a new web application. You must deploy the infrastructure as code, set up monitoring, enforce compliance, automate backups, and harden security. This lab ties together everything from Labs 1-5.

---

## Phase 1: Deploy Infrastructure with CloudFormation

Save as `capstone-stack.yaml`:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Capstone Lab - Production Web Application Stack

Parameters:
  EnvironmentName:
    Type: String
    Default: capstone-prod
  DBPassword:
    Type: String
    NoEcho: true
    MinLength: 8
    Description: RDS master password
  KeyPairName:
    Type: AWS::EC2::KeyPair::KeyName
  LatestAmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2

Resources:
  # ==================== NETWORKING ====================
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-vpc'
        - Key: Environment
          Value: Production
        - Key: Owner
          Value: SysOps

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-igw'

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-public-1'

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.2.0/24
      AvailabilityZone: !Select [1, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-public-2'

  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.3.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-private-1'

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.4.0/24
      AvailabilityZone: !Select [1, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-private-2'

  NATGatewayEIP:
    Type: AWS::EC2::EIP
    DependsOn: AttachGateway

  NATGateway:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NATGatewayEIP.AllocationId
      SubnetId: !Ref PublicSubnet1

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachGateway
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PublicSubnet1RouteAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet1
      RouteTableId: !Ref PublicRouteTable

  PublicSubnet2RouteAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet2
      RouteTableId: !Ref PublicRouteTable

  PrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC

  PrivateRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NATGateway

  PrivateSubnet1RouteAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet1
      RouteTableId: !Ref PrivateRouteTable

  PrivateSubnet2RouteAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet2
      RouteTableId: !Ref PrivateRouteTable

  # ==================== SECURITY GROUPS ====================
  ALBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: ALB Security Group
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0

  EC2SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: EC2 instances - only from ALB
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          SourceSecurityGroupId: !Ref ALBSecurityGroup

  RDSSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: RDS - only from EC2
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3306
          ToPort: 3306
          SourceSecurityGroupId: !Ref EC2SecurityGroup

  # ==================== IAM ====================
  EC2InstanceRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
        - arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

  EC2InstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - !Ref EC2InstanceRole

  # ==================== ALB ====================
  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Scheme: internet-facing
      SecurityGroups:
        - !Ref ALBSecurityGroup
      Subnets:
        - !Ref PublicSubnet1
        - !Ref PublicSubnet2
      LoadBalancerAttributes:
        - Key: access_logs.s3.enabled
          Value: 'true'
        - Key: access_logs.s3.bucket
          Value: !Ref LogBucket

  ALBTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Port: 80
      Protocol: HTTP
      VpcId: !Ref VPC
      HealthCheckPath: /
      HealthCheckIntervalSeconds: 30
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 3
      TargetType: instance

  ALBListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref ApplicationLoadBalancer
      Port: 80
      Protocol: HTTP
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref ALBTargetGroup

  # ==================== AUTO SCALING ====================
  LaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateData:
        ImageId: !Ref LatestAmiId
        InstanceType: t3.micro
        KeyName: !Ref KeyPairName
        IamInstanceProfile:
          Arn: !GetAtt EC2InstanceProfile.Arn
        SecurityGroupIds:
          - !Ref EC2SecurityGroup
        UserData:
          Fn::Base64: !Sub |
            #!/bin/bash -xe
            yum update -y
            yum install -y httpd
            systemctl start httpd
            systemctl enable httpd
            INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
            echo "<h1>Hello from $INSTANCE_ID in ${EnvironmentName}</h1>" > /var/www/html/index.html
        TagSpecifications:
          - ResourceType: instance
            Tags:
              - Key: Name
                Value: !Sub '${EnvironmentName}-web'
              - Key: Environment
                Value: Production
              - Key: Owner
                Value: SysOps

  AutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      LaunchTemplate:
        LaunchTemplateId: !Ref LaunchTemplate
        Version: !GetAtt LaunchTemplate.LatestVersionNumber
      MinSize: 2
      MaxSize: 4
      DesiredCapacity: 2
      VPCZoneIdentifier:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2
      TargetGroupARNs:
        - !Ref ALBTargetGroup
      HealthCheckType: ELB
      HealthCheckGracePeriod: 300
      Tags:
        - Key: Environment
          Value: Production
          PropagateAtLaunch: false

  ScalingPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AutoScalingGroupName: !Ref AutoScalingGroup
      PolicyType: TargetTrackingScaling
      TargetTrackingConfiguration:
        PredefinedMetricSpecification:
          PredefinedMetricType: ASGAverageCPUUtilization
        TargetValue: 70

  # ==================== RDS ====================
  DBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: Private subnets for RDS
      SubnetIds:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2

  Database:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Snapshot
    Properties:
      DBInstanceIdentifier: !Sub '${EnvironmentName}-db'
      Engine: mysql
      EngineVersion: '8.0'
      DBInstanceClass: db.t3.micro
      AllocatedStorage: 20
      MasterUsername: admin
      MasterUserPassword: !Ref DBPassword
      DBSubnetGroupName: !Ref DBSubnetGroup
      VPCSecurityGroups:
        - !Ref RDSSecurityGroup
      MultiAZ: true
      StorageEncrypted: true
      BackupRetentionPeriod: 7
      CopyTagsToSnapshot: true
      Tags:
        - Key: Environment
          Value: Production

  # ==================== S3 ====================
  LogBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      VersioningConfiguration:
        Status: Enabled
      LifecycleConfiguration:
        Rules:
          - Id: ArchiveOldLogs
            Status: Enabled
            Transitions:
              - TransitionInDays: 30
                StorageClass: STANDARD_IA
              - TransitionInDays: 90
                StorageClass: GLACIER
            ExpirationInDays: 365

  LogBucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref LogBucket
      PolicyDocument:
        Statement:
          - Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:root'
            Action: 's3:PutObject'
            Resource: !Sub '${LogBucket.Arn}/*'

  # ==================== SNS ====================
  AlertsTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: !Sub '${EnvironmentName}-alerts'
      KmsMasterKeyId: alias/aws/sns

Outputs:
  VPCID:
    Value: !Ref VPC
  ALBEndpoint:
    Value: !GetAtt ApplicationLoadBalancer.DNSName
  RDSEndpoint:
    Value: !GetAtt Database.Endpoint.Address
  LogBucketName:
    Value: !Ref LogBucket
  SNSTopicArn:
    Value: !Ref AlertsTopic
  ASGName:
    Value: !Ref AutoScalingGroup
```

### Deploy

```bash
aws cloudformation create-stack \
  --stack-name capstone-lab \
  --template-body file://capstone-stack.yaml \
  --parameters \
    ParameterKey=DBPassword,ParameterValue=MyS3cur3P@ss \
    ParameterKey=KeyPairName,ParameterValue=my-key-pair \
  --capabilities CAPABILITY_IAM

# Wait for completion (~10-15 min due to RDS Multi-AZ)
aws cloudformation wait stack-create-complete --stack-name capstone-lab

# Get outputs
aws cloudformation describe-stacks \
  --stack-name capstone-lab \
  --query 'Stacks[0].Outputs' --output table
```

---

## Phase 2: Configure Monitoring & Alerting

### Install CloudWatch Agent via SSM

```bash
# Install CloudWatch Agent on all instances
aws ssm send-command \
  --document-name "AWS-ConfigureAWSPackage" \
  --targets "Key=tag:Environment,Values=Production" \
  --parameters '{"action":["Install"],"name":["AmazonCloudWatchAgent"]}'

# Wait for install, then configure with SSM parameter
aws ssm put-parameter \
  --name "AmazonCloudWatch-capstone" \
  --type String \
  --value '{
    "metrics": {
      "metrics_collected": {
        "mem": {"measurement": ["mem_used_percent"]},
        "disk": {"measurement": ["used_percent"], "resources": ["/"]}
      }
    },
    "logs": {
      "logs_collected": {
        "files": {
          "collect_list": [
            {"file_path": "/var/log/httpd/access_log", "log_group_name": "/capstone/httpd/access", "log_stream_name": "{instance_id}"},
            {"file_path": "/var/log/httpd/error_log", "log_group_name": "/capstone/httpd/error", "log_stream_name": "{instance_id}"}
          ]
        }
      }
    }
  }'

aws ssm send-command \
  --document-name "AmazonCloudWatch-ManageAgent" \
  --targets "Key=tag:Environment,Values=Production" \
  --parameters '{"action":["configure"],"mode":["ec2"],"optionalConfigurationSource":["ssm"],"optionalConfigurationLocation":["AmazonCloudWatch-capstone"]}'
```

### Create CloudWatch Alarms

```bash
SNS_ARN=$(aws cloudformation describe-stacks --stack-name capstone-lab \
  --query 'Stacks[0].Outputs[?OutputKey==`SNSTopicArn`].OutputValue' --output text)
ASG_NAME=$(aws cloudformation describe-stacks --stack-name capstone-lab \
  --query 'Stacks[0].Outputs[?OutputKey==`ASGName`].OutputValue' --output text)

# CPU alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "capstone-HighCPU" \
  --metric-name CPUUtilization --namespace AWS/EC2 \
  --statistic Average --period 300 --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 --datapoints-to-alarm 2 \
  --dimensions "Name=AutoScalingGroupName,Value=$ASG_NAME" \
  --alarm-actions "$SNS_ARN" --ok-actions "$SNS_ARN" \
  --treat-missing-data notBreaching

# ALB 5xx errors alarm
ALB_ID=$(aws cloudformation describe-stacks --stack-name capstone-lab \
  --query 'Stacks[0].Outputs[?OutputKey==`ALBEndpoint`].OutputValue' --output text)

aws cloudwatch put-metric-alarm \
  --alarm-name "capstone-ALB-5xx" \
  --metric-name HTTPCode_Target_5XX_Count \
  --namespace AWS/ApplicationELB \
  --statistic Sum --period 300 --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions "$SNS_ARN" \
  --treat-missing-data notBreaching

# RDS free storage alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "capstone-RDS-LowStorage" \
  --metric-name FreeStorageSpace --namespace AWS/RDS \
  --statistic Average --period 300 \
  --threshold 5368709120 \
  --comparison-operator LessThanThreshold \
  --evaluation-periods 1 \
  --dimensions "Name=DBInstanceIdentifier,Value=capstone-prod-db" \
  --alarm-actions "$SNS_ARN"
```

### Create CloudWatch Dashboard

```bash
aws cloudwatch put-dashboard \
  --dashboard-name "Capstone-Production" \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric", "x": 0, "y": 0, "width": 12, "height": 6,
        "properties": {
          "title": "ASG CPU Utilization",
          "metrics": [["AWS/EC2", "CPUUtilization", "AutoScalingGroupName", "REPLACE_ASG_NAME"]],
          "period": 60, "stat": "Average"
        }
      },
      {
        "type": "metric", "x": 12, "y": 0, "width": 12, "height": 6,
        "properties": {
          "title": "ALB Request Count & Response Time",
          "metrics": [
            ["AWS/ApplicationELB", "RequestCount", "LoadBalancer", "REPLACE_ALB_ARN", {"stat": "Sum"}],
            [".", "TargetResponseTime", ".", ".", {"stat": "p99", "yAxis": "right"}]
          ],
          "period": 60
        }
      },
      {
        "type": "alarm", "x": 0, "y": 6, "width": 24, "height": 3,
        "properties": {
          "title": "Alarm Status",
          "alarms": ["arn:aws:cloudwatch:REGION:ACCOUNT:alarm:capstone-HighCPU", "arn:aws:cloudwatch:REGION:ACCOUNT:alarm:capstone-ALB-5xx", "arn:aws:cloudwatch:REGION:ACCOUNT:alarm:capstone-RDS-LowStorage"]
        }
      }
    ]
  }'
```

### Set Up CloudTrail

```bash
aws cloudtrail create-trail \
  --name "capstone-trail" \
  --s3-bucket-name "$(aws cloudformation describe-stacks --stack-name capstone-lab --query 'Stacks[0].Outputs[?OutputKey==`LogBucketName`].OutputValue' --output text)" \
  --s3-key-prefix cloudtrail \
  --is-multi-region-trail \
  --enable-log-file-validation

aws cloudtrail start-logging --name "capstone-trail"
```

### EventBridge Rules

```bash
# Root user login alert
aws events put-rule \
  --name "capstone-root-login" \
  --event-pattern '{
    "source": ["aws.signin"],
    "detail-type": ["AWS Console Sign In via CloudTrail"],
    "detail": {"userIdentity": {"type": ["Root"]}}
  }'

aws events put-targets --rule "capstone-root-login" \
  --targets "Id=sns,Arn=$SNS_ARN"

# Security group change alert
aws events put-rule \
  --name "capstone-sg-change" \
  --event-pattern '{
    "source": ["aws.ec2"],
    "detail-type": ["AWS API Call via CloudTrail"],
    "detail": {
      "eventSource": ["ec2.amazonaws.com"],
      "eventName": ["AuthorizeSecurityGroupIngress", "AuthorizeSecurityGroupEgress", "RevokeSecurityGroupIngress", "RevokeSecurityGroupEgress"]
    }
  }'

aws events put-targets --rule "capstone-sg-change" \
  --targets "Id=sns,Arn=$SNS_ARN"
```

---

## Phase 3: Enforce Compliance with AWS Config

```bash
# Enable Config recorder
aws configservice put-configuration-recorder \
  --configuration-recorder name=capstone-recorder,roleARN=arn:aws:iam::ACCOUNT:role/aws-service-role/config.amazonaws.com/AWSServiceRoleForConfig \
  --recording-group allSupported=true,includeGlobalResourceTypes=true

# Set up delivery channel
aws configservice put-delivery-channel \
  --delivery-channel name=capstone-channel,s3BucketName=LOGBUCKET

# Start recorder
aws configservice start-configuration-recorder --configuration-recorder-name capstone-recorder

# Add Config rules
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "restricted-ssh",
  "Source": {"Owner": "AWS", "SourceIdentifier": "INCOMING_SSH_DISABLED"}
}'

aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "required-tags",
  "Source": {"Owner": "AWS", "SourceIdentifier": "REQUIRED_TAGS"},
  "InputParameters": "{\"tag1Key\": \"Environment\", \"tag2Key\": \"Owner\"}"
}'

aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "rds-multi-az",
  "Source": {"Owner": "AWS", "SourceIdentifier": "RDS_MULTI_AZ_SUPPORT"}
}'

aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "encrypted-volumes",
  "Source": {"Owner": "AWS", "SourceIdentifier": "ENCRYPTED_VOLUMES"}
}'
```

### Auto-Remediation for restricted-ssh

```bash
# Create SSM Automation document for remediation
aws ssm create-document \
  --name "Custom-RevokeSSH" \
  --document-type Automation \
  --document-format YAML \
  --content '
description: Revoke SSH access from 0.0.0.0/0
schemaVersion: "0.3"
parameters:
  SecurityGroupId:
    type: String
mainSteps:
  - name: revokeSSH
    action: aws:executeAwsApi
    inputs:
      Service: ec2
      Api: RevokeSecurityGroupIngress
      GroupId: "{{ SecurityGroupId }}"
      IpPermissions:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          IpRanges:
            - CidrIp: "0.0.0.0/0"
'

# Set up Config auto-remediation
aws configservice put-remediation-configurations \
  --remediation-configurations '[{
    "ConfigRuleName": "restricted-ssh",
    "TargetType": "SSM_DOCUMENT",
    "TargetId": "Custom-RevokeSSH",
    "Parameters": {
      "SecurityGroupId": {
        "ResourceValue": {"Value": "RESOURCE_ID"}
      }
    },
    "Automatic": true,
    "MaximumAutomaticAttempts": 3,
    "RetryAttemptSeconds": 60
  }]'
```

---

## Phase 4: Automate Backups

### AWS Backup Plan

```bash
# Create backup vault
aws backup create-backup-vault --backup-vault-name capstone-vault

# Create backup plan
aws backup create-backup-plan --backup-plan '{
  "BackupPlanName": "capstone-daily-backup",
  "Rules": [
    {
      "RuleName": "DailyBackup",
      "TargetBackupVaultName": "capstone-vault",
      "ScheduleExpression": "cron(0 2 ? * * *)",
      "StartWindowMinutes": 60,
      "CompletionWindowMinutes": 180,
      "Lifecycle": {
        "DeleteAfterDays": 30
      },
      "CopyActions": [
        {
          "DestinationBackupVaultArn": "arn:aws:backup:us-west-2:ACCOUNT:backup-vault:capstone-vault-dr",
          "Lifecycle": {"DeleteAfterDays": 30}
        }
      ]
    }
  ]
}'

# Assign resources by tag
PLAN_ID=$(aws backup list-backup-plans --query 'BackupPlansList[?BackupPlanName==`capstone-daily-backup`].BackupPlanId' --output text)

aws backup create-backup-selection \
  --backup-plan-id $PLAN_ID \
  --backup-selection '{
    "SelectionName": "ProductionResources",
    "IamRoleArn": "arn:aws:iam::ACCOUNT:role/AWSBackupDefaultServiceRole",
    "ListOfTags": [
      {"ConditionType": "STRINGEQUALS", "ConditionKey": "Environment", "ConditionValue": "Production"}
    ]
  }'
```

### DLM Policy for EBS Snapshots

```bash
aws dlm create-lifecycle-policy \
  --description "Capstone EBS snapshots every 12 hours" \
  --state ENABLED \
  --execution-role-arn "arn:aws:iam::ACCOUNT:role/AWSDataLifecycleManagerDefaultRole" \
  --policy-details '{
    "PolicyType": "EBS_SNAPSHOT_MANAGEMENT",
    "ResourceTypes": ["VOLUME"],
    "TargetTags": [{"Key": "Environment", "Value": "Production"}],
    "Schedules": [{
      "Name": "Every12Hours",
      "CreateRule": {"Interval": 12, "IntervalUnit": "HOURS", "Times": ["00:00"]},
      "RetainRule": {"Count": 5},
      "CopyTags": true,
      "CrossRegionCopyRules": [{
        "TargetRegion": "us-west-2",
        "Encrypted": true,
        "RetainRule": {"Interval": 7, "IntervalUnit": "DAYS"}
      }]
    }]
  }'
```

---

## Phase 5: Security Hardening

### Enable GuardDuty

```bash
aws guardduty create-detector --enable --finding-publishing-frequency FIFTEEN_MINUTES
```

### Enable Security Hub

```bash
aws securityhub enable-security-hub \
  --enable-default-standards

# Check enabled standards
aws securityhub get-enabled-standards
```

### Store RDS Password in Secrets Manager

```bash
aws secretsmanager create-secret \
  --name "capstone/prod/rds-password" \
  --description "RDS master password for capstone DB" \
  --secret-string '{"username":"admin","password":"MyS3cur3P@ss","engine":"mysql","host":"REPLACE_RDS_ENDPOINT","port":3306,"dbname":"myapp"}'

# Enable automatic rotation (requires Lambda rotation function)
aws secretsmanager rotate-secret \
  --secret-id "capstone/prod/rds-password" \
  --rotation-lambda-arn "arn:aws:lambda:us-east-1:ACCOUNT:function:SecretsManagerRDSMySQLRotation" \
  --rotation-rules AutomaticallyAfterDays=30
```

### Verify Encryption

```bash
# Check EBS encryption
echo "=== EBS Volumes ==="
aws ec2 describe-volumes \
  --filters "Name=tag:Environment,Values=Production" \
  --query 'Volumes[*].[VolumeId,Encrypted,State]' --output table

# Check RDS encryption
echo "=== RDS Instances ==="
aws rds describe-db-instances \
  --db-instance-identifier capstone-prod-db \
  --query 'DBInstances[0].[DBInstanceIdentifier,StorageEncrypted,KmsKeyId]' --output table

# Check S3 encryption
echo "=== S3 Buckets ==="
aws s3api get-bucket-encryption --bucket LOGBUCKET
```

---

## Phase 6: Validate & Test

### Test 1: ASG Self-Healing

```bash
# Get an instance ID from the ASG
INSTANCE_ID=$(aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names $ASG_NAME \
  --query 'AutoScalingGroups[0].Instances[0].InstanceId' --output text)

# Terminate it
aws ec2 terminate-instances --instance-ids $INSTANCE_ID

# Watch ASG replace it (check every 30 seconds)
watch -n 30 "aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names $ASG_NAME \
  --query 'AutoScalingGroups[0].Instances[*].[InstanceId,HealthStatus,LifecycleState]' --output table"
```

### Test 2: Trigger CPU Alarm

```bash
# Install stress tool and max out CPU
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Environment,Values=Production" \
  --parameters 'commands=["yum install -y stress","stress --cpu 4 --timeout 600"]' \
  --comment "CPU stress test"

# Monitor the alarm state
watch -n 30 "aws cloudwatch describe-alarms \
  --alarm-names capstone-HighCPU \
  --query 'MetricAlarms[0].[AlarmName,StateValue,StateReason]' --output table"
```

### Test 3: Config Compliance

```bash
# Add a non-compliant SSH rule to a security group
SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=*capstone*EC2*" \
  --query 'SecurityGroups[0].GroupId' --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp --port 22 --cidr 0.0.0.0/0

echo "Added SSH 0.0.0.0/0 - Config should detect and auto-remediate..."

# Check Config compliance (may take a few minutes)
sleep 120
aws configservice get-compliance-details-by-config-rule \
  --config-rule-name restricted-ssh \
  --compliance-types NON_COMPLIANT \
  --query 'EvaluationResults[*].[EvaluationResultIdentifier.EvaluationResultQualifier.ResourceId,ComplianceType]'

# Verify auto-remediation removed the rule
aws ec2 describe-security-groups \
  --group-ids $SG_ID \
  --query 'SecurityGroups[0].IpPermissions'
```

### Test 4: Verify Backups

```bash
# Check AWS Backup jobs
aws backup list-backup-jobs \
  --by-backup-vault-name capstone-vault \
  --by-state COMPLETED \
  --query 'BackupJobs[*].[ResourceArn,BackupJobId,State,CompletionDate]' --output table
```

### Test 5: Verify CloudTrail

```bash
# Look up recent events
aws cloudtrail lookup-events \
  --max-results 10 \
  --query 'Events[*].[EventName,Username,EventTime]' --output table
```

---

## Challenge Questions

### Question 1

During the capstone deployment, the CloudFormation stack fails because the RDS instance takes too long to create. The stack rolls back. What should the administrator check or modify?

**A)** Add a `DependsOn` attribute to the RDS resource
**B)** Increase the stack creation timeout
**C)** CloudFormation has no overall timeout; check the specific error in stack events. RDS Multi-AZ can take 15-20 minutes which is normal
**D)** Use a smaller instance class to speed up creation

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** CloudFormation does NOT have an overall stack creation timeout. Individual resources may have timeouts (like `CreationPolicy`), but RDS Multi-AZ provisioning taking 15-20 minutes is normal. Check stack events for the actual error. The stack likely failed for a different reason (wrong parameter, permission issue, etc.).
</details>

---

### Question 2

The ALB is showing healthy targets but users report 504 Gateway Timeout errors. The EC2 instances are running and responding to health checks. What is the MOST likely cause?

**A)** The security group doesn't allow traffic from the ALB
**B)** The application takes longer than the ALB idle timeout (60 seconds default)
**C)** The health check path is wrong
**D)** The instances are in a different AZ than the ALB

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** 504 errors mean the ALB timed out waiting for a response from the target. If health checks pass but real requests time out, the application likely takes longer than the ALB's idle timeout (default 60 seconds). Solution: increase the ALB idle timeout or optimize the application. The security group is working (health checks pass), ruling out A. Health check works, ruling out C. Cross-AZ is default behavior, ruling out D.
</details>

---

### Question 3

After enabling AWS Config auto-remediation for the `restricted-ssh` rule, the administrator notices that a security group keeps getting SSH re-added by an automation script. Config remediates it, then the script adds it back, creating an infinite loop. How should this be resolved?

**A)** Disable the Config rule
**B)** Fix the automation script that's adding the SSH rule, and use a Config rule to detect if it recurs
**C)** Increase the remediation retry delay
**D)** Add an exception for that security group in the Config rule

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** The root cause is the automation script that keeps adding the non-compliant SSH rule. Fix the source of the problem, not the symptom. Disabling the Config rule (A) removes the safety net. Increasing delay (C) doesn't solve the loop. While adding an exception (D) might work, it leaves a non-compliant security group.
</details>

---

### Question 4

A GuardDuty finding shows "UnauthorizedAccess:EC2/TorIPCaller" for one of the capstone EC2 instances. What does this mean and what action should be taken?

**A)** The instance is being used as a Tor exit node; isolate it immediately
**B)** An API call to the EC2 instance was made from a Tor exit node IP; investigate the instance for compromise
**C)** The EC2 instance is accessing Tor websites; this is informational only
**D)** The instance's security group was modified from a Tor IP; review CloudTrail

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** This finding indicates that an AWS API was invoked from a Tor exit node IP address, targeting the EC2 instance. This suggests potential unauthorized access. Steps: (1) Isolate the instance by changing its security group, (2) Check CloudTrail for suspicious API calls, (3) Investigate the instance for malware or unauthorized changes, (4) Rotate any credentials the instance had access to.
</details>

---

### Question 5

The cross-region backup copy to us-west-2 is failing. The backup vault in us-west-2 exists. What is the MOST likely cause?

**A)** Cross-region backup is not supported for EC2 instances
**B)** The KMS key used for encryption is a regional key that doesn't exist in us-west-2
**C)** The IAM role doesn't have permissions in the destination region
**D)** The destination vault must have the same name as the source vault

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** If the source backup is encrypted with a KMS key that doesn't exist in the destination region, the copy fails. Solutions: use an AWS-managed key, use a multi-region KMS key, or create a KMS key in the destination region and specify it in the copy action. Cross-region backup is supported for EC2/EBS (A is wrong). The vaults can have different names (D is wrong).
</details>

---

### Question 6

During a VPC troubleshooting exercise, an EC2 instance in the private subnet can reach the internet (for yum updates) but a newly launched instance in the same subnet cannot. Both have the same security group and IAM role. What should be checked?

**A)** The new instance doesn't have a public IP
**B)** The NAT Gateway may have hit its elastic IP limit
**C)** The new instance may not have the correct route table associated with its subnet
**D)** Check that the new instance was assigned to the correct subnet and that the subnet's route table has a route to the NAT Gateway

<details>
<summary>Show Answer</summary>

**Correct Answer: D**

**Explanation:** Since one instance works and another doesn't in the "same subnet," verify the new instance is actually in the same subnet. If it is, check: (1) route table association (maybe it was changed), (2) NACL rules (could have a deny rule added), (3) security group outbound rules. Private instances don't need public IPs (A is irrelevant). NAT Gateway doesn't have an EIP limit issue in this scenario (B).
</details>

---

### Question 7

The CloudWatch alarm for RDS FreeStorageSpace shows INSUFFICIENT_DATA even though the RDS instance is running and healthy. What is the MOST likely cause?

**A)** The RDS instance doesn't send metrics to CloudWatch
**B)** The alarm dimension (DBInstanceIdentifier) doesn't match the actual instance identifier
**C)** Enhanced monitoring needs to be enabled
**D)** The alarm evaluation period is too short

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** INSUFFICIENT_DATA almost always means CloudWatch can't find any data points for the specified metric + dimensions combination. The most common cause is a typo in the dimension value (DBInstanceIdentifier). Verify the exact identifier matches. RDS does send metrics to CloudWatch by default (A is wrong). Enhanced monitoring provides OS-level metrics to a different namespace (C is wrong). Evaluation period doesn't cause INSUFFICIENT_DATA (D is wrong).
</details>

---

### Question 8

The operations team wants to implement a blue/green deployment for the capstone application. Currently, the ASG is behind the ALB. What is the recommended approach?

**A)** Create a second ASG with the new version, create a second target group, add it to the ALB listener, then switch traffic using listener rules
**B)** Update the launch template and let the ASG gradually replace instances
**C)** Use CloudFormation to delete and recreate the stack
**D)** Stop all instances, update the AMI, and start them again

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** Blue/green deployment: (1) Create a new ASG (green) with the updated launch template, (2) Create a new target group and register the green ASG, (3) Add the green target group to the ALB listener with a small weight (10%), (4) Gradually shift traffic, (5) Deregister the blue target group when confident. Option B is a rolling update, not blue/green. C and D cause downtime.
</details>

---

### Question 9

Security Hub shows a CRITICAL finding: "S3.4 - S3 buckets should have server-side encryption enabled." But the LogBucket in the CloudFormation template has SSE-S3 encryption. Why is Security Hub flagging this?

**A)** Security Hub requires SSE-KMS, not SSE-S3
**B)** The bucket policy overrides the encryption setting
**C)** Security Hub checks are eventually consistent and may not have evaluated the latest state yet
**D)** The finding might be for a different S3 bucket in the account

<details>
<summary>Show Answer</summary>

**Correct Answer: D**

**Explanation:** Security Hub evaluates ALL resources in the account, not just those in the capstone stack. The finding might be for another S3 bucket that doesn't have encryption. Always check the specific resource ARN in the finding details. SSE-S3 does satisfy the S3.4 control (A is wrong). Bucket policies don't override encryption (B is wrong).
</details>

---

### Question 10

During the stress test, the CPU alarm triggered and the Auto Scaling group scaled from 2 to 4 instances. After the stress test ended, how long will it take for the ASG to scale back to 2?

**A)** Immediately when CPU drops below the threshold
**B)** After the target tracking scaling policy's cooldown period (default 300 seconds) and when the metric consistently shows the target can be met with fewer instances
**C)** It will never scale in; you must manually set desired capacity
**D)** After exactly 15 minutes (the default scale-in cooldown)

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Target tracking scaling policies manage both scale-out and scale-in. After CPU drops, the policy waits for the cooldown period and then gradually scales in (one instance at a time) to maintain the target (70% CPU). Scale-in is intentionally slower and more conservative than scale-out to maintain availability. The exact timing depends on how quickly the metric stabilizes.
</details>

---

### Question 11

The company's security team requires that ALL CloudTrail log files must be provably untampered for audit compliance. Which two features together provide this guarantee?

**A)** S3 versioning + S3 Object Lock
**B)** CloudTrail log file integrity validation + S3 Object Lock (Compliance mode)
**C)** CloudTrail encryption + S3 bucket policy
**D)** CloudTrail organization trail + KMS encryption

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Log file integrity validation creates SHA-256 digest files that can mathematically prove logs haven't been altered. S3 Object Lock in Compliance mode prevents anyone (including root) from deleting or modifying objects for the retention period. Together, they provide tamper detection AND tamper prevention. Versioning alone (A) doesn't prevent deletion. Encryption (C, D) protects confidentiality, not integrity.
</details>

---

### Question 12

An on-call engineer receives an SNS notification from the capstone ALB 5xx alarm at 2 AM. They need to quickly identify which URL paths are generating errors. What is the FASTEST investigation method?

**A)** Check the ALB access logs in S3 using Athena
**B)** SSH into each instance and check Apache error logs
**C)** Run a CloudWatch Logs Insights query against the httpd access log group
**D)** Check CloudWatch metrics for the target group

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** CloudWatch Logs Insights provides instant interactive querying. Query: `parse @message '* * * * "* * *" * *' as ip, id, user, ts, method, path, proto, status, size | filter status >= 500 | stats count(*) by path | sort count desc`. ALB logs in S3 (A) are near real-time but require Athena setup. SSH (B) is slow. CloudWatch metrics (D) don't show URL paths.
</details>

---

### Question 13

The capstone RDS instance in us-east-1 fails. Multi-AZ automatically promotes the standby. The application resumes but users report that some data from the last few seconds is missing. Is this expected?

**A)** No, Multi-AZ provides zero data loss with synchronous replication
**B)** Yes, Multi-AZ uses asynchronous replication so some data loss is expected
**C)** It depends on whether PITR was enabled
**D)** Data loss occurs because the DNS TTL hasn't expired

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** Multi-AZ uses synchronous replication. When the primary fails, the standby has an identical copy of the data. There should be zero data loss. If users report missing data, investigate the application (e.g., in-flight transactions that weren't committed, application caching). DNS is the TTL concern (D) but that affects connectivity, not data. Read replicas use async replication (that's where data loss can occur), but Multi-AZ standby does not.
</details>

---

### Question 14

The DLM lifecycle policy is creating EBS snapshots but the cross-region copy to us-west-2 is not working. The error says "KMS key not found." The source volumes use the default `aws/ebs` KMS key. What is the issue?

**A)** DLM cannot copy encrypted snapshots cross-region
**B)** The default `aws/ebs` key is region-specific and cannot be used to encrypt snapshots in another region. Specify a customer-managed KMS key in the cross-region copy rule
**C)** The IAM role for DLM doesn't have KMS permissions
**D)** You must create an identical KMS key alias in the destination region

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** AWS-managed KMS keys (like `aws/ebs`) are region-specific and cannot be shared across regions. For cross-region copy, you must specify a customer-managed KMS key (CMK) in the destination region for re-encryption. DLM can copy encrypted snapshots cross-region (A is wrong), but the destination must have a valid KMS key specified.
</details>

---

### Question 15

After completing all phases, the team wants to ensure they can recreate the entire environment in a disaster recovery region with minimal effort. What is the BEST approach?

**A)** Manually document all steps and have an engineer repeat them
**B)** Export AMIs and RDS snapshots to the DR region
**C)** Store the CloudFormation template in version control, use cross-region backups, and create a CloudFormation stack in the DR region from the same template
**D)** Set up continuous replication of all resources using AWS DataSync

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** Infrastructure as Code (CloudFormation) is the foundation of DR strategy. With the template in version control, the team can deploy an identical stack in any region. Cross-region backups (Phase 4) ensure data is available. The combination provides both infrastructure AND data recovery. Manual documentation (A) is error-prone. AMI/snapshot export (B) doesn't recreate networking, ALB, etc. DataSync (D) is for file-level replication, not full infrastructure.
</details>

---

## Cleanup Steps (IMPORTANT - follow order to avoid dependency errors)

```bash
# 1. Delete EventBridge rules (targets first)
for RULE in capstone-root-login capstone-sg-change; do
  aws events remove-targets --rule $RULE --ids sns 2>/dev/null
  aws events delete-rule --name $RULE 2>/dev/null
done

# 2. Delete CloudWatch alarms
aws cloudwatch delete-alarms --alarm-names \
  capstone-HighCPU capstone-ALB-5xx capstone-RDS-LowStorage

# 3. Delete CloudWatch dashboard
aws cloudwatch delete-dashboards --dashboard-names Capstone-Production

# 4. Stop CloudTrail and delete
aws cloudtrail stop-logging --name capstone-trail
aws cloudtrail delete-trail --name capstone-trail

# 5. Delete Config rules and recorder
for RULE in restricted-ssh required-tags rds-multi-az encrypted-volumes; do
  aws configservice delete-config-rule --config-rule-name $RULE 2>/dev/null
done
aws configservice stop-configuration-recorder --configuration-recorder-name capstone-recorder
aws configservice delete-configuration-recorder --configuration-recorder-name capstone-recorder

# 6. Delete SSM documents and parameters
aws ssm delete-document --name Custom-RevokeSSH 2>/dev/null
aws ssm delete-parameter --name AmazonCloudWatch-capstone 2>/dev/null

# 7. Delete AWS Backup resources
aws backup delete-backup-plan --backup-plan-id $PLAN_ID

# 8. Delete DLM policy
POLICY_ID=$(aws dlm get-lifecycle-policies --query 'Policies[0].PolicyId' --output text)
aws dlm delete-lifecycle-policy --policy-id $POLICY_ID

# 9. Delete Secrets Manager secret
aws secretsmanager delete-secret --secret-id capstone/prod/rds-password --force-delete-without-recovery

# 10. Disable GuardDuty
DETECTOR_ID=$(aws guardduty list-detectors --query 'DetectorIds[0]' --output text)
aws guardduty delete-detector --detector-id $DETECTOR_ID

# 11. Disable Security Hub
aws securityhub disable-security-hub

# 12. Delete CloudFormation stack (this deletes VPC, ALB, ASG, RDS, S3, etc.)
aws cloudformation delete-stack --stack-name capstone-lab
aws cloudformation wait stack-delete-complete --stack-name capstone-lab

# 13. Verify everything is cleaned up
echo "Check for any remaining resources in the AWS console"
```

> **WARNING:** NAT Gateways and RDS Multi-AZ instances incur hourly charges. Make sure the CloudFormation stack deletion completes successfully. If it fails, check stack events for resources that couldn't be deleted (e.g., non-empty S3 bucket).
