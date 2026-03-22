# Lab 5: Automating Data Snapshots for Archiving and Data Recovery

## Estimated Time: 2 hours

## Objectives
By the end of this lab, you will be able to:
- Create and manage EBS snapshots with AWS Data Lifecycle Manager (DLM)
- Implement RDS backup strategies and perform point-in-time recovery
- Configure AWS Backup for centralized backup management
- Set up S3 data protection with versioning, replication, and Object Lock
- Automate snapshot management using Lambda and EventBridge
- Understand exam-critical concepts for backup and recovery scenarios

---

## Prerequisites
- AWS CLI configured with appropriate credentials
- IAM permissions for EC2, RDS, S3, AWS Backup, Lambda, and EventBridge
- At least one running EC2 instance with an EBS volume
- Basic understanding of Python for Lambda functions

---

## Part 1: EBS Snapshots

### 1.1 Create Manual EBS Snapshot

**EXAM CALLOUT:** EBS snapshots are incremental - only blocks that have changed since the last snapshot are saved. The first snapshot is a full copy.

List your EBS volumes:
```bash
aws ec2 describe-volumes \
  --query 'Volumes[*].[VolumeId,Size,State,AvailabilityZone]' \
  --output table
```

Create a snapshot of a specific volume:
```bash
VOLUME_ID="vol-0123456789abcdef0"  # Replace with your volume ID

aws ec2 create-snapshot \
  --volume-id $VOLUME_ID \
  --description "Manual snapshot for Lab 5" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=Lab5-Manual-Snapshot},{Key=Environment,Value=Test}]'
```

Monitor snapshot progress:
```bash
SNAPSHOT_ID="snap-0123456789abcdef0"  # Replace with your snapshot ID

aws ec2 describe-snapshots \
  --snapshot-ids $SNAPSHOT_ID \
  --query 'Snapshots[0].[SnapshotId,State,Progress,StartTime]' \
  --output table
```

### 1.2 Copy Snapshot Cross-Region

**EXAM CALLOUT:** Cross-region snapshot copies are useful for disaster recovery and compliance requirements.

```bash
# Copy snapshot to another region
aws ec2 copy-snapshot \
  --region us-west-2 \
  --source-region us-east-1 \
  --source-snapshot-id $SNAPSHOT_ID \
  --description "Cross-region copy for DR" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=Lab5-DR-Copy},{Key=Purpose,Value=DisasterRecovery}]'
```

### 1.3 Create Volume from Snapshot in Different AZ

**EXAM CALLOUT:** You can restore snapshots to any AZ within the same region, enabling quick recovery and testing.

```bash
# Create a volume from snapshot in a different AZ
aws ec2 create-volume \
  --snapshot-id $SNAPSHOT_ID \
  --availability-zone us-east-1b \
  --volume-type gp3 \
  --iops 3000 \
  --throughput 125 \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=Lab5-Restored-Volume},{Key=Source,Value=Snapshot}]'
```

### 1.4 Amazon Data Lifecycle Manager (DLM)

**EXAM CALLOUT:** DLM automates snapshot creation, retention, and cross-region copies. It's cost-effective and reduces operational overhead.

Create an IAM role for DLM:
```bash
# Create trust policy
cat > dlm-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "dlm.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role
aws iam create-role \
  --role-name AWSDataLifecycleManagerDefaultRole \
  --assume-role-policy-document file://dlm-trust-policy.json

# Attach the managed policy
aws iam attach-role-policy \
  --role-name AWSDataLifecycleManagerDefaultRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSDataLifecycleManagerServiceRole
```

Create a DLM lifecycle policy (full JSON):
```bash
cat > dlm-policy.json <<EOF
{
  "ExecutionRoleArn": "arn:aws:iam::123456789012:role/AWSDataLifecycleManagerDefaultRole",
  "Description": "Lab5 - EBS Snapshot Policy every 12 hours, retain 7",
  "State": "ENABLED",
  "PolicyDetails": {
    "ResourceTypes": [
      "VOLUME"
    ],
    "TargetTags": [
      {
        "Key": "Backup",
        "Value": "true"
      }
    ],
    "Schedules": [
      {
        "Name": "Every-12-hours",
        "CopyTags": true,
        "TagsToAdd": [
          {
            "Key": "CreatedBy",
            "Value": "DLM"
          },
          {
            "Key": "LifecyclePolicy",
            "Value": "Lab5-Policy"
          }
        ],
        "CreateRule": {
          "Interval": 12,
          "IntervalUnit": "HOURS",
          "Times": [
            "00:00"
          ]
        },
        "RetainRule": {
          "Count": 7
        },
        "FastRestoreRule": {
          "Count": 1,
          "Interval": 1,
          "IntervalUnit": "DAYS",
          "AvailabilityZones": [
            "us-east-1a"
          ]
        },
        "CrossRegionCopyRules": [
          {
            "TargetRegion": "us-west-2",
            "Encrypted": true,
            "CopyTags": true,
            "RetainRule": {
              "Interval": 7,
              "IntervalUnit": "DAYS"
            }
          }
        ]
      }
    ]
  }
}
EOF

# Create the lifecycle policy
aws dlm create-lifecycle-policy \
  --cli-input-json file://dlm-policy.json
```

Tag volumes for automatic backup:
```bash
# Tag a volume to be backed up by DLM
aws ec2 create-tags \
  --resources $VOLUME_ID \
  --tags Key=Backup,Value=true
```

List DLM policies:
```bash
aws dlm get-lifecycle-policies \
  --query 'Policies[*].[PolicyId,Description,State]' \
  --output table
```

### 1.5 EBS Snapshot Archive

**EXAM CALLOUT:** EBS Snapshot Archive provides up to 75% cost savings for long-term retention. Restore time is 24-72 hours, suitable for compliance archives.

```bash
# Archive a snapshot
aws ec2 modify-snapshot-tier \
  --snapshot-id $SNAPSHOT_ID \
  --storage-tier archive

# Check archive status
aws ec2 describe-snapshots \
  --snapshot-ids $SNAPSHOT_ID \
  --query 'Snapshots[0].[SnapshotId,StorageTier,RestoreExpiryTime]' \
  --output table

# Restore from archive (temporary restore for 72 hours)
aws ec2 restore-snapshot-tier \
  --snapshot-id $SNAPSHOT_ID
```

### 1.6 Recycle Bin for EBS Snapshots

**EXAM CALLOUT:** Recycle Bin protects against accidental deletion by retaining deleted snapshots for a specified period (1 day to 1 year).

```bash
# Create retention rule for EBS snapshots
cat > recycle-bin-rule.json <<EOF
{
  "RetentionPeriod": {
    "RetentionPeriodValue": 7,
    "RetentionPeriodUnit": "DAYS"
  },
  "Description": "Retain deleted EBS snapshots for 7 days",
  "ResourceType": "EBS_SNAPSHOT",
  "ResourceTags": [
    {
      "ResourceTagKey": "Environment",
      "ResourceTagValue": "Production"
    }
  ]
}
EOF

aws rbin create-rule \
  --cli-input-json file://recycle-bin-rule.json

# List retention rules
aws rbin list-rules \
  --resource-type EBS_SNAPSHOT \
  --query 'Rules[*].[Identifier,Description,Status,RetentionPeriod]' \
  --output table
```

---

## Part 2: RDS Backup & Recovery

### 2.1 Automated Backups Configuration

**EXAM CALLOUT:** RDS automated backups enable point-in-time recovery (PITR). Retention period: 1-35 days. Backups are stored in S3 (managed by AWS).

```bash
# Modify DB instance to configure backup settings
aws rds modify-db-instance \
  --db-instance-identifier mydb-instance \
  --backup-retention-period 7 \
  --preferred-backup-window "03:00-04:00" \
  --apply-immediately

# View backup configuration
aws rds describe-db-instances \
  --db-instance-identifier mydb-instance \
  --query 'DBInstances[0].[BackupRetentionPeriod,PreferredBackupWindow,LatestRestorableTime]' \
  --output table
```

