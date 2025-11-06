# ✅ CCI Lite v1.3 — Phase 2 : S3 & KMS (Updated)

### Date

2025-11-06

### Region

`eu-central-1`

### Environment

Production

---

## 1. Purpose

This phase establishes secure S3 storage for CCI Lite and configures encryption using a single customer-managed KMS key (`alias/cci-lite-master-key`). Updates below align all buckets to the same CMK and correct the KMS policy principals for Transcribe and EventBridge integrations.

---

## 2. Buckets Configuration

Three buckets are used for pipeline input, results, and Athena query staging. Each bucket is now explicitly set to use the CMK `alias/cci-lite-master-key` for encryption.

| Bucket Name                                        | Purpose                                     | Encryption                      | Notes                                            |
| -------------------------------------------------- | ------------------------------------------- | ------------------------------- | ------------------------------------------------ |
| `cci-lite-input-<accountid>-eu-central-1`          | Raw audio uploads (trigger for job init)    | SSE-KMS (`cci-lite-master-key`) | Event notifications enabled for `/calls/` prefix |
| `cci-lite-results-<accountid>-eu-central-1`        | Transcribe + Bedrock processed JSON results | SSE-KMS (`cci-lite-master-key`) | Accessible by Glue, Athena, and QuickSight       |
| `cci-lite-athena-staging-<accountid>-eu-central-1` | Athena query output + temporary data        | SSE-KMS (`cci-lite-master-key`) | Set as Athena workgroup query results bucket     |

Lifecycle rules remain:

* Input bucket → Retain 1 day
* Results bucket → Retain 30 days
* Staging bucket → Retain 7 days

---

## 3. KMS Configuration

**Key Alias:** `alias/cci-lite-master-key`
**Key ARN:** `arn:aws:kms:eu-central-1:591338347562:key/79d23f7f-9420-4398-a848-93876d0250e5`

### Updated Policy

Revised to remove invalid IAM role principal and replace with correct AWS service principals. This ensures Amazon Transcribe, EventBridge, and CloudWatch Logs can encrypt/decrypt objects as part of pipeline operations.

```json
{
  "Version": "2012-10-17",
  "Id": "cci-lite-master-key-policy-updated",
  "Statement": [
    {
      "Sid": "AllowRootFullAccess",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::591338347562:root" },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "AllowServiceAccess",
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "s3.amazonaws.com",
          "lambda.amazonaws.com",
          "glue.amazonaws.com",
          "sns.amazonaws.com",
          "transcribe.amazonaws.com",
          "events.amazonaws.com",
          "logs.eu-central-1.amazonaws.com"
        ]
      },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowCoreRolesToUseKey",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::591338347562:role/cci-lite-lambda-role",
          "arn:aws:iam::591338347562:role/cci-lite-glue-role",
          "arn:aws:iam::591338347562:role/cci-lite-quicksight-role"
        ]
      },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 4. Validation Checklist

| Check                                                                       | Result                    |
| --------------------------------------------------------------------------- | ------------------------- |
| Buckets exist and accessible                                                | ✅                         |
| Encryption alias applied to all                                             | ✅ (`cci-lite-master-key`) |
| Lifecycle rules enforced                                                    | ✅                         |
| KMS policy valid (no invalid principals)                                    | ✅                         |
| Service principals added for S3, Lambda, SNS, Transcribe, EventBridge, Logs | ✅                         |

---

## 5. Outputs / Notes

* All storage locations now share a single CMK for unified encryption policy.
* Transcribe and EventBridge can safely encrypt/decrypt intermediate files.
* Glue, Athena, and QuickSight maintain read-only access through `cci-lite-glue-role` and `cci-lite-quicksight-role`.
* Policy tested and saved successfully without validation errors.

---

## 6. Next Phase

Proceed to **Phase 3 — IAM Roles**
Tasks:

* Create and attach IAM roles for Lambda, Glue, and QuickSight.
* Ensure each role has KMS decrypt permissions using this same key.

---

*End of updated Phase 2 (S3 & KMS) documentation — aligned with CMK alias and valid service principals.*
