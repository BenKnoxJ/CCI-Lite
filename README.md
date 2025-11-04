# Conversant Conversational Intelligence (CCI Lite) — AWS Implementation Plan

## Overview

This document defines the **end-to-end implementation plan** for deploying **CCI Lite** on AWS using a **console-first approach**. The goal is to achieve a fully functional, demo-ready environment that processes uploaded audio recordings into AI-enriched JSON insights (transcripts, sentiment, summaries, QA, and tone scores).

The design is **single-account, multi-tenant**, fully **serverless**, and leverages **managed AWS services only** — ensuring security, cost efficiency, and scalability for future SaaS replication.

---

## Architecture Summary

**Core Services:**
S3 → Lambda → Transcribe → Comprehend → Bedrock → S3 → Glue → Athena → QuickSight

**Supporting Services:**
KMS (encryption), IAM (access), CloudWatch (logging/monitoring), SSM Parameter Store (configuration)

**Regions:**
`eu-central-1` (preferred) or `us-east-1` for Bedrock model access.

**Primary Output:**
JSON results in the canonical schema `cci-lite-v1.0.json` per call.

---

# Phase 1 — Foundation Setup (Environment & Security)

### Objective

Prepare a secure, isolated AWS environment for CCI Lite that supports multi-tenant operation.

### Steps

1. **Create KMS Key (Customer Managed Key)**

   * Service: AWS KMS → Create key → Symmetric → AES-256.
   * Name: `cci-lite-master-key`
   * Add usage policy for Lambda, S3, Transcribe, Comprehend, and Glue.
   * Reason: Ensures encryption at rest and compliant data isolation.

2. **Set Up S3 Buckets**

   * Create two buckets:

     * `cci-lite-input` → audio uploads.
     * `cci-lite-results` → processed analytics.
   * Enable default encryption (SSE-KMS using `cci-lite-master-key`).
   * Enable versioning and lifecycle policies:

     * Input: delete after 1 day.
     * Results: delete after 30 days.
   * Folder structure:

     ```
     s3://cci-lite-input/<tenant_id>/calls/
     s3://cci-lite-results/<tenant_id>/YYYY/MM/DD/
     ```
   * Reason: Provides tenant-level partitioning and ensures short-term retention.

3. **Create IAM Roles and Policies**

   * **Lambda Execution Role:**

     * Permissions: S3 (read/write), Transcribe, Comprehend, Bedrock runtime, Glue, Athena, KMS.
     * Name: `cci-lite-lambda-role`.
   * **QuickSight Access Role:**

     * Grant access to Athena and `cci-lite-results` S3 bucket.
   * Reason: Defines least privilege access for compute and analytics.

4. **Set Up SSM Parameter Store**

   * Create parameters:

     * `/cci-lite/config/models` → Bedrock model IDs.
     * `/cci-lite/config/tenants/<tenant_id>` → tenant-specific metadata.
   * Reason: Centralized configuration management for SaaS deployment.

5. **Enable CloudWatch Logging**

   * Create a log group `cci-lite-logs`.
   * Configure retention (30 days).
   * Reason: Enables visibility, debugging, and future alerting.

---

# Phase 2 — Core Processing Pipeline

### Objective

Build the event-driven serverless pipeline to process audio files and produce AI-enriched insights.

### Steps

1. **Create Lambda Function: `cci-lite-processor`**

   * Runtime: Python 3.12 or Node.js 20.x.
   * Role: `cci-lite-lambda-role`.
   * Handler: `lambda_function.lambda_handler`.
   * Environment variables:

     ```
     INPUT_BUCKET=cci-lite-input
     OUTPUT_BUCKET=cci-lite-results
     KMS_KEY_ID=arn:aws:kms:region:account:key/xxxx
     MODEL_ID_CLAUDE=anthropic.claude-3-haiku
     ```
   * Timeout: 5 minutes.
   * Memory: 1024 MB.

2. **Attach S3 Trigger to Lambda**

   * Source: `cci-lite-input` bucket.
   * Event: `s3:ObjectCreated:*`.
   * Prefix filter: `calls/`
   * Reason: Automatically triggers pipeline when new call file is uploaded.

3. **Implement Lambda Logic (Console Summary)**

   1. **Start Transcribe Job**

      * `StartTranscriptionJob` using S3 URI.
      * Output to temporary path in `cci-lite-results/tmp/`.
      * Wait for completion via polling or SNS notification.
   2. **Invoke Comprehend**

      * Run `detect_sentiment`, `detect_key_phrases`, `detect_entities`.
   3. **Invoke Bedrock (Claude 3 Haiku)**

      * Generate summary, tone score, QA scoring, and action items.
   4. **Write Unified JSON Result**

      * Store in `s3://cci-lite-results/<tenant_id>/YYYY/MM/DD/<call_id>.json`.
      * Conform to schema `cci-lite-v1.0.json`.

   * Reason: Each stage builds on previous data; no local storage required.

4. **Test the Pipeline**

   * Upload a sample `.wav` file manually into `s3://cci-lite-input/demo-tenant/calls/`.
   * Verify Transcribe job creation and JSON output in results bucket.
   * Confirm redaction and sentiment accuracy.

