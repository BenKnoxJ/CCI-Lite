# CCI Lite — Environment Overview

## Global
Region: eu-central-1  
Account ID: <your_account_id>  
KMS Key ARN: arn:aws:kms:eu-central-1:591338347562:key/79d23f7f-9420-4398-a848-93876d0250e5  
Naming Pattern: cci-lite-<resource>-<accountid>-eu-central-1  

## Buckets
| Name | Purpose | Encryption | Lifecycle | Tag:tenant_id |
|------|----------|-------------|------------|----------------|
| cci-lite-input-... | Incoming audio | SSE-KMS | 30d | required |
| cci-lite-results-... | Transcribe + AI output | SSE-KMS | 90d | required |
| cci-lite-athena-staging-... | Athena temp | SSE-KMS | N/A | N/A |

## IAM Roles
| Role | Attached Policies | Notes |
|------|--------------------|-------|
| cci-lite-lambda-role | AWSLambdaBasicExecutionRole, TranscribeFullAccess, ComprehendFullAccess, BedrockInvokeModel, S3FullAccess, KMSDecrypt | Used by all Lambdas |
| cci-lite-glue-role | AWSGlueServiceRole, S3Access, KMSDecrypt | For crawlers |
| cci-lite-quicksight-role | QuickSightAccess, AthenaAccess, S3ReadOnly, KMSDecrypt | For dashboards |

## Event Rules / Topics
| Name | Source | Target |
|------|---------|--------|
| S3ObjectCreatedToJobInit | s3:ObjectCreated:* | Lambda: cci-lite-job-init |
| TranscribeCompletedToResultHandler | SNS: TranscribeJobStatusTopic | Lambda: cci-lite-result-handler |

## DynamoDB
Table: cci-lite-config  
Keys: tenant_id (PK), config_type (SK)  
Purpose: per-tenant QA and prompt configuration

## Glue / Athena
Database: cci-lite-db  
Crawler: cci-lite-results-crawler  
Athena Workgroups: cci-lite-shared, tenant-<tenant_id>

## Tagging Standards
Environment: prod  
Project: CCI-Lite  
Owner: Conversant  
TenantID: <value>  
CostCentre: <optional>

