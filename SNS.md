# ✅ CCI Lite v1.3 — Phase 5 : SNS (Transcribe Job Status Topic)

### Date

2025-11-06

### Region

`eu-central-1`

### Environment

Production

---

## 1. Purpose

This phase establishes the **Amazon SNS topic** that notifies Lambda when an Amazon Transcribe job completes. The topic is encrypted with the same customer-managed KMS key to maintain data security and multi-tenant isolation. It will be used in subsequent phases by the Lambda functions (`cci-lite-job-init` and `cci-lite-result-handler`).

---

## 2. SNS Configuration

**Topic Name:** `TranscribeJobStatusTopic.fifo`
**Type:** FIFO (First-In-First-Out)
**Encryption:** Customer-managed KMS → `alias/cci-lite-master-key`
**Region:** eu-central-1
**Tags:**
`Project=CCI-Lite`, `Environment=Production`, `Version=v1.3`, `Region=eu-central-1`

**Topic ARN:**
`arn:aws:sns:eu-central-1:591338347562:TranscribeJobStatusTopic.fifo`

---

## 3. IAM Role Updates

The `cci-lite-lambda-role` was updated to include permissions for publishing and subscribing to the SNS topic.

### Updated SNS Access Policy

```json
{
  "Sid": "SNSAccess",
  "Effect": "Allow",
  "Action": [
    "sns:Publish",
    "sns:Subscribe",
    "sns:GetTopicAttributes",
    "sns:SetTopicAttributes",
    "sns:ListSubscriptionsByTopic",
    "sns:Unsubscribe"
  ],
  "Resource": "arn:aws:sns:eu-central-1:591338347562:TranscribeJobStatusTopic.fifo"
}
```

This block ensures Lambda can both publish messages (when initiating Transcribe jobs) and receive notifications (once subscribed in the result handler stage).

---

## 4. KMS Integration

* **Encryption Key:** `arn:aws:kms:eu-central-1:591338347562:key/79d23f7f-9420-4398-a848-93876d0250e5`
* **Alias:** `alias/cci-lite-master-key`
* The key policy already authorizes SNS service access, no changes required.

---

## 5. Validation Checklist

| Check                                      | Result |
| ------------------------------------------ | ------ |
| SNS Topic created                          | ✅      |
| FIFO suffix (`.fifo`) confirmed            | ✅      |
| Encryption key applied                     | ✅      |
| IAM permissions for Lambda updated         | ✅      |
| Region alignment                           | ✅      |
| Topic ARN captured for Lambda subscription | ✅      |

---

## 6. Outputs / Notes

* SNS topic now acts as the **event bridge** between Amazon Transcribe and Lambda.
* All encryption remains centralized under the same KMS key.
* The Lambda subscriber will be added during the **Result Handler** phase.
* Optional (future): add DLQ (SQS FIFO) or CloudWatch delivery status logging for fault tolerance.

---

## 7. Next Phase

Proceed to **Phase 6 — Glue / Athena Integration**
Tasks:

* Create Glue Database: `cci-lite-db`
* Create Glue Crawler: `cci-lite-results-crawler`
* Target: `s3://cci-lite-results-<accountid>-eu-central-1/demo-tenant/`
* Assign Role: `cci-lite-glue-role`
* Run crawler once to build Athena tables for QuickSight.

---

*End of Phase 5 update.*
