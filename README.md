# Conversant CCI Lite — End‑to‑End AWS Console Click‑Through Guide (v1.1)

> **Scope**: Single‑account, multi‑tenant, fully serverless pipeline using **S3 → EventBridge → Lambda (Init) → Transcribe (PII redaction) → SNS/EventBridge → Lambda (Result) → Comprehend (batch) → Bedrock → S3(JSON) → Glue → Athena → QuickSight**. No Step Functions. Implements all fixes noted in the v1.1 plan (failover events, decoupled jobs, KMS context, RLS, throttling, cleanup, budgets).

> **Outcome**: A working environment that turns uploaded audio into AI‑enriched JSON and dashboards, with tenant isolation and cost guardrails.

---

## 0) Prerequisites and Naming

Pick concise names and stick to them:

* **Region**: `eu-central-1` (Frankfurt). Bedrock calls will be cross‑region to `us-east-1` if needed.
* **Tenant example**: `demo-tenant`
* **Buckets**:

  * `cci-lite-input-<accountid>-eu-central-1`
  * `cci-lite-results-<accountid>-eu-central-1`
  * `cci-lite-athena-staging-<accountid>-eu-central-1`
* **KMS key**: `cci-lite-master-key`
* **Roles**:

  * Lambda exec: `cci-lite-lambda-role`
  * Glue crawler: `cci-lite-glue-role`
  * QuickSight access: `cci-lite-quicksight-role`
* **Lambdas**:

  * `cci-lite-job-init`
  * `cci-lite-result-handler`
* **SNS topic**: `TranscribeJobStatusTopic`
* **EventBridge rules**:

  * `S3ObjectCreatedToJobInit`
  * `TranscribeCompletedToResultHandler`
* **DynamoDB table**: `cci-lite-config`
* **Glue DB**: `cci-lite-db`
* **Glue Crawler**: `cci-lite-results-crawler`
* **Athena Workgroups**: `cci-lite-shared`, `tenant-<tenant_id>` (e.g., `tenant-demo-tenant`)

> **Tip**: Replace `<accountid>` and `<tenant_id>` everywhere. Avoid spaces and uppercase characters.

---

## 1) KMS — Customer Managed Key with Tenant Context

**Console path**: KMS → Keys → Create key → Symmetric → Next.

1. **Key type**: Symmetric, Encrypt/Decrypt.
2. **Alias**: `alias/cci-lite-master-key`.
3. **Key admins**: Your admin role.
4. **Key users**: Add Lambda, Glue, Athena, Transcribe, Comprehend, SNS, EventBridge service roles you will create later.
5. Create the key and note the **Key ARN**.

**Add a key policy condition for tenant context (after roles exist):**

* KMS → `alias/cci-lite-master-key` → Key policy → Edit → add condition allowing decryption only when `kms:EncryptionContext:tenant_id` matches caller’s `aws:PrincipalTag/tenant_id` **for production**. For this guide, we scope by roles and buckets first. You can add context enforcement later once tenant tags are in place.

---

## 2) S3 — Buckets, Encryption, Versioning, Lifecycle

**Console path**: S3 → Create bucket.

Create three buckets:

1. **Input**: `cci-lite-input-<accountid>-eu-central-1`

   * Block Public Access: **On**.
   * Default encryption: **AWS KMS** → choose `cci-lite-master-key`.
   * Versioning: **Enable**.
   * Lifecycle rule: `delete-input-after-1d` → Filter: prefix `*/calls/` → Expiration: 1 day.

2. **Results**: `cci-lite-results-<accountid>-eu-central-1`

   * Same security settings.
   * Lifecycle rule: `delete-results-after-30d` → Expiration: 30 days.

3. **Athena staging**: `cci-lite-athena-staging-<accountid>-eu-central-1`

   * Same security settings. No lifecycle needed.

**Folder structure (create as needed):**

```
s3://cci-lite-input-.../<tenant_id>/calls/
s3://cci-lite-results-.../<tenant_id>/YYYY/MM/DD/
```

**Enable EventBridge for S3** (failover to Lambda trigger):

* S3 → `cci-lite-input-...` → Properties → Event notifications → **Send events to EventBridge** = **On**.

---

## 3) IAM — Roles and Policies

### 3.1 Lambda Execution Role — `cci-lite-lambda-role`

**Console path**: IAM → Roles → Create role → Trusted entity: AWS service → **Lambda** → Next.

