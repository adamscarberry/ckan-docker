# S3 Bucket Folder Structure for CKAN Deployment

## Overview

Logical organization for CKAN resources, backups, logs, and configuration in AWS S3.

## Folder Structure

```
s3://my-ckan-bucket/
│
├── ckan-resources/
│   ├── datasets/
│   │   ├── {dataset-id}/
│   │   │   ├── resource-1.csv
│   │   │   ├── resource-2.xlsx
│   │   │   └── resource-3.json
│   │   └── {dataset-id}/
│   │       └── ...
│   └── uploads/
│       ├── {year}/{month}/{day}/
│       │   └── {filename}
│       └── ...
│
├── solr-backups/
│   ├── daily/
│   │   ├── solr-index-2026-02-08.tar.gz
│   │   ├── solr-index-2026-02-07.tar.gz
│   │   └── solr-index-2026-02-06.tar.gz
│   ├── weekly/
│   │   ├── solr-index-week-06-2026.tar.gz
│   │   └── solr-index-week-05-2026.tar.gz
│   └── monthly/
│       ├── solr-index-2026-02.tar.gz
│       └── solr-index-2026-01.tar.gz
│
├── logs/
│   ├── ckan-web/
│   │   ├── 2026/02/08/
│   │   │   ├── app-logs-00001.gz
│   │   │   └── app-logs-00002.gz
│   │   └── ...
│   ├── solr/
│   │   ├── 2026/02/08/
│   │   │   └── solr-logs-00001.gz
│   │   └── ...
│   ├── datapusher/
│   │   ├── 2026/02/08/
│   │   │   └── datapusher-logs-00001.gz
│   │   └── ...
│   └── celery/
│       ├── 2026/02/08/
│       │   └── celery-logs-00001.gz
│       └── ...
│
├── config/
│   ├── solr-schema.xml
│   ├── solr-solrconfig.xml
│   ├── ckan-extensions/
│   │   ├── custom-extension-v1.0.tar.gz
│   │   └── another-extension-v2.1.tar.gz
│   └── cloudformation-templates/
│       └── ckan-deployment.yaml
│
├── database-backups/
│   ├── rds-snapshots-log.json
│   └── manual-backups/
│       ├── ckan-db-2026-02-08-backup.sql.gz
│       └── datastore-db-2026-02-08-backup.sql.gz
│
└── tmp/
    ├── processing/
    │   └── {temporary-files-during-uploads}
    └── cache/
        └── {temporary-cache-files}
```

## Detailed Breakdown

### ckan-resources/ (User Data)

**Purpose:** CKAN resource files uploaded by users

```
ckan-resources/datasets/{dataset-id}/
├─ Stores actual resource files (CSV, Excel, JSON, etc.)
├─ Organized by dataset for easy retrieval
└─ Parallel structure mirrors CKAN database

ckan-resources/uploads/{year}/{month}/{day}/
├─ Alternative: time-based organization for new uploads
├─ Good for lifecycle policies (older data → cheaper storage class)
└─ Easier to clean up old temporary files
```

### solr-backups/ (Search Index Backups)

**Purpose:** Disaster recovery for Solr index

```
solr-backups/daily/
├─ Keep last 7-30 days of backups
├─ Script: Run daily backup job via Lambda or Celery
└─ Format: solr-index-YYYY-MM-DD.tar.gz

solr-backups/weekly/
├─ Keep last 8-12 weeks
├─ Keep every Sunday (or Monday) backup
└─ Format: solr-index-week-WW-YYYY.tar.gz

solr-backups/monthly/
├─ Keep last 12 months
├─ First day of month backup
└─ Format: solr-index-YYYY-MM.tar.gz
```

### logs/ (Application Logs)

**Purpose:** Centralized log storage from CloudWatch

```
logs/{service}/{year}/{month}/{day}/
├─ ckan-web/ → CKAN application logs
├─ solr/ → Solr indexing/search logs
├─ datapusher/ → DataPusher processing logs
└─ celery/ → Celery worker task logs

Organization by date allows:
├─ Easy cleanup (S3 Lifecycle policy: delete after 90 days)
├─ Quick retrieval (query by date range)
└─ Compliance (retention policies)
```

### config/ (Configuration & Code)

**Purpose:** Version control for infrastructure code

```
config/solr-schema.xml
├─ Solr core schema configuration
└─ Track changes to search index structure

config/ckan-extensions/
├─ Custom CKAN extensions as tarballs
├─ Versioned (v1.0, v2.1, etc.)
└─ Easy to rollback or test new versions

config/cloudformation-templates/
├─ IaC templates for reproducibility
└─ Version all infrastructure changes
```

### database-backups/ (RDS Manual Backups)

**Purpose:** Additional RDS backup snapshots

```
database-backups/manual-backups/
├─ ckan-db-YYYY-MM-DD-backup.sql.gz
│  └─ CKAN metadata database export
│
└─ datastore-db-YYYY-MM-DD-backup.sql.gz
   └─ Datastore database export

Note: RDS automated snapshots stay in RDS service
These are additional full exports for archive/compliance
```

### tmp/ (Temporary Files)

**Purpose:** Scratch space for processing

```
tmp/processing/
├─ Intermediate files during DataPusher conversion
├─ Lifecycle: Delete after 24-48 hours
└─ Example: CSV being converted to JSON

tmp/cache/
├─ Temporary cache files
└─ Lifecycle: Delete after 7 days
```

