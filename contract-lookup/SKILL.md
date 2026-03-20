---
name: contract-lookup
description: "Look up contract details, workspace configuration, and related data for debugging support issues. Use this skill when you need to check a contract's status, type, kind, version history, workflow configuration, or workspace settings to help diagnose a support issue. Trigger on phrases like 'check contract', 'contract details', 'workspace info', 'contract status', 'contract type', 'contract ID', 'WSID', or when any other triage skill needs underlying contract or workspace data to complete its diagnosis."
---

# Contract & Workspace Lookup

This skill helps gather the contract and workspace context needed for debugging. Many triage workflows require knowing the contract's current state, type, kind, and configuration.

## Available MCP Tools

You have FOUR categories of tools for contract investigation:

### 1. SpotDraft API (MCP Tools) — Contract Data

- **`get_contract_list`** — Search for contracts by filters (status, type, client name, etc.)
- **`get_contract_status`** — Get current status (composite ID like T-123 or H-123)
- **`get_contract_key_pointers`** — Get metadata fields for a contract
- **`get_contract_content`** — Get text content of a contract
- **`get_contract_activity_log`** — View activity history (comments, status changes)
- **`get_contract_approvals`** — Check approval status and workflow
- **`get_contract_types`** — List all contract types in the workspace
- **`get_templates`** / **`get_template_details`** — Check template configuration
- **`get_counter_parties`** / **`get_counter_party_details`** — Look up counterparty info

### 2. BigQuery Django Models — Contract State

**✅ MCP has access to BOTH `spotdraft-qa` AND `spotdraft-prod` projects.** The earlier note saying prod BQ access is unavailable was incorrect — it works fine. Use the environment table below to pick the right project and dataset.

Use the correct `projectId` and dataset per the environment table below.

**Full contract state (status, kind, workflow):**
```sql
-- QA: SELECT ... FROM `spotdraft-qa.qa_india_public.contracts_v3_contractv3`
-- Prod USA: SELECT ... FROM `spotdraft-prod.prod_usa_db.public_contracts_v3_contractv3`
SELECT
  c.id, c.status, c.contract_kind, c.contract_type_id,
  c.created_by_workspace_id, c.workflow_status,
  c.created, c.modified, c.execution_date, c.editor_client
FROM `{project}.{dataset}.{prefix}contracts_v3_contractv3` c
WHERE c.id = {contract_id}
```

**Version history (for version mismatch / wrong version selected issues):**
```sql
SELECT
  cv.id, cv.version_number, cv.sub_version_number, cv.action,
  cv.is_current, cv.is_deleted, cv.deleted_at, cv.root_version_id,
  cv.version_description,
  cv.docx_version IS NOT NULL as has_docx,
  cv.pdf_version IS NOT NULL as has_pdf,
  cv.created, cv.created_by_workspace_id
FROM `spotdraft-qa.qa_india_public.contracts_v3_contractversion` cv
WHERE cv.contract_id = {contract_id}
ORDER BY cv.version_number DESC, cv.sub_version_number DESC
LIMIT 20
```

**Signature setup state (for signing workflow issues):**
```sql
SELECT
  ss.id, ss.is_completed, ss.sent_for_signature, ss.signing_method,
  ss.is_aes_enabled, ss.is_deleted, ss.mark_executed_after_signatures,
  ss.signing_order_confirmed, ss.created, ss.modified
FROM `spotdraft-qa.qa_india_public.contracts_v3_contractsignaturesetup` ss
WHERE ss.contract_id = {contract_id}
  AND ss.is_deleted = FALSE
```

**Approval state for a contract:**
```sql
SELECT
  a.id, a.name, a.current_state, a.order, a.is_required,
  a.enforcement_point_type, a.breakpoint_type, a.approval_type,
  a.linked_to_entity_type, a.is_deleted
FROM `spotdraft-qa.qa_india_public.approvals_v5_approvalv5` a
WHERE a.linked_to_entity_id = {contract_id}
  AND a.linked_to_entity_type = 'CONTRACT'
  AND a.is_deleted = FALSE
ORDER BY a.order
```

**Workflow for a workspace:**
```sql
SELECT
  w.id, w.title, w.type, w.status, w.public_id,
  w.last_published, w.unpublished_changes_present, w.is_deleted
FROM `spotdraft-qa.qa_india_public.workflow_v1_workflow` w
WHERE w.tenant_workspace_id = {wsid}
  AND w.is_deleted = FALSE
```