Attach policies (inline JSON where needed):

**S3 access (input/results/athena-staging):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject","s3:PutObject","s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::cci-lite-input-<accountid>-eu-central-1",
        "arn:aws:s3:::cci-lite-input-<accountid>-eu-central-1/*",
        "arn:aws:s3:::cci-lite-results-<accountid>-eu-central-1",
        "arn:aws:s3:::cci-lite-results-<accountid>-eu-central-1/*",
        "arn:aws:s3:::cci-lite-athena-staging-<accountid>-eu-central-1",
        "arn:aws:s3:::cci-lite-athena-staging-<accountid>-eu-central-1/*"
      ]
    }
  ]
}
```

**KMS decrypt/encrypt:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["kms:Encrypt","kms:Decrypt","kms:GenerateDataKey"],
      "Resource": "<KMS_KEY_ARN>"
    }
  ]
}
```

**Transcribe:**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "transcribe:StartTranscriptionJob",
      "transcribe:GetTranscriptionJob",
      "transcribe:ListTranscriptionJobs"
    ],
    "Resource": "*"
  }]
}
```

**Comprehend (batch):**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "comprehend:BatchDetectSentiment",
      "comprehend:BatchDetectEntities",
      "comprehend:BatchDetectKeyPhrases"
    ],
    "Resource": "*"
  }]
}
```

**Bedrock runtime (cross‑region):**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "bedrock:InvokeModel",
      "bedrock:InvokeModelWithResponseStream"
    ],
    "Resource": "*"
  }]
}
```

**SNS + EventBridge + SQS (used by handlers):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect":"Allow","Action":["sns:Publish","sns:Subscribe","sns:CreateTopic","sns:GetTopicAttributes"],"Resource":"*"},
    {"Effect":"Allow","Action":["events:PutEvents","events:DescribeRule","events:PutRule","events:PutTargets"],"Resource":"*"},
    {"Effect":"Allow","Action":["sqs:SendMessage","sqs:ReceiveMessage","sqs:DeleteMessage","sqs:GetQueueAttributes"],"Resource":"*"}
  ]
}
```

**CloudWatch Logs/X‑Ray:** Attach AWS managed `AWSLambdaBasicExecutionRole` and `AWSXRayDaemonWriteAccess`.

> You can merge the above into a single inline policy or split per service. Keep the scope to the named resources where possible.

### 3.2 Glue Crawler Role — `cci-lite-glue-role`

**Console path**: IAM → Roles → Create role → **Glue** → Next.

* Attach `AWSGlueServiceRole`.
* Inline policy for S3 results + KMS:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect":"Allow","Action":["s3:ListBucket","s3:GetObject"],"Resource":[
      "arn:aws:s3:::cci-lite-results-<accountid>-eu-central-1",
      "arn:aws:s3:::cci-lite-results-<accountid>-eu-central-1/*"
    ]},
    {"Effect":"Allow","Action":["kms:Decrypt"],"Resource":"<KMS_KEY_ARN>"}
  ]
}
```

### 3.3 QuickSight Role — `cci-lite-quicksight-role`

If using Enterprise edition with identity‑based access to Athena:

* Trust: `quicksight.amazonaws.com`
* Permissions: `athena:*` on workgroups, `s3:GetObject` on results and staging, `glue:Get*` on Data Catalog, KMS decrypt on key.

---

## 4) DynamoDB — Config Table for Tenants

**Console path**: DynamoDB → Tables → Create table.

* Table name: `cci-lite-config`
* Partition key: `pk` (String)
* Sort key: `sk` (String)
* On‑demand capacity.

**Seed minimal items** (use Console → Explore table items → Create item):

* Tenant metadata:

  * `pk = TENANT#demo-tenant`, `sk = META#v1`, other attributes: `{ "name": "Demo Tenant", "bedrockModel": "anthropic.claude-3-haiku-20240307" }`
* Model config:

  * `pk = CONFIG#MODELS`, `sk = bedrock`, `{ "default": "anthropic.claude-3-haiku-20240307" }`

---

## 5) SNS — Topic for Transcribe Job Status

**Console path**: SNS → Topics → Create topic.

* Type: Standard
* Name: `TranscribeJobStatusTopic`
* Encryption: KMS key `cci-lite-master-key`.

Copy the **Topic ARN**.

---

## 6) EventBridge — Rules

### 6.1 S3 Object Created → Lambda #1 (Job Init)