### 2.2 Manual DB Snapshots

**EXAM CALLOUT:** Manual snapshots persist indefinitely until explicitly deleted, even after the DB instance is deleted. Use for long-term retention.

```bash
# Create manual snapshot
aws rds create-db-snapshot \
  --db-snapshot-identifier mydb-manual-snapshot-$(date +%Y%m%d-%H%M%S) \
  --db-instance-identifier mydb-instance \
  --tags Key=Type,Value=Manual Key=Purpose,Value=BeforeMaintenance

# List snapshots
aws rds describe-db-snapshots \
  --db-instance-identifier mydb-instance \
  --query 'DBSnapshots[*].[DBSnapshotIdentifier,SnapshotCreateTime,Status,SnapshotType]' \
  --output table
```

### 2.3 Point-in-Time Recovery (PITR)

**EXAM CALLOUT:** PITR allows restore to any second within the retention period. Creates a NEW DB instance; cannot restore over existing instance.

```bash
# Restore to a specific time
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier mydb-instance \
  --target-db-instance-identifier mydb-restored-$(date +%Y%m%d) \
  --restore-time "2026-03-22T10:30:00Z" \
  --db-subnet-group-name mydb-subnet-group \
  --publicly-accessible false

# Restore to latest restorable time
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier mydb-instance \
  --target-db-instance-identifier mydb-restored-latest \
  --use-latest-restorable-time \
  --db-subnet-group-name mydb-subnet-group
```

### 2.4 Restore from Snapshot

**EXAM CALLOUT:** Restoring from snapshot creates a NEW DB instance. You must update application endpoints.

```bash
# Restore from snapshot
SNAPSHOT_ID="mydb-manual-snapshot-20260322-103000"

aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier mydb-from-snapshot \
  --db-snapshot-identifier $SNAPSHOT_ID \
  --db-instance-class db.t3.medium \
  --db-subnet-group-name mydb-subnet-group \
  --publicly-accessible false \
  --tags Key=RestoredFrom,Value=$SNAPSHOT_ID

# Monitor restore progress
aws rds describe-db-instances \
  --db-instance-identifier mydb-from-snapshot \
  --query 'DBInstances[0].[DBInstanceIdentifier,DBInstanceStatus,Endpoint.Address]' \
  --output table
```

### 2.5 Copy Snapshot Cross-Region

**EXAM CALLOUT:** Cross-region snapshot copies provide disaster recovery capabilities. Encrypted snapshots can be copied across regions.

```bash
# Copy snapshot to another region
aws rds copy-db-snapshot \
  --region us-west-2 \
  --source-db-snapshot-identifier arn:aws:rds:us-east-1:123456789012:snapshot:$SNAPSHOT_ID \
  --target-db-snapshot-identifier mydb-dr-copy-uswest2 \
  --kms-key-id arn:aws:kms:us-west-2:123456789012:key/your-kms-key-id \
  --copy-tags

# Verify copy in target region
aws rds describe-db-snapshots \
  --region us-west-2 \
  --db-snapshot-identifier mydb-dr-copy-uswest2 \
  --query 'DBSnapshots[0].[DBSnapshotIdentifier,Status,PercentProgress]' \
  --output table
```

### 2.6 Export Snapshot to S3

**EXAM CALLOUT:** RDS snapshot export to S3 exports data in Apache Parquet format for analytics. Useful for data lakes and long-term archival.

```bash
# Create IAM role for export task (trust policy)
cat > rds-export-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "export.rds.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create role
aws iam create-role \
  --role-name RDSSnapshotExportRole \
  --assume-role-policy-document file://rds-export-trust-policy.json

# Attach policy for S3 access
cat > rds-export-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject*",
        "s3:GetObject*",
        "s3:DeleteObject*"
      ],
      "Resource": [
        "arn:aws:s3:::my-rds-exports-bucket/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-rds-exports-bucket"
      ]
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name RDSSnapshotExportRole \
  --policy-name S3Access \
  --policy-document file://rds-export-policy.json

# Export snapshot to S3
aws rds start-export-task \
  --export-task-identifier mydb-export-$(date +%Y%m%d) \
  --source-arn arn:aws:rds:us-east-1:123456789012:snapshot:$SNAPSHOT_ID \
  --s3-bucket-name my-rds-exports-bucket \
  --s3-prefix mydb-exports/ \
  --iam-role-arn arn:aws:iam::123456789012:role/RDSSnapshotExportRole \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/your-kms-key-id

# Monitor export progress
aws rds describe-export-tasks \
  --query 'ExportTasks[*].[ExportTaskIdentifier,Status,PercentProgress,FailureCause]' \
  --output table
```

### 2.7 Aurora Backtrack

**EXAM CALLOUT:** Aurora Backtrack allows you to rewind a database to a specific time WITHOUT creating a new cluster. Available only for Aurora MySQL. Maximum backtrack window: 72 hours.

```bash
# Enable backtrack when creating Aurora cluster
aws rds create-db-cluster \
  --db-cluster-identifier myaurora-cluster \
  --engine aurora-mysql \
  --engine-version 5.7.mysql_aurora.2.11.2 \
  --master-username admin \
  --master-user-password MyPassword123! \
  --backtrack-window 259200 \
  --backup-retention-period 7

# Backtrack to a specific time
aws rds backtrack-db-cluster \
  --db-cluster-identifier myaurora-cluster \
  --backtrack-to "2026-03-22T10:00:00Z" \
  --force

# View backtrack history
aws rds describe-db-cluster-backtracks \
  --db-cluster-identifier myaurora-cluster \
  --query 'DBClusterBacktracks[*].[BacktrackIdentifier,BacktrackTo,Status]' \
  --output table
```

### 2.8 RDS Multi-AZ Failover Behavior

**EXAM CALLOUT:** Multi-AZ failover is automatic and typically completes in 60-120 seconds. Backups are taken from standby to avoid I/O impact on primary.

```bash
# Enable Multi-AZ
aws rds modify-db-instance \
  --db-instance-identifier mydb-instance \
  --multi-az \
  --apply-immediately

# Force failover for testing
aws rds reboot-db-instance \
  --db-instance-identifier mydb-instance \
  --force-failover

# Monitor failover
aws rds describe-events \
  --source-identifier mydb-instance \
  --source-type db-instance \
  --duration 60 \
  --query 'Events[*].[Date,Message]' \
  --output table
```

---

## Part 3: AWS Backup

**EXAM CALLOUT:** AWS Backup provides centralized backup management across AWS services (EC2, EBS, RDS, DynamoDB, EFS, Aurora, Storage Gateway, etc.). Supports cross-account and cross-region backups.

### 3.1 Create Backup Vault

```bash
# Create backup vault
aws backup create-backup-vault \
  --backup-vault-name Lab5-Production-Vault \
  --backup-vault-tags Key=Environment,Value=Production Key=ManagedBy,Value=AWSBackup

# Create vault in another region for DR
aws backup create-backup-vault \
  --region us-west-2 \
  --backup-vault-name Lab5-DR-Vault \
  --backup-vault-tags Key=Environment,Value=DR Key=ManagedBy,Value=AWSBackup

# List backup vaults
aws backup list-backup-vaults \
  --query 'BackupVaultList[*].[BackupVaultName,BackupVaultArn,NumberOfRecoveryPoints]' \
  --output table
```

### 3.2 Create Backup Plan with Multiple Rules

**EXAM CALLOUT:** Backup plans define WHEN and HOW to back up resources. Rules specify schedule, lifecycle, and copy actions.