**All contracts with version counts for a workspace:**
```sql
SELECT
  c.id, c.status, c.contract_kind, c.created_by_workspace_id,
  COUNT(cv.id) as version_count, MAX(cv.version_number) as max_version
FROM `spotdraft-qa.qa_india_public.contracts_v3_contractv3` c
JOIN `spotdraft-qa.qa_india_public.contracts_v3_contractversion` cv ON cv.contract_id = c.id
WHERE c.created_by_workspace_id = {wsid}
  AND (cv.is_deleted = FALSE OR cv.is_deleted IS NULL)
GROUP BY c.id, c.status, c.contract_kind, c.created_by_workspace_id
ORDER BY version_count DESC
LIMIT 20
```

### Django Model Table Schemas

**`contracts_v3_contractv3`:** id, created, modified, execution_date, workflow_id, contract_data_id, contract_template_id, created_by_id, status (STRING), public_id, frozen_template_id, contract_type_id, workflow_name, contract_kind (STRING), created_by_workspace_id (INT), editor_client, workflow_status

**`contracts_v3_contractversion`:** id, created, modified, version_number (INT), sub_version_number (INT), action (STRING), docx_version (STRING), pdf_version (STRING), is_current (BOOL), meta_data (JSON), contract_id (INT), created_by_workspace_id (INT), is_deleted (BOOL), deleted_at, root_version_id (INT), version_description

**`contracts_v3_contractsignaturesetup`:** id, created, modified, contract_id (INT), is_completed (BOOL), sent_for_signature (BOOL), signing_method, is_aes_enabled (BOOL), is_deleted (BOOL), created_by_workspace (INT), mark_executed_after_signatures (BOOL), signing_order_confirmed (BOOL)

**`approvals_v5_approvalv5`:** id, created, modified, name, current_state, order (INT), is_required (BOOL), linked_to_entity_id (INT), linked_to_entity_type, breakpoint_type, enforcement_point_type, tenant_workspace_id (INT), is_deleted (BOOL), approval_type

**`workflow_v1_workflow`:** id, created, modified, title, type, status, public_id, tenant_workspace_id (INT), is_deleted (BOOL), last_published, unpublished_changes_present (BOOL)

### Environment-to-Dataset Mapping

| Environment | BQ Project | Dataset | Table Prefix | MCP Access |
|-------------|-----------|---------|--------------|------------|
| QA India | `spotdraft-qa` | `qa_india_public` | *(none)* | ✅ Yes |
| QA EU | `spotdraft-qa` | `qa_eu_public` | *(none)* | ✅ Yes |
| QA USA | `spotdraft-qa` | `qa_usa_public` | *(none)* | ✅ Yes |
| Dev India | `spotdraft-qa` | `dev_india_public` | *(none)* | ✅ Yes |
| **Prod India** | `spotdraft-prod` | `prod_india_db` | `public_` | ✅ Yes |
| **Prod EU** | `spotdraft-prod` | `prod_eu_db` | `public_` | ✅ Yes |
| **Prod USA** | `spotdraft-prod` | `prod_usa_db` | `public_` | ✅ Yes |
| **Prod MEA** | `spotdraft-prod` | `prod_mea_db` | *(none)* | ✅ Yes |

> **Note on prod table names:** Prod tables have a `public_` prefix, e.g. `prod_usa_db.public_contracts_v3_contractv3`. QA/Dev tables have no prefix.

> **⚠️ BigQuery parallel query bug:** Do NOT fire multiple `execute_sql` calls in the same message — all return 500 errors. Always run BQ queries sequentially (one per message).

### 3. GCP Cloud Logging (MCP tool: `list_log_entries`) — Backend Errors

**✅ Prod logs ARE accessible** via `resourceNames: ["projects/spotdraft-prod"]`. Use `orderBy: "timestamp desc"`.

**⚠️ Critical: Prod uses `jsonPayload`, not `textPayload`**
- QA clusters: use `textPayload:"{term}"`
- Prod clusters (`prod-usa`, `prod-india`, etc.): use `jsonPayload.message:"{term}"`
- The `logName` for prod stderr is `projects/spotdraft-prod/logs/stderr`

**Prod cluster names** (use in `resource.labels.cluster_name`):
- `prod-usa` — US cluster (WSID prefix 14xxxx)
- `prod-india` — India cluster
- *(EU prod cluster name not yet confirmed — omit filter or try `prod-eu`)*

