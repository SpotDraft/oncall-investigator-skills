---
name: contract-signing-triage
description: "Diagnose and resolve contract signing, execution, and document generation errors in SpotDraft. Use this skill when support reports issues like: contract signing failures, 'Word server raised an error', 'Document Generation Unsuccessful', 'SoftTimeLimitExceeded', pre-sign PDF or prepare-for-signing failures, signature fields overlapping or retained after editing, contracts stuck in signing state, unable to send for signing, unable to execute contract, document preview failures, PDF upload errors, version selection issues when moving contracts between stages, or any error during the sign/execute workflow. Also trigger on 'signing error', 'execution failed', 'signature fields', 'contract preview', 'document generation failure', 'version mismatch', or 'template editable' issues."
---

# Contract Signing & Execution Triage

This covers the second most common category of issues — problems that occur during signing, execution, document generation, or template-to-editable flows.

## Available MCP Tools for Investigation

You have direct access to GCP Cloud Logging and BigQuery for `spotdraft-qa`. Use these to investigate BEFORE asking humans to check dashboards.

### BigQuery Django Models — Contract & Signing State

**⚠️ MCP only has access to `spotdraft-qa` project.** For QA India contracts, use `qa_india_public`. For other envs, see the environment mapping below.

**Full contract state + signature setup (run first for any signing issue):**
```sql
SELECT
  c.id, c.status, c.contract_kind, c.workflow_status,
  c.created, c.modified, c.execution_date,
  ss.id as sig_setup_id, ss.is_completed, ss.sent_for_signature,
  ss.signing_method, ss.is_deleted as sig_deleted,
  ss.mark_executed_after_signatures, ss.signing_order_confirmed
FROM `spotdraft-qa.qa_india_public.contracts_v3_contractv3` c
LEFT JOIN `spotdraft-qa.qa_india_public.contracts_v3_contractsignaturesetup` ss
  ON ss.contract_id = c.id AND ss.is_deleted = FALSE
WHERE c.id = {contract_id}
```

**Version history (essential for version mismatch diagnosis):**
```sql
SELECT
  cv.id, cv.version_number, cv.sub_version_number, cv.action,
  cv.is_current, cv.is_deleted, cv.deleted_at,
  cv.root_version_id, cv.version_description,
  cv.docx_version IS NOT NULL as has_docx,
  cv.pdf_version IS NOT NULL as has_pdf,
  cv.created
FROM `spotdraft-qa.qa_india_public.contracts_v3_contractversion` cv
WHERE cv.contract_id = {contract_id}
ORDER BY cv.version_number DESC, cv.sub_version_number DESC
LIMIT 20
```

What to look for in version history:
- Multiple rows with `is_current = TRUE` → version consistency bug
- Alternating `EXECUTION_PDF` / `UPLOADED_TPP_PDF` pattern with many retries → ConvertAPI/document merging repeatedly failing
- Unexpected `action` values for the stage (`WORKING_EDITABLE` appearing after move to SIGN stage → version mismatch)
- `is_deleted = TRUE` on the expected current version → wrong version selected

**Approval state for a contract:**
```sql
SELECT
  a.id, a.name, a.current_state, a.order, a.enforcement_point_type,
  a.breakpoint_type, a.approval_type, a.is_deleted
FROM `spotdraft-qa.qa_india_public.approvals_v5_approvalv5` a
WHERE a.linked_to_entity_id = {contract_id}
  AND a.linked_to_entity_type = 'CONTRACT'
  AND a.is_deleted = FALSE
ORDER BY a.order
```

### Environment-to-Dataset Mapping

| Environment | BQ Project | Dataset | MCP Access |
|-------------|-----------|---------|------------|
| QA India | `spotdraft-qa` | `qa_india_public` | ✅ Yes |
| QA EU | `spotdraft-qa` | `qa_eu_public` | ✅ Yes |
| QA USA | `spotdraft-qa` | `qa_usa_public` | ✅ Yes |
| Dev India | `spotdraft-qa` | `dev_india_public` | ✅ Yes |
| **Prod IN/EU/US** | `spotdraft-prod` | `prod_{region}_db` | ❌ Use admin panel |
| **Prod MEA** | `spotdraft-prod` | `prod_mea_db` | ❌ Use admin panel |

> Prod tables have a `public_` prefix, e.g. `prod_india_db.public_contracts_v3_contractversion`. QA/Dev have no prefix.

