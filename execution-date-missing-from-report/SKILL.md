---
name: execution-date-missing-from-report
description: "Diagnose why the Execution Date column (or any key pointer column) is missing from a generated contract report in SpotDraft. Use this skill when support reports: 'Execution Date not in report', 'column missing from generated report', 'report does not show Execution Date', 'column visible in repository but absent in download', 'All Columns selected but field missing from export', or any mention of a metadata/key pointer column that appears in the repository view or View Columns but does not appear in the generated/exported report. Also covers the frontend columns_filter category routing bug where the Execution Date column is sent under KEY_POINTERS instead of GENERAL."
---

# Execution Date (or Key Pointer Column) Missing from Generated Report

## What This Skill Diagnoses

Two distinct, independently occurring root causes can make a key pointer column disappear from generated reports even when the field is visible in the repository view:

1. **Key pointer `is_visible = False` (most common for Execution Date):** The "Execution Date" key pointer for the workspace has been toggled to `is_visible = False` via the contract metadata settings page. The backend filters key pointers by `is_visible=True` before including them in report columns. The UI still shows the column in the repository view and "View Columns" — this mismatch is a known UX gap.

2. **Frontend `columns_filter` category routing bug:** When "Select visible columns" is used, the frontend sends `"Execution Date"` in `columns_filter` under `KEY_POINTERS` with the display label `"Execution Date"`, instead of sending it under `GENERAL` with the slug `"execution_date"`. The backend only recognizes `execution_date` under `GENERAL` as the standard execution date column and ignores it when it appears under `KEY_POINTERS`.

**Critical UX nuance (confirmed Mar 2026):** For non-metadata-enabled workspaces, the Execution Date column only appears in reports when the associated key pointer is active (`is_visible = True`). Even when disabled, the field remains visible in the repository view and in "View Columns", creating user confusion.

---

## Available MCP Tools for Investigation

### BigQuery — Key Pointer State

**⚠️ Run BQ queries one at a time** — parallel `execute_sql` calls return 500 errors.

**Check key pointer `is_visible` status for a workspace (run this first):**
```sql
-- Prod USA
SELECT
  kp.id, kp.label, kp.is_visible, kp.is_deleted,
  kp.well_known_type, kp.key_pointer_type,
  kp.created, kp.modified
FROM `spotdraft-prod.prod_usa_db.public_historic_contracts_keypointer` kp
WHERE kp.created_by_workspace_id = {wsid}
  AND kp.is_deleted = FALSE
  AND LOWER(kp.label) LIKE '%execution%'
ORDER BY kp.created DESC
```

```sql
-- Prod India
SELECT
  kp.id, kp.label, kp.is_visible, kp.is_deleted,
  kp.well_known_type, kp.key_pointer_type,
  kp.created, kp.modified
FROM `spotdraft-prod.prod_india_db.public_historic_contracts_keypointer` kp
WHERE kp.created_by_workspace_id = {wsid}
  AND kp.is_deleted = FALSE
  AND LOWER(kp.label) LIKE '%execution%'
ORDER BY kp.created DESC
```

**Check all key pointers with `is_visible = False` for a workspace:**
```sql
SELECT id, label, is_visible, is_deleted, well_known_type, key_pointer_type, modified
FROM `spotdraft-prod.prod_usa_db.public_historic_contracts_keypointer`
WHERE created_by_workspace_id = {wsid}
  AND is_deleted = FALSE
  AND is_visible = FALSE
ORDER BY modified DESC
```

**Check the ElasticContractSearchReportRun record to inspect `columns_filter` payload sent by the frontend:**
```sql
SELECT
  r.id, r.status, r.created, r.modified,
  r.columns_filter, r.columns_filter_type,
  r.total_contracts, r.created_by_workspace_id
FROM `spotdraft-prod.prod_usa_db.public_contracts_v3_elasticcontractsearchreportrun` r
WHERE r.created_by_workspace_id = {wsid}
ORDER BY r.created DESC
LIMIT 10
```

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