**Prod: All errors containing a contract ID or error message:**
```
logName="projects/spotdraft-prod/logs/stderr"
resource.labels.cluster_name="prod-usa"
jsonPayload.message:"{contract_id_or_error_string}"
severity >= ERROR
```

**Prod: Search for a specific error type (e.g. Tray download_link failures):**
```
logName="projects/spotdraft-prod/logs/stderr"
resource.labels.cluster_name="prod-usa"
jsonPayload.message:"Current contract version not found"
```

**Prod: Narrow by time window (always do this — prod is very high volume):**
```
logName="projects/spotdraft-prod/logs/stderr"
resource.labels.cluster_name="prod-usa"
jsonPayload.message:"{error_string}"
timestamp >= "2026-03-03T02:00:00Z"
timestamp <= "2026-03-03T04:00:00Z"
```

**QA: All errors for a specific contract ID:**
```
resource.type="k8s_container" labels."k8s-pod/app"="spotdraft-qa-django-app" textPayload:"{contract_id}" severity >= ERROR
```

**QA: All activity for a contract (info + errors):**
```
resource.type="k8s_container" labels."k8s-pod/app"="spotdraft-qa-django-app" textPayload:"{contract_id}"
```

**QA: Async/deferred task errors for a contract:**
```
resource.type="k8s_container" labels."k8s-pod/app"="spotdraft-qa-django-app-deffered-tasks" textPayload:"{contract_id}" severity >= ERROR
```

### 4. BigQuery Operational Logs — API Errors & Task Failures

**QA** uses `projectId: "spotdraft-qa"`. **Prod** uses `projectId: "spotdraft-prod"`.

#### Prod USA — Request/Response Logs

> **⚠️ `request_response_logs_prod_usa.logs` is empty (new partitioned table with 0 rows).** Use `prod_usa_db.public_core_requestresponselog` instead.

**Prod: Check API request/response history for a contract (correct table):**
```sql
SELECT start_timestamp, request_method, request_path, response_status,
       request_header, response_content
FROM `spotdraft-prod.prod_usa_db.public_core_requestresponselog`
WHERE start_timestamp >= TIMESTAMP('2026-03-03 02:00:00')
  AND start_timestamp <= TIMESTAMP('2026-03-03 03:00:00')
  AND workspace_id = {wsid}
  AND request_path LIKE '%{contract_id}%'
ORDER BY start_timestamp DESC
LIMIT 20
```

**Prod `public_core_requestresponselog` schema (confirmed):**
`start_timestamp` (TIMESTAMP), `request_method`, `request_path`, `response_status` (INT),
`request_header` (JSON string — contains `HTTP_CLIENT_ID`, `HTTP_CLIENT_SECRET`, `HTTP_USER_EMAIL`, `HTTP_USER_AGENT`),
`response_content`, `user_id` (null for API key auth), `workspace_id`

> Note: Column names differ from QA logs. It's `start_timestamp` not `inserted_at`, `response_status` not `status_code`, `request_path` not `path`.

#### Prod USA — Tray Integration Table

**Check active Tray integrations for a workspace:**
```sql
SELECT id, solution_instance_id, is_enabled, is_installed, created, modified
FROM `spotdraft-prod.prod_usa_db.public_tray_integrations_traysolutioninstance`
WHERE created_by_workspace_id = {wsid}
  AND is_enabled = TRUE
  AND is_installed = TRUE
```

> **⚠️ Column is `created_by_workspace_id`, NOT `workspace_id`** — using `workspace_id` returns no results.

#### QA — Request/Response Logs

**Check API request/response history for a contract:**
```sql
SELECT request_method, request_path, response_status, time_taken, response_content, inserted_at
FROM `spotdraft-qa.request_response_logs_qa_india.logs`
WHERE inserted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND request_path LIKE '%{contract_id}%'
ORDER BY inserted_at DESC
LIMIT 20
```

**Find all API errors for a workspace:**
```sql
SELECT request_path, response_status, COUNT(*) as error_count, AVG(time_taken) as avg_ms
FROM `spotdraft-qa.request_response_logs_qa_india.logs`
WHERE inserted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND workspace_id = '{wsid}'
  AND response_status >= 400
GROUP BY request_path, response_status
ORDER BY error_count DESC
LIMIT 20
```