### GCP Cloud Logging Queries

Use `list_log_entries` with `resourceNames: ["projects/spotdraft-qa"]` and `orderBy: "timestamp desc"`:

**Search Django app logs for signing/execution errors by contract ID:**
```
resource.type="k8s_container" labels."k8s-pod/app"="spotdraft-qa-django-app" textPayload:"{contract_id}" severity >= ERROR
```

**Search for Word server errors:**
```
resource.type="k8s_container" labels."k8s-pod/app"="spotdraft-qa-django-app" textPayload:("Word server raised an error" OR "ConvertAPI" OR "document generation" OR "document conversion")
```

**Search for signing-specific errors:**
```
resource.type="k8s_container" labels."k8s-pod/app"="spotdraft-qa-django-app" textPayload:("signature" OR "signing" OR "execution" OR "Cannot edit a signature setup")
```

**Search deferred tasks worker for async task failures:**
```
resource.type="k8s_container" labels."k8s-pod/app"="spotdraft-qa-django-app-deffered-tasks" textPayload:("{contract_id}" OR "signing" OR "execution") severity >= ERROR
```

**Prod — pre-sign PDF / SoftTimeLimitExceeded (use when incident is on prod):** Prod uses `jsonPayload`, not `textPayload`. Use `resourceNames: ["projects/spotdraft-prod"]`. Cluster label for US prod: `resource.labels.cluster_name="prod-usa"`. Deferred tasks workload: `sd-apps-django-app-deffered-tasks`.
```
logName="projects/spotdraft-prod/logs/stderr"
resource.labels.cluster_name="prod-usa"
labels."k8s-pod/app"="sd-apps-django-app-deffered-tasks"
jsonPayload.message=~"SoftTimeLimitExceeded|save_pre_sign_pdf|cf_exporter|generate_docx_for_payload"
```
Always add a time window (e.g. `timestamp >= "YYYY-MM-DDTHH:MM:00Z"`) to limit volume.

**Check nginx for upstream timeouts on contract endpoints:**
```
resource.type="k8s_container" resource.labels.namespace_name="kube-system" textPayload:"upstream timed out" textPayload:"/api/v3/contracts/{contract_id}"
```

### BigQuery Operational Log Queries

Use `projectId: "spotdraft-qa"` for all queries.

**Check DLQ for failed signing/execution tasks:**
```sql
SELECT task_name, workspace_id, created, signature
FROM `spotdraft-qa.dlq_qa_india.dlq`
WHERE created >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND (task_name LIKE '%sign%' OR task_name LIKE '%execut%' OR task_name LIKE '%convert%' OR task_name LIKE '%document%')
ORDER BY created DESC
LIMIT 20
```

**Check DLQ for a specific workspace's failed tasks:**
```sql
SELECT task_name, COUNT(*) as failures, MAX(created) as last_failure
FROM `spotdraft-qa.dlq_qa_india.dlq`
WHERE created >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND workspace_id = {wsid}
GROUP BY task_name
ORDER BY failures DESC
LIMIT 20
```

**Check request/response logs for 500 errors on contract endpoints:**
```sql
SELECT request_path, response_status, time_taken, workspace_id, inserted_at, response_content
FROM `spotdraft-qa.request_response_logs_qa_india.logs`
WHERE inserted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND response_status >= 500
  AND request_path LIKE '%contract%'
ORDER BY inserted_at DESC
LIMIT 20
```

**Check request/response logs for a specific contract ID:**
```sql
SELECT request_path, request_method, response_status, time_taken, response_content, inserted_at
FROM `spotdraft-qa.request_response_logs_qa_india.logs`
WHERE inserted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND request_path LIKE '%{contract_id}%'
ORDER BY inserted_at DESC
LIMIT 20
```

### BigQuery Datasets by Cluster
- QA India: `spotdraft-qa.dlq_qa_india.dlq`, `spotdraft-qa.request_response_logs_qa_india.logs`
- QA Europe: `spotdraft-qa.dlq_qa_europe.dlq`, `spotdraft-qa.request_response_logs_qa_europe.logs`
- QA USA: `spotdraft-qa.dlq_qa_usa.dlq`, `spotdraft-qa.request_response_logs_qa_usa.logs`

## Diagnostic Framework

### Step 1: Identify the Error Type

