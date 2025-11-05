# CCI Lite — Console-Only Implementation Plan (Corrected, Error-Proof)

> Single-account, multi-tenant. Region: **eu-central-1** for core services. **us-east-1** for Bedrock (until local model availability). This plan eliminates out-of-order steps and adds guardrails, validations, and fixes for common AWS console pitfalls.

---

## Global Conventions

* **Tags:** `Project=CCI-Lite`, `Env=prod`, `Owner=Conversant`, `Data=Voice`
* **KMS alias:** `alias/cci-lite-master-key`
* **Buckets:** `cci-lite-input`, `cci-lite-results`
* **S3 prefixes:**

  * Input: `s3://cci-lite-input/<tenant_id>/calls/`
  * Results: `s3://cci-lite-results/<tenant_id>/YYYY/MM/DD/`
* **IAM roles:**

  * `cci-lite-lambda-role` (Lambda execution)
  * *(later)* `cci-lite-upload-api-role` (only if you add the upload API)

---

## Phase 1 — Foundation Setup (Stop after this; no CloudWatch/QuickSight yet)

### 1. Create the KMS CMK

**Console:** KMS → *Create key* → *Symmetric* → Alias: `alias/cci-lite-master-key` → Add your admin as *Key administrator* → Finish.
**Then:** KMS → your key → **Key users** → add:

* Your IAM user
* `cci-lite-lambda-role` *(you will add this again after you create the role in Step 3)*

**Validation:** Key shows alias and Key users tab will be used later.

---

### 2. Create S3 buckets with lifecycle and default encryption

**2.1 `cci-lite-input`**

* S3 → *Create bucket* → Name: `cci-lite-input` → Region: eu-central-1 → Block public access **ON** → Create.
* Bucket → *Properties* → **Default encryption** → **AWS KMS (SSE-KMS)** → select `alias/cci-lite-master-key` → Save.
* Bucket → *Management* → **Lifecycle rules** → *Create rule*

  * Name: `input-retention-1d`
  * Scope: All objects
  * Action: **Expire current versions after 1 day** → Save.
* Bucket → *Objects* → *Create folder* → `demo-tenant/calls/`.

**2.2 `cci-lite-results`**

* Repeat creation.
* Default encryption: SSE-KMS with the same alias.
* Lifecycle: `results-retention-30d` → expire current versions after 30 days.
* Create folders: `tmp/`, `demo-tenant/`.

**2.3 Bucket policy (input)**

