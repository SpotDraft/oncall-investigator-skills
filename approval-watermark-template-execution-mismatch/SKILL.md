---
name: approval-watermark-template-execution-mismatch
description: "Diagnose template or template-editable contracts where the approval/ready-to-sign watermark is visible during signing but missing from the executed platform copy, while TPP/upload-sign workflows retain it after execution. Use when support reports inconsistent watermark behavior across Template vs TPP, 'watermark disappears after execution', 'approved by legal missing on executed contract', or PRE_SIGN_PDF and executed download do not match. Trigger phrases: approval watermark template executed contract, ready_to_sign_watermark, PRE_SIGN_PDF vs WITHOUT_STAMP_SPACE, watermark visible before sign but gone after execution, SPD-42223."
---

# Approval Watermark Missing on Executed Template Contracts

## What This Skill Debugs

This runbook covers the specific mismatch where **approval / ready-to-sign watermarking** behaves differently by workflow:

- **TPP / upload-sign / editable flows** keep the watermark after execution
- **Template / template-editable flows** show the watermark at signing time, but the **executed platform copy** loses it

This is not usually a random rendering issue. The important split is in **how the executed PDF is produced**:

- **Editable / TPP path** signs the already-generated **`PRE_SIGN_PDF`**, so the watermark survives into the final copy
- **Template path** historically regenerated the executed PDF from scratch and, before the Mar 2026 fix, did **not** re-apply the watermark before generating the executed copy

Confirmed fix pattern from incident `#incident-20260311-medium-doceree-inconsistent-watermark-behavior-between-templat` / Jira `SPD-42223`:

- move watermark application into `get_pre_sign_pdf_for_template_contract()`
- let `save_executed_pdf_for_template_contract()` consume that already-watermarked PDF
- regression-test Native signing, DocuSign, HelloSign, and Aadhaar/native-sign variants before prod rollout

## When To Use / Not Use

Use this skill when:

- support says watermark is visible in **signing stage** but gone after **execution**
- incident compares **Template** vs **TPP** and only template loses the watermark
- you need to confirm whether the executed artifact came from **`PRE_SIGN_PDF`** or a fresh template regeneration path
- customer reports **platform executed copy** and **other download paths** behaving differently

Do not use this skill for:

- watermark fully missing in **all** stages and workflows -> first verify the workflow setting / flag is actually enabled
- generic signing failures, document generation errors, or signature-field issues without a watermark symptom -> use `contract-signing-triage`
- DocuSign execution-date population issues -> use `docusign-execution-date-blank-pre-sign`

## Key Identifiers To Collect

- `{contract_id}`
- `{workspace_id}` / `{wsid}`
- `{cluster}`
- `{contract_kind}` if already known (`TEMPLATE`, `TEMPLATE_EDITABLE`, `UPLOAD_SIGN`, `UPLOAD_EDITABLE`)
- whether the mismatch is on:
  - executed platform view / executed download
  - counterparty-side download
  - pre-sign/signing-stage PDF

## Available MCP Tools

### Slack

- Read the incident channel and related discussion threads.
- Search prior conversations with terms like `watermark template execution`, `ready_to_sign_watermark`, `SPD-42223`.
- Useful prior signals from Mar 2026:
  - product/support confirmed intended behavior is that watermark should remain after execution for **both** template and TPP
  - PR review explicitly recommended moving the watermark call into `get_pre_sign_pdf_for_template_contract()`
  - reviewer called out regression coverage for Native signing, DocuSign, and HelloSign

### BigQuery

Run queries **sequentially**.
Use the environment mapping from `contract-lookup`:

- QA: `spotdraft-qa.{dataset}.contracts_v3_*`
- Prod: `spotdraft-prod.{dataset}.public_contracts_v3_*`

**Contract + kind check:**
```sql
SELECT
  c.id,
  c.status,
  c.contract_kind,
  c.workflow_status,
  c.created_by_workspace_id,
  c.execution_date,
  c.created,
  c.modified
FROM `{project}.{dataset}.{prefix}contracts_v3_contractv3` c
WHERE c.id = {contract_id};
```