Ask the support person for:
- **Contract ID and link** (e.g., `https://app.spotdraft.com/contracts/v2/{id}`)
- **Workspace ID (WSID) and Cluster** (IN, US, ME, EU)
- **Exact error message** shown in the UI
- **What action was being performed** — sending for signing, executing, previewing, uploading new version?
- **Contract kind** — template, template-editable, upload-sign, upload-editable?
- **Screenshots** if available

### Step 2: Run Automated Queries

1. **Query BQ Django models** — Get full contract state + signature setup + version history (for QA contracts)
2. **Search GCP logs** for the contract ID in Django app and deferred task worker logs (QA only)
3. **Search DLQ** for failed async tasks related to the contract or workspace
4. **Search request/response logs** for 500 errors on the contract's API endpoints
5. **Check nginx logs** for upstream timeouts on the contract's endpoints

### Step 3: Check Async Task Logs

Most signing/execution operations are async. Check the admin panel:
```
https://api.{cluster}.spotdraft.com/admin/core/asynctask/?q={contract_id}
```

Look for:
- Task status (SUCCESS, FAILURE, PENDING)
- Error messages in task details
- Whether the task is stuck or has timed out

### Step 4: Correlate Findings

Cross-reference BQ Django model data with GCP log errors, DLQ failures, and API response codes:

| BQ Signal | Log Signal | DLQ Signal | API Signal | Likely Issue |
|-----------|-----------|-----------|-----------|-------------|
| Multiple `is_current=TRUE` versions | — | — | — | Version consistency bug |
| `EXECUTION_PDF`/`UPLOADED_TPP_PDF` repeated many times | "ConvertAPI returned 500" | `merge_documents_task` failures | 500 on `/execute` | ConvertAPI merging failing repeatedly |
| Sig setup `sent_for_signature=TRUE`, `is_completed=FALSE` | "Cannot edit a signature setup" | — | 400 on `/sign` | Race condition in signing flow |
| — | "Word server raised an error" | `document_conversion_task` failures | 500 on `/preview` | Document processing failure |
| — | Nginx "upstream timed out" | — | No response logged | Worker overloaded/killed |
| — | "SoftTimeLimitExceeded" in deffered-tasks | GENERATE_PRE_SIGN_PDF FAILED/TIMED_OUT | "Document Generation Unsuccessful" | Pre-sign PDF timeout (cf-exporter path) |

## Common Issue Patterns

### "Word server raised an error"
**What it means:** The document processing service failed during conversion or manipulation.

**Automated investigation:**
1. Query BQ: Check contract kind — is this TEMPLATE or TEMPLATE_EDITABLE? (affects conversion path)
2. Search GCP logs: `textPayload:("Word server raised an error") textPayload:"{contract_id}"`
3. Check DLQ: Look for `document_conversion_task` or similar task failures
4. Check if other contracts in the same workspace are also affected

**Common causes:**
- Corrupted document template
- Special characters or formatting the Word server can't handle
- Word server overload (check if other customers are affected)

**Resolution:**
1. If document is corrupted, ask customer to re-upload a clean version
2. If systemic Word server issue, check for service health and escalate

### Contract Stuck in Signing — ConvertAPI 500

**This pattern was seen in QA (contract 413213, WS 8498):** ConvertAPI returned 500 during document merging with `USE_CONVERT_API_FOR_MERGING` feature flag enabled. The version history shows alternating EXECUTION_PDF / UPLOADED_TPP_PDF rows (one pair per failed retry attempt). Jira: SPD-33571.

**Automated investigation:**
1. Query version history — look for alternating EXECUTION_PDF/UPLOADED_TPP_PDF pattern with many rows
2. Search GCP logs: `textPayload:("ConvertAPI" OR "convert_api") textPayload:"{contract_id}"`
3. Check DLQ for merge-related task failures
4. Check if it's specific to the contract's document content

**Resolution:**
- Retry the operation (may be intermittent)
- If persistent, check if it's the specific document causing failure
- Check feature flag `USE_CONVERT_API_FOR_MERGING` status

### Signature Fields Retained After Editing Template Contract
**What it means:** When a template contract is edited (creating template-editable), signature variables persist.

**This is a known regression** (Feb 2026 — affected Allen, Chargebee, Stedi, Rain, ADP, Amagi, and others).

**Workaround:** Advise customer to manually remove existing signature fields while editing.
**Fix status:** Check Jira for SPD ticket related to "signature fields retained".