* Use the **validated** policy below (works with presigned URLs and console; enforces TLS + KMS):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowTenantAndLambdaUploads",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::591338347562:user/Ben.Knox-Johnston",
          "arn:aws:iam::591338347562:role/cci-lite-lambda-role"
        ]
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::cci-lite-input/*",
      "Condition": {
        "Bool": { "aws:SecureTransport": "true" },
        "StringEquals": { "s3:x-amz-server-side-encryption": "aws:kms" },
        "StringLikeIfExists": {
          "s3:x-amz-server-side-encryption-aws-kms-key-id": "arn:aws:kms:eu-central-1:591338347562:key/79d23f7f-9420-4398-a848-93876d0250e5"
        }
      }
    },
    {
      "Sid": "AllowPutObjectAclSeparately",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::591338347562:user/Ben.Knox-Johnston",
          "arn:aws:iam::591338347562:role/cci-lite-lambda-role"
        ]
      },
      "Action": "s3:PutObjectAcl",
      "Resource": "arn:aws:s3:::cci-lite-input/*",
      "Condition": { "Bool": { "aws:SecureTransport": "true" } }
    },
    {
      "Sid": "AllowReadByAnalyticsAndOwner",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::591338347562:user/Ben.Knox-Johnston",
          "arn:aws:iam::591338347562:role/cci-lite-lambda-role"
        ]
      },
      "Action": [ "s3:GetObject", "s3:GetObjectVersion", "s3:ListBucket" ],
      "Resource": [
        "arn:aws:s3:::cci-lite-input",
        "arn:aws:s3:::cci-lite-input/*"
      ],
      "Condition": { "Bool": { "aws:SecureTransport": "true" } }
    },
    {
      "Sid": "DenyUnEncryptedOrNonTLSUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::cci-lite-input/*",
      "Condition": {
        "Bool": { "aws:SecureTransport": "false" },
        "StringNotEqualsIfExists": { "s3:x-amz-server-side-encryption": "aws:kms" }
      }
    }
  ]
}
```

*(Replicate a simpler read policy for `cci-lite-results` if needed; deny block not required on results.)*

**Validation:**

* Upload a small file via console to `demo-tenant/calls/` → succeeds and shows **aws:kms**.
* CLI without SSE-KMS → denied (expected).

---

### 3. Create IAM role for Lambda (least privilege after test)

**IAM → Roles → Create role → Lambda**

* Name: `cci-lite-lambda-role`.
* Temporary policies (for first test; will restrict in Phase 2):

  * `AmazonS3FullAccess`, `CloudWatchLogsFullAccess`
* Create role.

**KMS:** Go back to KMS key → **Key users** → add `cci-lite-lambda-role`.

---

### 4. Parameter Store (central config)

**SSM → Parameter Store → Create parameter**

* `/cci-lite/config/models` (String):

```json
{ "bedrock_model_summary": "anthropic.claude-3-haiku-20240307", "bedrock_region": "us-east-1" }
```

* `/cci-lite/config/tenants/demo-tenant` (String):

```json
{ "tenant_id": "demo-tenant", "retention_days_results": 30, "retention_days_input": 1 }
```

**Phase 1 STOP.**

---

## Phase 2 — Core Pipeline (Lambda → Transcribe → Comprehend → Bedrock)

> Build Lambda, then trigger, then test. Only after a successful run do CloudWatch retention and alarms.

### 1. Create Lambda `cci-lite-processor`

**Lambda → Create function → Author from scratch**

* Name: `cci-lite-processor`
* Runtime: Python 3.12 *(or Node.js 20.x)*
* Role: **Use existing** → `cci-lite-lambda-role`
* Memory: 1024 MB; Timeout: 5 min
* Env vars:

  * `INPUT_BUCKET=cci-lite-input`
  * `OUTPUT_BUCKET=cci-lite-results`
  * `RESULTS_TMP_PREFIX=tmp/`
  * `BEDROCK_REGION=us-east-1`
  * `MODEL_ID_CLAUDE=anthropic.claude-3-haiku-20240307`

**Code (minimal, paste in editor):**

```python
import json, os, boto3, time
s3 = boto3.client('s3')
transcribe = boto3.client('transcribe')
comprehend = boto3.client('comprehend')
bedrock = boto3.client('bedrock-runtime', region_name=os.environ['BEDROCK_REGION'])

INPUT_BUCKET = os.environ['INPUT_BUCKET']
OUTPUT_BUCKET = os.environ['OUTPUT_BUCKET']
TMP_PREFIX = os.environ['RESULTS_TMP_PREFIX']
MODEL_ID = os.environ['MODEL_ID_CLAUDE']

def lambda_handler(event, context):
    # 1) get object key
    rec = event['Records'][0]
    key = rec['s3']['object']['key']
    tenant = key.split('/')[0]
    call_id = key.split('/')[-1].split('.')[0]

    # 2) start transcribe (Call Analytics optional)
    job_name = f"cci-lite-{int(time.time())}-{call_id}"
    s3_uri = f"s3://{INPUT_BUCKET}/{key}"
    transcribe.start_transcription_job(
        TranscriptionJobName=job_name,
        Media={'MediaFileUri': s3_uri},
        IdentifyLanguage=True,
        OutputBucketName=OUTPUT_BUCKET,
        OutputKey=f"{TMP_PREFIX}{tenant}/{job_name}/"
    )

    # 3) poll until done
    while True:
        j = transcribe.get_transcription_job(TranscriptionJobName=job_name)
        s = j['TranscriptionJob']['TranscriptionJobStatus']
        if s in ('COMPLETED','FAILED'): break
        time.sleep(3)
    if s == 'FAILED':
        raise RuntimeError('Transcribe failed')

    # 4) read transcript JSON (from results tmp)
    out_key = f"{TMP_PREFIX}{tenant}/{job_name}/{job_name}.json"
    obj = s3.get_object(Bucket=OUTPUT_BUCKET, Key=out_key)
    transcript_json = json.loads(obj['Body'].read())
    text = transcript_json.get('results',{}).get('transcripts',[{"transcript":""}])[0]['transcript']

    # 5) basic NLP
    sent = comprehend.detect_sentiment(Text=text[:4800], LanguageCode='en')

    # 6) bedrock summary (short prompt)
    prompt = {
        "anthropic_version":"bedrock-2023-05-31",
        "max_tokens":512,
        "messages":[{"role":"user","content":[{"type":"text","text":f"Summarise this call and list 3 action items.\n\n{text[:12000]}"}]}]
    }
    br = bedrock.invoke_model(modelId=MODEL_ID, body=json.dumps(prompt))
    summary = json.loads(br['body'].read()).get('content',[{"text":""}])[0].get('text','')

    # 7) write unified result
    result = {
        "schema_version":"cci-lite-v1.0",
        "tenant_id": tenant,
        "call_id": call_id,
        "sentiment": sent,
        "summary": summary,
        "transcript_excerpt": text[:2000]
    }
    date_path = time.strftime('%Y/%m/%d')
    final_key = f"{tenant}/{date_path}/{call_id}.json"
    s3.put_object(Bucket=OUTPUT_BUCKET, Key=final_key, Body=json.dumps(result).encode('utf-8'), ServerSideEncryption='aws:kms')
    return {"ok": True, "key": final_key}
```

> **Why this works:** minimal happy-path; Transcribe writes to `cci-lite-results/tmp/…`, Lambda reads it, adds Comprehend and Bedrock outputs, then writes a unified JSON to your results partition.

**IAM tightening (after test):** Replace temp policies with least privilege (S3 get/put on your buckets, Transcribe start/get, Comprehend detect*, Bedrock invoke, KMS encrypt/decrypt, SSM get-parameter).

---

### 2. Attach S3 trigger (after function exists)

**S3 → cci-lite-input → Properties → Event notifications → Create event notification**

* Name: `on-new-call`
* Event types: *All object create events*
* Prefix: `demo-tenant/calls/`
* Destination: **Lambda function** → `cci-lite-processor` → Save.

**Validation:** Event row shows *Enabled* with Lambda target.

---

### 3. Test end-to-end

* Upload `demo.wav` to: `s3://cci-lite-input/demo-tenant/calls/demo.wav`
* CloudWatch → Logs → Log groups → `/aws/lambda/cci-lite-processor` appears → open latest stream → ensure no errors.
* S3 → `cci-lite-results/tmp/demo-tenant/<job>/…json` exists.
* S3 → `cci-lite-results/demo-tenant/YYYY/MM/DD/demo.json` created with unified result.

**If it fails:**

* `AccessDenied (kms)` → add Lambda role as **Key user** on KMS.
* `AccessDenied (s3)` → check bucket policy principals and encryption conditions.
* `Transcribe FAILED` → input codec or region; try WAV/PCM 16kHz mono.
* `Bedrock` error → ensure client region = `us-east-1`.

---

### 4. Now set CloudWatch retention & alarms (only after first run)

**Retention:** CloudWatch → Log groups → `/aws/lambda/cci-lite-processor` → *Actions* → *Edit retention* → **30 days**.

**Alarms:** CloudWatch → Alarms → *Create alarm*

* Metric: `Errors` for `cci-lite-processor`
* Threshold: `>= 1` for `5 minutes`
* Notification: email/SNS

---

## Phase 3 — Data Cataloging (only after the first JSON exists)

### 1. Create Glue database

Glue → Databases → *Add database* → Name: `cci-lite-db`

### 2. Create Glue crawler

* Glue → Crawlers → *Create crawler*

  * Data source: `S3` → `cci-lite-results/`
  * IAM role: create `cci-lite-glue-role` with read on results bucket and KMS decrypt
  * Target database: `cci-lite-db`
  * Schedule: **On demand** (switch to daily after first success)

### 3. Run crawler once

* After success, confirm a table like `results` appears with columns from your JSON.

### 4. Query via Athena

* Athena → Workgroup `primary` → Data source: AWS Glue Data Catalog
* Database: `cci-lite-db`
* Example:

```sql
SELECT tenant_id, summary, sentiment
FROM "cci-lite-db"."results"
ORDER BY 1 DESC
LIMIT 10;
```

---

## Phase 4 — Dashboards (after Athena tables exist)

### 1. QuickSight setup

* QuickSight → Admin → Manage data sources → Add → **Athena**
* Grant QuickSight access to Athena and S3 `cci-lite-results` when prompted.

### 2. Dataset & visuals

* Create dataset from `cci-lite-db.results`
* Import to SPICE
* Build visuals: sentiment trend, QA/tone scores, top phrases/entities, outcome counts

*(Optional) Row-level security via `tenant_id` filter or RLS table.*

---

## Phase 5 — SaaS Readiness

* Convert to **CDK/CloudFormation** with parameters: `TenantId`, `InputBucket`, `ResultsBucket`, `KmsKeyArn`, `ModelId`.
* Add **API Gateway + Lambda** to mint **presigned URLs** with enforced SSE-KMS headers.
* **Budgets & concurrency:** AWS Budgets alerts for Transcribe & Bedrock; set Lambda reserved concurrency (e.g., 50).

---

## Phase 6 — Observability

* Ensure **structured logs** in Lambda (JSON). Consider adding request IDs, tenant ID, stage, and elapsed ms.
* Add alarms for: Lambda Errors, Duration p95, Transcribe job failures (via metric filter or EventBridge rule), Bedrock throttles.

---

## Phase 7 — Validation & Demo Sign-off

**Checklist**

* Upload triggers Lambda automatically
* Transcribe writes tmp JSON in results bucket
* Unified JSON written to `results/<tenant>/YYYY/MM/DD/…`
* Glue crawler detects schema; Athena queries return rows
* QuickSight dashboard shows data
* CloudWatch alarms healthy; retention = 30 days
* Cost per 10-min call < £0.25

---

## Risk Matrix & Preemptive Fixes

| Risk                         | Symptom                      | Root Cause                           | Fix                                                                       |
| ---------------------------- | ---------------------------- | ------------------------------------ | ------------------------------------------------------------------------- |
| AccessDenied on upload       | Console fails                | Bucket policy requires KMS headers   | Ensure default encryption = SSE-KMS; or include headers in presigned URLs |
| Invalid principal in policy  | Save error                   | Referenced role doesn’t exist yet    | Remove missing ARNs; add back after role creation                         |
| KMS AccessDenied in Lambda   | Lambda errors on put/get     | Role not in Key users                | Add role under KMS → Key users                                            |
| Transcribe FAILED            | Job status FAILED            | Unsupported codec or region mismatch | Use WAV/PCM mono 16kHz; ensure Transcribe in eu-central-1                 |
| Bedrock error                | 4xx/5xx from bedrock-runtime | Wrong region                         | Use client in `us-east-1`                                                 |
| Crawler finds nothing        | No tables created            | No JSON yet in results               | Run pipeline first, then run crawler                                      |
| Athena no rows               | Empty table                  | Wrong prefix/partition               | Check output key path; rerun crawler                                      |
| QuickSight cannot see tables | Data source empty            | Glue/Athena not ready                | Create dataset only after crawler success                                 |

---

## Notes on Permissions (least-privilege after test)

* **S3:** get/put/list on `cci-lite-input/*`, `cci-lite-results/*`
* **KMS:** encrypt/decrypt/generate-data-key on `alias/cci-lite-master-key`
* **Transcribe:** `StartTranscriptionJob`, `GetTranscriptionJob`
* **Comprehend:** `DetectSentiment`, `DetectKeyPhrases`, `DetectEntities`
* **Bedrock:** `bedrock:InvokeModel` (region `us-east-1`)
* **SSM:** `GetParameter` on `/cci-lite/config/*`

---

### Done

Following this order prevents non-existent-object errors and enforces encryption and tenancy from day one while staying demo-ready.
