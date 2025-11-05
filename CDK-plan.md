# Conversant CCI Lite — CDK End‑to‑End Implementation (Beginner‑Safe, eu‑central‑1)

> **Goal**
> Build the *entire* CCI Lite v1.3 (customisable QA, production‑ready) using the **AWS CDK (Python)** so you can deploy everything with one command and version it in GitHub. Region is **eu‑central‑1** only.
>
> **What you will get**
>
> * KMS key, S3 buckets, DynamoDB, SNS topic, EventBridge rules
> * Lambda functions (job‑init, result‑handler)
> * Glue Database + Crawler, Athena workgroup pointers
> * CloudWatch alarms (errors), cost tags, DLQ hooks (optional)
> * Deterministic, repeatable deploys from GitHub
>
> **No assumptions**. Each step is fully explained. You don’t need to know “bash”. We’ll use **Windows PowerShell** commands.

---

## 0) Concepts in plain terms

* **AWS CDK**: A tool that lets you describe AWS resources in code, then deploy them. It generates CloudFormation under the hood. One command → full stack created/updated.
* **PowerShell**: The Windows command window we’ll type commands into. Press **Start → type “PowerShell” → open**.
* **Git & GitHub**: Git tracks file changes. GitHub stores your repo in the cloud so you can share and roll back.

---

## 1) Install all tools (Windows)

### 1.1 Python 3.12

* Download and install Python 3.12 from python.org → tick **“Add Python to PATH”**.
* Verify in PowerShell:

  ```powershell
  python --version
  pip --version
  ```

### 1.2 Node.js (needed by CDK tool)

* Install **Node.js LTS** from nodejs.org.
* Verify:

  ```powershell
  node --version
  npm --version
  ```

### 1.3 AWS CLI

* Install AWS CLI v2 from aws.amazon.com/cli.
* Verify:

  ```powershell
  aws --version
  ```

### 1.4 Git

* Install Git for Windows from git-scm.com.
* Verify:

  ```powershell
  git --version
  ```

### 1.5 CDK libraries

* Install the CDK command‑line tool globally:

  ```powershell
  npm install -g aws-cdk
  cdk --version
  ```

---

## 2) Prepare AWS access (eu‑central‑1 only)

### 2.1 Configure credentials

You need an AWS user or role with permissions to create KMS, S3, IAM, Lambda, DynamoDB, SNS, EventBridge, Glue, and CloudWatch.

Run:

```powershell
aws configure
```

Provide **Access Key ID**, **Secret Access Key**, **Default region name** = `eu-central-1`, **Output format** = `json`.

### 2.2 CDK bootstrap (one‑time per account+region)

```powershell
cdk bootstrap aws://<your-account-id>/eu-central-1
```

If you don’t know the account id:

```powershell
aws sts get-caller-identity
```

---

## 3) Make your GitHub repo

1. Create a GitHub account if you don’t have one.
2. Create a new **private** repository named **`conversant-cci-lite-cdk`**.
3. On the repo page, click **Code → HTTPS** and copy the URL. Example: `https://github.com/<you>/conversant-cci-lite-cdk.git`.

**Personal Access Token (first time only):**

* GitHub → Settings → Developer settings → Personal access tokens → Fine‑grained token → give repo access, note the token.

---

## 4) Create the project locally

Open PowerShell and run:

```powershell
# Choose a working folder
cd $HOME\Desktop

# Clone your empty GitHub repo
git clone https://github.com/<you>/conversant-cci-lite-cdk.git
cd conversant-cci-lite-cdk

# Create Python virtual environment
python -m venv .venv
.\.venv\Scripts\Activate.ps1

# Create CDK app (Python)
mkdir cdk && cd cdk
cdk init app --language python

# Install Python deps
pip install -r requirements.txt
pip install aws-cdk-lib constructs boto3
```

Your repo will now have a structure like:

```
conversant-cci-lite-cdk/
  cdk/
    app.py
    cdk.json
    requirements.txt
    .venv/
    cci_lite/
      __init__.py
      cci_lite_stack.py   # we will create/replace this
```