**Version history:**
```sql
SELECT
  cv.id,
  cv.version_number,
  cv.sub_version_number,
  cv.action,
  cv.is_current,
  cv.is_deleted,
  cv.created,
  cv.pdf_version IS NOT NULL AS has_pdf,
  cv.docx_version IS NOT NULL AS has_docx
FROM `{project}.{dataset}.{prefix}contracts_v3_contractversion` cv
WHERE cv.contract_id = {contract_id}
ORDER BY cv.created DESC
LIMIT 20;
```

**Generated exports:**
```sql
SELECT
  ce.id,
  ce.file_type,
  ce.is_contract_executed,
  ce.created,
  ce.modified
FROM `{project}.{dataset}.{prefix}contracts_v3_contractexport` ce
WHERE ce.contract_id = {contract_id}
ORDER BY ce.created DESC
LIMIT 20;
```

What to look for:

- **Template bug signature**: `PRE_SIGN_PDF` exists and shows watermark, but the executed artifact (`WITHOUT_STAMP_SPACE` / `EXECUTION_PDF` path) was produced via template regeneration and ends up watermarkless
- **Editable / TPP behavior**: executed copy derives from existing `PRE_SIGN_PDF`, so watermark survives naturally

### Admin

Known useful admin check from the incident thread:

- `https://api.{cluster}.spotdraft.com/admin/contracts_v3/contractversion/?contract_id={contract_id}`

Use it to compare:

- whether the current execution version exists
- whether the incident is on template vs TPP path
- whether there was a recent re-generation after send-for-sign

If needed, search `ContractExport` in admin by the same contract ID and compare `PRE_SIGN_PDF` vs executed export rows.

### Jira

- Primary ticket from the incident: `SPD-42223`
- The ticket / PR review added an explicit regression note for Native signing, DocuSign, and HelloSign before prod rollout

### Logs / Groundcover

- Usually lower-signal than code path + admin/version evidence for this issue class
- Use only to correlate exact timing of pre-sign generation / signing / executed export regeneration if needed

## Investigation Workflow

1. **Confirm the symptom precisely**

- Is the watermark present in the **signing-stage** document?
- Is it missing only from the **executed platform copy**?
- Is the comparison actually **Template vs TPP**, or is the watermark missing in both?

2. **Confirm intended behavior**

- For this issue class, the intended behavior is that watermark remains visible after execution for both **Template** and **TPP**
- If support is asking whether disappearing watermark is expected for template, the Mar 2026 answer was **no**

3. **Identify the contract path**

- Query or inspect `contract_kind`
- Treat these buckets differently:
  - **Template path**: `TEMPLATE`, `TEMPLATE_EDITABLE`
  - **Editable / TPP path**: upload / third-party-paper variants using existing pre-sign PDF

4. **Inspect versions and exports**

- Check `ContractVersion` rows for the contract
- Check generated exports:
  - `PRE_SIGN_PDF`
  - executed export (`WITHOUT_STAMP_SPACE`)
- If the watermark appears pre-sign but disappears after execution only on template contracts, this skill applies

5. **Map the code path**

- **Editable / TPP executed flow**: `save_executed_pdf_for_editable_contract()`
  - reads existing `PRE_SIGN_PDF`
  - signs that file
  - final executed PDF keeps watermark already present in the pre-sign artifact
- **Template executed flow**: `save_executed_pdf_for_template_contract()`
  - regenerates template PDF via `get_pre_sign_pdf_for_template_contract()`
  - updates signature spans
  - calls `get_signed_pdf_content(...)`
  - before the fix, this path did not guarantee watermark was applied to the regenerated PDF used for execution

6. **Check whether the fix is present in the environment**

Known fix from Mar 20, 2026:

- commit: `7b8f8f2765`
- PR: `#12297`
- summary: apply `AddReadyToSignWatermarkToContractUseCase` inside `get_pre_sign_pdf_for_template_contract()`