**Check DLQ for failed tasks related to a workspace:**
```sql
SELECT task_name, COUNT(*) as failures, MAX(created) as last_failure
FROM `spotdraft-qa.dlq_qa_india.dlq`
WHERE created >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND workspace_id = {wsid}
GROUP BY task_name
ORDER BY failures DESC
LIMIT 20
```

**Check access control events for a workspace:**
```sql
SELECT event_name, event_outcome, permission_name, role_name, context_status, datetime
FROM `spotdraft-qa.access_control_logs_qa_india.logs`
WHERE datetime >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND workspace_id = '{wsid}'
ORDER BY datetime DESC
LIMIT 20
```

### Operational Log Table Schemas

**QA DLQ (`dlq_qa_india.dlq`):** id, task_name, workspace_id (INT), throttle_rate, throttle_duration, signature, dedupe_id, throttle_key, celery_task_id, created (TIMESTAMP, partitioned), modified

**QA Request/Response Logs (`request_response_logs_qa_india.logs`):** time_taken (FLOAT), start_timestamp, hostname, request_method, request_path, post_data, get_data, request_header, cookie_data, session_data, user_id, response_status (INT), response_content, workspace_id (STRING), inserted_at (TIMESTAMP, partitioned)

**QA Access Control Logs (`access_control_logs_qa_india.logs`):** client_ip, user_id, workspace_id, event_name, event_outcome, event_category, more_data, datetime (TIMESTAMP, partitioned), permission_name, role_id, role_name, reason, target_org_user_id, context_status, context_id, context_contract_type_id, context_principal, context_object, http_useragent, cluster_url, cluster_id

**Prod Request/Response Log (`prod_usa_db.public_core_requestresponselog`):** start_timestamp, request_method, request_path, response_status (INT), request_header, response_content, user_id, workspace_id

### Operational Log Datasets by Cluster
| Cluster | DLQ | Request/Response | Access Control |
|---------|-----|-----------------|----------------|
| QA India | `spotdraft-qa.dlq_qa_india.dlq` | `spotdraft-qa.request_response_logs_qa_india.logs` | `spotdraft-qa.access_control_logs_qa_india.logs` |
| QA Europe | `spotdraft-qa.dlq_qa_europe.dlq` | `spotdraft-qa.request_response_logs_qa_europe.logs` | `spotdraft-qa.access_control_logs_qa_europe.logs` |
| QA USA | `spotdraft-qa.dlq_qa_usa.dlq` | `spotdraft-qa.request_response_logs_qa_usa.logs` | `spotdraft-qa.access_control_logs_qa_usa.logs` |
| Dev India | `spotdraft-qa.dlq_dev_india.dlq` | `spotdraft-qa.request_response_logs_dev_india.logs` | `spotdraft-qa.access_control_logs_dev_india.logs` |
| **Prod USA** | `spotdraft-prod.dlq_prod_usa.*` | `spotdraft-prod.prod_usa_db.public_core_requestresponselog` | *(not confirmed)* |

## Gathering Initial Context

When a support person reports an issue, you typically need:

1. **Contract ID** — numeric ID or composite ID (T-123 for template, H-123 for historical)
2. **Contract link** — `https://app.spotdraft.com/contracts/v2/{id}`
3. **Workspace ID (WSID)** — identifies the customer's workspace
4. **Cluster** — IN, US, ME, or EU (determines which API endpoint and BQ dataset to use)

## Investigation Workflow

1. **Start with SpotDraft API** — Use `get_contract_status` and `get_contract_activity_log` for any cluster
2. **Query BQ Django models** — Use `qa_india_public` (or equivalent QA dataset) to get contract state, version history, sig setup, and approvals directly — no admin panel needed for QA contracts
3. **Check GCP logs** — Search for the contract ID in Django app logs (QA only)
4. **Check operational logs** — Query DLQ for failed tasks, request/response logs for API errors (QA only)
5. **For prod contracts** — Use admin panel URLs below; BQ prod access is not available via MCP

## Understanding Contract Kinds

| Kind | What it means |
|------|--------------|
| TEMPLATE | Created from a template, not yet edited |
| TEMPLATE_EDITABLE | Created from template, then edited |
| UPLOAD_SIGN | Uploaded document sent directly for signing |
| UPLOAD_EDITABLE | Uploaded document that was edited |
| UPLOAD_EXECUTED | Uploaded already-executed contract |
| **CLICKWRAP** | Self-execution contract — created, executed, and completed atomically/simultaneously. Fires Tray/integration webhooks the instant it's created. High-risk for race conditions with `download_link`. |
| HISTORICAL_CLICKWRAP | Historical version of a clickwrap contract |
| EXPRESS_TEMPLATE | Created from an express template |

