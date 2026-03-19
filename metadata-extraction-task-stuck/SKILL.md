---
name: metadata-extraction-task-stuck
description: "Diagnose and remediate metadata extraction (SDC / Smart Data Capture / Key Pointers) async tasks that are stuck perpetually in IN_PROGRESS state, showing 0% or never advancing. Use when: a customer reports metadata extraction has been running for hours or days without completing; an AsyncTask of type EXTRACT_SUGGESTIONS is stuck with started=NULL; a contract shows extraction 'in progress' and the customer cannot re-trigger it; bulk metadata extraction was triggered and some contracts are stuck. Trigger phrases: 'extraction running forever', 'metadata extraction stuck', 'SDC stuck', 'key pointers not extracting', 'extraction at 0%', 'cannot re-run extraction', 'bulk extraction stuck', 'extraction task perpetually running'. Recurring pattern: hit Prompt Therapy (Feb 2026) and Cedar (March 2026) with identical root cause — bulk extraction on unsupported contract kind/status. Jira: SPD-42477."
---

# Metadata Extraction (SDC) AsyncTask Stuck

## What This Skill Diagnoses

Bulk metadata extraction (Key Pointers / SDC) creates `AsyncTask` records of type `EXTRACT_SUGGESTIONS` **before** checking if the contract's `contract_kind` + `display_status` combination is supported. When an unsupported contract is included (e.g., DRAFT, VOIDED), the pipeline raises `KPExtractionNotAllowedException` — but this is uncaught in the outer use case. The `started` field is never set. The timeout cronjob requires `started IS NOT NULL` to mark the task `TIMED_OUT`, so the task stays `IN_PROGRESS` forever.

The customer cannot re-trigger extraction because there is a **one-extraction-per-contract limit** that checks for existing `IN_PROGRESS` tasks.

**The stuck cycle:**
```
Bulk extraction → AsyncTask created (started=NULL) → pipeline raises KPExtractionNotAllowedException
  → exception uncaught → started never set → timeout cronjob never fires → stuck forever
  → customer tries re-run → blocked by IN_PROGRESS check
```

**Unsupported contract combinations** (from `CONTRACT_KIND_DISPLAY_STATUS_MAP_SUPPORTED`):
- `CLICKWRAP` — no statuses supported
- `HISTORICAL_CLICKWRAP` — no statuses supported
- `TEMPLATE` with `DRAFT`, `REDLINING`, `SIGN` — only `EXECUTED` is supported
- Any other kind/status not in the supported map

**This is a recurring pattern.** Two confirmed incidents in Q1 2026:
- Feb 25, 2026: Prompt Therapy (WS 274685, IN cluster)
- March 17–18, 2026: Cedar (WS 63672, US cluster)

---

## Available MCP Tools

### BigQuery — Identify Stuck Tasks

> ⚠️ Run BQ queries **one at a time** — parallel `execute_sql` calls return 500 errors.

**Find all stuck EXTRACT_SUGGESTIONS tasks for a workspace (QA India example):**
```sql
SELECT at.id, at.object_id as contract_id, at.task_status, at.started, at.created,
       c.contract_kind, c.status
FROM `spotdraft-qa.qa_india_public.core_asynctask` at
LEFT JOIN `spotdraft-qa.qa_india_public.contracts_v3_contractv3` c ON c.id = at.object_id
WHERE at.task_type = 'EXTRACT_SUGGESTIONS'
  AND at.task_status = 'IN_PROGRESS'
  AND at.started IS NULL
  AND at.workspace_id = {workspace_id}
ORDER BY at.created DESC
LIMIT 50
```