> Prod tables have a `public_` prefix. QA/Dev tables have no prefix.

### Admin URLs for Key Pointer Management

| Action | URL |
|--------|-----|
| Key pointer admin (workspace) | `https://api.{cluster}.spotdraft.com/admin/historic_contracts/keypointer/?created_by_workspace_id={wsid}` |
| Key pointer detail | `https://api.{cluster}.spotdraft.com/admin/historic_contracts/keypointer/{kp_id}/change/` |
| Metadata settings page (customer-facing) | `https://app.spotdraft.com/workspace-settings/metadata` |

---

## When to Use This Skill

**Use when:**
- Customer reports that a column (especially "Execution Date") appears in the repository view or "View Columns" but is absent from the generated or downloaded report
- Customer uses "All Columns" or "Only columns visible in current view" and still doesn't see the Execution Date field in the exported report
- Column appeared in previous reports but has stopped appearing after someone adjusted workspace settings

**Do not use for:**
- DocuSign (or external sign) template contracts where **`contract$execution_date`** is null in the **pre-sign / cfexporter payload** and the date is blank on the signed PDF → use **`docusign-execution-date-blank-pre-sign`**
- Execution Date field not being captured/extracted on individual contracts → different issue (key pointer value not populated from contract metadata or execution event)
- Report failing to generate entirely → use `contract-lookup` or check GCP logs / DLQ for task failures
- Specific contract's execution date field missing in the contract sidebar (not the report) → check key pointer value extraction separately

---

## Architectural Background

### Two types of columns in a generated report

**GENERAL columns** are standard system-level columns identified by slug (e.g., `"execution_date"`, `"title"`, `"contract_type"`). They are defined in `GeneralSearchReportColumn` enum in `contracts_v3/search_v2/domain/domain_models.py`.

**KEY_POINTERS columns** are workspace-specific metadata fields identified by their display label (e.g., `"My Custom Field"`). They are stored in the `historic_contracts_keypointer` table.

The Execution Date is **not** in the `GeneralSearchReportColumn` enum — it is treated as a key pointer column that maps to the `execution_date` slug when recognized properly.

### The `columns_filter` structure

When generating a report with selected columns, the frontend sends a `columns_filter` array. Each entry specifies a `category` (`GENERAL` or `KEY_POINTERS`) and a `column` value:

```json
// Correct — sends Execution Date as a GENERAL column with slug
{"category": "GENERAL", "column": "execution_date"}

// Bug — sends Execution Date as a KEY_POINTERS column with display label
{"category": "KEY_POINTERS", "column": "Execution Date"}
```

The backend (`generate_search_report_for_contract_batch_use_case.py`) processes `GENERAL` and `KEY_POINTERS` column lists separately. When `execution_date` is received under `KEY_POINTERS`, it's treated as a custom field label lookup — and if no key pointer with the label "Execution Date" is active, the column is silently dropped.

### Key pointer `is_visible` filtering

The backend filters key pointers by `is_visible=True` in three places in `contract_key_pointers_service.py`:

```python
filters = filters & Q(key_pointer__is_visible=True)           # line 219
key_pointers = key_pointers.filter(key_pointer__is_visible=True)  # line 283
contract_key_pointers_service.fetch_minimal_contract_key_pointers_for_id_list(...)  # line 1978 — also filters is_visible=True
```

When a workspace admin disables a key pointer via the metadata settings page (toggling `is_visible = False`), it stops appearing in report output even though the UI still shows it in the repository grid and "View Columns" panel.

---

## Step-by-Step Investigation Workflow

### Step 1: Gather identifiers

From the support ticket or incident channel, collect:
- **Workspace ID (WSID)** and **Cluster** (IN / US / EU / ME)
- **Exact column name** the customer reports as missing (e.g., "Execution Date")
- **Report type**: one-time report or recurring/scheduled report
- **Column selection method**: "All Columns", "Only columns visible in current view", or manually selected columns