```bash
# Create comprehensive backup plan
cat > backup-plan.json <<EOF
{
  "BackupPlanName": "Lab5-Comprehensive-Backup-Plan",
  "Rules": [
    {
      "RuleName": "DailyBackups",
      "TargetBackupVaultName": "Lab5-Production-Vault",
      "ScheduleExpression": "cron(0 2 * * ? *)",
      "StartWindowMinutes": 60,
      "CompletionWindowMinutes": 120,
      "Lifecycle": {
        "DeleteAfterDays": 7,
        "MoveToColdStorageAfterDays": 3
      },
      "RecoveryPointTags": {
        "Frequency": "Daily",
        "ManagedBy": "AWSBackup"
      },
      "EnableContinuousBackup": true
    },
    {
      "RuleName": "WeeklyBackups",
      "TargetBackupVaultName": "Lab5-Production-Vault",
      "ScheduleExpression": "cron(0 2 ? * SUN *)",
      "StartWindowMinutes": 60,
      "CompletionWindowMinutes": 180,
      "Lifecycle": {
        "DeleteAfterDays": 28
      },
      "RecoveryPointTags": {
        "Frequency": "Weekly",
        "RetentionType": "Medium"
      }
    },
    {
      "RuleName": "MonthlyBackupsWithCrossRegion",
      "TargetBackupVaultName": "Lab5-Production-Vault",
      "ScheduleExpression": "cron(0 2 1 * ? *)",
      "StartWindowMinutes": 60,
      "CompletionWindowMinutes": 240,
      "Lifecycle": {
        "DeleteAfterDays": 365
      },
      "RecoveryPointTags": {
        "Frequency": "Monthly",
        "RetentionType": "Long"
      },
      "CopyActions": [
        {
          "DestinationBackupVaultArn": "arn:aws:backup:us-west-2:123456789012:backup-vault:Lab5-DR-Vault",
          "Lifecycle": {
            "DeleteAfterDays": 365
          }
        }
      ]
    }
  ],
  "AdvancedBackupSettings": [
    {
      "ResourceType": "EC2",
      "BackupOptions": {
        "WindowsVSS": "enabled"
      }
    }
  ]
}
EOF

# Create the backup plan
BACKUP_PLAN_ID=$(aws backup create-backup-plan \
  --backup-plan file://backup-plan.json \
  --query 'BackupPlanId' \
  --output text)

echo "Backup Plan ID: $BACKUP_PLAN_ID"
```

### 3.3 Assign Resources Using Tags

**EXAM CALLOUT:** Tag-based resource assignment dynamically includes resources matching specified tags. Simplifies management at scale.

```bash
# Create IAM role for AWS Backup
cat > backup-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "backup.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name AWSBackupDefaultServiceRole \
  --assume-role-policy-document file://backup-trust-policy.json

aws iam attach-role-policy \
  --role-name AWSBackupDefaultServiceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForBackup

aws iam attach-role-policy \
  --role-name AWSBackupDefaultServiceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForRestores

# Create backup selection with tag-based assignment
cat > backup-selection.json <<EOF
{
  "SelectionName": "Production-Resources",
  "IamRoleArn": "arn:aws:iam::123456789012:role/AWSBackupDefaultServiceRole",
  "Resources": [],
  "ListOfTags": [
    {
      "ConditionType": "STRINGEQUALS",
      "ConditionKey": "Environment",
      "ConditionValue": "Production"
    }
  ]
}
EOF

aws backup create-backup-selection \
  --backup-plan-id $BACKUP_PLAN_ID \
  --backup-selection file://backup-selection.json

# Tag resources for backup
aws ec2 create-tags \
  --resources vol-0123456789abcdef0 \
  --tags Key=Environment,Value=Production

aws rds add-tags-to-resource \
  --resource-name arn:aws:rds:us-east-1:123456789012:db:mydb-instance \
  --tags Key=Environment,Value=Production
```

### 3.4 Cross-Account Backup with AWS Organizations

**EXAM CALLOUT:** Cross-account backup centralizes backup management and provides additional protection against account compromise.

```bash
# Enable cross-account backup in management account
aws backup put-backup-vault-access-policy \
  --backup-vault-name Lab5-Production-Vault \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "AWS": "arn:aws:iam::987654321098:root"
        },
        "Action": [
          "backup:CopyIntoBackupVault"
        ],
        "Resource": "*"
      }
    ]
  }'

# In source account, create backup plan with cross-account copy
cat > cross-account-backup-plan.json <<EOF
{
  "BackupPlanName": "CrossAccount-Backup",
  "Rules": [
    {
      "RuleName": "DailyCrossAccountCopy",
      "TargetBackupVaultName": "Default",
      "ScheduleExpression": "cron(0 3 * * ? *)",
      "Lifecycle": {
        "DeleteAfterDays": 7
      },
      "CopyActions": [
        {
          "DestinationBackupVaultArn": "arn:aws:backup:us-east-1:123456789012:backup-vault:Lab5-Production-Vault",
          "Lifecycle": {
            "DeleteAfterDays": 30
          }
        }
      ]
    }
  ]
}
EOF

aws backup create-backup-plan \
  --backup-plan file://cross-account-backup-plan.json
```

### 3.5 AWS Backup Audit Manager

**EXAM CALLOUT:** Backup Audit Manager helps meet compliance requirements by continuously monitoring backup activity and generating audit reports.

```bash
# Create audit framework
cat > audit-framework.json <<EOF
{
  "FrameworkName": "Lab5-Compliance-Framework",
  "FrameworkDescription": "Compliance framework for production backups",
  "FrameworkControls": [
    {
      "ControlName": "BACKUP_RECOVERY_POINT_MINIMUM_RETENTION_CHECK",
      "ControlInputParameters": [
        {
          "ParameterName": "requiredRetentionDays",
          "ParameterValue": "7"
        }
      ]
    },
    {
      "ControlName": "BACKUP_PLAN_MIN_FREQUENCY_AND_MIN_RETENTION_CHECK",
      "ControlInputParameters": [
        {
          "ParameterName": "requiredFrequencyUnit",
          "ParameterValue": "days"
        },
        {
          "ParameterName": "requiredFrequencyValue",
          "ParameterValue": "1"
        },
        {
          "ParameterName": "requiredRetentionDays",
          "ParameterValue": "7"
        }
      ]
    },
    {
      "ControlName": "BACKUP_RESOURCES_PROTECTED_BY_BACKUP_PLAN"
    },
    {
      "ControlName": "BACKUP_RECOVERY_POINT_ENCRYPTED"
    },
    {
      "ControlName": "BACKUP_RECOVERY_POINT_MANUAL_DELETION_DISABLED"
    }
  ]
}
EOF

aws backup create-framework \
  --cli-input-json file://audit-framework.json

# Generate compliance report
aws backup list-frameworks \
  --query 'Frameworks[*].[FrameworkName,FrameworkStatus,NumberOfControls]' \
  --output table
```

### 3.6 Restore Testing

```bash
# Start on-demand backup job
aws backup start-backup-job \
  --backup-vault-name Lab5-Production-Vault \
  --resource-arn arn:aws:ec2:us-east-1:123456789012:volume/vol-0123456789abcdef0 \
  --iam-role-arn arn:aws:iam::123456789012:role/AWSBackupDefaultServiceRole

# List recovery points
aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name Lab5-Production-Vault \
  --query 'RecoveryPoints[*].[RecoveryPointArn,ResourceType,CreationDate,Status]' \
  --output table

# Restore from recovery point
RECOVERY_POINT_ARN="arn:aws:backup:us-east-1:123456789012:recovery-point:abcd1234-5678-90ab-cdef-EXAMPLE11111"

aws backup start-restore-job \
  --recovery-point-arn $RECOVERY_POINT_ARN \
  --iam-role-arn arn:aws:iam::123456789012:role/AWSBackupDefaultServiceRole \
  --metadata '{"volumeType":"gp3","availabilityZone":"us-east-1a"}' \
  --resource-type EBS

# Monitor restore job
aws backup list-restore-jobs \
  --query 'RestoreJobs[*].[RestoreJobId,Status,PercentDone,ResourceType]' \
  --output table
```

---

## Part 4: S3 Data Protection

### 4.1 Enable S3 Versioning

**EXAM CALLOUT:** Versioning protects against accidental deletion and overwrites. Once enabled, it can only be suspended (not disabled). Delete markers are created for deletions.