**Scan for stuck tasks across all workspaces (QA):**
```sql
SELECT at.id, at.object_id as contract_id, at.workspace_id, at.task_status,
       at.started, at.created, c.contract_kind, c.status
FROM `spotdraft-qa.qa_india_public.core_asynctask` at
LEFT JOIN `spotdraft-qa.qa_india_public.contracts_v3_contractv3` c ON c.id = at.object_id
WHERE at.task_type = 'EXTRACT_SUGGESTIONS'
  AND at.task_status = 'IN_PROGRESS'
  AND at.started IS NULL
  AND at.created < TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 DAY)
ORDER BY at.created ASC
LIMIT 100
```

**Check if any metadata values were actually extracted onto the stuck contracts:**
```sql
SELECT kp.id, kp.contract_id, kp.key, kp.value, kp.created
FROM `spotdraft-qa.qa_india_public.key_pointers_keypointsuggestion` kp
WHERE kp.contract_id IN ({contract_id_1}, {contract_id_2})
ORDER BY kp.created DESC
LIMIT 50
```
> If this returns 0 rows, no data was extracted — no cleanup needed. (This was confirmed for Cedar: VOIDED at 0%, DRAFT never started.)

### Environment-to-Dataset Mapping

| Environment | BQ Project | Dataset | Table Prefix |
|-------------|-----------|---------|--------------|
| QA India | `spotdraft-qa` | `qa_india_public` | *(none)* |
| QA EU | `spotdraft-qa` | `qa_eu_public` | *(none)* |
| QA USA | `spotdraft-qa` | `qa_usa_public` | *(none)* |
| **Prod India** | `spotdraft-prod` | `prod_india_db` | `public_` |
| **Prod EU** | `spotdraft-prod` | `prod_eu_db` | `public_` |
| **Prod USA** | `spotdraft-prod` | `prod_usa_db` | `public_` |
| **Prod MEA** | `spotdraft-prod` | `prod_mea_db` | *(none)* |

### Admin Panel URLs

| Purpose | URL |
|---------|-----|
| AsyncTask detail | `https://api.{cluster}.spotdraft.com/admin/core/asynctask/{task_id}/change/` |
| AsyncTask list by contract | `https://api.{cluster}.spotdraft.com/admin/core/asynctask/?q={contract_id}` |
| AsyncTask list by workspace | `https://api.{cluster}.spotdraft.com/admin/core/asynctask/?created_by_workspace_id={workspace_id}` |
| Contract detail | `https://api.{cluster}.spotdraft.com/admin/contracts_v3/contractv3/{contract_id}/change/` |

### Sentry

Search Sentry for `KPExtractionNotAllowedException` to find affected contracts: `https://spotdraft.sentry.io/issues/?query=KPExtractionNotAllowedException`

---

## When to Use This Skill

**Use when:**
- Customer says metadata extraction has been running for hours/days with no progress
- AsyncTask type is `EXTRACT_SUGGESTIONS`, status `IN_PROGRESS`, `started` is `NULL`
- Customer cannot re-trigger extraction (blocked by one-extraction-per-contract limit)
- Bulk extraction was triggered and only some contracts are stuck (likely unsupported stages included)

**Do not use for:**
- Extraction completed but data is wrong/missing → different issue
- Extraction never triggered → check permissions or feature flag
- Individual (non-bulk) extraction stuck with `started IS NOT NULL` → check pipeline/beam logs

---

## Step-by-Step Investigation Workflow

### Step 1: Identify Stuck Tasks

Get the workspace ID and cluster from support. Check the admin panel:
```
https://api.{cluster}.spotdraft.com/admin/core/asynctask/?created_by_workspace_id={workspace_id}
```
Filter for `task_type = EXTRACT_SUGGESTIONS`, `task_status = IN_PROGRESS`.

**Confirm the stuck pattern:** `started IS NULL`. If `started` is set and the task is still IN_PROGRESS, this is a different issue (beam pipeline hanging, not the eligibility bug).

**Also check:** What is the `contract_kind` and `display_status` of the stuck contracts? Draft, Voided, or other unsupported stages confirm this root cause.

### Step 2: Confirm Trigger — Was This a Bulk Operation?

