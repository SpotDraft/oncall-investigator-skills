---
name: clickwrap-tray-salesforce-sync-triage
description: "Triage clickthrough (clickwrap) contracts that exist in SpotDraft with salesforce_id but Salesforce fields stay blank — Tray shows no logs for KP Created / reverse sync. Covers Tray solution disabled, Native vs Tray conflict for clickthrough, and bulk retrigger of KP Created webhooks. Use when: clickwrap not syncing to Salesforce, Abnormal-style Tray integration, empty Tray logs, reverse sync clickthrough, KP created webhook, US_147206 pattern. Trigger phrases: clickthrough Salesforce blank fields, Tray solution disabled, retrigger KP created, native integration clickwrap conflict."
---

# Clickwrap → Salesforce (Tray) Sync Triage

## What this skill debugs

**Post-execution** issues: the **clickwrap contract** is created in SpotDraft (often with `salesforce_id` / external link), but **mapped Key Pointer (KP) fields** on the **Salesforce** Account or Opportunity stay **empty**. Customer may report **no entries in Tray logs** for that `contract_id`.

**Common root cause (production-confirmed pattern):** The **Tray.io solution instance** that listens for **KP Created** (or equivalent reverse-sync webhooks) was **disabled** for a date range. Contracts created while disabled **never** triggered a successful Tray run — enabling Tray again **does not retroactively** send webhooks.

**Architectural constraint:** **Native** reverse sync expects contracts tied to a **workflow**. **Clickthrough** audits are often tied to a **contract type only**, not a workflow — so **Native reverse sync can fail** for clickthrough while **Tray-based** custom sync works. **Do not enable Native and Tray reverse sync together** for the same flow without explicit integration-team approval (incident: mutual exclusion required — **Tray on, Native off** for mitigation).

---

## When to use / when not to use

**Use when:**

- **CLICKWRAP** / clickthrough contract + **Salesforce** sync-back / KP fields
- **Empty Tray logs** for contract id filter
- Discussion of **solution instance disabled**, **re-enable**, **retrigger** webhooks

**Do not use for:**

- **400** on clickwrap **submit** API → **clickwrap-400-submission-triage**
- **download_link** 404 / `ContractVersionNotFound` right after create → **contract-lookup** (CLICKWRAP + Tray race)
- **HubSpot / SFDC questionnaire** intake → other skills

---

## Investigation workflow

### Step 1: Confirm contract shape

1. `{contract_id}` — open in app + Django admin.
2. Confirm **kind** = CLICKWRAP / clickthrough path.
3. Check **ExternalIntegrationContractDetail** (or equivalent) — `salesforce_id` may be present **without** KP sync having run.

### Step 2: Tray solution instance status

1. In **Tray.io**, open the workspace workflow used for **reverse sync** / **KP Created** (customer-specific; get link from CS or integration docs).
2. Confirm **solution instance is enabled** and **not paused** — note **who disabled** and **when** (GCP audit / Tray audit if available).

**From incident thread (example timeline — use as pattern only):**

- Solution **disabled** → sync stops immediately for new contracts.
- Solution **re-enabled** → **new** contracts may sync; **old** ones need **retrigger**.

### Step 3: Native vs Tray

Ask **integrations / IE**: Is **Native Salesforce reverse sync** enabled for this customer?

- If **Native** is on for clickthrough path and **Tray** was disabled, behavior can be **confusing** (Native may not support clickthrough KP sync the same way).
- **Mitigation pattern from incident:** **Re-enable Tray** for reverse sync and **disable Native** reverse sync until **CI-2100**-class work ties clickthrough audits to a workflow.

### Step 4: BigQuery — Tray table (optional)

Use **contract-lookup** skill — **Prod USA** Tray table:

```sql
SELECT id, solution_instance_id, is_enabled, is_installed, created, modified
FROM `spotdraft-prod.prod_usa_db.public_tray_integrations_traysolutioninstance`
WHERE created_by_workspace_id = {workspace_id}
  AND is_enabled = TRUE
  AND is_installed = TRUE
```