**Console path**: EventBridge → Rules → Create rule.

* Name: `S3ObjectCreatedToJobInit`
* Event bus: default
* Rule type: **Rule with an event pattern**
* Pattern: **AWS events or EventBridge partner events** → **S3** → **Object Created** → bucket equals `cci-lite-input-…`
* Add **condition** on key prefix: you cannot filter by prefix directly in pattern; add a **target input transformer** or filter in Lambda. Keep Lambda filter simple by checking key startswith `<tenant_id>/calls/`.
* Target: Lambda function `cci-lite-job-init`

### 6.2 Transcribe Completed → Lambda #2 (Result Handler)

* Name: `TranscribeCompletedToResultHandler`
* Pattern source: **Transcribe** → **Transcription Job State Change** → `TranscriptionJobStatus = COMPLETED`
* Target: Lambda `cci-lite-result-handler`

> Optional: Add a second rule for `FAILED` to a dead‑letter queue or notification Lambda.

---

## 7) SQS — Throttling Queues (Optional but Recommended)

Create one queue `cci-lite-analysis-queue` to buffer Comprehend/Bedrock work when many jobs finish at once.

* SQS → Create queue → Standard → Name `cci-lite-analysis-queue`.
* Encryption: KMS key.
* Configure Lambda #2 to push work to SQS, and a third Lambda `cci-lite-analyzer` to process messages at a controlled concurrency (e.g., reserved concurrency = 10). If you prefer simplicity, keep analysis inside Lambda #2 and rely on its reserved concurrency.

---

## 8) Lambda Functions

### 8.1 Shared Settings

* **Runtime**: Python 3.12
* **Role**: `cci-lite-lambda-role`
* **Env vars** (both functions, adjust per function):

```
INPUT_BUCKET=cci-lite-input-<accountid>-eu-central-1
OUTPUT_BUCKET=cci-lite-results-<accountid>-eu-central-1
ATHENA_STAGING_BUCKET=cci-lite-athena-staging-<accountid>-eu-central-1
KMS_KEY_ARN=<KMS_KEY_ARN>
SNS_TOPIC_ARN=<SNS_TOPIC_ARN>
BEDROCK_MODEL_ID=anthropic.claude-3-haiku-20240307
BEDROCK_REGION=us-east-1
DDB_TABLE=cci-lite-config
LOG_LEVEL=INFO
```

* **Monitoring**: Enable X‑Ray tracing. Set log retention 30d.

> Reserved concurrency: set 50 across functions to cap spend. For analyzer, set 10.

### 8.2 `cci-lite-job-init` — Start Transcribe with PII Redaction

**Console path**: Lambda → Create function → Author from scratch.

* Name: `cci-lite-job-init`
* Timeout: 2 minutes
* Memory: 1024 MB

**Code (minimal reference outline, paste into console editor and adjust):**

```python
import os, json, urllib.parse
import boto3

s3 = boto3.client('s3')
transcribe = boto3.client('transcribe')
sns_arn = os.environ['SNS_TOPIC_ARN']

def lambda_handler(event, context):
    # Handle both EventBridge S3 event and direct S3 trigger (defensive)
    records = event.get('Records') or []
    if not records and 'detail' in event:
        # EventBridge S3 event
        detail = event['detail']
        bucket = detail['bucket']['name']
        key = detail['object']['key']
    else:
        r = records[0]
        bucket = r['s3']['bucket']['name']
        key = urllib.parse.unquote_plus(r['s3']['object']['key'])

    # Only process /calls/
    if '/calls/' not in key:
        return {"skipped": True, "key": key}

    tenant_id = key.split('/')[0]
    job_name = f"cci_{tenant_id}_{key.split('/')[-1].replace('.', '_')}"
    media_uri = f"s3://{bucket}/{key}"

    transcribe.start_transcription_job(
        TranscriptionJobName=job_name,
        Media={"MediaFileUri": media_uri},
        LanguageCode="en-GB",
        OutputBucketName=os.environ['OUTPUT_BUCKET'],
        OutputKey=f"tmp/{tenant_id}/",
        Settings={
            "ShowSpeakerLabels": False,
            "ChannelIdentification": False
        },
        ContentRedaction={
            "RedactionType": "PII",
            "RedactionOutput": "redacted"
        },
        JobExecutionSettings={
            "AllowDeferredExecution": True,
            "DataAccessRoleArn": os.environ.get('TRANSCRIBE_ROLE_ARN', '')
        },
        Subtitles={"Formats": ["vtt"]},
        Tags=[{"Key":"tenant_id","Value":tenant_id}],
        OutputEncryptionKMSKeyId=os.environ['KMS_KEY_ARN'],
        Notifications={
            "Completed": sns_arn,
            "Failed": sns_arn
        }
    )

    return {"started": True, "job": job_name, "tenant": tenant_id}
```