Ask the support person: was metadata extraction triggered in bulk or individually? The root cause (no pre-flight guard) only affects the **bulk extraction path**.

The Cedar customer's statement: *"I only selected the executed contracts then proceeded with bulk-select"* — the UI selection somehow included non-executed contracts. This is a known footgun.

### Step 3: Check Whether Any Data Was Extracted

Customer may ask if incorrect metadata values were extracted onto the unintended contracts. Check:
- Was the task for a VOIDED contract? Extraction runs at 0% and never writes data.
- Was the task for a DRAFT/TEMPLATE contract that hasn't passed questionnaire filling? Extraction never started.

Query `key_pointers_keypointsuggestion` (see BigQuery section) for the stuck contract IDs. If 0 rows → no data was extracted → no cleanup needed. Tell the customer.

### Step 4: Run the Cleanup Script

**Run from the managerie/Django shell for the affected cluster.** Always dry-run first:

```python
from datetime import timedelta
from django.utils import timezone
from core.async_task.domain.domain_models import AsyncTaskStatus, AsyncTaskType
from core.models import AsyncTask

# DRY RUN — check what will be cancelled
cutoff_date = timezone.now() - timedelta(days=1)
stuck_tasks = AsyncTask.objects.filter(
    task_type=AsyncTaskType.EXTRACT_SUGGESTIONS.value,
    created__lt=cutoff_date,
    started__isnull=True,
    task_status=AsyncTaskStatus.IN_PROGRESS.value,
)
total_count = stuck_tasks.count()
print(f"Found {total_count} stuck tasks")
for t in stuck_tasks:
    print(f"  Task {t.id} | Contract {t.object_id} | Created {t.created} | WS {t.workspace_id}")

# CANCEL (uncomment after reviewing dry run output)
# updated_count = stuck_tasks.update(task_status=AsyncTaskStatus.CANCELLED.value)
# print(f"✅ Successfully cancelled {updated_count} tasks.")
```

**Workspace-scoped (for specific customer during triage):**
```python
stuck_for_ws = AsyncTask.objects.filter(
    created_by_workspace_id={workspace_id},
    task_type=AsyncTaskType.EXTRACT_SUGGESTIONS.value,
    task_status=AsyncTaskStatus.IN_PROGRESS.value,
    started__isnull=True,
)
print(stuck_for_ws.count(), [(t.id, t.object_id, t.created) for t in stuck_for_ws])
```