```bash
# Create S3 bucket
BUCKET_NAME="lab5-data-protection-$(date +%s)"

aws s3api create-bucket \
  --bucket $BUCKET_NAME \
  --region us-east-1

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket $BUCKET_NAME \
  --versioning-configuration Status=Enabled

# Verify versioning status
aws s3api get-bucket-versioning \
  --bucket $BUCKET_NAME

# Upload files and test versioning
echo "Version 1" > test-file.txt
aws s3 cp test-file.txt s3://$BUCKET_NAME/

echo "Version 2" > test-file.txt
aws s3 cp test-file.txt s3://$BUCKET_NAME/

# List all versions
aws s3api list-object-versions \
  --bucket $BUCKET_NAME \
  --prefix test-file.txt \
  --query '{Versions:Versions[*].[Key,VersionId,IsLatest,LastModified]}' \
  --output table
```

### 4.2 S3 Lifecycle Policies

**EXAM CALLOUT:** Lifecycle policies automate transitions between storage classes and expiration. Transitions: S3 Standard -> S3-IA (min 30 days) -> Glacier (min 90 days).

```bash
# Create lifecycle policy
cat > lifecycle-policy.json <<EOF
{
  "Rules": [
    {
      "Id": "TransitionAndExpiration",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "documents/"
      },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        },
        {
          "Days": 180,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ],
      "Expiration": {
        "Days": 365
      },
      "NoncurrentVersionTransitions": [
        {
          "NoncurrentDays": 30,
          "StorageClass": "GLACIER"
        }
      ],
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 90
      },
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }
  ]
}
EOF

aws s3api put-bucket-lifecycle-configuration \
  --bucket $BUCKET_NAME \
  --lifecycle-configuration file://lifecycle-policy.json

# Verify lifecycle policy
aws s3api get-bucket-lifecycle-configuration \
  --bucket $BUCKET_NAME
```

### 4.3 Cross-Region Replication (CRR)

**EXAM CALLOUT:** CRR requires versioning enabled on both source and destination buckets. Useful for compliance, lower latency, and disaster recovery. Existing objects NOT replicated by default.

```bash
# Create destination bucket in another region
DEST_BUCKET_NAME="lab5-crr-destination-$(date +%s)"

aws s3api create-bucket \
  --bucket $DEST_BUCKET_NAME \
  --region us-west-2 \
  --create-bucket-configuration LocationConstraint=us-west-2

# Enable versioning on destination
aws s3api put-bucket-versioning \
  --bucket $DEST_BUCKET_NAME \
  --versioning-configuration Status=Enabled

# Create IAM role for replication
cat > replication-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name S3ReplicationRole \
  --assume-role-policy-document file://replication-trust-policy.json

# Create replication policy
cat > replication-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetReplicationConfiguration",
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::$BUCKET_NAME"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObjectVersionForReplication",
        "s3:GetObjectVersionAcl",
        "s3:GetObjectVersionTagging"
      ],
      "Resource": "arn:aws:s3:::$BUCKET_NAME/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ReplicateObject",
        "s3:ReplicateDelete",
        "s3:ReplicateTags"
      ],
      "Resource": "arn:aws:s3:::$DEST_BUCKET_NAME/*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name S3ReplicationRole \
  --policy-name S3ReplicationPolicy \
  --policy-document file://replication-policy.json

# Configure replication on source bucket
cat > replication-config.json <<EOF
{
  "Role": "arn:aws:iam::123456789012:role/S3ReplicationRole",
  "Rules": [
    {
      "Id": "ReplicateAll",
      "Status": "Enabled",
      "Priority": 1,
      "DeleteMarkerReplication": {
        "Status": "Enabled"
      },
      "Filter": {},
      "Destination": {
        "Bucket": "arn:aws:s3:::$DEST_BUCKET_NAME",
        "ReplicationTime": {
          "Status": "Enabled",
          "Time": {
            "Minutes": 15
          }
        },
        "Metrics": {
          "Status": "Enabled",
          "EventThreshold": {
            "Minutes": 15
          }
        },
        "StorageClass": "STANDARD_IA",
        "EncryptionConfiguration": {
          "ReplicaKmsKeyID": "arn:aws:kms:us-west-2:123456789012:key/your-kms-key-id"
        }
      },
      "SourceSelectionCriteria": {
        "SseKmsEncryptedObjects": {
          "Status": "Enabled"
        },
        "ReplicaModifications": {
          "Status": "Enabled"
        }
      }
    }
  ]
}
EOF

aws s3api put-bucket-replication \
  --bucket $BUCKET_NAME \
  --replication-configuration file://replication-config.json

# Verify replication configuration
aws s3api get-bucket-replication \
  --bucket $BUCKET_NAME
```

### 4.4 Same-Region Replication (SRR)

**EXAM CALLOUT:** SRR use cases include: log aggregation, compliance requirements, live replication between production and test accounts.

```bash
# SRR configuration (similar to CRR but destination in same region)
SRR_DEST_BUCKET="lab5-srr-destination-$(date +%s)"

aws s3api create-bucket \
  --bucket $SRR_DEST_BUCKET \
  --region us-east-1

aws s3api put-bucket-versioning \
  --bucket $SRR_DEST_BUCKET \
  --versioning-configuration Status=Enabled

# Update replication config with same-region destination
cat > srr-config.json <<EOF
{
  "Role": "arn:aws:iam::123456789012:role/S3ReplicationRole",
  "Rules": [
    {
      "Id": "SRRForCompliance",
      "Status": "Enabled",
      "Priority": 1,
      "Filter": {
        "Prefix": "compliance/"
      },
      "Destination": {
        "Bucket": "arn:aws:s3:::$SRR_DEST_BUCKET",
        "StorageClass": "STANDARD_IA"
      }
    }
  ]
}
EOF

# Note: For demo purposes only - typically you wouldn't have both CRR and SRR on same bucket
# This would replace the previous replication configuration
```

### 4.5 S3 Object Lock

**EXAM CALLOUT:**
- **Governance Mode**: Users with special permissions can override retention
- **Compliance Mode**: NO ONE can override, not even root account. For regulatory requirements
- Object Lock requires versioning
- Can only be enabled during bucket creation

```bash
# Create bucket with Object Lock enabled
LOCK_BUCKET="lab5-object-lock-$(date +%s)"

aws s3api create-bucket \
  --bucket $LOCK_BUCKET \
  --region us-east-1 \
  --object-lock-enabled-for-bucket

# Configure default retention
aws s3api put-object-lock-configuration \
  --bucket $LOCK_BUCKET \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {
      "DefaultRetention": {
        "Mode": "GOVERNANCE",
        "Days": 30
      }
    }
  }'

# Upload object with specific retention
aws s3api put-object \
  --bucket $LOCK_BUCKET \
  --key important-document.pdf \
  --body important-document.pdf \
  --object-lock-mode COMPLIANCE \
  --object-lock-retain-until-date "2026-12-31T23:59:59Z"

# Upload object with legal hold
aws s3api put-object \
  --bucket $LOCK_BUCKET \
  --key legal-document.pdf \
  --body legal-document.pdf \
  --object-lock-legal-hold-status ON

# View object retention
aws s3api get-object-retention \
  --bucket $LOCK_BUCKET \
  --key important-document.pdf

# View legal hold status
aws s3api get-object-legal-hold \
  --bucket $LOCK_BUCKET \
  --key legal-document.pdf
```

### 4.6 Legal Hold

**EXAM CALLOUT:** Legal Hold provides indefinite protection independent of retention period. Can be placed and removed freely (with proper permissions).

```bash
# Place legal hold
aws s3api put-object-legal-hold \
  --bucket $LOCK_BUCKET \
  --key sensitive-data.json \
  --legal-hold Status=ON

# Remove legal hold (requires s3:PutObjectLegalHold permission)
aws s3api put-object-legal-hold \
  --bucket $LOCK_BUCKET \
  --key sensitive-data.json \
  --legal-hold Status=OFF
```

### 4.7 MFA Delete

**EXAM CALLOUT:** MFA Delete requires MFA authentication to delete object versions or disable versioning. Only bucket owner (root account) can enable MFA Delete.

