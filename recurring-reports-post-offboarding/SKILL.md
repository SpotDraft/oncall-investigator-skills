---
name: recurring-reports-post-offboarding
description: "Diagnose and stop scheduled recurring reports still firing after a trial has expired or a workspace has been churned/offboarded. Trigger on: 'customer still receiving scheduled reports', 'recurring report after trial ended', 'delete recurring reports', 'reports firing after workspace deactivated', 'stop recurring email report', 'purge recurring report schedule', 'tenant receiving reports after trial expiry'."
---

# Recurring Reports Firing After Trial Expiry or Workspace Offboarding

This skill diagnoses the class of incidents where users continue receiving scheduled recurring report emails after their trial has expired or their workspace has been churned — and provides the mitigation to stop them immediately.

## What This Skill Debugs

Churned or expired workspaces can leak emails through **two independent email systems**. Both must be checked.

**System 1 — `RecurringReportSchedule` → `SCHEDULED_GENERATED_CONTRACT_SEARCH_REPORT`**

`RecurringReportSchedule` rows are not deleted or deactivated when:
1. A **trial workspace expires** — `UpdateExpiredTrialSignupsUseCase` and `PurgeTrialDataUseCase` (pre-fix) did not clean up `RecurringReportSchedule` rows.
2. A **workspace is churned/offboarded without the `delete_workspace` management command** — the soft-delete chain in `SoftDeletionModel` does not fire Django's DB-level `CASCADE`, leaving `RecurringReportSchedule` rows alive.

The daily cron (`send_due_today_recurring_report_cron_task`) has **no trial-status or workspace-active guard** — it picks up all `RecurringReportSchedule` rows where `next_send_at` falls today and `is_deleted=False`, regardless of workspace state.

**System 2 — `EmailConfig` + `EXPIRING_CONTRACTS_EMAIL_REPORT_ENABLED` FF → `CONTRACT_EXPIRATION_REPORT`**

The weekly contract expiry reminder system is independent of `RecurringReportSchedule`. It fires based on `emails_emailconfig` rows and the workspace-level feature flag `EXPIRING_CONTRACTS_EMAIL_REPORT_ENABLED`. When a workspace is churned without running `delete_workspace`, `EmailConfig` rows survive and the feature flag remains enabled — so expiry reminder emails keep firing weekly even after all `RecurringReportSchedule` rows have been soft-deleted.

> **VSCO trap (Mar 2026):** After soft-deleting all `RecurringReportSchedule` rows for WS 27382 (Feb 27 mitigation), the customer still received a `CONTRACT_EXPIRATION_REPORT` email on Mar 10. The two systems are independent — stopping one does not stop the other.

## Available MCP Tools

| Tool | Use |
|------|-----|
| **BigQuery MCP (`execute_sql`)** | Inspect `contracts_v3_recurringreportschedule` rows for the affected workspace. Run queries sequentially — parallel BQ calls fail. |
| **Slack MCP** | Confirm symptom details, read past incident threads, check if similar recurrences occurred |
| **Atlassian MCP (Jira)** | Check fix status for related Jira tickets |
| **Groundcover / GCP Logs** | Confirm cron is firing (no errors — the cron works correctly, this is a product gap) |
| **Admin panel** | Inspect and manually delete/disable recurring report schedules |

> **⚠️ No runtime errors in GCP Logs** — this issue leaves no exception traces. The cron fires and sends correctly from the system's perspective. Do not waste time searching logs for errors.

## Environment-to-Dataset Mapping

| Cluster | BQ Project | Dataset | Table |
|---------|-----------|---------|-------|
| Prod EU | `spotdraft-prod` | `prod_eu_db` | `public_contracts_v3_recurringreportschedule` |
| Prod India | `spotdraft-prod` | `prod_india_db` | `public_contracts_v3_recurringreportschedule` |
| Prod USA | `spotdraft-prod` | `prod_usa_db` | `public_contracts_v3_recurringreportschedule` |
| QA India | `spotdraft-qa` | `qa_india_public` | `contracts_v3_recurringreportschedule` |
| QA EU | `spotdraft-qa` | `qa_eu_public` | `contracts_v3_recurringreportschedule` |