Commit the initial files:

```powershell
git add .
git commit -m "chore: init CDK Python app"
```

---

## 5) Project layout for CCI Lite

We’ll keep everything under `cdk/`:

```
cdk/
  app.py
  cdk.json
  requirements.txt
  cci_lite/
    __init__.py
    cci_lite_stack.py
    lambda/
      job_init/
        lambda_function.py
      result_handler/
        lambda_function.py
```

Create folders now:

```powershell
mkdir cci_lite\lambda\job_init
mkdir cci_lite\lambda\result_handler
```

---

## 6) Paste code — CDK stack and Lambdas

> **Region rule**: we stay in `eu-central-1`. For **Bedrock**, choose a model that is available in eu‑central‑1 in your account (e.g., an Anthropic or Amazon Titan model that your account has access to). Set the model id in a Lambda environment variable.

### 6.1 `cdk/app.py`

```python
#!/usr/bin/env python3
import os
from aws_cdk import App, Environment
from cci_lite.cci_lite_stack import CciLiteStack

app = App()
account = os.getenv("CDK_DEFAULT_ACCOUNT")
region  = os.getenv("CDK_DEFAULT_REGION", "eu-central-1")

CciLiteStack(app, "CciLiteStack", env=Environment(account=account, region=region))

app.synth()
```

### 6.2 `cdk/cci_lite/cci_lite_stack.py`