### Step 2: Check `is_visible` status of the key pointer (BQ — run first)

Run the key pointer visibility query for the workspace. Look for:
- Is there a row with `label LIKE '%execution%'` (or the missing column name)?
- Is `is_visible = FALSE`?
- When was `modified`? (Check if someone recently toggled it off via the metadata settings page)

**Decision tree:**

| BQ result | Conclusion | Action |
|-----------|-----------|--------|
| `is_visible = FALSE` | Key pointer was disabled — **this is the root cause** | Re-enable the key pointer (see mitigation below) |
| `is_visible = TRUE` but column still missing | Look at `columns_filter` routing bug (Step 3) | Check the report run record |
| No matching key pointer row | Key pointer was deleted or never created for this workspace | Check with customer if the field was recently deleted |
| `is_deleted = TRUE` | Key pointer was deleted | Cannot restore without data migration; escalate |

### Step 3: Inspect the `columns_filter` payload (BQ)

Query the most recent `ElasticContractSearchReportRun` records for the workspace to see the raw `columns_filter` JSON sent by the frontend.

Look for:
- Is `"Execution Date"` appearing under `"category": "KEY_POINTERS"` instead of `"category": "GENERAL"` with `"column": "execution_date"`?
- Is `columns_filter` empty? (If empty, all columns should be included — so the bug must be elsewhere)

**Example of the bug:**
```json
// Buggy payload (FE sends label under wrong category)
{"category": "KEY_POINTERS", "column": "Execution Date"}

// Correct payload (what BE expects for the standard Execution Date column)
{"category": "GENERAL", "column": "execution_date"}
```

### Step 4: Check if the workspace is metadata-enabled

For **non-metadata-enabled workspaces**, the Execution Date must come through a key pointer. If that key pointer is disabled, the column won't appear regardless of column selection settings.

The metadata-enabled flag is stored at the workspace level — check via admin:
```
https://api.{cluster}.spotdraft.com/admin/sd_organizations/companyprofile/{wsid}/change/
```

Look for `is_metadata_enabled` or equivalent flag. If the workspace is non-metadata-enabled, the key pointer `is_visible` state directly controls report output.

---

## Confirming Root Cause

| Evidence | Root Cause |
|----------|-----------|
| `historic_contracts_keypointer.is_visible = FALSE` for Execution Date KP in the workspace | Key pointer was disabled by an admin via metadata settings — WAI but confusing |
| `columns_filter` in `elasticcontractsearchreportrun` shows `"KEY_POINTERS": "Execution Date"` instead of `"GENERAL": "execution_date"` | Frontend routing bug |
| Both of the above | Both issues compounding |
| `is_visible = TRUE` and correct `columns_filter` category, but column still absent | Escalate — possible data pipeline or Elastic sync issue |

---

## Mitigation / Workaround

### Re-enable the key pointer (fastest unblock)

1. Identify the key pointer ID from the BQ query above.
2. Go to the Django admin for the key pointer:
   ```
   https://api.{cluster}.spotdraft.com/admin/historic_contracts/keypointer/{kp_id}/change/
   ```
3. Set `is_visible = True` and save.
4. Ask the customer to regenerate the report.

**⚠️ Before toggling:** Confirm with the customer or CS team whether the key pointer was intentionally disabled (as Sonu did in the Mar 2026 incident — the customer had turned it off themselves and the team confirmed before re-enabling).

### If it's the frontend routing bug

- The customer can work around it by selecting "All Columns" and then deselecting the ones they don't want, rather than starting from "Only visible columns in current view".
- Alternatively, report the issue to the frontend team with the `columns_filter` payload evidence from BQ.

---

## Known Issue Patterns

### Pattern 1: Key pointer disabled via metadata settings (confirmed root cause — Mar 2026)