---

# Phase 3 — Data Cataloging and Querying

### Objective

Enable structured querying and reporting from processed JSON outputs.

### Steps

1. **Create AWS Glue Database**

   * Name: `cci-lite-db`.
   * Reason: Logical catalog for Athena and QuickSight.

2. **Create Glue Crawler**

   * Source: `s3://cci-lite-results/`
   * Schedule: Daily.
   * IAM Role: `cci-lite-lambda-role` (reuse).
   * Output: Tables partitioned by `tenant_id` and `call_date`.
   * Reason: Keeps schema updated automatically.

3. **Query via Athena**

   * Database: `cci-lite-db`.
   * Example query:

     ```sql
     SELECT tenant_id, call_date, sentiment.overall, qa_score, tone_score
     FROM "cci-lite-db"."results"
     WHERE call_date > current_date - interval '7' day;
     ```
   * Reason: Verify schema and validate output fields.

---

# Phase 4 — Dashboard and Visualization

### Objective

Build QuickSight dashboards for sentiment, QA, and summary insights.

### Steps

1. **Set Up QuickSight**

   * Connect QuickSight to Athena.
   * Create dataset using `cci-lite-db` tables.
   * Enable SPICE caching for performance.

2. **Create Dashboards**

   * Key visuals:

     * Average sentiment per day.
     * Top key phrases and entities.
     * Average QA score by agent.
     * Call outcomes (Resolved / Escalated).
     * Tone distribution chart.
   * Reason: Enables demo-ready visual storytelling.

3. **Apply Row-Level Security (Optional)**

   * Use `tenant_id` as filter to isolate dashboards per customer.
   * Reason: Supports multi-tenant visibility control.

---

# Phase 5 — SaaS Readiness & Expansion

### Objective

Prepare for onboarding real tenants and automating deployment.

### Steps

1. **Parameterize Deployment**

   * Convert setup into reusable **CloudFormation or CDK stack**.
   * Parameters:

     * `TenantId`, `InputBucket`, `ResultsBucket`, `KmsKeyArn`, `ModelId`.

2. **Add API Gateway (Future Phase)**

   * Endpoint: `/upload-audio` → generates pre-signed S3 URL.
   * Lambda backend authenticates via API key or Cognito.
   * Reason: Simplifies external uploads for SaaS users.

3. **Set Cost Guardrails**

   * AWS Budgets → Alerts for Transcribe, Bedrock usage.
   * Lambda concurrency limits: 50.
   * Reason: Prevent runaway costs in testing or demos.

4. **Demo Preparation**

   * Preload demo audio for 2–3 sample tenants.
   * Verify dashboard refresh in QuickSight.
   * Produce walkthrough script for live demo.

---

# Phase 6 — Observability & Maintenance

### Objective

Ensure long-term reliability, auditing, and easy debugging.

### Steps

1. **Enable CloudWatch Metrics & Alarms**

   * Metrics:

     * Lambda errors > 0
     * Transcribe job failures
     * Bedrock latency > 10s
   * Actions: Email or Slack notifications.

2. **Implement Logging Standards**

   * Structured logs JSON format:

     ```json
     {"tenant_id": "t01", "call_id": "c-001", "stage": "Transcribe", "status": "Success", "duration_ms": 2430}
     ```
   * Reason: Supports future metrics aggregation.

3. **Lifecycle Validation**

   * Monthly review: confirm S3 cleanup, IAM role validity, Glue schema updates.
   * Reason: Keeps environment lean and compliant.

---

# Phase 7 — Validation and Demo Sign-Off

### Objective

Complete functional verification and readiness checklist.

### Demo Checklist

| Item | Description                                   | Status |
| ---- | --------------------------------------------- | ------ |
| ✅    | Upload file triggers pipeline                 |        |
| ✅    | Transcribe returns accurate transcript        |        |
| ✅    | Comprehend produces sentiment and key phrases |        |
| ✅    | Bedrock generates summary and QA              |        |
| ✅    | JSON output follows `cci-lite-v1.0` schema    |        |
| ✅    | Glue/Athena query successful                  |        |
| ✅    | QuickSight dashboard updated                  |        |
| ✅    | Cost < £0.25 per 10-min call                  |        |

---

## Future Expansion Roadmap

| Stage                           | Description                                                       |
| ------------------------------- | ----------------------------------------------------------------- |
| **CCI Lite v2.0**               | Add upload API and basic tenant onboarding portal.                |
| **CCI Premium Integration**     | Extend to live call ingestion via Chime Voice Connector + SIPREC. |
| **Cross-Region Deployment**     | Deploy replicas via CloudFormation StackSets.                     |
| **Anomaly & Emotion Detection** | Integrate Comprehend custom classifiers or Titan fine-tuning.     |

---

### **Conclusion**

This phased implementation plan provides a complete, console-driven path to deploying and demoing **CCI Lite**. Each phase builds incrementally, ensuring that security, scalability, and SaaS readiness are inherent from the start. Once implemented, this environment can be cloned, automated, and extended into **CCI Premium** with zero architectural refactoring.