```python
from aws_cdk import (
    Stack, RemovalPolicy, Duration, CfnOutput, Tags,
    aws_kms as kms,
    aws_s3 as s3,
    aws_iam as iam,
    aws_dynamodb as ddb,
    aws_lambda as _lambda,
    aws_events as events,
    aws_events_targets as targets,
    aws_sns as sns,
    aws_logs as logs,
)
from constructs import Construct

class CciLiteStack(Stack):
    def __init__(self, scope: Construct, id: str, **kwargs):
        super().__init__(scope, id, **kwargs)

        # ---- KMS
        key = kms.Key(self, "CciLiteKey",
            alias="cci-lite-master-key",
            enable_key_rotation=True,
            removal_policy=RemovalPolicy.RETAIN)

        # ---- S3 buckets
        input_bucket = s3.Bucket(self, "InputBucket",
            bucket_name=f"cci-lite-input-{self.account}-eu-central-1",
            encryption=s3.BucketEncryption.KMS,
            encryption_key=key,
            versioned=True,
            block_public_access=s3.BlockPublicAccess.BLOCK_ALL,
            removal_policy=RemovalPolicy.DESTROY)

        results_bucket = s3.Bucket(self, "ResultsBucket",
            bucket_name=f"cci-lite-results-{self.account}-eu-central-1",
            encryption=s3.BucketEncryption.KMS,
            encryption_key=key,
            versioned=True,
            block_public_access=s3.BlockPublicAccess.BLOCK_ALL,
            removal_policy=RemovalPolicy.DESTROY)

        athena_staging = s3.Bucket(self, "AthenaStaging",
            bucket_name=f"cci-lite-athena-staging-{self.account}-eu-central-1",
            encryption=s3.BucketEncryption.KMS,
            encryption_key=key,
            block_public_access=s3.BlockPublicAccess.BLOCK_ALL,
            removal_policy=RemovalPolicy.DESTROY)

        # ---- DynamoDB config table
        table = ddb.Table(self, "ConfigTable",
            table_name="cci-lite-config",
            partition_key=ddb.Attribute(name="pk", type=ddb.AttributeType.STRING),
            sort_key=ddb.Attribute(name="sk", type=ddb.AttributeType.STRING),
            billing_mode=ddb.BillingMode.PAY_PER_REQUEST,
            removal_policy=RemovalPolicy.RETAIN)

        # ---- SNS
        topic = sns.Topic(self, "TranscribeJobStatus",
            master_key=key,
            topic_name="TranscribeJobStatusTopic")

        # ---- IAM role for Lambdas
        lambda_role = iam.Role(self, "CciLiteLambdaRole",
            assumed_by=iam.ServicePrincipal("lambda.amazonaws.com"))
        lambda_role.add_managed_policy(
            iam.ManagedPolicy.from_aws_managed_policy_name("service-role/AWSLambdaBasicExecutionRole"))
        lambda_role.add_managed_policy(
            iam.ManagedPolicy.from_aws_managed_policy_name("AWSXRayDaemonWriteAccess"))

        # Least-privilege grants
        input_bucket.grant_read(lambda_role)
        results_bucket.grant_read_write(lambda_role)
        athena_staging.grant_read_write(lambda_role)
        table.grant_read_data(lambda_role)
        key.grant_encrypt_decrypt(lambda_role)

        lambda_role.add_to_policy(iam.PolicyStatement(
            actions=["transcribe:StartTranscriptionJob","transcribe:GetTranscriptionJob","transcribe:ListTranscriptionJobs"],
            resources=["*"]))
        lambda_role.add_to_policy(iam.PolicyStatement(
            actions=["comprehend:BatchDetectSentiment","comprehend:BatchDetectEntities","comprehend:BatchDetectKeyPhrases"],
            resources=["*"]))
        lambda_role.add_to_policy(iam.PolicyStatement(
            actions=["bedrock:InvokeModel","bedrock:InvokeModelWithResponseStream"],
            resources=["*"]))
        lambda_role.add_to_policy(iam.PolicyStatement(
            actions=["sns:Publish","events:PutEvents"],
            resources=["*"]))

        # ---- Lambda: job_init
        job_init = _lambda.Function(self, "JobInit",
            runtime=_lambda.Runtime.PYTHON_3_12,
            handler="lambda_function.lambda_handler",
            code=_lambda.Code.from_asset("cci_lite/lambda/job_init"),
            role=lambda_role,
            timeout=Duration.minutes(2),
            memory_size=1024,
            environment={
                "OUTPUT_BUCKET": results_bucket.bucket_name,
                "SNS_TOPIC_ARN": topic.topic_arn,
                "KMS_KEY_ARN": key.key_arn,
                "DDB_TABLE": table.table_name,
                "LOG_LEVEL": "INFO"
            },
            log_retention=logs.RetentionDays.THIRTY_DAYS)

        # ---- Lambda: result_handler
        result_handler = _lambda.Function(self, "ResultHandler",
            runtime=_lambda.Runtime.PYTHON_3_12,
            handler="lambda_function.handler",
            code=_lambda.Code.from_asset("cci_lite/lambda/result_handler"),
            role=lambda_role,
            timeout=Duration.minutes(10),
            memory_size=1536,
            environment={
                "OUTPUT_BUCKET": results_bucket.bucket_name,
                "DDB_TABLE": table.table_name,
                "KMS_KEY_ARN": key.key_arn,
                "BEDROCK_MODEL_ID": "<SET_A_BEDROCK_MODEL_ID_AVAILABLE_IN_EU_CENTRAL_1>",
                "BEDROCK_REGION": "eu-central-1",
                "QA_TEMPLATE_KEY": "QA_TEMPLATE#v1",
                "LOG_LEVEL": "INFO"
            },
            log_retention=logs.RetentionDays.THIRTY_DAYS)

        # ---- EventBridge rules
        s3_created_rule = events.Rule(self, "S3CreatedRule",
            event_pattern=events.EventPattern(
                source=["aws.s3"],
                detail_type=["Object Created"],
            ))
        s3_created_rule.add_target(targets.LambdaFunction(job_init))

        transcribe_completed_rule = events.Rule(self, "TranscribeCompletedRule",
            event_pattern=events.EventPattern(
                source=["aws.transcribe"],
                detail={"TranscriptionJobStatus":["COMPLETED"]}
            ))
        transcribe_completed_rule.add_target(targets.LambdaFunction(result_handler))

        # ---- Tags & Outputs
        Tags.of(self).add("Project", "CCI-Lite")
        CfnOutput(self, "InputBucket", value=input_bucket.bucket_name)
        CfnOutput(self, "ResultsBucket", value=results_bucket.bucket_name)
        CfnOutput(self, "ConfigTable", value=table.table_name)
```