```bash
# Enable MFA Delete (requires MFA device serial and token)
# Note: This command must be run with root account credentials
aws s3api put-bucket-versioning \
  --bucket $BUCKET_NAME \
  --versioning-configuration Status=Enabled,MFADelete=Enabled \
  --mfa "arn:aws:iam::123456789012:mfa/root-account-mfa-device 123456"

# Delete object version with MFA
aws s3api delete-object \
  --bucket $BUCKET_NAME \
  --key protected-file.txt \
  --version-id "version-id-here" \
  --mfa "arn:aws:iam::123456789012:mfa/root-account-mfa-device 654321"
```

### 4.8 S3 Glacier Vault Lock

**EXAM CALLOUT:** Glacier Vault Lock enforces compliance controls with WORM (Write Once Read Many) model. Once locked, policy CANNOT be changed, even by AWS.

```bash
# Create Glacier vault
VAULT_NAME="lab5-compliance-vault"

aws glacier create-vault \
  --account-id - \
  --vault-name $VAULT_NAME

# Create vault lock policy
cat > vault-lock-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "deny-delete-before-retention",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "glacier:DeleteArchive",
      "Resource": "arn:aws:glacier:us-east-1:123456789012:vaults/$VAULT_NAME",
      "Condition": {
        "DateLessThan": {
          "glacier:ArchiveAgeInDays": "365"
        }
      }
    }
  ]
}
EOF

# Initiate vault lock (24 hour window to test)
LOCK_ID=$(aws glacier initiate-vault-lock \
  --account-id - \
  --vault-name $VAULT_NAME \
  --policy file://vault-lock-policy.json \
  --query 'lockId' \
  --output text)

echo "Lock ID: $LOCK_ID"
echo "You have 24 hours to test and complete the vault lock"

# After testing, complete vault lock (irreversible!)
# aws glacier complete-vault-lock \
#   --account-id - \
#   --vault-name $VAULT_NAME \
#   --lock-id $LOCK_ID
```

---

## Part 5: Automation with Lambda + EventBridge

### 5.1 Lambda Function for Automated EBS Snapshot Management

**EXAM CALLOUT:** Lambda + EventBridge provides serverless automation for backup tasks. Use CloudWatch Events for scheduling and SNS for notifications.

```python
# snapshot_manager.py
import boto3
import os
from datetime import datetime, timedelta

ec2 = boto3.client('ec2')
sns = boto3.client('sns')

# Environment variables
RETENTION_DAYS = int(os.environ.get('RETENTION_DAYS', 7))
SNS_TOPIC_ARN = os.environ.get('SNS_TOPIC_ARN')

def lambda_handler(event, context):
    try:
        # Create snapshots for tagged volumes
        created_snapshots = create_snapshots()

        # Cleanup old snapshots
        deleted_snapshots = cleanup_old_snapshots()

        # Send success notification
        message = f"""
        Snapshot Management Completed Successfully

        Created Snapshots: {len(created_snapshots)}
        {format_snapshot_list(created_snapshots)}

        Deleted Snapshots: {len(deleted_snapshots)}
        {format_snapshot_list(deleted_snapshots)}

        Retention Period: {RETENTION_DAYS} days
        Execution Time: {datetime.now().isoformat()}
        """

        send_notification(
            subject="EBS Snapshot Management - Success",
            message=message
        )

        return {
            'statusCode': 200,
            'body': {
                'created': len(created_snapshots),
                'deleted': len(deleted_snapshots)
            }
        }

    except Exception as e:
        error_message = f"Snapshot management failed: {str(e)}"
        send_notification(
            subject="EBS Snapshot Management - FAILURE",
            message=error_message
        )
        raise

def create_snapshots():
    """Create snapshots for volumes tagged with Backup=true"""
    created_snapshots = []

    # Find volumes tagged for backup
    volumes = ec2.describe_volumes(
        Filters=[
            {'Name': 'tag:Backup', 'Values': ['true']},
            {'Name': 'status', 'Values': ['in-use', 'available']}
        ]
    )['Volumes']

    print(f"Found {len(volumes)} volumes tagged for backup")

    for volume in volumes:
        volume_id = volume['VolumeId']
        volume_name = get_tag_value(volume.get('Tags', []), 'Name', volume_id)

        # Create snapshot
        snapshot = ec2.create_snapshot(
            VolumeId=volume_id,
            Description=f"Automated backup of {volume_name} - {datetime.now().isoformat()}"
        )

        snapshot_id = snapshot['SnapshotId']

        # Tag the snapshot
        tags = [
            {'Key': 'Name', 'Value': f"Auto-{volume_name}-{datetime.now().strftime('%Y%m%d-%H%M')}"},
            {'Key': 'AutomatedBackup', 'Value': 'true'},
            {'Key': 'CreatedDate', 'Value': datetime.now().isoformat()},
            {'Key': 'SourceVolume', 'Value': volume_id},
            {'Key': 'RetentionDays', 'Value': str(RETENTION_DAYS)}
        ]

        # Copy tags from volume
        for tag in volume.get('Tags', []):
            if tag['Key'] not in ['Name', 'Backup']:
                tags.append(tag)

        ec2.create_tags(
            Resources=[snapshot_id],
            Tags=tags
        )

        created_snapshots.append({
            'SnapshotId': snapshot_id,
            'VolumeId': volume_id,
            'VolumeName': volume_name
        })

        print(f"Created snapshot {snapshot_id} for volume {volume_id}")

    return created_snapshots

def cleanup_old_snapshots():
    """Delete automated snapshots older than retention period"""
    deleted_snapshots = []

    # Find automated snapshots
    snapshots = ec2.describe_snapshots(
        OwnerIds=['self'],
        Filters=[
            {'Name': 'tag:AutomatedBackup', 'Values': ['true']}
        ]
    )['Snapshots']

    print(f"Found {len(snapshots)} automated snapshots")

    cutoff_date = datetime.now() - timedelta(days=RETENTION_DAYS)

    for snapshot in snapshots:
        snapshot_id = snapshot['SnapshotId']
        start_time = snapshot['StartTime']

        # Remove timezone info for comparison
        if start_time.tzinfo:
            start_time = start_time.replace(tzinfo=None)

        # Check if snapshot is older than retention period
        if start_time < cutoff_date:
            try:
                snapshot_name = get_tag_value(snapshot.get('Tags', []), 'Name', snapshot_id)

                ec2.delete_snapshot(SnapshotId=snapshot_id)

                deleted_snapshots.append({
                    'SnapshotId': snapshot_id,
                    'Name': snapshot_name,
                    'Age': (datetime.now() - start_time).days
                })

                print(f"Deleted snapshot {snapshot_id} (age: {(datetime.now() - start_time).days} days)")

            except Exception as e:
                print(f"Failed to delete snapshot {snapshot_id}: {str(e)}")

    return deleted_snapshots

def get_tag_value(tags, key, default=''):
    """Extract tag value from tags list"""
    for tag in tags:
        if tag['Key'] == key:
            return tag['Value']
    return default

def format_snapshot_list(snapshots):
    """Format snapshot list for notification"""
    if not snapshots:
        return "None"

    lines = []
    for snap in snapshots:
        if 'VolumeId' in snap:
            lines.append(f"  - {snap['SnapshotId']} (Volume: {snap['VolumeId']}, Name: {snap['VolumeName']})")
        else:
            lines.append(f"  - {snap['SnapshotId']} (Name: {snap['Name']}, Age: {snap['Age']} days)")

    return '\n'.join(lines)

def send_notification(subject, message):
    """Send SNS notification"""
    if SNS_TOPIC_ARN:
        try:
            sns.publish(
                TopicArn=SNS_TOPIC_ARN,
                Subject=subject,
                Message=message
            )
            print(f"Notification sent: {subject}")
        except Exception as e:
            print(f"Failed to send notification: {str(e)}")
    else:
        print("SNS_TOPIC_ARN not configured, skipping notification")
```

### 5.2 Deploy Lambda Function