## Understanding Contract Statuses

| Status | What it means |
|--------|--------------|
| DRAFT | Initial creation, being filled out |
| REDLINING | Under negotiation/editing |
| SIGN | Sent for or ready for signature |
| EXECUTED | Fully signed and completed |
| ON_HOLD | Temporarily paused |
| VOIDED | Cancelled/invalidated |

## Known Prod Data Patterns

### CLICKWRAP + Tray "download_link" Race Condition (Mar 2026, WSID 147206)

**Symptom:** Tray "Get contract download link" workflow returns `ContractVersionNotFound` — "Current contract version not found for the contract: X". Contract shows COMPLETED in DB.

**Root cause:** CLICKWRAP contracts fire execution webhooks to Tray the instant they're created. Tray immediately calls `POST /api/v2/public/contracts/T-{id}/download_link`. The backend's `_get_current_contract_version()` queries for `ContractVersion.is_current=TRUE` — but this flag may not be committed yet, causing transient 404s. Confirmed for contract 1404237: `ContractVersion.created` matched the first error timestamp; `modified` (when `is_current` was finalized) was 77 minutes later.

**This is NOT:** wrong credentials, ConvertAPI failure, or a permanently broken contract. The `ContractVersionNotFoundV2` ("Last published Contract version not found") is a **different** error from a different code path — don't confuse them.

**Code path for the failing error:**
```
public/contract/views_v2.py:619  download_link()
  → contract_versions_service.py:237  _get_current_contract_version()
    → raises ContractVersionNotFound("Current contract version not found for the contract: X")
```

**How to diagnose:**
1. GCP logs: filter `jsonPayload.message:"Current contract version not found"` on `prod-usa` cluster
2. Check the Tray request source IPs — they should be AWS us-west-2 (`52.x.x.x`) for Tray.io
3. BQ: query `public_contracts_v3_contractv3` — check `contract_kind=CLICKWRAP`
4. BQ: query `public_contracts_v3_contractversion` — compare `created` vs `modified` timestamps; large gap = `is_current` was set late
5. BQ: query `public_core_requestresponselog` — confirm credentials are valid (requests reach app layer, not rejected at auth)

**Fix directions:** Delay CLICKWRAP webhook until `is_current=TRUE` is committed, OR add retry-with-backoff in `download_link` when no current version is found.

**Jira:** SPD-39109 (marked fixed Jan 2026 but race condition persists — prior fix addressed a different symptom)

**Related (different symptom):** Clickwrap exists with `salesforce_id` but **Salesforce KP fields stay blank** and **Tray shows no KP Created logs** — often **Tray solution instance disabled** for a date range; mitigation is **re-enable Tray** (resolve **Native vs Tray** conflict) and **bulk retrigger** webhooks. See **clickwrap-tray-salesforce-sync-triage**.

---

## Connecting to Other Triage Skills

Contract lookup is usually the first step:
- **For clickwrap → Salesforce KP sync / empty Tray logs** (not download_link 404) → **clickwrap-tray-salesforce-sync-triage**
- **For email/OTP issues** → Look up contract to find counterparty email, then use email-delivery-triage
- **For signing errors** → Check contract kind, status, version history and sig setup, then use contract-signing-triage
- **For approval issues** → Query `approvals_v5_approvalv5` directly for approval state
- **For version mismatch** → Query `contracts_v3_contractversion` — look for multiple `is_current=TRUE` rows or unexpected `action` values

## Key Admin URLs by Cluster (for prod contracts)

| Cluster | Admin URL |
|---------|-----------|
| IN | `https://api.in.spotdraft.com/admin/` |
| US | `https://api.us.spotdraft.com/admin/` |
| ME | `https://api.me.spotdraft.com/admin/` |
| EU | `https://api.eu.spotdraft.com/admin/` |

### Useful Admin Endpoints
- Contract details: `admin/core/contract/{id}/change/`
- Async tasks: `admin/core/asynctask/?q={contract_id}`
- Email audits: `admin/emails/emailaudit/?contract_id={id}`
- Workspace: `admin/core/workspace/{wsid}/change/`