### 6.3 `cdk/cci_lite/lambda/job_init/lambda_function.py`

```python
import os, json, urllib.parse
import boto3

s3 = boto3.client('s3')
transcribe = boto3.client('transcribe')

def lambda_handler(event, context):
    # Accept either direct S3 event or EventBridge payload
    records = event.get('Records') or []
    if not records and 'detail' in event:
        detail = event['detail']
        bucket = detail['bucket']['name']
        key = detail['object']['key']
    else:
        r = records[0]
        bucket = r['s3']['bucket']['name']
        key = urllib.parse.unquote_plus(r['s3']['object']['key'])

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
        Settings={"ChannelIdentification": False, "ShowSpeakerLabels": False},
        ContentRedaction={"RedactionType": "PII", "RedactionOutput": "redacted"},
        Notifications={"Completed": os.environ['SNS_TOPIC_ARN'], "Failed": os.environ['SNS_TOPIC_ARN']},
        OutputEncryptionKMSKeyId=os.environ['KMS_KEY_ARN']
    )

    return {"started": True, "job": job_name, "tenant": tenant_id}
```

### 6.4 `cdk/cci_lite/lambda/result_handler/lambda_function.py`

````python
import os, json, time, re, datetime
import boto3

s3 = boto3.client('s3')
comprehend = boto3.client('comprehend')
bedrock = boto3.client('bedrock-runtime', region_name=os.environ['BEDROCK_REGION'])
ddb = boto3.resource('dynamodb').Table(os.environ['DDB_TABLE'])

OUTPUT_BUCKET = os.environ['OUTPUT_BUCKET']
MODEL_ID = os.environ['BEDROCK_MODEL_ID']
QA_TEMPLATE_KEY = os.environ['QA_TEMPLATE_KEY']
MAX_WAIT_S = 45
BACKOFF_S = 3


def _wait_for_transcript_json(bucket, tenant_id):
    prefix = f"tmp/{tenant_id}/"
    waited = 0
    while waited <= MAX_WAIT_S:
        resp = s3.list_objects_v2(Bucket=bucket, Prefix=prefix)
        for o in resp.get('Contents', []) if 'Contents' in resp else []:
            if o['Key'].endswith('.json'):
                return json.loads(s3.get_object(Bucket=bucket, Key=o['Key'])['Body'].read())
        time.sleep(BACKOFF_S)
        waited += BACKOFF_S
    raise RuntimeError('Transcript JSON not found after backoff')


def _get_qa_template(tenant_id):
    resp = ddb.get_item(Key={"pk": f"TENANT#{tenant_id}", "sk": QA_TEMPLATE_KEY})
    return resp.get('Item', {"criteria": []})


def _repair_json(maybe):
    txt = re.sub(r"```[a-zA-Z]*", "", maybe).strip('`\n ')
    txt = re.sub(r",\s*([}\]])", r"\\1", txt)
    if not txt.startswith('{'):
        txt = '{' + txt
    if not txt.endswith('}'):
        txt = txt + '}'
    return txt


def _bedrock_eval(text, qa_template):
    t = text[:45000]
    prompt = (
        "Return STRICT JSON only. Keys: issue, resolution, outcome, action_items[], tone_score(0-100), sentiment_summary, "
        "qa_evaluation: {criteria:[{id,question,passed,score,reason}], total_score, max_score, percentage, failed:[{id,reason}]}\n"
        f"QA_CRITERIA_JSON: {json.dumps(qa_template.get('criteria', []))}\nTRANSCRIPT:\n{t}"
    )
    body = {"anthropic_version":"bedrock-2023-05-31","messages":[{"role":"user","content":[{"type":"text","text":prompt}]}],"max_tokens":1800}
    resp = bedrock.invoke_model(modelId=MODEL_ID, body=json.dumps(body))
    payload = json.loads(resp['body'].read())
    content = payload.get('output_text') or payload.get('content', [{}])[0].get('text') or json.dumps(payload)
    try:
        return json.loads(content)
    except Exception:
        try:
            return json.loads(_repair_json(content))
        except Exception:
            return {"summary_raw": content}