## S3 Lifecycle Policies

Apply these lifecycle rules to manage storage costs and automatic cleanup:

```json
{
  "Rules": [
    {
      "Id": "DeleteOldLogs",
      "Status": "Enabled",
      "Prefix": "logs/",
      "Expiration": {
        "Days": 90
      }
    },
    {
      "Id": "DeleteOldDailyBackups",
      "Status": "Enabled",
      "Prefix": "solr-backups/daily/",
      "Expiration": {
        "Days": 30
      }
    },
    {
      "Id": "DeleteOldWeeklyBackups",
      "Status": "Enabled",
      "Prefix": "solr-backups/weekly/",
      "Expiration": {
        "Days": 90
      }
    },
    {
      "Id": "DeleteTmpFiles",
      "Status": "Enabled",
      "Prefix": "tmp/",
      "Expiration": {
        "Days": 7
      }
    },
    {
      "Id": "TransitionOldResourcesStorage",
      "Status": "Enabled",
      "Prefix": "ckan-resources/",
      "Transitions": [
        {
          "Days": 90,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 180,
          "StorageClass": "GLACIER"
        }
      ]
    }
  ]
}
```

## Access Patterns

| Who                  | What                                    | Where               |
| -------------------- | --------------------------------------- | ------------------- |
| **CKAN Web**         | Read/write resource files               | `ckan-resources/`   |
| **DataPusher**       | Read/write temp files during conversion | `tmp/processing/`   |
| **Solr Backup Job**  | Write index backups                     | `solr-backups/`     |
| **Solr Restore Job** | Read index backups                      | `solr-backups/`     |
| **CloudWatch**       | Write application logs                  | `logs/`             |
| **Terraform/CDK**    | Read infrastructure templates           | `config/`           |
| **DBA/Admin**        | Manual RDS exports                      | `database-backups/` |

## IAM Policy Example

Restrict S3 access by component using IAM policies:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CKANResourceAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT:role/ckan-ecs-task-role"
      },
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::my-ckan-bucket/ckan-resources/*"
    },
    {
      "Sid": "SolrBackupAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT:role/solr-backup-job-role"
      },
      "Action": ["s3:PutObject", "s3:GetObject"],
      "Resource": "arn:aws:s3:::my-ckan-bucket/solr-backups/*"
    },
    {
      "Sid": "DataPusherTmpAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT:role/datapusher-ecs-task-role"
      },
      "Action": ["s3:PutObject", "s3:GetObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::my-ckan-bucket/tmp/*"
    },
    {
      "Sid": "CloudWatchLogsAccess",
      "Effect": "Allow",
      "Principal": {
        "Service": "logs.amazonaws.com"
      },
      "Action": ["s3:PutObject"],
      "Resource": "arn:aws:s3:::my-ckan-bucket/logs/*"
    }
  ]
}
```

## Naming Conventions

Follow these conventions for consistency:

- **Backup files:** `{service}-{YYYY-MM-DD}.tar.gz` or `.sql.gz`
- **Datasets:** Use dataset UUID or URL-safe slug
- **Logs:** `{service}-{timestamp}.gz`
- **Configurations:** Use meaningful names (e.g., `solr-schema.xml`, not `file1.xml`)
- **Versioned files:** Include version numbers (e.g., `extension-v1.0.tar.gz`)

## Cost Optimization Tips

1. **Use S3 Storage Classes:**
   - STANDARD: Recent resources and daily backups
   - STANDARD_IA: Resources older than 90 days
   - GLACIER: Resources older than 180 days

2. **Enable Versioning (optional):**
   - For config files (to track changes)
   - Disable for ckan-resources to save costs

3. **Set Up Lifecycle Rules:**
   - Automatically delete old logs after 90 days
   - Transition old backups to cheaper storage classes
   - Delete tmp files after 7 days

4. **Monitor Bucket Size:**
   - Use CloudWatch metrics to track growth
   - Alert if bucket size exceeds expected thresholds

5. **Enable S3 Intelligent-Tiering (optional):**
   - Automatically moves objects between access tiers
   - Good for unpredictable access patterns

## Backup & Recovery Procedures

### Daily Solr Index Backup

Schedule a daily task (Lambda or Celery):

```bash
# Backup current Solr index to S3
aws s3 cp /var/solr/data/ckan/index.zip \
  s3://my-ckan-bucket/solr-backups/daily/solr-index-$(date +%Y-%m-%d).tar.gz
```

### Restore Solr Index

When recovering from failure:

```bash
# Download backup from S3
aws s3 cp s3://my-ckan-bucket/solr-backups/daily/solr-index-2026-02-08.tar.gz \
  /var/solr/data/ckan-restore.tar.gz

# Extract and restore
tar -xzf /var/solr/data/ckan-restore.tar.gz -C /var/solr/data/

# Restart Solr container
```

### Manual RDS Database Backup

```bash
# Export CKAN database
aws rds describe-db-instances --db-instance-identifier ckan-db

# Use AWS Backup service or export snapshots to S3
aws rds start-export-task \
  --export-task-identifier ckan-db-export-2026-02-08 \
  --source-arn "arn:aws:rds:region:account:db:ckan-db" \
  --s3-bucket-name my-ckan-bucket \
  --s3-prefix database-backups/manual-backups/ \
  --iam-role-arn "arn:aws:iam::account:role/rds-export-role"
```