> **Note on prod table names:** Prod tables have a `public_` prefix. QA tables have no prefix.

> **⚠️ BigQuery parallel query bug:** Do NOT fire multiple `execute_sql` calls in the same message — all return 500 errors. Always run BQ queries sequentially (one per message).

## Key Table Schema

**`contracts_v3_recurringreportschedule`:**
`id`, `tenant_workspace_id` (INT), `created_by_org_user_id` (INT), `search_view_id` (INT), `schedule_expression` (VARCHAR, RRULE format), `schedule_expression_readable`, `next_send_at` (DATETIME), `last_sent_at` (DATETIME), `is_deleted` (BOOL), `cc_recipients` (ARRAY/JSON of emails), `to_recipients` (ARRAY), `columns_filter_type`, `attach_report_to_email` (BOOL), `report_file_format`, `notes`, `created`, `modified`

## Investigation Workflow

### Step 1 — Confirm the symptom

Get the workspace ID (WSID) and cluster from the support ticket. Confirm:
- Trial has ended or workspace is churned/deactivated
- Customer is receiving scheduled report emails (`SCHEDULED_GENERATED_CONTRACT_SEARCH_REPORT`)

### Step 2 — Count active recurring report schedules in BQ

```sql
-- Replace prod_eu_db with the correct dataset for your cluster
SELECT
  id, tenant_workspace_id, created_by_org_user_id,
  schedule_expression, schedule_expression_readable,
  next_send_at, last_sent_at, is_deleted, cc_recipients
FROM `spotdraft-prod.prod_eu_db.public_contracts_v3_recurringreportschedule`
WHERE tenant_workspace_id = {wsid}
  AND is_deleted = FALSE
ORDER BY created DESC
```

If rows exist with `is_deleted=FALSE` → root cause confirmed. These will keep firing.

### Step 3 — Check for cc_recipients scope (churned workspace variant)

For churned workspace incidents, also check if the recipient's email appears as a `cc_recipient` on schedules in **other** workspaces:

```sql
-- Check if a specific email is cc'd on other workspaces' schedules
SELECT id, tenant_workspace_id, cc_recipients, is_deleted
FROM `spotdraft-prod.prod_usa_db.public_contracts_v3_recurringreportschedule`
WHERE cc_recipients::text ILIKE '%{recipient_email}%'
  AND is_deleted = FALSE
```

### Step 3b — Check for CONTRACT_EXPIRATION_REPORT email configs (churned workspace variant)

Even after all `RecurringReportSchedule` rows are soft-deleted, the workspace may have active `EmailConfig` rows driving `CONTRACT_EXPIRATION_REPORT` emails. Check:

1. **Django Email Audit** — look up the specific email that the customer received:
   - `https://api.{cluster}.spotdraft.com/admin/emails/emailaudit/?q={recipient_email}`
   - Confirm the email type. If it is `CONTRACT_EXPIRATION_REPORT` (not `SCHEDULED_GENERATED_CONTRACT_SEARCH_REPORT`), the source is System 2.

2. **Check `EmailConfig` rows** for the workspace (via BigQuery):
```sql
SELECT id, workspace_id, email_type, contract_type_id, to_business_user, to_emails
FROM `spotdraft-prod.{dataset}.public_emails_emailconfig`
WHERE workspace_id = {wsid}
  AND email_type = 'CONTRACT_EXPIRATION_REPORT'
```
If rows exist → the expiry reminder system will keep firing regardless of `RecurringReportSchedule` state.

3. **Mitigation for System 2**: disable the feature flag `EXPIRING_CONTRACTS_EMAIL_REPORT_ENABLED` for the workspace (see Mitigation section below). This is the quickest stop — no script needed.

### Step 4 — Confirm workspace trial/churn status

Check whether this is:
- **Trial expiry** — `UpdateExpiredTrialSignupsUseCase` ran but `delete_by_workspace` wasn't called (pre-fix)
- **Workspace churn without `delete_workspace` command** — workspace deactivated through another path, leaving `SavedSearchView` and `RecurringReportSchedule` rows alive

