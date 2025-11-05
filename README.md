# Conversant Conversational Intelligence (CCI) Lite — Implementation Plan

## Overview

CCI Lite is the post-call analytics tier of Conversant Conversational Intelligence. It ingests audio recordings, applies AI-based transcription and sentiment analysis, and delivers structured insights without any change to customer telephony systems.

### Core Objective

Build a secure, serverless AWS-native pipeline that converts uploaded call recordings into actionable insights within minutes.

---

## Phase 1 — Core Infrastructure

**Goal:** Establish all foundational AWS resources.

**Components:**

* AWS KMS key (`cci-lite-key`) for encryption
* S3 buckets: `cci-lite-audio-input`, `cci-lite-results`, `cci-lite-logs`
* DynamoDB table: `cci-lite-calls`
* IAM roles: Lambda, Glue, EventBridge
* EventBridge rules placeholders
* CloudWatch log groups

**Outcome:** Secure, encrypted environment with IAM roles and buckets ready for automation.

---

## Phase 2 — Lambda Pipeline

**Goal:** Automate analysis from audio upload to AI summary.

**Flow:**

1. **Lambda 1 – S3 Trigger**

   * Triggered on audio upload
   * Starts Transcribe batch job
   * Records job metadata in DynamoDB

2. **Lambda 2 – Transcribe Monitor**

   * Triggered via EventBridge on job completion
   * Fetches transcript, calls Comprehend and Bedrock
   * Produces structured JSON (`cci-insight-v2.1-full.json`)
   * Stores results in S3, updates DynamoDB

3. **Lambda 3 – Retention Cleaner**

   * Runs daily
   * Deletes input and intermediate data after 72 hours

**Outcome:** Fully automated AI analysis pipeline.

---

## Phase 3 — Data Lake & Analytics

**Goal:** Make enriched data queryable and visual.

**Steps:**

1. Create Glue Crawler for `cci-lite-results`
2. Build Athena views for sentiment and QA summaries
3. Connect QuickSight dashboards for insights and trends

**Outcome:** Visual reporting and analytics for customer calls.

---

## Phase 4 — Tenant Management & API

**Goal:** Secure multi-tenant isolation and programmatic access.

**Tasks:**

* Add API Gateway + Lambda for metadata and query endpoints
* Implement JWT or Cognito authentication
* Enforce KMS encryption context per tenant

**Outcome:** API-driven multi-tenant design for integrations.

---

## Phase 5 — Testing & Simulation

**Goal:** Validate functionality and reliability.

**Steps:**

1. Upload sample WAV (8 kHz µ-law) files
2. Validate Transcribe and Bedrock outputs
3. Ensure JSON matches canonical schema
4. Confirm latency < 2 minutes per file

**Outcome:** Verified accuracy, stability, and speed.

---

## Phase 6 — Observability & CI/CD

**Goal:** Harden for production readiness.

**Steps:**

* Enable CloudWatch and Sentry
* Add GitHub Actions for deploy (SAM/CDK)
* Implement cost-control (lifecycle, concurrency limits)

**Outcome:** Production-grade, cost-optimized, observable system.

---

## Deliverable

> A fully functional, secure, and scalable post-call analytics pipeline that processes audio uploads into AI-enriched structured data and dashboards — forming the foundation for CCI Premium and future SaaS layers.