```bash
# Create SNS topic for notifications
SNS_TOPIC_ARN=$(aws sns create-topic \
  --name EBSSnapshotNotifications \
  --query 'TopicArn' \
  --output text)

# Subscribe email to topic
aws sns subscribe \
  --topic-arn $SNS_TOPIC_ARN \
  --protocol email \
  --notification-endpoint your-email@example.com

# Create IAM role for Lambda
cat > lambda-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name SnapshotManagerLambdaRole \
  --assume-role-policy-document file://lambda-trust-policy.json

# Create IAM policy for Lambda
cat > lambda-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeVolumes",
        "ec2:DescribeSnapshots",
        "ec2:CreateSnapshot",
        "ec2:DeleteSnapshot",
        "ec2:CreateTags",
        "ec2:DescribeTags"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sns:Publish"
      ],
      "Resource": "$SNS_TOPIC_ARN"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name SnapshotManagerLambdaRole \
  --policy-name SnapshotManagerPolicy \
  --policy-document file://lambda-policy.json

# Package Lambda function
zip snapshot_manager.zip snapshot_manager.py

# Create Lambda function
LAMBDA_ARN=$(aws lambda create-function \
  --function-name EBSSnapshotManager \
  --runtime python3.11 \
  --role arn:aws:iam::123456789012:role/SnapshotManagerLambdaRole \
  --handler snapshot_manager.lambda_handler \
  --zip-file fileb://snapshot_manager.zip \
  --timeout 300 \
  --memory-size 256 \
  --environment "Variables={RETENTION_DAYS=7,SNS_TOPIC_ARN=$SNS_TOPIC_ARN}" \
  --description "Automated EBS snapshot creation and cleanup" \
  --query 'FunctionArn' \
  --output text)

echo "Lambda Function ARN: $LAMBDA_ARN"
```

### 5.3 Create EventBridge Scheduled Rule

**EXAM CALLOUT:** EventBridge (formerly CloudWatch Events) triggers Lambda on schedule. Use cron expressions for precise scheduling.

```bash
# Create EventBridge rule (runs daily at 2 AM UTC)
aws events put-rule \
  --name DailyEBSSnapshotRule \
  --schedule-expression "cron(0 2 * * ? *)" \
  --state ENABLED \
  --description "Trigger EBS snapshot creation daily at 2 AM UTC"

# Add Lambda as target
aws events put-targets \
  --rule DailyEBSSnapshotRule \
  --targets "Id"="1","Arn"="$LAMBDA_ARN"

# Grant EventBridge permission to invoke Lambda
aws lambda add-permission \
  --function-name EBSSnapshotManager \
  --statement-id AllowEventBridgeInvoke \
  --action lambda:InvokeFunction \
  --principal events.amazonaws.com \
  --source-arn arn:aws:events:us-east-1:123456789012:rule/DailyEBSSnapshotRule

# Test Lambda function manually
aws lambda invoke \
  --function-name EBSSnapshotManager \
  --payload '{}' \
  response.json

cat response.json
```

### 5.4 CloudWatch Alarm for Backup Failures

**EXAM CALLOUT:** CloudWatch alarms monitor Lambda errors and trigger SNS notifications for failed backups.

```bash
# Create CloudWatch alarm for Lambda errors
aws cloudwatch put-metric-alarm \
  --alarm-name EBSSnapshotManagerFailures \
  --alarm-description "Alert on EBS snapshot manager Lambda failures" \
  --metric-name Errors \
  --namespace AWS/Lambda \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --dimensions Name=FunctionName,Value=EBSSnapshotManager \
  --alarm-actions $SNS_TOPIC_ARN \
  --treat-missing-data notBreaching

# Create alarm for Lambda duration (timeout warning)
aws cloudwatch put-metric-alarm \
  --alarm-name EBSSnapshotManagerDuration \
  --alarm-description "Alert when snapshot manager approaches timeout" \
  --metric-name Duration \
  --namespace AWS/Lambda \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 240000 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=FunctionName,Value=EBSSnapshotManager \
  --alarm-actions $SNS_TOPIC_ARN

# View Lambda logs
aws logs tail /aws/lambda/EBSSnapshotManager --follow
```

### 5.5 Enhanced Monitoring Dashboard

```bash
# Create CloudWatch dashboard
cat > dashboard.json <<EOF
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/Lambda", "Invocations", {"stat": "Sum", "label": "Invocations"}],
          [".", "Errors", {"stat": "Sum", "label": "Errors"}],
          [".", "Duration", {"stat": "Average", "label": "Avg Duration"}]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-east-1",
        "title": "EBS Snapshot Manager Performance",
        "period": 300,
        "dimensions": {
          "FunctionName": "EBSSnapshotManager"
        }
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/EC2", "SnapshotCreateVolume", {"stat": "Sum"}]
        ],
        "view": "timeSeries",
        "region": "us-east-1",
        "title": "EBS Snapshot Activity",
        "period": 3600
      }
    }
  ]
}
EOF

aws cloudwatch put-dashboard \
  --dashboard-name EBSSnapshotManagement \
  --dashboard-body file://dashboard.json
```

---

## Part 6: Review Questions

Test your knowledge with these exam-style questions:

<details>
<summary><strong>Question 1:</strong> Your company needs to implement a backup strategy for 500 EBS volumes across multiple environments (Dev, Test, Prod). Production volumes must be backed up every 12 hours and retained for 7 days, with copies in a secondary region retained for 14 days. What is the MOST operationally efficient solution?</summary>

**Answer:** Use Amazon Data Lifecycle Manager (DLM) with tag-based resource selection.

**Explanation:**
- Create a DLM lifecycle policy targeting volumes tagged with Environment=Production
- Configure schedule: every 12 hours with retention count of 7 snapshots
- Enable cross-region copy rules with 14-day retention in the secondary region
- DLM automatically handles snapshot creation, retention, and cross-region copying
- This is more efficient than Lambda functions for standard snapshot schedules
- DLM is AWS-managed, requires no server maintenance, and integrates with Fast Snapshot Restore

**Why not AWS Backup?** AWS Backup is also a valid solution but DLM is purpose-built for EBS and offers features like Fast Snapshot Restore that are EBS-specific.

**Why not Lambda?** Lambda adds operational overhead for a use case that DLM handles natively.
</details>

<details>
<summary><strong>Question 2:</strong> An RDS MySQL database was accidentally deleted at 10:30 AM today. The last manual snapshot was taken 3 days ago. Automated backups are enabled with a 7-day retention period. What is the BEST way to recover the database with minimal data loss?</summary>

**Answer:** Use Point-in-Time Recovery (PITR) to restore to 10:29 AM (just before deletion).

**Explanation:**
- RDS automated backups enable PITR to any second within the retention period
- PITR can restore to any time between the earliest restorable time and LatestRestorableTime (typically 5 minutes ago)
- This provides recovery with only minutes of potential data loss
- Command: `aws rds restore-db-instance-to-point-in-time --source-db-instance-identifier original-db --target-db-instance-identifier restored-db --restore-time "2026-03-22T10:29:00Z"`
- A new DB instance is created; update application connection strings

**Why not manual snapshot?** The manual snapshot is 3 days old, resulting in 3 days of data loss.

**EXAM TIP:** PITR creates a NEW instance. You cannot restore over the existing (deleted) instance.
</details>

<details>
<summary><strong>Question 3:</strong> Your organization requires backups to be immutable for compliance reasons, with a minimum 7-year retention. Backups should be protected from deletion by ANY user, including the root account. Which solution meets these requirements?</summary>

**Answer:** Use S3 with Object Lock in Compliance mode or S3 Glacier Vault Lock.

**Explanation:**
- **S3 Object Lock - Compliance Mode:** Once locked, objects cannot be deleted or overwritten by anyone, including root account, until retention period expires
- **S3 Glacier Vault Lock:** Provides WORM (Write Once Read Many) protection with a lockable policy that cannot be changed once finalized
- Compliance mode is required (not Governance mode, which allows override with special permissions)
- For 7-year retention, Glacier Deep Archive offers the lowest cost