### Contract Version Mismatch When Moving Between Stages
**What it means:** When moving a contract between stages, the system picks up the wrong version.

**Investigation:**
1. **Query BQ version history** — check for multiple `is_current=TRUE` rows, or unexpected `action` types for the stage
2. Look for `WORKING_EDITABLE` or `DRAFT_EDITABLE` versions appearing when contract is in SIGN stage
3. Check `root_version_id` — mismatched root versions indicate branching/split history
4. If QA: check GCP logs for the contract ID around the time of the stage transition
5. If prod: check admin panel async tasks for version-related failures

### Unable to Execute — "Cannot edit signature setup"
**What it means:** System tries to modify signature setup after contract has been sent for signing.

**BQ check:** Query signature setup — if `sent_for_signature = TRUE` and an edit is attempted, this error fires.
**Historical fix:** Overly aggressive safety check was reverted.

### Preview/Document Generation Failure
**What it means:** Contract preview or document generation fails, often due to variable type mismatches.

**Automated investigation:**
1. Query BQ: Check contract kind and template_id for clues about variable configuration
2. Search GCP logs for the contract ID with severity >= ERROR
3. Check DLQ for preview or document_generation task failures
4. Check request/response logs for 500s on preview endpoints

**Common causes:**
- Computed variables with incorrect types
- Recently changed SFDC mappings
- Template construction issues

### "Document Generation Unsuccessful: SoftTimeLimitExceeded()" (pre-sign PDF timeout)
**What it means:** User sees "Document Generation Unsuccessful: SoftTimeLimitExceeded()" when preparing a contract for signing (e.g. from Redlining stage). The Celery task that generates the pre-sign PDF hit its soft time limit while calling the cf-exporter service.

**System flow:** API (prepare for sign) → AsyncTaskService.create_task(GENERATE_PRE_SIGN_PDF) → `save_pre_sign_pdf_for_template_contract_task` (Celery) → contract_export_service.get_contract_docx_preview → `cf_exporter_client.generate_docx_for_payload` → sd-apps-cfexporter (HTTP) → task killed by Celery soft limit.

**Identifiers to collect:** `{contract_id}`, `{wsid}`, `{cluster}` (e.g. US → api.us.spotdraft.com). Async task ID is in admin URL or logs.

**Investigation steps:**
1. **Confirm the failure:** Open async task admin for the contract: `https://api.{cluster}.spotdraft.com/admin/core/asynctask/?q={contract_id}`. Look for task type `GENERATE_PRE_SIGN_PDF` with status `FAILED` or `TIMED_OUT` and failure reason or stack mentioning `SoftTimeLimitExceeded` or `billiard.exceptions.SoftTimeLimitExceeded`.
2. **Prod logs (if prod):** Use GCP Cloud Logging with `resourceNames: ["projects/spotdraft-prod"]`, filter on `sd-apps-django-app-deffered-tasks`, `jsonPayload.message` containing `SoftTimeLimitExceeded` or `generate_docx_for_payload`, and a narrow time window. Optionally check for `sd-apps-cfexporter` logs (e.g. Deprecated "attachModule") to see cf-exporter load or slowness.
3. **Rule out DocuSign auth:** In a past incident (Odessa), the same UI error was resolved after customer re-authenticated with DocuSign. Check Tray/integration status for the workspace if retry fails.
4. **Contributing factors:** Large or complex template (e.g. 500+ row dynamic table), high cf-exporter load (limited concurrency per pod), or deffered-tasks pod restart during the run (check for cloudsql-proxy socket close or pod eviction around the failure time).

**Immediate mitigation (unblock customer):**
1. **Retry the async task** — Either:
   - **Django admin:** Open the failed async task (e.g. `https://api.{cluster}.spotdraft.com/admin/core/asynctask/{async_task_id}/change/`). If the UI allows retry, use it; otherwise use the management command below.
   - **Management command (preferred when available):** In the API repo, run:
     ```bash
     python manage.py retry_pre_sign_pdf_async_task --contract-id {contract_id}
     # or by async task ID
     python manage.py retry_pre_sign_pdf_async_task --async-task-id {async_task_id}
     # preview only
     python manage.py retry_pre_sign_pdf_async_task --contract-id {contract_id} --dry-run
     ```
     Command validates task type GENERATE_PRE_SIGN_PDF and status FAILED/TIMED_OUT, then re-enqueues the Celery task. Run in the correct cluster environment (e.g. prod US).
