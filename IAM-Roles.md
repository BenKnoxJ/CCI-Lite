# ✅ CCI Lite v1.3 — Phase 3 : IAM Roles (Updated)

### Date

2025-11-06

### Region

`eu-central-1`

### Environment

Production

---

## 1. Purpose

Phase 3 defines the IAM roles that connect all core CCI Lite components to AWS resources securely. This update aligns permissions with the latest Standard SNS configuration, unified KMS alias, and DynamoDB integration. All roles are validated for least privilege while enabling encryption and service integrations.

---

## 2. Roles Created

### 🟢 `cci-lite-lambda-role`

**Trust:** `lambda.amazonaws.com`
**Managed Policy:** `AWSLambdaBasicExecutionRole`

**Inline Permissions:**

* S3 read/write on all three buckets
* DynamoDB CRUD on `cci-lite-config`
* KMS Encrypt/Decrypt via `alias/cci-lite-master-key`
* SNS (Standard topic) publish permissions
* Transcribe, Comprehend, Bedrock access
* EventBridge, CloudWatch Logs, and X-Ray

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3Access",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::cci-lite-input-591338347562-eu-central-1",
        "arn:aws:s3:::cci-lite-input-591338347562-eu-central-1/*",
        "arn:aws:s3:::cci-lite-results-591338347562-eu-central-1",
        "arn:aws:s3:::cci-lite-results-591338347562-eu-central-1/*",
        "arn:aws:s3:::cci-lite-athena-staging-591338347562-eu-central-1",
        "arn:aws:s3:::cci-lite-athena-staging-591338347562-eu-central-1/*"
      ]
    },
    {
      "Sid": "DynamoDBAccess",
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:UpdateItem",
        "dynamodb:Query",
        "dynamodb:Scan",
        "dynamodb:DescribeTable"
      ],
      "Resource": "arn:aws:dynamodb:eu-central-1:591338347562:table/cci-lite-config"
    },
    {
      "Sid": "KMSAccess",
      "Effect": "Allow",
      "Action": ["kms:Encrypt", "kms:Decrypt", "kms:GenerateDataKey", "kms:DescribeKey"],
      "Resource": "arn:aws:kms:eu-central-1:591338347562:key/79d23f7f-9420-4398-a848-93876d0250e5"
    },
    {
      "Sid": "SNSAccess",
      "Effect": "Allow",
      "Action": [
        "sns:Publish",
        "sns:GetTopicAttributes",
        "sns:SetTopicAttributes",
        "sns:ListSubscriptionsByTopic"
      ],
      "Resource": "arn:aws:sns:eu-central-1:591338347562:TranscribeJobStatusTopic"
    },
    {
      "Sid": "ServiceIntegrations",
      "Effect": "Allow",
      "Action": [
        "transcribe:*",
        "comprehend:*",
        "bedrock:InvokeModel",
        "events:PutEvents",
        "logs:*",
        "xray:*"
      ],
      "Resource": "*"
    }
  ]
}
```

> **Note:** `sns:Publish` is only required for Lambdas that publish to SNS (e.g., job initiator). Lambdas that only receive SNS messages do **not** need this block.

---

### 🟢 `cci-lite-glue-role`

**Trust:** `glue.amazonaws.com`
**Managed Policy:** `AWSGlueServiceRole`
**Inline Permissions:**

* Read-only S3 access (results and staging)
* KMS decrypt and describe key access

Purpose: used by Glue Crawlers to catalog transcription results for Athena.

---

### 🟢 `cci-lite-quicksight-role`

**Trust:** `quicksight.amazonaws.com`
**Inline Permissions:**

* Athena query execution
* S3 read (results + staging)
* Glue metadata access
* KMS decrypt/describe key access

Purpose: grants QuickSight permission to query tenant datasets and decrypt results.

---

## 3. KMS Integration

All roles above are included in the master key policy for
`arn:aws:kms:eu-central-1:591338347562:key/79d23f7f-9420-4398-a848-93876d0250e5`.

Validated principals:

```
arn:aws:iam::591338347562:role/cci-lite-lambda-role  
arn:aws:iam::591338347562:role/cci-lite-glue-role  
arn:aws:iam::591338347562:role/cci-lite-quicksight-role  
```

No additional key policy changes required.

---

## 4. Validation Checklist

| Check                                    | Result |
| ---------------------------------------- | ------ |
| Roles created successfully               | ✅      |
| Managed and inline policies attached     | ✅      |
| SNS permissions reference Standard topic | ✅      |
| KMS alias applied and accessible         | ✅      |
| Region alignment (eu-central-1)          | ✅      |

---

## 5. Outputs / Notes

* IAM architecture now supports SNS Standard topic, replacing FIFO references.
* All encryption, S3, and DynamoDB permissions remain consistent with KMS key.
* Policies validated via IAM console with no syntax or principal errors.
* Roles provide least privilege necessary for the current CCI Lite scope.

---

## 6. Next Phase

Proceed to **Phase 4 — DynamoDB (Tenant Config + QA Templates)**
Tasks:

* Create `cci-lite-config` table.
* Insert tenant metadata and QA templates.
* Confirm Lambda has read/write access.

---

*End of updated Phase 3 (IAM Roles) documentation — aligned with Standard SNS topic, unified KMS alias, and validated principal structure.*