**Why not AWS Backup with vault lock?** AWS Backup Vault Lock uses Governance mode by default and doesn't provide the same immutability guarantees as S3 Compliance mode.

**Why not EBS snapshots?** EBS snapshots can be deleted by users with appropriate permissions; they don't offer true immutability.

**EXAM TIP:** Compliance mode = no one can delete (including root). Governance mode = special permissions can override.
</details>

<details>
<summary><strong>Question 4:</strong> You are using AWS Backup to protect EC2 instances, EBS volumes, and RDS databases. The compliance team requires proof that all production resources tagged with Environment=Production are being backed up daily and retained for at least 30 days. How can you automate compliance reporting?</summary>

**Answer:** Use AWS Backup Audit Manager with a custom compliance framework.

**Explanation:**
- AWS Backup Audit Manager continuously evaluates backup activity against compliance requirements
- Create a framework with controls:
  - `BACKUP_RESOURCES_PROTECTED_BY_BACKUP_PLAN` - ensures resources are protected
  - `BACKUP_PLAN_MIN_FREQUENCY_AND_MIN_RETENTION_CHECK` - validates daily backups with 30-day retention
  - `BACKUP_RECOVERY_POINT_ENCRYPTED` - ensures backups are encrypted
- Audit Manager generates compliance reports automatically
- Use resource assignment by tags (Environment=Production) to dynamically include resources
- Reports can be exported to S3 for audit purposes

**EXAM TIP:** Audit Manager is specifically designed for backup compliance monitoring and reporting.
</details>

<details>
<summary><strong>Question 5:</strong> An application stores critical data in an S3 bucket. You need to protect against accidental deletion and enable recovery of deleted objects for up to 30 days. Objects older than 90 days should be moved to Glacier for cost optimization. What combination of features should you enable?</summary>

**Answer:** Enable S3 Versioning + Lifecycle policy with transitions + Noncurrent version expiration after 30 days.

**Explanation:**
- **S3 Versioning:** Protects against accidental deletion by keeping all versions. Delete operations create delete markers rather than permanently deleting
- **Lifecycle Policy:**
  ```json
  {
    "Rules": [{
      "Status": "Enabled",
      "Transitions": [
        {"Days": 90, "StorageClass": "GLACIER"}
      ],
      "NoncurrentVersionExpiration": {"NoncurrentDays": 30}
    }]
  }
  ```
- This configuration:
  - Moves current versions to Glacier after 90 days
  - Automatically deletes noncurrent versions (old/deleted) after 30 days
  - Provides 30-day recovery window for accidentally deleted objects

**Why not Object Lock?** Object Lock prevents deletion for compliance but doesn't automatically transition to Glacier based on age.

**EXAM TIP:** Versioning + Lifecycle policies = cost-effective protection with automated cleanup.
</details>

<details>
<summary><strong>Question 6:</strong> Your company has a Multi-AZ RDS instance. When does RDS take automated backups, and what is the impact on performance?</summary>

**Answer:** Automated backups are taken during the backup window from the standby instance in Multi-AZ deployments, resulting in no performance impact on the primary.

**Explanation:**
- **Multi-AZ behavior:** Backups are taken from the standby replica, so no I/O impact on primary
- **Single-AZ behavior:** Backups cause brief I/O suspension during snapshot creation
- Backup window should be set during low-traffic periods for single-AZ instances
- First backup is full; subsequent backups are incremental (transaction logs)
- Automated backups are stored in S3 (managed by AWS, not visible in your account)

**EXAM TIP:** Multi-AZ = backups from standby = no performance impact. This is a key benefit of Multi-AZ beyond just HA.
</details>

<details>
<summary><strong>Question 7:</strong> You need to replicate S3 data from us-east-1 to eu-west-1 for disaster recovery. Objects must be replicated within 15 minutes. Existing objects must also be replicated. What configuration is required?</summary>

**Answer:** Enable Cross-Region Replication (CRR) with S3 Replication Time Control (RTC) and use S3 Batch Replication for existing objects.

**Explanation:**
- **Prerequisites:**
  - Versioning must be enabled on both source and destination buckets
  - IAM role with replication permissions
- **S3 RTC (Replication Time Control):**
  - Guarantees 99.99% of objects replicated within 15 minutes
  - Provides replication metrics and event notifications
  - Required for the 15-minute SLA
- **Existing Objects:**
  - CRR only replicates NEW objects by default
  - Use S3 Batch Replication to replicate existing objects
  - Alternative: use S3 sync command as one-time copy
- **Configuration includes:**
  ```json
  {
    "ReplicationTime": {
      "Status": "Enabled",
      "Time": {"Minutes": 15}
    },
    "Metrics": {
      "Status": "Enabled",
      "EventThreshold": {"Minutes": 15}
    }
  }
  ```

**EXAM TIP:** Standard CRR doesn't guarantee replication time. RTC is required for SLA-backed replication timing.
</details>

<details>
<summary><strong>Question 8:</strong> Your Lambda function manages EBS snapshots but occasionally times out when there are many volumes to snapshot. The function is configured with 3-minute timeout and 512 MB memory. What is the BEST solution?</summary>

**Answer:** Refactor the Lambda function to use Step Functions for orchestration, processing volumes in batches with parallel execution.

**Explanation:**
- **Problem:** Lambda has a maximum timeout of 15 minutes, but processing many volumes sequentially may exceed this
- **Solution:**
  - Use AWS Step Functions to orchestrate snapshot workflow
  - Process volumes in parallel or batches
  - Each Lambda invocation handles a subset of volumes
  - Step Functions can run for up to 1 year
- **Alternative approaches:**
  - Increase Lambda timeout to maximum (15 minutes)
  - Increase memory (more memory = more CPU)
  - Use DynamoDB to track progress and resume on failure
  - Split into multiple Lambda functions

**Why not just increase timeout?** This only works if processing completes within 15 minutes. Step Functions provides better scalability.

**EXAM TIP:** For long-running workflows, use Step Functions to orchestrate multiple Lambda invocations.
</details>

<details>
<summary><strong>Question 9:</strong> What is the difference between EBS snapshot standard storage and EBS Snapshot Archive?</summary>

**Answer:**

**EBS Snapshot Storage (Standard):**
- Stored in S3 (managed by AWS, incremental storage)
- Instant restore capability - can create volumes immediately
- Use for snapshots that may need quick restore
- Higher cost per GB

**EBS Snapshot Archive:**
- 75% cost reduction compared to standard snapshot storage
- Restore time: 24 to 72 hours
- Designed for quarterly or yearly accessed snapshots
- Good for compliance archives and long-term retention
- Minimum storage duration: 90 days

**Use cases:**
- Standard: Daily/weekly backups, DR snapshots, testing
- Archive: Quarterly/annual snapshots, compliance archives, cold storage

**Command to archive:**
```bash
aws ec2 modify-snapshot-tier --snapshot-id snap-xxx --storage-tier archive
```

**EXAM TIP:** Archive tier is about cost optimization for infrequently accessed snapshots, similar to S3 Glacier relationship to S3 Standard.
</details>

<details>
<summary><strong>Question 10:</strong> Your organization uses AWS Organizations with multiple accounts. The security team wants to enforce that all production backups are automatically copied to a central backup account. How can you implement this?</summary>

**Answer:** Use AWS Backup with cross-account backup copy in the backup plan, combined with AWS Organizations integration for centralized management.

**Explanation:**

**Steps:**
1. **Enable AWS Backup in Organizations:**
   - Designate a central backup account
   - Enable AWS Backup policies in Organizations

2. **Configure backup vault in central account:**
   ```bash
   aws backup put-backup-vault-access-policy \
     --backup-vault-name Central-Backup-Vault \
     --policy '{
       "Statement": [{
         "Effect": "Allow",
         "Principal": {"AWS": "arn:aws:iam::SOURCE-ACCOUNT:root"},
         "Action": "backup:CopyIntoBackupVault",
         "Resource": "*"
       }]
     }'
   ```