> If your account/region requires a Transcribe data access role, create it and set `TRANSCRIBE_ROLE_ARN` env var.

### 8.3 `cci-lite-result-handler` — Fetch Transcript, Analyze, Write Unified JSON

* Name: `cci-lite-result-handler`
* Trigger: EventBridge rule `TranscribeCompletedToResultHandler` (add target now)
* Timeout: 5 minutes
* Memory: 1536 MB

**Add EventBridge trigger**: Lambda → Triggers → Add trigger → EventBridge → choose rule.

**Code (outline):**

```python
import os, json, boto3, datetime, urllib.parse

s3 = boto3.client('s3')
transcribe = boto3.client('transcribe')
comprehend = boto3.client('comprehend')
bedrock = boto3.client('bedrock-runtime', region_name=os.environ['BEDROCK_REGION'])

OUTPUT_BUCKET = os.environ['OUTPUT_BUCKET']
MODEL_ID = os.environ['BEDROCK_MODEL_ID']


def _load_transcript_from_s3(output_bucket, job_name, tenant_id):
    # Transcribe puts JSON under the OutputKey prefix; list and fetch the JSON
    prefix = f"tmp/{tenant_id}/"
    resp = s3.list_objects_v2(Bucket=output_bucket, Prefix=prefix)
    for o in resp.get('Contents', []):
        if o['Key'].endswith('.json'):
            body = s3.get_object(Bucket=output_bucket, Key=o['Key'])['Body'].read()
            return json.loads(body)
    raise RuntimeError('Transcript JSON not found')


def _bedrock_summarize(text):
    prompt = (
        "You are analyzing a customer service call. Return JSON with fields: "
        "issue, resolution, outcome, action_items[], qa_score(0-100), tone_score(0-100), sentiment_summary.\n"
        f"Transcript:\n{text[:15000]}"
    )
    body = {"anthropic_version":"bedrock-2023-05-31","messages":[{"role":"user","content":[{"type":"text","text":prompt}]}],"max_tokens":1500}
    resp = bedrock.invoke_model(modelId=MODEL_ID, body=json.dumps(body))
    payload = json.loads(resp['body'].read())
    # Extract text content depending on provider structure
    content = payload.get('output_text') or payload.get('content', [{}])[0].get('text') or json.dumps(payload)
    # Expect JSON in content; be defensive
    try:
        return json.loads(content)
    except:
        return {"summary_raw": content}


def lambda_handler(event, context):
    detail = event.get('detail', {})
    job_name = detail.get('TranscriptionJobName') or detail.get('TranscriptionJob')['TranscriptionJobName']

    # Derive tenant from job name convention if not provided
    parts = job_name.split('_')
    tenant_id = parts[1] if len(parts) > 2 else 'unknown'

    # Get transcript JSON from S3 output
    transcript = _load_transcript_from_s3(OUTPUT_BUCKET, job_name, tenant_id)

    # Compose a single text string for NLP
    items = transcript.get('results', {}).get('transcripts', [])
    call_text = '\n'.join([t.get('transcript','') for t in items])[:45000]

    # Comprehend batch APIs expect list of up to 25 docs; use 1 doc
    comp_in = [call_text]
    sentiment = comprehend.batch_detect_sentiment(TextList=comp_in, LanguageCode='en')
    phrases = comprehend.batch_detect_key_phrases(TextList=comp_in, LanguageCode='en')
    entities = comprehend.batch_detect_entities(TextList=comp_in, LanguageCode='en')

    # Bedrock summary
    ai = _bedrock_summarize(call_text)

    # Unified JSON (cci-lite-v1.1)
    now = datetime.datetime.utcnow()
    out = {
        "tenant_id": tenant_id,
        "call_id": job_name,
        "call_date": now.strftime('%Y-%m-%d'),
        "sentiment": sentiment.get('ResultList',[{}])[0],
        "key_phrases": phrases.get('ResultList',[{}])[0],
        "entities": entities.get('ResultList',[{}])[0],
        "ai_summary": ai,
        "source": {
            "transcribe_job": job_name,
            "pii_redacted": True
        }
    }

    key = f"{tenant_id}/{now.year:04d}/{now.month:02d}/{now.day:02d}/{job_name}.json"
    s3.put_object(Bucket=OUTPUT_BUCKET, Key=key, Body=json.dumps(out).encode('utf-8'), ServerSideEncryption='aws:kms', SSEKMSKeyId=os.environ['KMS_KEY_ARN'])

    return {"written": f"s3://{OUTPUT_BUCKET}/{key}"}
```

