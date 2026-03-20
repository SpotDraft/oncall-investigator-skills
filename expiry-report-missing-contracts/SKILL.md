---
name: expiry-report-missing-contracts
description: "Diagnose why the Weekly Expiry Reminder Report (CONTRACT_EXPIRATION_REPORT) is missing contracts for a recipient. Trigger on: 'contracts missing from expiry report', 'expiry reminder report incomplete', 'weekly expiry report not showing all contracts', 'expiring contracts not in report', 'customer not receiving all expiry reminders', 'report missing contracts after metadata cleanup'. Covers two distinct causes: missing/misconfigured EmailConfig routing rules (primary) and Elasticsearch sync lag after metadata changes (transient/secondary)."
---

# Weekly Expiry Reminder Report — Missing Contracts

## What This Skill Diagnoses

Contracts expiring within the report window are indexed in Elasticsearch but do not appear in the report email received by the customer.

Two distinct, independently occurring root causes:

1. **Missing/misconfigured `EmailConfig` routing (primary):** The `CONTRACT_EXPIRATION_REPORT` email is routed per contract based on `to_business_user` and `to_emails` fields in `emails_emailconfig`. A recipient only gets a contract if they are the contract's Business User (`to_business_user=True`) **or** their email is explicitly in `to_emails` for that contract type's config. Contracts assigned to other BUs are silently not sent — this is WAI, not a bug.

2. **Elasticsearch sync lag (transient):** After a bulk metadata change (e.g., deleting duplicate key pointers, running a metadata cleanup), affected contracts lose their `expiring_at` indexed value temporarily. If the weekly report cron runs before re-sync completes, those contracts are absent from the ES query result. Self-heals when re-sync finishes.

> **Key investigative pivot:** If the ES query returns the expected number of contracts but the recipient still reports missing → root cause is EmailConfig routing (cause 1). If ES query itself returns fewer contracts than expected → investigate re-sync lag (cause 2).

## Investigation Workflow

### Step 1 — Confirm ES query returns expected contracts

Run the backend Elasticsearch query for the workspace on the appropriate cluster. Check:
- Does ES return the number of contracts the customer expects?
- What are the `last_re_sync_at` values for contracts that are missing?

Known ES index name format: `contracts-{timestamp}` (Prod IN cluster confirmed: `contracts-20250312110313772786`).

Key fields to check in the ES source:
- `expiring_at` — must be populated for the contract to appear
- `last_re_sync_at` — if recently after the report run date, sync lag is the cause
- `created_by_workspace` — confirm workspace scope

If ES returns fewer contracts than expected **and** `last_re_sync_at` values are after the report date → **Cause 2: sync lag** (see mitigation below). If ES returns the full expected set → proceed to Step 2.

### Step 2 — Audit EmailConfig routing for the missing contracts

Run this query against the affected workspace (in Metabase or via Django Admin):

```sql
SELECT
    c.id AS contract_id,
    c.contract_type_id,
    ct.name AS contract_type_name,
    ec.id AS email_config_id,
    ec.to_business_user,
    ec.to_emails,
    CASE
        WHEN ec.id IS NULL THEN 'NO CONFIG'
        WHEN ec.contract_type_id IS NULL THEN 'catch-all config'
        ELSE 'per-type config'
    END AS config_scope
FROM contracts_v3_contractv3 c
JOIN contracts_contracttype ct ON ct.id = c.contract_type_id
LEFT JOIN emails_emailconfig ec
    ON (ec.contract_type_id = c.contract_type_id OR ec.contract_type_id IS NULL)
    AND ec.workspace_id = {wsid}
    AND ec.email_type = 'CONTRACT_EXPIRATION_REPORT'
WHERE c.id IN ({contract_ids})
ORDER BY config_scope, c.id
```

> Replace `{contract_ids}` with the comma-separated list of missing contract IDs from the customer's report.

### Step 3 — Interpret the 3 contract routing buckets

For each contract in the result, apply this decision tree:

| Scenario | Behaviour | Root Cause |
|----------|-----------|------------|
| Recipient IS the BU **and** `to_business_user=True` | ✅ Sent correctly | — |
| Recipient IS NOT the BU; a different BU is assigned; recipient not in `to_emails` | ❌ Not sent to recipient | Missing `to_emails` entry in EmailConfig |
| Recipient IS the BU **but** `to_business_user=False` and not in `to_emails` | ❌ Silently dropped | `to_business_user=False` + no `to_emails` fallback |
| `config_scope = 'NO CONFIG'` | ❌ Dropped entirely | No EmailConfig for this contract type |

**Confirmed example (Dvara Holdings, WSID 122251, March 2026):**
- 18 contracts sent correctly (arvind IS BU, `to_business_user=True`)
- 34 not sent (BU = other users; no `to_emails` entry for arvind)
- 2 not sent (arvind IS BU but `to_business_user=False` on those 2 contract types; not in `to_emails`)

### Step 4 — Confirm whether missing is WAI or misconfiguration

Ask support/CS: Does the customer expect the recipient to receive **all workspace contracts** regardless of who the Business User is?

- **Yes** → Missing EmailConfig. The workspace has not configured a workspace-wide catch-all `to_emails` recipient. This is a configuration gap, not a code bug. Fix: add the recipient's email to `to_emails` in the relevant EmailConfigs.
- **No** → The report is working as intended. Explain the routing logic to the customer.

## Mitigation

### Cause 1 — Missing EmailConfig routing

Two types of fix depending on the bucket:

**For contracts where recipient is NOT the BU (34-contract pattern):**
Add the recipient email to `to_emails` field in `emails_emailconfig` for those contract types.
- Admin URL: `https://api.{cluster}.spotdraft.com/admin/emails/emailconfig/?workspace_id={wsid}&email_type=CONTRACT_EXPIRATION_REPORT`
- Metabase: Query the `emails_emailconfig` table filtered by `workspace_id={wsid}` and `email_type='CONTRACT_EXPIRATION_REPORT'`

**For contracts where recipient IS the BU but `to_business_user=False` (2-contract pattern):**
Enable `to_business_user=True` for those contract types' EmailConfig rows, OR add the recipient to `to_emails`.

> ⚠️ Confirm intent with CS before modifying EmailConfig — adding a recipient to `to_emails` means they will receive expiry reminders for ALL contracts of that type going forward, regardless of BU assignment.

### Cause 2 — Elasticsearch sync lag

No active fix needed — the re-sync is self-healing. Verify by checking `last_re_sync_at` against the report date:
- If `last_re_sync_at > report_date` for missing contracts → those contracts were not indexed at report time
- Next week's report will include them once re-sync completes

If the customer needs an immediate re-send, an engineer must manually trigger re-sync for the affected contracts or re-run the weekly report for those specific contracts (no self-service tooling exists for this as of March 2026 — escalate to engineering).

> **Observability gap:** Weekly expiry reminder reports are generated in-memory with no backup/logging of which contract IDs were included. There is no way to audit a past report's exact content. Jira tracking: SPD-42457.

## EmailConfig Schema

Table: `emails_emailconfig`

| Column | Type | Purpose |
|--------|------|---------|
| `id` | INT | Primary key |
| `workspace_id` | INT | Target workspace |
| `email_type` | VARCHAR | `CONTRACT_EXPIRATION_REPORT` for expiry reminders |
| `contract_type_id` | INT \| NULL | Per-type config (NULL = catch-all for all contract types in workspace) |
| `to_business_user` | BOOL | Send to the contract's assigned Business User |
| `to_emails` | ARRAY/JSON | Explicit email addresses to always include |

A contract is included in a recipient's report if **any** of these are true:
- Recipient is the BU **and** `to_business_user=True`
- Recipient's email is in `to_emails` for that contract type's config
- A catch-all config (`contract_type_id IS NULL`) exists with `to_business_user=True` and the recipient is the BU, **or** recipient is in its `to_emails`

## Admin & Diagnostic URLs