### Step 5 — Mitigate immediately

Choose one of the options below:

**Option A (preferred): Django shell soft-delete** — run on the correct cluster:
```python
from contracts_v3.saved_search_view.data.models import RecurringReportSchedule

# Soft-deletes all recurring report schedules for the workspace
deleted_count, _ = RecurringReportSchedule.objects.filter(
    tenant_workspace_id={wsid}
).delete()
print(f"Deleted {deleted_count} recurring report schedules for workspace {wsid}")
```

> **Note:** `RecurringReportSchedule(SDBaseOrgUserDBModel)` uses soft delete — `delete()` sets `is_deleted=True`, does NOT remove the DB row. The default `.objects` manager filters `is_deleted=False`, so soft-deleted records become invisible to the daily cron. This is sufficient to stop reports.

**Option B (no Django shell access): Admin Email Templates** — disable the email type for the workspace from the admin panel:
- Navigate to `https://api.{cluster}.spotdraft.com/admin/emails/emailtemplate/`
- Find `SCHEDULED_GENERATED_CONTRACT_SEARCH_REPORT` for the workspace
- Disable it for WSID `{wsid}`

> This was used for Superior Pipeline Services (WSID 525076, US cluster, Nov 2025) as a quick workaround.

**Option C: Stop `CONTRACT_EXPIRATION_REPORT` emails (System 2)**

If the email audit shows `CONTRACT_EXPIRATION_REPORT` as the email type, the source is the expiry reminder system — not `RecurringReportSchedule`. Mitigation:

1. **Disable the feature flag `EXPIRING_CONTRACTS_EMAIL_REPORT_ENABLED` for the workspace** — this disables expiry report emails entirely for the workspace. No Django shell required; done via the feature flag admin UI or Kratos config.
   - Confirmed fix for VSCO WS 27382 (US cluster, Mar 21 2026): Sandeep disabled `EXPIRING_CONTRACTS_EMAIL_REPORT_ENABLED` for `US_27382` and customer received no further emails.

2. Alternatively, delete the `EmailConfig` rows for `CONTRACT_EXPIRATION_REPORT` for that workspace via Django admin:
   - `https://api.{cluster}.spotdraft.com/admin/emails/emailconfig/?workspace_id={wsid}&email_type=CONTRACT_EXPIRATION_REPORT`

### Step 6 — Verify with BQ after mitigation

```sql
SELECT id, is_deleted, next_send_at
FROM `spotdraft-prod.{dataset}.public_contracts_v3_recurringreportschedule`
WHERE tenant_workspace_id = {wsid}
  AND is_deleted = FALSE
```

Should return zero rows after the script runs.

### Step 7 — Watch for recurrence (churned workspace variant)

After the soft-delete, confirm with the customer that no further emails arrive in the next 24–48 hours. For the churned-workspace class:
- If the recurrence continues after soft-delete, the remaining schedule is likely owned by a **different workspace/org user** with the customer's email in `cc_recipients`
- Run the `cc_recipients` scan in Step 3 to find it

## System / Code Flow

```
Cron: send_due_today_recurring_report_cron_task
  → FetchDueTodayAndTriggerProcessRecurringReportScheduleUseCase.execute()
    → contracts_v3/saved_search_view/domain/use_cases/fetch_due_today_and_trigger_process_recurring_report_schedule_use_case.py
    → RecurringReportSchedule.objects.filter(
          next_send_at__gte=today_min,
          next_send_at__lte=today_max,
          is_deleted=False          ← NO trial/workspace-active guard here
      )
      → bulk_update(next_send_at, last_sent_at)
      → process_recurring_schedule_and_trigger_report_generation_task.delay() [Celery]
          → SendScheduledGeneratedSearchReportEmailUseCase
              → Email: type SCHEDULED_GENERATED_CONTRACT_SEARCH_REPORT
                  → sent to org users + cc_recipients
```

**Failure point:** No guard against expired trial workspaces or deactivated workspaces. The cron has no awareness of `demo_access_end_date` or `TrialStatusEnum.EXPIRED`.