2. After retry, ask the customer to try "Prepare for signing" again; verify in workspace that the contract can proceed to signing.
3. If retry fails again: check DocuSign auth for the workspace; inspect template for very large dynamic tables or heavy content; consider temporary scale-up of cf-exporter if under load.

**Code path (for reference):** Failure is in `contracts_v3/clients/cf_exporter_client.py` — `generate_docx_for_payload` (has `@retry(tries=3)` but SoftTimeLimitExceeded is in `fatal_exceptions`, so no retry on timeout). Celery task has no explicit SoftTimeLimitExceeded handler and marks the async task as FAILED.

**Related incidents / Jira:** Formation Bio (Mar 2026, contract 1422552, WSID 144671, US) — mitigated by retrying pre-sign PDF task; Amagi (Jul 2025) — high payload size → cf-exporter load, temp fix was scaling cf-exporter; Odessa (May 2025) — DocuSign auth expiry, resolved after re-auth. Jira: SPD-42443 (Formation Bio incident).

### Signature Hidden by Interactive PDF Overlay (Lemfi — Mar 2026)

**Symptom:** Signature is not visible in the signed or executed contract PDF. `GENERATE_EXECUTED_PDF` AsyncTask completes successfully with no errors. Pre-sign PDF export (`PRE_SIGN_PDF`) contains only a stamp/box with the rest of the page blank (but text is selectable). The PDF renders correctly in Mac Preview but appears blank in Adobe/Chrome.

**Root cause:** The source PDF uploaded by the customer contains an **interactive stamp or form field annotation** (e.g. an AcroForm box / stamp widget) that:
1. Renders as an opaque overlay in strict renderers (Adobe, Chrome)
2. Sits on top of the signature placement coordinates — hiding the signature visually
3. The underlying document content and signature ARE present (selectable) but invisible

**Key diagnostic signals:**
- `AsyncTask` for `GENERATE_EXECUTED_PDF` → `status=COMPLETED`, no `error_message`
- `ContractExport` for `PRE_SIGN_PDF` → only the interactive stamp is visible; rest of page blank but selectable
- `contracts_v3_signatureinputfieldvalue.value` → `null` (media reference ID exists but value not stored)
- Signature placed at correct coordinates (cfexporter converts 96 DPI → 72 DPI, so always check actual pts: `x_px * 0.75`)
- File renders fully in Mac Preview; opening and saving from Preview fixes Adobe rendering

**Rule out before concluding:** Coordinates off-page — cfexporter scales pixel coords to PDF points at 0.75× ratio. An `x1=633.05px` becomes `474.8pt` which is within A4 (595pt). Do NOT assume off-page unless the calculated pt value exceeds page dimensions.

**Mitigation (immediate unblock):**
1. Download the latest signed version of the contract from Django admin: `https://api.{cluster}.spotdraft.com/admin/contracts_v3/contractversion/?contract_id={contract_id}`
2. Open it in **Mac Preview**, move the interactive stamp box to an empty area to expose the signature
3. Save from Preview — this re-renders the PDF stripping the overlay issue
4. Attach audit trail and regenerate the executed contract version from this fixed PDF

**Long-term fix:** Detect and strip interactive form field annotations (AcroForm widgets, stamp boxes) from source PDFs during upload or pre-sign validation. The PDF interactive layer structure should be flattened before signing so overlays cannot hide signature placements. Open Jira task to add this to the PDF validation pipeline.

**Confirmed example:** Lemfi (WSID 244718, EU cluster, contract 372981, March 2026). Incident: [#incident-20260317-high-lemfi-signature-not-visible-in-executed-contract](https://spotdraft.slack.com/archives/C0ALPNYE4FR)

## Escalation Triggers
- Multiple customers reporting the same signing issue → likely a regression, escalate as Critical
- Error involves data loss (executed version replaced) → escalate immediately
- Word server errors affecting multiple workspaces → service-level issue, escalate to infra

## Key Admin URLs
- Contract details: `https://api.{cluster}.spotdraft.com/admin/core/contract/{id}/change/`
- Async tasks: `https://api.{cluster}.spotdraft.com/admin/core/asynctask/?q={contract_id}`
- Email audits: `https://api.{cluster}.spotdraft.com/admin/emails/emailaudit/?contract_id={id}`