**Customer:** PWHL (WSID 500152, Cluster: US) + internal workspace WSID 1062168 (Cluster: IN)

**Symptom:** "Execution Date" column missing from one-time generated report even when "All Columns" selected. Column visible in repository view and in "View Columns".

**Root cause:** `historic_contracts_keypointer.is_visible = FALSE` for the "Execution Date" key pointer in the workspace. The visibility was toggled off via the contract metadata settings page.

**Metabase diagnostic query used (table: `historic_contracts_keypointer`, workspace 500152, label = "Execution Date"):**
```sql
SELECT id, label, is_visible, is_deleted, well_known_type, created, modified
FROM `spotdraft-prod.prod_usa_db.public_historic_contracts_keypointer`
WHERE created_by_workspace_id = 500152
  AND label = 'Execution Date'
```

**Resolution:** Re-enable the key pointer after confirming with the customer that the disable was unintentional.

**Jira:** SPD-42327

**UX gap:** UI shows the column in the repository view and "View Columns" even when the key pointer is disabled — this creates a false expectation that the column will appear in reports. Feedback raised to product team Mar 2026.

### Pattern 2: Frontend sends Execution Date under wrong `columns_filter` category (FE bug — Mar 2026)

**Symptom:** When user selects "Only columns visible in current view", the Execution Date is omitted from the report. BQ shows `is_visible = TRUE` for the key pointer.

**Root cause (Anjali Swami's finding):** The frontend sends `{"category": "KEY_POINTERS", "column": "Execution Date"}` instead of `{"category": "GENERAL", "column": "execution_date"}`. The backend only includes the standard Execution Date column when it receives the slug `"execution_date"` under `"GENERAL"`. When it arrives as `"KEY_POINTERS": "Execution Date"`, the backend looks for a custom KP with that label and may or may not find it, so the column gets dropped.

**Workaround:** Ask the customer to use "All Columns" selection rather than "Only columns visible in current view."

### Pattern 3: Individual contract execution dates not populated in report (PhonePe — Jan 2024)

Different issue class from the above. The column appears in the report header, but cells are blank for specific contracts. Causes: key pointer value was never extracted for those contracts (Upload-Sign contracts where execution date was not auto-populated), key pointer was deleted and re-created, or a Postgres connectivity failure during the execution event task. Check key pointer values at the contract level rather than the key pointer visibility at the workspace level.

---

## Escalation Triggers

- Customer is a key stakeholder and the report is blocking a business-critical process → re-enable immediately, escalate to CS
- The key pointer `is_deleted = TRUE` → cannot be re-enabled from admin; requires engineering to restore data
- Multiple workspaces report the same column missing from all new reports → likely a regression in the columns_filter routing logic; escalate to engineering
- `is_visible = TRUE` and correct `columns_filter` category, but the column still doesn't appear → escalate to BE with BQ evidence

---

## Related Incidents & Jira

| Reference | Details |
|-----------|---------|
| **Rootly Incident #2996** | PWHL — Execution Date Column Missing from Generated Report (Mar 13, 2026) |
| **Jira SPD-42327** | Original Jira ticket for this incident |
| **Slack channel** | `#incident-20260313-medium-pwhl-execution-date-column-missing-from-generated-repor` (C0AL05HEWQP) |
| **PhonePe incident (Jan 2024)** | Execution date not extracted for 56 contracts (IN/5092) — different root cause (value not populated vs column missing) |
| **FIGS incident (Oct 2024)** | Standard "Execution Date" key pointer disabled for workspace — same pattern as PWHL |

---

## Connections to Other Skills

- **contract-lookup** — Use to gather contract context and check workspace settings before diagnosing
- **incident-lookup** — Search for prior similar incidents if pattern is new or affects multiple customers
- **docusign-execution-date-blank-pre-sign** — DocuSign execution date null in document payload / blank on PDF (not repository report columns)