**Soft-delete cascade trap:** `SavedSearchView` (FK from `RecurringReportSchedule`, `on_delete=CASCADE`) uses `SoftDeletionModel` — so `SavedSearchView.delete()` only sets `is_deleted=True` on the view. Django's DB-level `CASCADE` never fires. This means simply marking a workspace as churned via non-`delete_workspace` paths leaves `RecurringReportSchedule` rows alive with `is_deleted=False`.

**`delete_workspace` command** (`sd_organizations_v2/management/commands/delete_workspace.py`) calls `hard_delete()` on `SavedSearchView` (line 598–604), which properly cascades to `RecurringReportSchedule` — but only if the workspace offboarding goes through this command.

## Key Code Paths

| File | Purpose |
|------|---------|
| `contracts_v3/saved_search_view/data/models.py` | `RecurringReportSchedule` model — `tenant_workspace_id` field |
| `contracts_v3/saved_search_view/data/db_repository.py` | `RecurringReportScheduleDbRepository` — `delete_for_view()` scoped to org_user + view; `delete_by_workspace()` needed but not yet implemented |
| `contracts_v3/saved_search_view/data/abstract_repository.py` | `RecurringReportScheduleAbstractRepository` — abstract interface |
| `contracts_v3/saved_search_view/domain/use_cases/fetch_due_today_and_trigger_process_recurring_report_schedule_use_case.py` | Daily cron use case — no trial guard |
| `sd_organizations_v2/trial/domain/use_cases/update_expired_trial_signups_use_case.py` | Trial expiry — now calls `delete_by_workspace()` as part of the permanent fix |
| `sd_organizations_v2/trial/domain/use_cases/purge_trial_data.py` | Trial data purge — deletes contracts, contract types, templates, counterparties — but NOT recurring reports (pre-fix) |
| `sd_organizations_v2/management/commands/delete_workspace.py` | Workspace deletion command — uses `hard_delete()` on `SavedSearchView` which cascades properly |

## Known Issue Patterns

### Pattern 1: Expired Trial Workspace (GC Europe, Mar 2026)

**Symptom:** Customer workspace trial ended but users still receive daily/weekly recurring report emails.

**Root cause:** `UpdateExpiredTrialSignupsUseCase.execute()` marked trials as `EXPIRED` but did not delete `RecurringReportSchedule` rows. `PurgeTrialDataUseCase` also skipped recurring reports.

**Confirmed example:** WSID 258204, EU cluster. Jira: SPD-42456 (Permanently Fixed).

**Fix status:** `UpdateExpiredTrialSignupsUseCase` has been updated to call `delete_by_workspace(workspace_id)` after marking a trial `EXPIRED`. The `RecurringReportScheduleAbstractRepository` and `RecurringReportScheduleDbRepository` methods are part of the fix in progress.

### Pattern 2: Churned Workspace Without `delete_workspace` Command (VSCO, Feb–Mar 2026)

**Symptom:** Customer workspace was deactivated/churned. Customer continues receiving scheduled reports. Soft-deleting all `RecurringReportSchedule` rows for the workspace appeared to fix it, but a different email type recurred weeks later.

**Root cause (phase 1 — Feb):** Workspace offboarded via non-standard path (not `delete_workspace` command). `SavedSearchView` and `RecurringReportSchedule` rows were never deleted. 4 active `RecurringReportSchedule` rows soft-deleted Feb 27 via script.

**Root cause (phase 2 — Mar 10 recurrence):** The Mar 10 email was **not** a `SCHEDULED_GENERATED_CONTRACT_SEARCH_REPORT` — it was a `CONTRACT_EXPIRATION_REPORT` (email audit ID: `77c795e6-b621-4e8d-ae6d-77d442dc1bec`). This is System 2 (expiry reminder), which fires from `emails_emailconfig` rows independently of `RecurringReportSchedule`. Those `EmailConfig` rows were still present and `EXPIRING_CONTRACTS_EMAIL_REPORT_ENABLED` was still enabled for the workspace.

