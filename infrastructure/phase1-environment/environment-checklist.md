# CCI Lite – Phase 1 Validation Log

**Date:** 2025-11-04  
**Region:** eu-central-1  
**Deployed by:** <your name>

## Verified Resources
- ✅ **KMS:** alias `cci-lite-key`, rotation enabled, policy grants Lambda and QuickSight roles.
- ✅ **S3:** bucket `cci-lite-data`
  - Folders: `audio-input/`, `transcripts/`, `enriched/`, `metadata/`
  - Lifecycle rules applied (3 d / 30 d / 90 d)
  - KMS encryption enforced (`cci-lite-key`)
  - Versioning enabled
- ✅ **DynamoDB:**
  - `cci-lite-tenants` → Partition Key = tenant_id  
  - `cci-lite-calls` → Partition Key = tenant_id#date, Sort Key = call_id  
  - Encryption: `cci-lite-key`
- ✅ **IAM Roles:**
  - `CciLiteLambdaRole` → attached `CciLiteLambdaPolicy`
  - `CciLiteQuickSightRole` → attached `CciLiteQuickSightPolicy`
- ✅ **CloudWatch:** dashboard placeholder created (`cci-lite-overview`)
- ✅ **SNS (optional):** `cci-lite-alerts` topic created for alarm notifications

**Validation result:** All foundational resources created, encrypted, and active.