3. **Create backup plan with cross-account copy:**
   ```json
   {
     "Rules": [{
       "RuleName": "CrossAccountCopy",
       "TargetBackupVaultName": "Default",
       "ScheduleExpression": "cron(0 2 * * ? *)",
       "CopyActions": [{
         "DestinationBackupVaultArn": "arn:aws:backup:us-east-1:CENTRAL-ACCOUNT:backup-vault:Central-Backup-Vault"
       }]
     }]
   }
   ```

4. **Use Organizations policies to enforce:**
   - Create backup policy at OU level
   - Automatically applies to all accounts in OU
   - Ensures compliance without manual configuration

**Benefits:**
- Centralized backup management and visibility
- Protection against account compromise (backups in separate account)
- Simplified compliance and auditing
- Automated enforcement through Organizations policies

**EXAM TIP:** Cross-account backup is a key feature for enterprise security. The central backup account should have restricted access and separate from production accounts.
</details>

---

## Cleanup Steps

To avoid ongoing charges, clean up the resources created in this lab:

```bash
# 1. Delete Lambda function
aws lambda delete-function --function-name EBSSnapshotManager

# 2. Delete EventBridge rule
aws events remove-targets --rule DailyEBSSnapshotRule --ids 1
aws events delete-rule --name DailyEBSSnapshotRule

# 3. Delete CloudWatch alarms
aws cloudwatch delete-alarms \
  --alarm-names EBSSnapshotManagerFailures EBSSnapshotManagerDuration

# 4. Delete CloudWatch dashboard
aws cloudwatch delete-dashboards --dashboard-names EBSSnapshotManagement

# 5. Delete SNS topic and subscriptions
aws sns delete-topic --topic-arn $SNS_TOPIC_ARN

# 6. Delete EBS snapshots
SNAPSHOT_IDS=$(aws ec2 describe-snapshots \
  --owner-ids self \
  --filters Name=tag:AutomatedBackup,Values=true \
  --query 'Snapshots[*].SnapshotId' \
  --output text)

for snap in $SNAPSHOT_IDS; do
  aws ec2 delete-snapshot --snapshot-id $snap
done

# 7. Delete DLM lifecycle policies
POLICY_IDS=$(aws dlm get-lifecycle-policies \
  --query 'Policies[*].PolicyId' \
  --output text)

for policy in $POLICY_IDS; do
  aws dlm delete-lifecycle-policy --policy-id $policy
done

# 8. Delete Recycle Bin rules
RULE_IDS=$(aws rbin list-rules \
  --resource-type EBS_SNAPSHOT \
  --query 'Rules[*].Identifier' \
  --output text)

for rule in $RULE_IDS; do
  aws rbin delete-rule --identifier $rule
done

# 9. Delete AWS Backup resources
# Delete backup selections
SELECTION_IDS=$(aws backup list-backup-selections \
  --backup-plan-id $BACKUP_PLAN_ID \
  --query 'BackupSelectionsList[*].SelectionId' \
  --output text)

for selection in $SELECTION_IDS; do
  aws backup delete-backup-selection \
    --backup-plan-id $BACKUP_PLAN_ID \
    --selection-id $selection
done

# Delete backup plans
aws backup delete-backup-plan --backup-plan-id $BACKUP_PLAN_ID

# Delete recovery points (optional - may want to keep for safety)
# aws backup delete-recovery-point --backup-vault-name Lab5-Production-Vault --recovery-point-arn <arn>

# Delete backup vaults (must be empty first)
aws backup delete-backup-vault --backup-vault-name Lab5-Production-Vault
aws backup delete-backup-vault --backup-vault-name Lab5-DR-Vault --region us-west-2

# Delete audit frameworks
aws backup delete-framework --framework-name Lab5-Compliance-Framework

# 10. Delete S3 buckets and objects
# Delete all versions and delete markers
aws s3api delete-bucket --bucket $BUCKET_NAME --region us-east-1
aws s3api delete-bucket --bucket $DEST_BUCKET_NAME --region us-west-2
aws s3api delete-bucket --bucket $SRR_DEST_BUCKET --region us-east-1
aws s3api delete-bucket --bucket $LOCK_BUCKET --region us-east-1

# For versioned buckets, you may need to delete all versions first:
# aws s3api list-object-versions --bucket $BUCKET_NAME | \
#   jq -r '.Versions[] | .Key + " " + .VersionId' | \
#   while read key version; do
#     aws s3api delete-object --bucket $BUCKET_NAME --key "$key" --version-id "$version"
#   done

# 11. Delete IAM roles and policies
aws iam delete-role-policy \
  --role-name SnapshotManagerLambdaRole \
  --policy-name SnapshotManagerPolicy

aws iam delete-role --role-name SnapshotManagerLambdaRole

aws iam detach-role-policy \
  --role-name AWSDataLifecycleManagerDefaultRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSDataLifecycleManagerServiceRole

aws iam delete-role --role-name AWSDataLifecycleManagerDefaultRole

aws iam delete-role-policy \
  --role-name RDSSnapshotExportRole \
  --policy-name S3Access

aws iam delete-role --role-name RDSSnapshotExportRole

aws iam delete-role-policy \
  --role-name S3ReplicationRole \
  --policy-name S3ReplicationPolicy

aws iam delete-role --role-name S3ReplicationRole

aws iam detach-role-policy \
  --role-name AWSBackupDefaultServiceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForBackup

aws iam detach-role-policy \
  --role-name AWSBackupDefaultServiceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForRestores

aws iam delete-role --role-name AWSBackupDefaultServiceRole

# 12. Delete RDS snapshots (if created)
RDS_SNAPSHOTS=$(aws rds describe-db-snapshots \
  --query 'DBSnapshots[?contains(DBSnapshotIdentifier, `lab5`)].DBSnapshotIdentifier' \
  --output text)

for snapshot in $RDS_SNAPSHOTS; do
  aws rds delete-db-snapshot --db-snapshot-identifier $snapshot
done

# 13. Clean up local files
rm -f dlm-trust-policy.json dlm-policy.json recycle-bin-rule.json
rm -f backup-plan.json backup-selection.json backup-trust-policy.json
rm -f lifecycle-policy.json replication-trust-policy.json replication-policy.json
rm -f replication-config.json srr-config.json vault-lock-policy.json
rm -f lambda-trust-policy.json lambda-policy.json snapshot_manager.zip
rm -f cross-account-backup-plan.json audit-framework.json dashboard.json
rm -f response.json

echo "Cleanup completed!"
```

---

## Key Exam Takeaways

1. **EBS Snapshots:**
   - Incremental, block-level, stored in S3
   - DLM for automated lifecycle management
   - Snapshot Archive for 75% cost savings (24-72 hour restore)
   - Recycle Bin for deletion protection

2. **RDS Backup:**
   - Automated backups enable PITR (1-35 day retention)
   - Manual snapshots persist indefinitely
   - Restore creates NEW instance
   - Multi-AZ backups from standby (no performance impact)
   - Aurora Backtrack for rewind without restore

3. **AWS Backup:**
   - Centralized backup across services
   - Tag-based resource assignment
   - Cross-account and cross-region capabilities
   - Audit Manager for compliance reporting

4. **S3 Protection:**
   - Versioning for deletion protection
   - Object Lock: Governance (can override) vs Compliance (cannot override)
   - CRR/SRR with RTC for guaranteed replication time
   - MFA Delete requires root account
   - Glacier Vault Lock for immutable archives

5. **Automation:**
   - Lambda + EventBridge for custom backup logic
   - CloudWatch alarms for failure detection
   - SNS for notifications
   - Step Functions for long-running workflows

**Remember:** Always consider cost, RTO/RPO requirements, and compliance needs when choosing backup solutions!

---

## Additional Resources

- [AWS Backup Documentation](https://docs.aws.amazon.com/aws-backup/)
- [EBS Snapshots User Guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSSnapshots.html)
- [RDS Backup and Restore](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_CommonTasks.BackupRestore.html)
- [S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)
- [Data Lifecycle Manager](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/snapshot-lifecycle.html)

---

**End of Lab 5**