**Run across all regions** after the platform fix lands (Hemil's note: must also re-run after the fix deploys to prod to catch any new stuck tasks):

| Region | Cluster Admin | Scale (March 2026) |
|--------|--------------|---------------------|
| US | `api.us.spotdraft.com` | 58 stuck total, 26 for Cedar |
| IN | `api.in.spotdraft.com` | 26 stuck |
| EU | `api.eu.spotdraft.com` | 4 stuck |
| ME | `api.me.spotdraft.com` | 0 stuck |

> Reference runbook: https://docs.google.com/document/d/112ockx5UzHZYuE42rPZ6fARhfODFBJrIVWibNs6nFLY/edit?usp=sharing

After cancellation, the customer can re-trigger extraction on the affected contracts individually (on supported-stage contracts only).

---

## Code Path & Architecture

```
Bulk trigger
  → create_bulk_action_for_metadata_backfill_use_case.py
      → filters: contract_type_id, contract_ids, workspace_id
      → NO display_status guard ← BUG #1: unsupported stages pass through

  → create_metadata_extraction_async_task_use_case.py (L269-303)
      → AsyncTask.create() ← task created BEFORE eligibility check
      → trigger_suggestions_beam_pipeline.execute()
          → trigger_suggestions_pipeline_use_case.py (L352-368)
              → KPExtractionNotAllowedException raised for unsupported kind/status
      → exception not caught ← BUG #2: no except block
      → started never set

Timeout cronjob
  → check_and_retry_stuck_async_task_objects()
      → requires started IS NOT NULL ← BUG #3: can't timeout unstarted tasks
      → task stuck forever
```

**Fix PRs (from Feb 2026 incident):**
- PR #12185: Add `display_status` pre-flight guard in `create_metadata_extraction_async_task_use_case` before AsyncTask creation
- PR #12186: Update cronjob to also cancel `IN_PROGRESS` tasks where `started IS NULL` and `created` exceeds timeout

> ⚠️ **Check PR merge status** before citing as fix — PR #12185 was in review during the Feb incident and was not deployed before the Cedar (March) incident. Confirm with engineering before closing.

---

## Confirming vs. Disproving the Root Cause

| Evidence | Confirms | Disproves |
|----------|---------|-----------|
| `started IS NULL` on stuck AsyncTask | Eligibility failure (task never kicked off pipeline) | — |
| Contract kind/status is unsupported (DRAFT, VOIDED, etc.) | Bulk extraction included unsupported stage | — |
| Trigger was a bulk operation | Bulk path has no display_status guard | Individual extraction — different code path |
| Sentry shows `KPExtractionNotAllowedException` for contract IDs | Root cause confirmed | — |
| `started IS NOT NULL` but task stuck | Different issue — beam pipeline hanging | This pattern |
| Task IS progressing (>0%) but slow | Different issue — pipeline performance | This pattern |

---

## Secondary: Checking for Accidentally Extracted Metadata

If the customer asks whether any unwanted metadata values landed on the unintended contracts:

1. Query `key_pointers_keypointsuggestion` for the stuck contract IDs (see BigQuery section)
2. If 0 rows → no extraction happened → tell the customer it's safe, no cleanup needed
3. If rows exist → escalate to engineering for a targeted deletion script

In both confirmed incidents (Feb/March 2026), no data was extracted:
- VOIDED contracts: extraction starts at 0% and immediately fails
- DRAFT/TEMPLATE contracts not past questionnaire: extraction never begins

---

## Escalation Triggers

- Multiple customers reporting simultaneously → check if a recent deployment changed bulk extraction behavior; scan all regions
- `started IS NOT NULL` and task still stuck → beam pipeline issue, not eligibility bug; check GCP Dataflow/Beam logs
- Customer reports extraction finished but returned incorrect data → different issue entirely
- PR #12185/#12186 status unclear → escalate to engineering to confirm merge and prod deployment

---

## Related Incidents & Jira

| Reference | Details |
|-----------|---------|
| **Rootly #3008** | Cedar — Metadata extraction running perpetually (March 17–18, 2026) |
| **Rootly Incident (Feb 2026)** | Prompt Therapy — SDC extraction stuck (Feb 25–27, 2026) |
| **Jira SPD-42477** | Cedar incident ticket — High, Blocked On Client |
| **Slack channel** | `#incident-20260317-high-cedar-metadata-extraction-running-perpetually-on-contract` (C0AMBQ2G3S8) |
| **Feb incident channel** | `#incident-20260225-critical-prompt-therapy-sdc-extraction-stuck` (C0AJ08ZCRRN) |
| **PR #12185** | `django-rest-api` — Do not create async task if SDC is not supported |
| **PR #12186** | `django-rest-api` — Cancel unstarted AsyncTasks in timeout cronjob |
| **Cedar known contracts** | Contract `1319820` (Draft, AsyncTask `540721`), Contract `1319878` (Voided, AsyncTask `540717`) — WSID `63672`, US |
| **Prompt Therapy known contract** | Contract `136490` — WSID `274685`, IN cluster |
| **Runbook doc** | https://docs.google.com/document/d/112ockx5UzHZYuE42rPZ6fARhfODFBJrIVWibNs6nFLY/edit?usp=sharing |

---

## Connections to Other Skills

- **incident-lookup** — Use to find prior incidents for the same workspace before diagnosing
- **contract-lookup** — Use to retrieve contract kind, status, and workspace context for the stuck contracts