> For heavy loads, push a message to SQS with `{ bucket, key }` and move Comprehend/Bedrock to `cci-lite-analyzer`.

---

## 9) CloudWatch — Logs, Metrics, Alarms

**Console path**: CloudWatch → Logs → Log groups.

* Confirm both Lambdas produce logs.

**Alarms**: CloudWatch → Alarms → Create alarm

* Metric: Lambda Errors ≥ 1 (per function) → action: email/Slack.
* Metric: `bedrock` latency (custom log metric filter if desired) > 10s.
* Metric: Transcribe failures (use EventBridge rule for FAILED → SNS/email).

Retention: 30 days per log group.

---

## 10) Glue — Database and Crawler

**Console path**: Glue → Data Catalog → Databases → Add database → `cci-lite-db`.

**Crawler**: Glue → Crawlers → Create crawler

* Name: `cci-lite-results-crawler`
* Data source: S3 → `cci-lite-results-...`
* IAM role: `cci-lite-glue-role`
* Schedule: Daily
* Output: Database `cci-lite-db` → Create table
* Classifiers: JSON (default). Ensure it infers `tenant_id` and partitions by path (`tenant_id` + `YYYY/MM/DD`). If not, keep table unpartitioned initially; you can promote to partitioned via ETL later.

Run the crawler once and verify a table appears (e.g., `results`).

---

## 11) Athena — Workgroups, Query, and Staging

**Console path**: Athena → Workgroups → Create workgroup

* `cci-lite-shared` → Query result location: `s3://cci-lite-athena-staging-.../shared/`
* For each tenant: `tenant-demo-tenant` → Result location `s3://cci-lite-athena-staging-.../tenant-demo-tenant/`

**Settings**: Enable **enforce workgroup settings**, set bytes scanned limit if desired.

**Query** (Athena → Query editor → Workgroup `cci-lite-shared`, Database `cci-lite-db`):

```sql
SELECT tenant_id, call_date,
       sentiment.ResultList[1].Sentiment AS overall,
       ai_summary.qa_score AS qa_score,
       ai_summary.tone_score AS tone_score
FROM "cci-lite-db"."results"
WHERE call_date >= date_format(current_date - interval '7' day, '%Y-%m-%d');
```

Adjust JSON accessors to match crawler output; use `json_extract_scalar` if stored as strings.

---

## 12) QuickSight — Dataset, RLS, Dashboard

**Console path**: QuickSight → Datasets → New dataset → Athena → choose workgroup `cci-lite-shared` → select `cci-lite-db` table.

* Import to SPICE.
* Create a **RLS dataset** with columns (`user_email`, `tenant_id`). Attach RLS to the main dataset.
* Build visuals: sentiment over time, QA score by agent (if supplied later), tone distribution, top key phrases.
* Schedule SPICE refresh **daily**.

---

## 13) Budgets — Cost Guardrails

**Console path**: AWS Budgets → Create budget → **Cost budget**.

* Scope: Services Transcribe, Bedrock, Lambda.
* Thresholds: e.g., £50 and £100 with email alerts.

**Lambda reserved concurrency**: Lambda → Functions → each → Concurrency → Reserve → 50 (or lower for analyzer = 10).

---

## 14) CloudTrail — Data Events and Audit

**Console path**: CloudTrail → Trails → Create trail.

* Log **Data events** for S3 (read/write) on the three buckets.
* Management events: Read/Write.
* Encryption: KMS key.

---

## 15) Security Hardening (After Smoke Tests)

* Add **KMS encryption context** requirements once you tag principals by tenant:

  * Tag tenant‑scoped roles/users with `tenant_id`.
  * Update KMS key policy conditions to require `kms:EncryptionContext:tenant_id == ${aws:PrincipalTag/tenant_id}`.
* S3 bucket policies to only allow access if `s3:ExistingObjectTag/tenant_id` matches caller tag.
* Add object tag `tenant_id` when writing results.