> Column is **`created_by_workspace_id`**, not `workspace_id` — see contract-lookup.

### Step 5: GCP logs

Incident referenced **Cloud Logging** short links for Tray disable/enable events — use similar filters for **audit** who toggled integration flags (exact log names depend on current infra; ask `#integrations-oncall` if unsure).

---

## Mitigation: bulk retrigger KP Created (clickthrough)

After Tray is **enabled** and **Native** conflict is resolved:

1. Resolve **numeric** `{workspace_id}` (e.g. support may say `US_147206` → **147206** — verify in admin / BQ).
2. Define **date range** of affected contracts (example from incident: **2026-02-04** through **2026-02-26**).
3. Run management command **dry run first** (command ships in **django-rest-api** — verify current name/flags in repo):

```bash
# Dry run (preview only)
python manage.py retrigger_kp_created_webhook_for_clickthrough_contracts \
    --workspace_id={workspace_id} \
    --start_date=YYYY-MM-DD \
    --end_date=YYYY-MM-DD \
    --dry_run

# Execute
python manage.py retrigger_kp_created_webhook_for_clickthrough_contracts \
    --workspace_id={workspace_id} \
    --start_date=YYYY-MM-DD \
    --end_date=YYYY-MM-DD
```

**Operational notes from incident:**

- Script/command may require **auth user** and **org user** ids aligned with **clickwrap creator** from **MergeBase (MB)** — confirm with author of the PR before prod run.
- **Code review / merge:** e.g. [django-rest-api PR #12411](https://github.com/SpotDraft/django-rest-api/pull/12411) (comment thread referenced in Slack); initial implementation in [PR #12269](https://github.com/SpotDraft/django-rest-api/pull/12269).
- Internal doc (if present in repo): `docs/incidents/ABNORMAL_SECURITY_RETRIGGER_KP_WEBHOOK_MITIGATION.md`

4. **Verify:** Tray logs for sample `contract_id`s + Salesforce fields populated.

---

## Long-term fix

| Tracker | Purpose |
|---------|---------|
| **CI-2100** | Associate **clickthrough audits** with a **workflow** so **Native** integration can replace Tray for reverse sync ([Jira](https://spotdraft.atlassian.net/browse/CI-2100)) |

Until then, document **Tray dependency** for this customer’s clickthrough → SFDC KP sync.

---

## Confirm / disprove

| Evidence | Conclusion |
|----------|------------|
| Tray disabled in window; no logs for contract | **Missing webhooks** — retrigger after fix |
| Tray enabled; logs show 4xx/5xx | **Payload / auth** — different triage |
| Native on + Tray off; clickthrough only | **Wrong integration path** for KP sync |
| download_link 404 at create time | **Race** — contract-lookup CLICKWRAP section |

---

## Escalation triggers

- **Many workspaces** affected after a **platform** change → integrations platform team.
- **Single customer**, confirmed Tray gap → runbook + CS; **IE** for Tray/Native toggles.
- **Command fails** or wrong user context → **django-rest-api** owners + **dry run** only until fixed.

---

## Related incidents & Jira

| Reference | Notes |
|-----------|--------|
| **SPD-42002** | Task / incident ticket linked from Rootly |
| **Rootly #2954** | [Abnormal Security — retrigger KP Created](https://rootly.com/account/incidents/2954-abnormal-security-retrigger-kp-created-webhook-for-clickthrough-contracts) |
| **Slack** | `#incident-20260305-medium-abnormal-security-retrigger-kp-created-webhook-for-clic` (C0AJQH11XHT) |
| **Example WS** | **147206** (US) — verify every time |
| **Example contract** | **1390887** (from integrations thread — example only) |

---

## Connections to other skills

- **contract-lookup** — `workspace_id`, CLICKWRAP + Tray **download_link** race (different symptom).
- **clickwrap-400-submission-triage** — Contract **never created** / API **400**.
- **incident-lookup** — Row for Tray download_link race on WSID 147206.

---

## Operational note

> Run **BigQuery** queries **one at a time** (repo README — parallel BQ MCP calls may 500).