| Purpose | URL |
|---------|-----|
| EmailConfig admin | `https://api.{cluster}.spotdraft.com/admin/emails/emailconfig/?workspace_id={wsid}` |
| Email audit (past sends) | `https://api.{cluster}.spotdraft.com/admin/emails/emailaudit/?q={recipient_email}` |
| Workspace admin | `https://api.{cluster}.spotdraft.com/admin/core/workspace/{wsid}/change/` |
| Metabase (IN) — EmailConfig query | `https://metabase.in.spotdraft.com/question/` (build inline query on `emails_emailconfig` table, filter by `workspace_id` and `email_type`) |

## Confirming vs Ruling Out

| Check | Confirms | Rules out |
|-------|----------|-----------|
| ES query returns fewer contracts than customer expects | ES sync lag (Cause 2) | EmailConfig routing (Cause 1) |
| ES query returns full expected set; recipient still gets subset | EmailConfig routing (Cause 1) | ES sync lag |
| `last_re_sync_at` on missing contracts > report date | Sync lag | — |
| `config_scope = 'per-type config'`; recipient not in `to_emails`; recipient ≠ BU | Missing EmailConfig entry | — |
| `to_business_user = False`; recipient IS BU | Wrong flag on EmailConfig | — |

## Known Issue Patterns

### Pattern 1: Missing EmailConfig for workspace-level recipient (Dvara Holdings — Mar 2026)

**Symptom:** Recipient receives 18 of 54 expiring contracts. Others go to other BU addresses.

**Root cause:** No EmailConfig entry adding recipient to `to_emails` for the 34 contract types where they are not the BU.

**Confirmed IDs:** WSID 122251, IN cluster. Jira: SPD-42256. Incident: [#incident-20260312-high-weekly-expiry-reminder-report-missing-contracts-expiring-](https://spotdraft.slack.com/archives/C0ALHU60N2V)

**Resolution:** This is WAI — explained to customer that EmailConfig must be set up to route to them. Mitigation: add recipient email to `to_emails` in relevant EmailConfig rows.

### Pattern 2: ES sync lag after metadata cleanup (Dvara Holdings — Mar 2026, transient)

**Symptom:** 6 contracts missing from March 9 report. All 6 had `last_re_sync_at` values of March 13 — after the report ran.

**Trigger:** Bulk metadata cleanup (SPD-41927) deleted duplicate Expiry Date key pointer fields and unset `is_global` flag across 228 contract types. Affected contracts lost their `expiring_at` ES value temporarily.

**Self-healed:** Re-sync completed March 13. March 16 report returned all 54 contracts in ES.

## Escalation Triggers

- ES returns expected contracts; EmailConfig looks correct; recipient still gets subset → check if email audit shows delivery to a forwarding address or alias that the customer doesn't control
- Customer wants all workspace contracts routed to one recipient permanently → requires adding email to `to_emails` on every EmailConfig for the workspace (or creating a catch-all config with `contract_type_id=NULL`)
- Large-scale re-sync lag (many workspaces affected after a platform migration) → escalate to engineering to trigger batch re-sync

## Related Incidents & Jira

| Reference | Details |
|-----------|---------|
| SPD-42256 | Dvara Holdings — Weekly Expiry Report missing contracts |
| SPD-41927 | Metadata cleanup (duplicate Expiry Date fields removed across 228 contract types) — the trigger for the ES sync lag |
| SPD-42457 | Observability gap: weekly expiry report attachment generated in-memory with no logging |
| [Incident channel C0ALHU60N2V](https://spotdraft.slack.com/archives/C0ALHU60N2V) | Full investigation thread |

## Connections to Other Skills

- **`email-delivery-triage`** — if EmailConfig is correct but emails aren't arriving → check Postmark suppression
- **`contract-lookup`** — for workspace and cluster identification
- **`recurring-reports-post-offboarding`** — if the issue is recurring scheduled reports continuing after churn/trial expiry (different report type: `RecurringReportSchedule`, not `CONTRACT_EXPIRATION_REPORT`)