If the environment is on code older than this fix, the template-vs-TPP mismatch is expected.

7. **Validate adjacent behavior before closing**

- Reviewer guidance from the PR thread required regression on:
  - Native signing
  - DocuSign
  - HelloSign
- QA thread additionally confirmed testing on DocuSign and Aadhaar/native flow

8. **Handle the “counterparty download still has watermark” claim carefully**

- This was reported early in the incident but not conclusively proven as a separate code path
- Treat it as a **verification task**, not a default assumption
- First hypothesis from engineering discussion: the user may be seeing a **pre-sign** artifact rather than the executed export

## Code Paths / Architecture

### Watermark application

- `contracts_v3/contract_status/domain/use_cases/add_ready_to_sign_watermark_to_contract_use_case.py`
  - checks workflow settings
  - skips excluded contract classes
  - inserts watermark text into the PDF when `ready_to_sign_watermark_enabled` is true

### Editable / TPP behavior

- `contracts_v3/services/contract_v3_service.py`
  - `save_pre_sign_pdf_for_editable_contract()`
  - `save_executed_pdf_for_editable_contract()`

Key idea:

- watermark is added to the pre-sign artifact
- executed copy is created by signing that same artifact

### Template behavior

- `contracts_v3/services/contract_v3_service.py`
  - `get_pre_sign_pdf_for_template_contract()`
  - `save_pre_sign_pdf_for_template_contract()`
  - `save_executed_pdf_for_template_contract()`
  - `create_execution_pdf_contract_version()`

Key idea:

- template execution regenerates the PDF, updates signature spans, then generates the final signed PDF
- the durable fix was to apply watermark inside `get_pre_sign_pdf_for_template_contract()` so every downstream template execution path consumes the same transformed PDF

### Download behavior

- `contracts_v3/services/contract_service.py`
  - `get_exported_copy_for_the_contract()`

Useful nuance:

- `PRE_SIGN_PDF` requests are handled specially and can be signed on read
- executed downloads depend on the generated executed export unless regeneration is triggered

## Confirm / Disprove Matrix

| Observation | Interpretation |
|---|---|
| Template contract shows watermark before signing but not after execution; TPP keeps it | Classic signature of this issue class |
| Both template and TPP are missing watermark | Likely workflow setting / watermark config issue, not this mismatch |
| Admin/version history shows template execution path and environment predates fix `7b8f8f2765` | Root cause likely confirmed |
| `PRE_SIGN_PDF` has watermark but executed `WITHOUT_STAMP_SPACE` does not | Strong evidence for template execution-path mismatch |
| Watermark remains visible after moving back to redlining | QA thread treated this as pre-existing behavior, not a regression introduced by the fix |

## Mitigation / Workaround

Before the fix is deployed:

- there is no clean settings-only way to make template executed copies retain the watermark
- communicate current behavior clearly to support/customer
- if the customer wants watermark hidden everywhere, they must disable the watermark setting at the workflow level from the start

After the fix:

- verify template executed copies now match TPP behavior
- regression-test provider-specific signing flows before prod rollout

## Escalation Triggers

- Fix commit is present, but template executed copy still drops the watermark
- Provider-specific flow breaks after the watermark change (DocuSign, HelloSign, Native/Aadhaar)
- Counterparty-side artifact still behaves differently after confirming both users are viewing the same executed export
- Watermark disappears only for a specific workspace/workflow despite same code level, suggesting workflow-setting or environment drift

## Related Incidents / References

- Rootly incident `2981` — Doceree inconsistent watermark behavior between template and TPP workflows
- Jira `SPD-42223`
- GitHub PR `SpotDraft/django-rest-api#12297`
- Slack incident channel `C0AKHQNLNLX`

## Connections To Other Skills

- `contract-lookup` for environment mapping and contract/version baseline queries
- `contract-signing-triage` for generic signing/execution failures
- `docusign-execution-date-blank-pre-sign` for a different template pre-sign / execution mismatch that also depends on generated PDF timing