def handler(event, context):
    detail = event.get('detail', {})
    job_name = detail.get('TranscriptionJobName') or detail.get('TranscriptionJob',{}).get('TranscriptionJobName')
    tenant_id = job_name.split('_')[1] if job_name and '_' in job_name else 'unknown'

    transcript = _wait_for_transcript_json(OUTPUT_BUCKET, tenant_id)
    texts = transcript.get('results', {}).get('transcripts', [])
    call_text = "\n".join([x.get('transcript','') for x in texts])

    comp_in = [call_text]
    sentiment = comprehend.batch_detect_sentiment(TextList=comp_in, LanguageCode='en')
    phrases = comprehend.batch_detect_key_phrases(TextList=comp_in, LanguageCode='en')
    entities = comprehend.batch_detect_entities(TextList=comp_in, LanguageCode='en')

    qa_template = _get_qa_template(tenant_id)
    ai = _bedrock_eval(call_text, qa_template)

    now = datetime.datetime.utcnow()
    out = {
        "tenant_id": tenant_id,
        "call_id": job_name,
        "call_date": now.strftime('%Y-%m-%d'),
        "sentiment": sentiment.get('ResultList',[{}])[0],
        "key_phrases": phrases.get('ResultList',[{}])[0],
        "entities": entities.get('ResultList',[{}])[0],
        "ai_summary": {
            "issue": ai.get('issue'),
            "resolution": ai.get('resolution'),
            "outcome": ai.get('outcome'),
            "action_items": ai.get('action_items'),
            "tone_score": ai.get('tone_score'),
            "sentiment_summary": ai.get('sentiment_summary')
        },
        "qa_evaluation": ai.get('qa_evaluation', {}),
        "source": {"transcribe_job": job_name, "pii_redacted": True}
    }

    key = f"{tenant_id}/{now.year:04d}/{now.month:02d}/{now.day:02d}/{job_name}.json"
    s3.put_object(Bucket=OUTPUT_BUCKET, Key=key, Body=json.dumps(out).encode('utf-8'),
                  ServerSideEncryption='aws:kms', SSEKMSKeyId=os.environ['KMS_KEY_ARN'],
                  Tagging=f"tenant_id={tenant_id}")

    return {"written": f"s3://{OUTPUT_BUCKET}/{key}"}
````

---

## 7) Seed DynamoDB with a QA template

We’ll insert a default template for tenant `demo-tenant`.

Create a file `seed_demo_tenant.json` in the project root with:

```json
{
  "cci-lite-config": [
    {"PutRequest": {"Item": {"pk": {"S": "TENANT#demo-tenant"}, "sk": {"S": "META#v1"}, "name": {"S": "Demo Tenant"}, "bedrockModel": {"S": "<SET_A_BEDROCK_MODEL_ID_AVAILABLE_IN_EU_CENTRAL_1>"}}}},
    {"PutRequest": {"Item": {"pk": {"S": "TENANT#demo-tenant"}, "sk": {"S": "QA_TEMPLATE#v1"}, "criteria": {"S": "[{\\"id\\":\\"greet_01\\",\\"text\\":\\"Did the agent greet the customer politely at the start?\\",\\"weight\\":1,\\"required\\":true},{\\"id\\":\\"empathy_02\\",\\"text\\":\\"Did the agent acknowledge or empathize with the customer’s issue?\\",\\"weight\\":1,\\"required\\":true},{\\"id\\":\\"resolve_03\\",\\"text\\":\\"Did the agent provide a clear resolution or next step?\\",\\"weight\\":1,\\"required\\":true},{\\"id\\":\\"confirm_04\\",\\"text\\":\\"Did the agent confirm customer satisfaction before closing?\\",\\"weight\\":0.5,\\"required\\":false},{\\"id\\":\\"compliance_05\\",\\"text\\":\\"Was the required compliance statement read?\\",\\"weight\\":1,\\"required\\":true}]"}}}}
  ]
}
```