**Confirmed example:** WSID 27382 (VSCO), US cluster. Jira: SPD-41569.

**Resolution (Mar 21):** Sandeep disabled the feature flag `EXPIRING_CONTRACTS_EMAIL_REPORT_ENABLED` for workspace `US_27382`. Customer confirmed no further emails.

**Key lesson:** A churned workspace requires stopping BOTH email systems — `RecurringReportSchedule` rows (System 1) AND `EmailConfig` / `EXPIRING_CONTRACTS_EMAIL_REPORT_ENABLED` (System 2). Stopping System 1 alone is insufficient.

### Pattern 3: Deactivated Individual User (krazybee, Dec 2025)

**Symptom:** A single user's recurring report continues after their account is deactivated in the workspace.

**Root cause:** `RecurringReportSchedule` rows created by the deactivated org_user are not cleaned up on user deactivation.

**Mitigation:** Same Django shell script with `created_by_org_user_id={org_user_id}` filter instead of `tenant_workspace_id`.

## Confirming vs Ruling Out

| Check | Confirms issue | Rules out |
|-------|---------------|-----------|
| BQ: `is_deleted=FALSE` RecurringReportSchedule rows for WSID | ✅ System 1 (recurring schedule) active | — |
| Email audit: type = `SCHEDULED_GENERATED_CONTRACT_SEARCH_REPORT` | ✅ System 1 source | — |
| Email audit: type = `CONTRACT_EXPIRATION_REPORT` | ✅ System 2 (expiry reminder) source | ✅ Not System 1 |
| BQ: `is_deleted=FALSE` rows = 0 but customer still gets emails | — → check System 2 | ✅ System 1 stopped |
| BQ: `emails_emailconfig` rows exist with `email_type='CONTRACT_EXPIRATION_REPORT'` | ✅ System 2 will keep firing | — |
| `EXPIRING_CONTRACTS_EMAIL_REPORT_ENABLED` FF is ON for workspace | ✅ System 2 active | — |
| GCP Logs: errors in either email task | — | ✅ Not this issue (no errors expected) |
| `delete_workspace` command in past job history | — | ✅ CMD would have hard-deleted correctly |
| Recurrence after soft-delete → check email audit type first | ✅ System 2 if type = CONTRACT_EXPIRATION_REPORT | — |

## Mitigation Summary

| Scenario | Who runs it | Where |
|----------|------------|-------|
| System 1: Django shell — soft-delete all schedules for workspace | On-call engineer | EU/US/IN cluster Django shell |
| System 1: Admin Email Templates — disable SCHEDULED_GENERATED email type | Support/On-call | Django admin email templates page |
| System 1: Django shell — soft-delete schedules by org user | On-call engineer | Cluster Django shell (for user deactivation case) |
| System 2: Disable `EXPIRING_CONTRACTS_EMAIL_REPORT_ENABLED` FF for workspace | On-call engineer | Feature flag admin / Kratos config |
| System 2: Delete `EmailConfig` rows for `CONTRACT_EXPIRATION_REPORT` | On-call engineer | Django admin: `emails/emailconfig/?workspace_id={wsid}` |

**Django shell scripts:**

```python
# Case 1: All schedules for a workspace
from contracts_v3.saved_search_view.data.models import RecurringReportSchedule
deleted_count, _ = RecurringReportSchedule.objects.filter(
    tenant_workspace_id={wsid}
).delete()
print(f"Deleted {deleted_count} schedules for workspace {wsid}")

# Case 2: Schedules by a specific org user
deleted_count, _ = RecurringReportSchedule.objects.filter(
    created_by_org_user_id={org_user_id}
).delete()
print(f"Deleted {deleted_count} schedules for org_user {org_user_id}")

# Case 3: Confirm cleanup
from contracts_v3.saved_search_view.data.models import RecurringReportSchedule
remaining = RecurringReportSchedule.objects.filter(
    tenant_workspace_id={wsid}, is_deleted=False
).count()
print(f"Remaining active schedules: {remaining}")  # should be 0
```

## Escalation Triggers

