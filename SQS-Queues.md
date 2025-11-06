# Phase 6 — SQS Queues (Analysis + DLQ)

### Overview

This phase adds two SQS queues to improve reliability, durability, and throughput in the CCI Lite pipeline.
Rather than invoking Lambda directly from SNS, messages are first written to an encrypted SQS queue.
This isolates Lambda from SNS bursts, ensures message durability, and enables dead-letter recovery.

---

### Architecture

```
Transcribe → SNS (TranscribeJobStatusTopic) → SQS (cci-lite-analysis-queue) → Lambda (cci-lite-result-handler)
                                                            ↳
                                                             DLQ (cci-lite-dlq)
```

---

### Resources Created

| Name                      | Type           | Purpose                                             | Encryption                            | Linked DLQ     | Subscribed From            |
| ------------------------- | -------------- | --------------------------------------------------- | ------------------------------------- | -------------- | -------------------------- |
| `cci-lite-analysis-queue` | SQS (Standard) | Primary buffer between SNS and Lambda               | SSE-KMS (`alias/cci-lite-master-key`) | `cci-lite-dlq` | `TranscribeJobStatusTopic` |
| `cci-lite-dlq`            | SQS (Standard) | Catches messages that fail after 5 receive attempts | SSE-KMS (`alias/cci-lite-master-key`) | —              | Analysis Queue             |

---

### Configuration — `cci-lite-analysis-queue`

| Setting                  | Value                                                              |
| ------------------------ | ------------------------------------------------------------------ |
| Type                     | Standard                                                           |
| Visibility timeout       | 30 s                                                               |
| Message retention period | 4 days                                                             |
| Delivery delay           | 0 s                                                                |
| Encryption               | SSE-KMS (`alias/cci-lite-master-key`)                              |
| DLQ                      | `cci-lite-dlq` (max receives = 5)                                  |
| Tags                     | `Project=CCI-Lite`, `Environment=Production`, `Type=AnalysisQueue` |

**Access policy**

```json
{
  "Version": "2012-10-17",
  "Id": "cci-lite-analysis-queue-policy",
  "Statement": [
    {
      "Sid": "AllowAccountFullControl",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::591338347562:root" },
      "Action": "SQS:*",
      "Resource": "arn:aws:sqs:eu-central-1:591338347562:cci-lite-analysis-queue"
    },
    {
      "Sid": "AllowSNSTopicToSend",
      "Effect": "Allow",
      "Principal": { "Service": "sns.amazonaws.com" },
      "Action": "SQS:SendMessage",
      "Resource": "arn:aws:sqs:eu-central-1:591338347562:cci-lite-analysis-queue",
      "Condition": {
        "ArnEquals": {
          "aws:SourceArn": "arn:aws:sns:eu-central-1:591338347562:TranscribeJobStatusTopic"
        }
      }
    }
  ]
}
```

---

### Configuration — `cci-lite-dlq`

| Setting                  | Value                                                    |
| ------------------------ | -------------------------------------------------------- |
| Type                     | Standard                                                 |
| Visibility timeout       | 30 s                                                     |
| Message retention period | 14 days                                                  |
| Encryption               | SSE-KMS (`alias/cci-lite-master-key`)                    |
| Tags                     | `Project=CCI-Lite`, `Type=DLQ`, `Environment=Production` |

**Access policy**

```json
{
  "Version": "2012-10-17",
  "Id": "cci-lite-dlq-policy",
  "Statement": [
    {
      "Sid": "AllowAccountFullAccess",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::591338347562:root" },
      "Action": "SQS:*",
      "Resource": "arn:aws:sqs:eu-central-1:591338347562:cci-lite-dlq"
    }
  ]
}
```

---

### KMS Key Update

Added `"sqs.amazonaws.com"` to the KMS key policy to allow SQS encryption/decryption:

```json
{
  "Sid": "AllowSQSServiceAccess",
  "Effect": "Allow",
  "Principal": { "Service": "sqs.amazonaws.com" },
  "Action": ["kms:Encrypt", "kms:Decrypt", "kms:GenerateDataKey*"],
  "Resource": "*"
}
```

---

### Lambda and SNS Integration

1. SNS topic (`TranscribeJobStatusTopic`) has a subscription to the `cci-lite-analysis-queue`.
2. Lambda (`cci-lite-result-handler`) polls the queue as an SQS trigger.
3. Failed messages after 5 receives automatically flow to `cci-lite-dlq`.

---

### Operational Benefits

* **Durability:** no lost messages during Lambda errors or scaling events.
* **Scalability:** queue absorbs SNS bursts without throttling.
* **Recoverability:** DLQ captures failed events for re-processing.
* **Security:** KMS encryption end-to-end (S3 / SNS / SQS / Lambda).

---

### Validation Checklist

* [x] `cci-lite-analysis-queue` created and encrypted with CMK.
* [x] `cci-lite-dlq` created and linked (max receives = 5).
* [x] SNS subscription confirmed.
* [x] Lambda trigger added.
* [x] KMS key policy updated with SQS service principal.

---

**Phase 8 complete — SQS buffering and DLQ enabled.**
This finalises the core ingestion pipeline before proceeding to Glue/Athena analytics.
