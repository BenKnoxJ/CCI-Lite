# CCI Lite — Corrected and Final Implementation Plan (Console-Only)

> Region: **eu-central-1** for all core services. **us-east-1** for Bedrock (until local model availability).
> Single-account, multi-tenant. Designed for deterministic setup and error-free execution.

---

## Global Standards

* **Tags:** `Project=CCI-Lite`, `Env=prod`, `Owner=Conversant`, `Data=Voice`
* **KMS alias:** `alias/cci-lite-master-key`
* **Buckets:** `cci-lite-input`, `cci-lite-results`
* **IAM roles:**

  * `cci-lite-lambda-role` (Lambda execution)
  * `cci-lite-transcribe-role` (Transcribe output role)
* **S3 prefixes:**

  * Input: `s3://cci-lite-input/<tenant_id>/calls/`
  * Results: `s3://cci-lite-results/<tenant_id>/YYYY/MM/DD/`

---

## Phase 1 — Foundation Setup

### 1. Create KMS Key

**KMS → Create key → Symmetric.**
Alias: `alias/cci-lite-master-key`. Add your IAM user as administrator.

Then under **Key Users**, add:

* Your IAM user
* (later) `cci-lite-lambda-role`
* (later) `cci-lite-transcribe-role`

**Validation:** Key alias appears, and Key Users are visible.

---

### 2. Create S3 Buckets

#### 2.1 `cci-lite-input`

* Region: eu-central-1
* Block public access: ON
* Encryption: **SSE-KMS** using `alias/cci-lite-master-key`
* Lifecycle: `input-retention-1d` → expire current versions after **1 day** and delete markers.
* Folder: `demo-tenant/calls/`

#### 2.2 `cci-lite-results`

* Region: eu-central-1
* Encryption: **SSE-KMS** using same key
* Lifecycle: `results-retention-30d` → expire after **30 days** and delete markers.
* Folders: `tmp/`, `demo-tenant/`

#### 2.3 Bucket Policies

**Input bucket:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowLambdaAndUserUploads",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::ACCOUNT_ID:user/Ben.Knox-Johnston",
          "arn:aws:iam::ACCOUNT_ID:role/cci-lite-lambda-role"
        ]
      },
      "Action": ["s3:PutObject", "s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::cci-lite-input", "arn:aws:s3:::cci-lite-input/*"],
      "Condition": {
        "Bool": {"aws:SecureTransport": "true"},
        "StringEquals": {"s3:x-amz-server-side-encryption": "aws:kms"}
      }
    },
    {
      "Sid": "DenyUnencryptedOrNonTLS",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::cci-lite-input/*",
      "Condition": {
        "Bool": {"aws:SecureTransport": "false"},
        "StringNotEqualsIfExists": {"s3:x-amz-server-side-encryption": "aws:kms"}
      }
    }
  ]
}
```

**Results bucket:** similar but no deny block required.

**Validation:** Upload a file manually. Confirm `aws:kms` encryption. CLI upload without SSE-KMS → denied.

---

### 3. IAM Roles

#### 3.1 Lambda Role: `cci-lite-lambda-role`

**Permissions (initial):**

* `AmazonS3FullAccess`
* `CloudWatchLogsFullAccess`

After first test, replace with:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow", "Action": ["s3:GetObject", "s3:PutObject"], "Resource": ["arn:aws:s3:::cci-lite-*/*"]},
    {"Effect": "Allow", "Action": ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"], "Resource": "*"},
    {"Effect": "Allow", "Action": ["transcribe:*", "comprehend:*"], "Resource": "*"},
    {"Effect": "Allow", "Action": ["bedrock:InvokeModel"], "Resource": "*"}
  ]
}
```

#### 3.2 Transcribe Role: `cci-lite-transcribe-role`

Required for Transcribe to write job output.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "transcribe.amazonaws.com"},
      "Action": "sts:AssumeRole"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject"],
      "Resource": "arn:aws:s3:::cci-lite-results/*"
    }
  ]
}
```

**Add both roles as Key Users** in KMS.

---

### 4. Parameter Store

**/cci-lite/config/models**:

```json
{"bedrock_model_summary": "anthropic.claude-3-haiku-20240307", "bedrock_region": "us-east-1"}
```

**/cci-lite/config/tenants/demo-tenant**:

```json
{"tenant_id": "demo-tenant", "retention_days_results": 30, "retention_days_input": 1}
```

**Phase 1 Complete. Do not configure CloudWatch or QuickSight yet.**

---

## Phase 2 — Core Pipeline

### 1. Lambda Function `cci-lite-processor`

**Runtime:** Python 3.12
**Memory:** 1024 MB
**Timeout:** 5 minutes
**Trigger:** S3 event (suffix: `.wav`, `.mp3`) on `cci-lite-input`

**Env Vars:**

