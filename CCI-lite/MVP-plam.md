# CCI Lite v1 Build Runbook

## 0. Scope

Deploy a working CCI Lite demo pipeline using AWS managed services:

* Upload audio → S3 input bucket
* Pipeline: Transcribe → Comprehend → Bedrock → Results JSON
* Persisted in S3, query via Glue + Athena
* Dashboard in QuickSight

---

## 1. Prerequisites

* AWS Account with permissions for KMS, S3, IAM, Lambda, SNS, EventBridge, Glue, Athena, QuickSight.
* Region: **eu-central-1**.
* Bedrock model access (Haiku or similar) confirmed.

---

## 2. Phase 1 — Foundation

### 2.1 KMS Key

* Create key alias: `alias/cci-lite-master-key`.
* Type: Symmetric, Encrypt/Decrypt.
* Grant access to: S3, Lambda, Glue, QuickSight, Logs.
* Record ARN in `environment-overview.md`.

### 2.2 S3 Buckets

| Name                                               | Purpose                    | Lifecycle                                     |
| -------------------------------------------------- | -------------------------- | --------------------------------------------- |
| `cci-lite-input-<accountid>-eu-central-1`          | Incoming audio files       | Delete after 30 days                          |
| `cci-lite-results-<accountid>-eu-central-1`        | Transcription & AI outputs | Transition to Glacier or delete after 90 days |
| `cci-lite-athena-staging-<accountid>-eu-central-1` | Athena query results       | Optional cleanup                              |

Settings: SSE-KMS encryption, block public access, optional versioning.

### 2.3 IAM Roles

#### a) `cci-lite-lambda-role`

* Trust: Lambda
* Managed: AWSLambdaBasicExecutionRole
* Inline permissions: Transcribe, Comprehend, Bedrock invoke, S3 (buckets only), KMS encrypt/decrypt

#### b) `cci-lite-glue-role`

* Trust: Glue
* Policies: AWSGlueServiceRole + S3 + KMS decrypt

#### c) `cci-lite-quicksight-role`

* Athena + S3 read + KMS decrypt

### 2.4 Parameter Store

Create:

```
/cci-lite/kms-key-arn
/cci-lite/input-bucket
/cci-lite/results-bucket
/cci-lite/athena-staging-bucket
```

Store ARNs and names for reference.

---

## 3. Phase 2 — Core Pipeline

### 3.1 `cci-lite-job-init` Lambda

* Runtime: Python 3.11
* Role: `cci-lite-lambda-role`
* Env vars: `INPUT_BUCKET`, `RESULTS_BUCKET`, `KMS_KEY_ARN`
* Trigger: EventBridge from S3 ObjectCreated events

**Logic:**

* Parse `tenant_id` and `call_id` from key.
* Start Transcribe job using `StartTranscriptionJob`.
* Output to results bucket prefix `tenant/<tenant_id>/calls/<call_id>/transcribe.json`.

### 3.2 EventBridge Rule

* Name: `cci-lite-s3-input-to-job-init`
* Source: `aws.s3`
* Detail-type: `Object Created`
* Target: Lambda `cci-lite-job-init`

### 3.3 Transcribe → SNS → `cci-lite-result-handler`

* Create SNS topic: `TranscribeJobStatusTopic`
* Subscribe `cci-lite-result-handler` Lambda
* Transcribe job completion publishes to SNS

### 3.4 `cci-lite-result-handler` Lambda

* Runtime: Python 3.11
* Role: `cci-lite-lambda-role`
* Env vars: `RESULTS_BUCKET`, `BEDROCK_MODEL_ID`, `COMPREHEND_REGION`

**Logic:**

1. Parse job name from SNS message.
2. Get transcript file from S3.
3. Run Comprehend (sentiment + key phrases).
4. Run Bedrock (summary, actions, QA checks).
5. Build unified JSON result.
6. Write to results bucket with KMS encryption.

Output path:
`s3://cci-lite-results-.../tenant/<tenant_id>/calls/<call_id>/cci-lite-output.json`

---

## 4. Phase 3 — Glue & Athena

### 4.1 Glue

* Database: `cci-lite-db`
* Crawler: `cci-lite-results-crawler`
* Source: results bucket (`tenant/` prefix)
* Role: `cci-lite-glue-role`

### 4.2 Athena

* Workgroup: `cci-lite-shared`
* Query result location: `cci-lite-athena-staging-...`
* Test query:

```sql
SELECT tenant_id, call_id, sentiment.overallSentiment, qa
FROM "cci-lite-db"."cci_lite_results"
LIMIT 10;
```

---

## 5. Phase 4 — QuickSight Dashboard

* Connect QuickSight to Athena.
* Create dataset: `cci-lite-db. cci_lite_results`.
* Build visuals:

  * KPI: Call count
  * Bar: Sentiment by Count
  * Table: Call summary, action items, QA
* Filter by tenant_id for isolation.

---

## 6. Phase 5 — Optional Enhancements

* Add DynamoDB table `cci-lite-config` for per-tenant QA/prompt control.
* Make Lambdas read tenant config dynamically.
* Add cost tags: `Project=CCI-Lite`, `Tenant=<id>`.

---

## 7. Validation Checklist

* ✅ S3 buckets encrypted and functional
* ✅ KMS policy grants correct access
* ✅ IAM roles operational
* ✅ File upload triggers pipeline
* ✅ JSON output generated
* ✅ Athena query succeeds
* ✅ QuickSight dashboard visible

---

## 8. Next Steps

After validation:

* Harden IAM policies.
* Add CloudWatch alarms.
* Integrate budget alerts.
* Document lessons learned in GitHub.