- Reports continue after mitigation script ran → check `cc_recipients` across other workspaces (VSCO pattern)
- Django shell access unavailable → escalate to on-call eng with admin panel access to disable email type
- Fix is not yet deployed and new trials are expiring frequently → add defensive guard in `FetchDueTodayAndTriggerProcessRecurringReportScheduleUseCase` to skip schedules for `TrialStatusEnum.EXPIRED` workspaces

## Long-Term Fix (Status)

**System 1 (RecurringReportSchedule):**
1. ✅ **`delete_by_workspace(workspace_id)`** added to `RecurringReportScheduleAbstractRepository` and `RecurringReportScheduleDbRepository` — hooks into `UpdateExpiredTrialSignupsUseCase.execute()` after marking trial `EXPIRED`.
2. ⬜ **`PurgeTrialDataUseCase`** — should also call `delete_by_workspace` for completeness.
3. ⬜ **Defensive guard in cron** — `FetchDueTodayAndTriggerProcessRecurringReportScheduleUseCase` should skip schedules for workspaces with `TrialStatusEnum.EXPIRED` or deactivated status.
4. ⬜ **Workspace offboarding process** — ensure all churn paths go through `delete_workspace` management command (or equivalent cleanup) so `hard_delete()` cascades correctly.
5. ⬜ **`cc_recipients` validation** — prevent schedules from sending to emails of deactivated users across workspaces.

**System 2 (CONTRACT_EXPIRATION_REPORT / EmailConfig):**
6. ⬜ **Disable `EXPIRING_CONTRACTS_EMAIL_REPORT_ENABLED` as part of workspace churn offboarding** — the workspace churn process should automatically disable this FF (and ideally delete `EmailConfig` rows) when a workspace is churned/deactivated.
7. ⬜ **Add active-workspace guard to expiry reminder pipeline** — before sending `CONTRACT_EXPIRATION_REPORT`, verify the `tenant_workspace` is still active/not-churned.

## Related Incidents & Jira

| Incident | Channel | WSID | Cluster | Jira | Status |
|----------|---------|------|---------|------|--------|
| GC Europe — trial expiry | [#incident-20260317-medium-gc-europe-need-help-in-deleting-recurring-reports](https://spotdraft.slack.com/archives/C0ALZGSKFS9) | 258204 | EU | SPD-42456 | Permanently Fixed |
| VSCO — workspace churn (System 1) | [#incident-20260319-medium-vsco-...](https://spotdraft.slack.com/archives/C0AHXSXACAC) | 27382 | US | SPD-41569 | Mitigated Feb 27 (RecurringReportSchedule soft-deleted) |
| VSCO — workspace churn recurrence (System 2) | same channel | 27382 | US | SPD-41569 | Mitigated Mar 21 (EXPIRING_CONTRACTS_EMAIL_REPORT_ENABLED FF disabled) |
| Superior Pipeline Services — expired trial | #eng (Nov 2025) | 525076 | US | — | Workaround via admin |
| krazybee — deactivated user | #cs-support (Dec 2025) | — | — | — | Manual deletion |

## Admin Panel URLs by Cluster

| Cluster | Admin URL |
|---------|-----------|
| EU | `https://api.eu.spotdraft.com/admin/` |
| US | `https://api.us.spotdraft.com/admin/` |
| IN | `https://api.in.spotdraft.com/admin/` |
| ME | `https://api.me.spotdraft.com/admin/` |

**Useful admin paths:**
- Recurring report schedules: `admin/contracts_v3/recurringreportschedule/?tenant_workspace_id={wsid}`
- Email templates (to disable email type): `admin/emails/emailtemplate/?q=SCHEDULED_GENERATED`
- Email audit (to confirm delivery): `admin/emails/emailaudit/?q={recipient_email}`
- Workspace: `admin/core/workspace/{wsid}/change/`

## Connections to Other Skills

- **`contract-lookup`** — for workspace state, trial status, and cluster identification
- **`email-delivery-triage`** — if the issue is about wrong delivery or bounced emails (not this skill's scope)
- **`incident-lookup`** — to find past Rootly incidents for this workspace or similar pattern