---

## 16) Validation — End‑to‑End Tests

1. Upload a `.wav` file to `s3://cci-lite-input-.../demo-tenant/calls/<anyname>.wav`.
2. EventBridge rule fires → `cci-lite-job-init` starts Transcribe.
3. Transcribe completes → SNS → EventBridge → `cci-lite-result-handler` starts.
4. Check CloudWatch logs for both Lambdas: no errors.
5. Confirm JSON written under `s3://cci-lite-results-.../demo-tenant/YYYY/MM/DD/<jobname>.json`.
6. Run Glue crawler; confirm or update schema.
7. Run Athena query; verify rows.
8. Open QuickSight dashboard; confirm visuals update.
9. Confirm budgets and alarms status.

**Success criterion**: Cost per 10‑min call ≤ £0.25, JSON matches `cci-lite-v1.1` fields, dashboard renders.

---

## 17) Troubleshooting Checklist

* **No Lambda #1 trigger**: Check EventBridge rule is enabled, S3 bucket has EventBridge delivery enabled, and object path includes `/calls/`.
* **Transcribe access denied**: Ensure Transcribe can read S3 input and write to results `tmp/` prefix; add Transcribe service role if required.
* **No completion event**: Verify SNS topic ARN wired in Transcribe job, and EventBridge rule filters `COMPLETED`.
* **Results missing**: Inspect `cci-lite-result-handler` logs; verify it finds the transcript JSON under `tmp/<tenant_id>/`.
* **Comprehend throttling**: Reduce concurrency or insert SQS buffer; ensure batch API used.
* **Bedrock errors**: Confirm model access in `us-east-1`, add retry with exponential backoff; reduce prompt size.
* **Glue schema wrong**: Use a Glue ETL job to flatten JSON or convert to Parquet; rerun crawler.
* **Athena permission denied**: Ensure workgroup result location and bucket/kms permissions are correct.
* **QuickSight empty**: Refresh SPICE, check RLS mapping.

---

## 18) Optional Enhancements (Safe to Add Later)

* Third Lambda `cci-lite-analyzer` off an SQS queue for high‑volume scaling.
* Daily Glue ETL to Parquet and partitioned tables for faster Athena.
* DLQ for both Lambdas via SQS to capture failed events.
* EventBridge rule for `FAILED` Transcribe jobs → notify via SNS/Email.

---

## 19) Rollback & Cleanup

If you need to reset the environment:

* Disable EventBridge rules.
* Delete Lambda triggers and functions.
* Delete SNS topic.
* Delete Glue crawler and DB.
* Empty and delete S3 buckets.
* Delete DynamoDB table.
* Delete CloudWatch log groups and alarms.
* Delete Budgets (optional).
* Delete CloudTrail trail (optional).
* Finally, schedule KMS key deletion (after verifying nothing depends on it).

---

## 20) Operator Runbook (Daily)

* Check CloudWatch alarms state.
* Confirm budgets under threshold.
* Review Glue crawler last run success.
* Spot‑check last 24h Athena queries.
* Validate QuickSight SPICE refresh.

---

### Appendix A — Minimal RLS Mapping CSV (QuickSight)

```
user_email,tenant_id
alice@example.com,demo-tenant
bob@partner.co,partner-tenant
```

### Appendix B — Example Unified JSON (cci-lite-v1.1)

```json
{
  "tenant_id": "demo-tenant",
  "call_id": "cci_demo-tenant_sample_wav",
  "call_date": "2025-11-05",
  "sentiment": {"ResultList": [{"Sentiment": "NEUTRAL", "SentimentScore": {"Positive":0.1,"Negative":0.2,"Neutral":0.7,"Mixed":0.0}}]},
  "key_phrases": {"ResultList": [{"KeyPhrases": [{"Text": "order number"}]}]},
  "entities": {"ResultList": [{"Entities": [{"Text": "Alice","Type":"PERSON"}]}]},
  "ai_summary": {
    "issue": "Billing query",
    "resolution": "Explained invoice line items",
    "outcome": "Resolved",
    "action_items": ["Email revised invoice"],
    "qa_score": 82,
    "tone_score": 73,
    "sentiment_summary": "Mostly neutral with brief negative"
  },
  "source": {"transcribe_job": "cci_demo-tenant_sample_wav", "pii_redacted": true}
}
```

---

**Done.** Follow the sections in order. We can now run the flow step by step and validate each stage before moving on.