```
INPUT_BUCKET=cci-lite-input
OUTPUT_BUCKET=cci-lite-results
TMP_PREFIX=tmp/
BEDROCK_REGION=us-east-1
MODEL_ID_CLAUDE=anthropic.claude-3-haiku-20240307
TRANSCRIBE_ROLE_ARN=arn:aws:iam::ACCOUNT_ID:role/cci-lite-transcribe-role
```

**Code:**

```python
import json, os, time, boto3
s3 = boto3.client('s3')
transcribe = boto3.client('transcribe')
comprehend = boto3.client('comprehend')
bedrock = boto3.client('bedrock-runtime', region_name=os.environ['BEDROCK_REGION'])

def lambda_handler(event, context):
    record = event['Records'][0]
    key = record['s3']['object']['key']
    tenant = key.split('/')[0]
    call_id = key.split('/')[-1].split('.')[0]

    job_name = f"cci-lite-{int(time.time())}-{call_id}"
    media_uri = f"s3://{os.environ['INPUT_BUCKET']}/{key}"

    transcribe.start_transcription_job(
        TranscriptionJobName=job_name,
        Media={'MediaFileUri': media_uri},
        IdentifyLanguage=True,
        OutputBucketName=os.environ['OUTPUT_BUCKET'],
        OutputKey=f"{os.environ['TMP_PREFIX']}{tenant}/{job_name}/",
        OutputEncryptionKMSKeyId='alias/cci-lite-master-key',
        DataAccessRoleArn=os.environ['TRANSCRIBE_ROLE_ARN']
    )

    while True:
        status = transcribe.get_transcription_job(TranscriptionJobName=job_name)
        s = status['TranscriptionJob']['TranscriptionJobStatus']
        if s in ('COMPLETED', 'FAILED'):
            break
        time.sleep(5)

    if s == 'COMPLETED':
        transcript_uri = status['TranscriptionJob']['Transcript']['TranscriptFileUri']
        tr_data = json.loads(s3.get_object(Bucket=os.environ['OUTPUT_BUCKET'], Key=f"{os.environ['TMP_PREFIX']}{tenant}/{job_name}/transcription.json")['Body'].read())
        text = tr_data['results']['transcripts'][0]['transcript']

        sentiment = comprehend.detect_sentiment(Text=text, LanguageCode='en')

        prompt = f"Summarise this call:\n{text}\nReturn JSON with summary, sentiment, and action items."
        br_resp = bedrock.invoke_model(
            modelId=os.environ['MODEL_ID_CLAUDE'],
            body=json.dumps({"prompt": prompt, "max_tokens": 500})
        )
        ai_output = json.loads(br_resp['body'].read())

        final = {
            "tenant": tenant,
            "call_id": call_id,
            "summary": ai_output.get('completion', ''),
            "sentiment": sentiment,
            "timestamp": time.strftime('%Y-%m-%dT%H:%M:%SZ', time.gmtime())
        }

        out_key = f"{tenant}/{time.strftime('%Y/%m/%d')}/{call_id}.json"
        s3.put_object(Bucket=os.environ['OUTPUT_BUCKET'], Key=out_key, Body=json.dumps(final), ServerSideEncryption='aws:kms')

        return {"status": "success", "result_key": out_key}
    else:
        raise Exception(f"Transcription failed: {status}")
```

**Validation:** Upload a short `.wav` file to `demo-tenant/calls/` and verify a JSON result under `cci-lite-results/demo-tenant/YYYY/MM/DD/`.

---

### 2. CloudWatch

After first successful test, enable:

* Log retention: 30 days
* Alarms:

  * `Errors > 0`
  * `Duration > 250000 ms`

---

## Phase 3 — Data Cataloging & Analytics

1. Create Glue Database: `cci_lite`
2. Create Glue Crawler pointing to `s3://cci-lite-results/`

   * Include all tenants.
   * Schedule: manual (for now).
3. Run crawler once → verify Athena table creation.
4. Athena: confirm `summary`, `sentiment`, and `timestamp` columns.
5. QuickSight: connect to Athena dataset and build visuals.

---

## Phase 4 — Observability

* Structured logging with JSON.
* Include `call_id`, `tenant`, `status`, and `duration_ms` per execution.
* Enable basic dashboard in CloudWatch Metrics.

---

## Phase 5 — SaaS Readiness

* Parameterize tenant IDs.
* Introduce optional API Gateway endpoint for file uploads.
* Add per-tenant quotas and cost alerts via AWS Budgets.

---

## Phase 6 — Validation Checklist

* ✅ File upload triggers Lambda automatically.
* ✅ Transcribe output written to results bucket.
* ✅ Sentiment + summary JSON created.
* ✅ Queryable in Athena.
* ✅ QuickSight visual refresh works.
* ✅ CloudWatch metrics normal.

**At this stage, CCI Lite is production-ready and conforms to AWS security and compliance best practices.**