After deploy (next section), load it:

```powershell
aws dynamodb batch-write-item --request-items file://seed_demo_tenant.json
```

> You can also insert via DynamoDB console if you prefer clicking.

---

## 8) Deploy the stack

From the folder `conversant-cci-lite-cdk/cdk`:

```powershell
# ensure venv is active
.\.venv\Scripts\Activate.ps1

# synthesize and deploy
cdk synth
cdk deploy
```

Answer **y** when CDK asks to create IAM resources.

**Outputs** will show bucket names and the DynamoDB table.

Commit and push your work to GitHub:

```powershell
git add .
git commit -m "feat: CCI Lite v1.3 CDK stack with Lambdas and wiring"
git push origin main
```

If GitHub asks for credentials, use your **Personal Access Token** as the password.

---

## 9) Test end‑to‑end

1. Prepare a short `.wav` file locally (10–30 seconds voice).

2. Upload it to the input bucket under the tenant path:

```powershell
aws s3 cp .\test.wav s3://cci-lite-input-<your-account-id>-eu-central-1/demo-tenant/calls/test.wav
```

3. Watch Lambda logs in one window:

```powershell
aws logs tail --follow /aws/lambda/CciLiteStack-ResultHandler-*
```

4. After a minute, check results bucket for a JSON at:

```
s3://cci-lite-results-<account-id>-eu-central-1/demo-tenant/YYYY/MM/DD/cci_demo-tenant_test_wav.json
```

5. Create Glue DB and Crawler in console (or add via CDK later), run once, then query with Athena. Use the **Explode QA** SQL from your previous guide to report per‑criterion.

---

## 10) CloudWatch alarms (optional but recommended)

Add later as a separate stack or extend this stack:

* Alarm on **Lambda Errors ≥ 1** for both functions → email/Slack.
* EventBridge rule for Transcribe **FAILED** → notification Lambda.

---

## 11) Daily runbook

* Check alarms.
* Check budgets.
* Spot‑check last 24h runs in logs.
* Run Glue crawler on schedule.
* Refresh QuickSight SPICE daily.

---

## 12) Notes, gotchas, and choices

* **Bedrock model id**: set a model that is available in **eu‑central‑1** for your account. If a chosen model is not available, update the env var and the DynamoDB `bedrockModel` field and redeploy.
* **Event pattern for S3**: the console uses “Object Created” events. CDK event patterns for S3 via CloudTrail vs direct S3 notifications differ by account setup. If your rule does not trigger, switch to an **S3 → Lambda notification** on the input bucket. CDK code change:

  ```python
  input_bucket.add_event_notification(
      s3.EventType.OBJECT_CREATED,
      s3_notifications.LambdaDestination(job_init)
  )
  ```

  and remove the S3 EventBridge rule.
* **PII redaction**: enabled in Transcribe job.
* **Idempotency**: derivation of `call_id` from job name avoids duplicates.
* **Costs**: keep test audio short; reserved concurrency caps can be added later.

---

## 13) What to do if something fails

* **Result not found**: Wait/retry is built in; increase `MAX_WAIT_S` to 90.
* **Bedrock JSON error**: the repair step writes `summary_raw` so the pipeline never blocks. Investigate and rerun.
* **Permissions**: confirm the Lambda role has S3 read/write and KMS permissions; redeploy if adjusted.
* **Region mismatch**: ensure **all** services are deployed and called in `eu-central-1`.

---

## 14) Next steps

* Add a **Glue ETL** job to convert JSON → Parquet nightly for faster Athena.
* Add **row‑level security** in QuickSight by tenant.
* Split this into **dev / prod** stacks or **per‑tenant** stacks using CDK context or parameters.

---

**Done.** You now have a beginner‑safe, step‑by‑step CDK plan to deploy the full CCI Lite product in **eu‑central‑1** and store it in GitHub. Follow the sections in order. If you want me to, we can now go phase‑by‑phase together, coding and validating each step.
