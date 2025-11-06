# ✅ CCI Lite v1.3 — Phase 4 : DynamoDB (Tenant Config + QA Templates)

### Date

2025-11-06

### Region

`eu-central-1`

### Environment

Production

---

## 1. Purpose

Phase 4 introduces DynamoDB as the tenant configuration and QA template store. It defines the schema that Bedrock, Lambda, and analytics components use to identify tenants, select models, and evaluate QA scores. Each tenant record provides metadata (model, name, etc.), while QA templates define scoring criteria.

---

## 2. Table Configuration

**Table Name:** `cci-lite-config`
**Partition Key:** `pk` (String)
**Sort Key:** `sk` (String)
**Billing Mode:** On-Demand
**Encryption:** Customer-managed KMS (`alias/cci-lite-master-key`)
**Tags:**
`Project=CCI-Lite`, `Environment=Production`, `Tenant=demo-tenant`, `Version=v1.3`, `Region=eu-central-1`

---

## 3. IAM Role Update (Lambda)

**Role:** `cci-lite-lambda-role`

Added inline DynamoDB permissions:

```json
{
  "Sid": "DynamoDBAccess",
  "Effect": "Allow",
  "Action": [
    "dynamodb:PutItem",
    "dynamodb:GetItem",
    "dynamodb:UpdateItem",
    "dynamodb:Query",
    "dynamodb:Scan",
    "dynamodb:DescribeTable"
  ],
  "Resource": "arn:aws:dynamodb:eu-central-1:591338347562:table/cci-lite-config"
}
```

KMS permissions already existed; no additional changes required.

---

## 4. Seed Data

Two initial records were inserted for baseline configuration and QA testing.

### Tenant Metadata

```json
{
  "pk": { "S": "TENANT#demo-tenant" },
  "sk": { "S": "META#v1" },
  "name": { "S": "Demo Tenant" },
  "bedrockModel": { "S": "anthropic.claude-3-haiku-20240307" },
  "created_at": { "S": "2025-11-06T12:00:00Z" }
}
```

### QA Template — Customer Support Standard v1

```json
{
  "pk": { "S": "TENANT#demo-tenant" },
  "sk": { "S": "QA_TEMPLATE#customer_support_v1" },
  "title": { "S": "Customer Support Standard QA v1" },
  "description": { "S": "Standard evaluation form for general support calls, covering greeting, empathy, resolution, compliance, and closure." },
  "criteria": {
    "L": [
      { "M": { "id": { "S": "greet_01" }, "text": { "S": "Did the agent greet the customer politely and introduce themselves?" }, "weight": { "N": "1" }, "required": { "BOOL": true } } },
      { "M": { "id": { "S": "verify_02" }, "text": { "S": "Did the agent verify the customer’s identity and account details correctly?" }, "weight": { "N": "1" }, "required": { "BOOL": true } } },
      { "M": { "id": { "S": "empathy_03" }, "text": { "S": "Did the agent acknowledge the customer’s concern and show empathy?" }, "weight": { "N": "1" }, "required": { "BOOL": false } } },
      { "M": { "id": { "S": "resolve_04" }, "text": { "S": "Did the agent provide an accurate resolution or clear next steps?" }, "weight": { "N": "2" }, "required": { "BOOL": true } } },
      { "M": { "id": { "S": "compliance_05" }, "text": { "S": "Did the agent follow the company’s data protection and disclosure policies?" }, "weight": { "N": "1.5" }, "required": { "BOOL": true } } },
      { "M": { "id": { "S": "close_06" }, "text": { "S": "Did the agent confirm satisfaction and close the call professionally?" }, "weight": { "N": "1" }, "required": { "BOOL": false } } }
    ]
  },
  "scoring": {
    "M": {
      "pass_threshold": { "N": "0.75" },
      "notes": { "S": "Empathy and closure are bonus items; failure in required items marks call as failed." }
    }
  },
  "created_at": { "S": "2025-11-06T12:30:00Z" }
}
```

---

## 5. Validation Checklist

| Check                                           | Result                          |
| ----------------------------------------------- | ------------------------------- |
| Table created successfully                      | ✅                               |
| Encryption key applied                          | ✅ (`alias/cci-lite-master-key`) |
| Lambda IAM policy updated                       | ✅                               |
| Tenant meta inserted                            | ✅                               |
| QA template inserted                            | ✅                               |
| Query (`pk=TENANT#demo-tenant`) returns 2 items | ✅                               |

---

## 6. Key Outputs

* DynamoDB now centralizes tenant metadata and QA templates.
* The QA form is realistic, simulating a production support environment.
* `cci-lite-lambda-role` has CRUD access to this table for runtime evaluation.
* Encryption and IAM integration remain consistent with prior phases.

---

## 7. Next Phase

Proceed to **Phase 5 — Glue / Athena Integration**
Tasks:

* Create Glue Database: `cci-lite-db`
* Create Crawler: `cci-lite-results-crawler`
* Target: `s3://cci-lite-results-<accountid>-eu-central-1/demo-tenant/`
* Assign Role: `cci-lite-glue-role`
* Run crawler to populate Athena schema

---

*End of Phase 4 update.*
