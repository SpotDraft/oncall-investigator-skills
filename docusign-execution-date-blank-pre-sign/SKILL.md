---
name: docusign-execution-date-blank-pre-sign
description: "Diagnose DocuSign template contracts where contract$execution_date is null in the cfexporter payload before send, so the execution date is blank on the document even after signing. Use when support reports: 'Execution date not populating after contract execution', 'execution date empty on DocuSign PDF', 'contract$execution_date null', 'Practice Services Agreement execution date', DocuSign + computed execution date variable, or API shows null for execution date in contract_details / contract_data while Spotdraft Sign works. Trigger phrases: DocuSign execution date missing, pre-sign PDF execution date, cfexporter execution_date null, DOCUSIGN_PRE_POPULATE_EXECUTION_DATE."
---

# DocuSign: Execution Date Blank on Document (Pre-Sign cfexporter Payload)

## What This Skill Debugs

**Issue class:** For **DocuSign** (and similar external signing) **template** workflows, `contract$execution_date` is **null** in the payload used to generate the pre-sign document. The field is only set in the database **after** execution; the document sent to DocuSign is generated **before** execution, so the execution-date variable stays empty on the signed PDF. This is separate from “execution date column missing from a repository report” (see `execution-date-missing-from-report`).

**Product nuance:** Externally signed documents cannot be modified after they return from DocuSign, so you cannot “inject” the true execution date into the PDF post-signing. The supported engineering workaround is to **pre-fill** a date **at send time** (feature flag), which is the **sending date**, not necessarily the date all parties finish signing.

**Confirmed engineering mechanism (Mar 2026):** `get_cfexporter_payload_for_contract` builds `blanks_json` via `CFExporterPayloadGenerator.convert_blanks_data_for_unsigned_contract`. Blanks from questionnaire data are merged into `flattened_data`, then `contract_variables` (from `self.contract.execution_date` in the DB) are merged. For an unsigned contract, DB `execution_date` is typically **null**, so flattened keys such as `contract$execution_date` end up **null** in the payload sent to CF exporter—unless overridden later in the pre-sign path.

**Mitigation shipped:** Workspace feature flag **`DOCUSIGN_PRE_POPULATE_EXECUTION_DATE`**. When enabled and the contract uses DocuSign, `save_pre_sign_pdf_for_template_contract` computes a formatted “today” (workspace timezone + date format) and `get_contract_docx_preview` sets `cfexporter_payload["blanks_json"]["contract$execution_date"]` **after** the payload is built and **before** `generate_docx_for_payload`—so the generated pre-sign PDF includes a visible date.

---

## Available MCP Tools for Investigation

### Slack

- Search this incident class: prior discussion **Storj** DocuSign execution date (`#incident-20250211-high-storj-execution-date-not-populating-via-docusign`).
- **Rootly:** [Incident 2962 — SiteRx execution date](https://rootly.com/account/incidents/2962-siterx-execution-date-not-populating-after-contract-execution-in-practice-services-agreement-workflow) (example customer: SiteRx, WSID **31514**, cluster **US**, workflow **Practice Services Agreement**).

### BigQuery — Contract Signing Method and Execution Date

**⚠️ Run BQ queries one at a time** — parallel `execute_sql` calls may return 500 errors.

**Contract row (replace dataset for cluster):**
```sql
SELECT
  c.id,
  c.public_id,
  c.status,
  c.execution_date,
  c.created,
  c.modified
FROM `spotdraft-prod.prod_usa_db.public_contracts_v3_contractv3` c
WHERE c.id = {contract_id};
```

**Signature setup — confirm DocuSign:**
```sql
SELECT
  ss.id,
  ss.signing_method,
  ss.sent_for_signature,
  ss.is_completed,
  ss.mark_executed_after_signatures
FROM `spotdraft-prod.prod_usa_db.public_contracts_v3_contractsignaturesetup` ss
WHERE ss.contract_id = {contract_id}
  AND ss.is_deleted = FALSE;
```

Look for `signing_method` matching DocuSign (e.g. `DOCU_SIGN` / `DOCU_SIGN_V2` — confirm enum values in code for your query window).

### GCP Cloud Logging

**Log line emitted when pre-population runs (successful path):**
- Message pattern: `Pre-populating execution_date for DocuSign contract` (see `contracts_v3.services.contract_v3_service.save_pre_sign_pdf_for_template_contract`).

**Example filter shape (adjust project and IDs):**
```text
SEARCH("Pre-populating execution_date for DocuSign contract")
jsonPayload.contract_id="{contract_id}"
```
Or filter by async task id when debugging a specific pre-sign run:
```text
SEARCH("Pre-populating execution_date for DocuSign contract")
jsonPayload.task_id="{task_id}"
```

**Known example from QA investigation (Mar 2026):** task_id `3a492abf-7124-4278-911a-164eda102b1c` in `spotdraft-qa` logs (link was shared in the incident channel).

> Prod vs QA: prefer `jsonPayload.*` field filters in structured logs; some environments may differ—verify field names in a sample log row.

### Groundcover

- Use for traces/logs around **GENERATE_PRE_SIGN_PDF** / `save_pre_sign_pdf` / `cf_exporter` if your cluster is wired to Groundcover.
- Correlate with `contract_id` and timestamp of “send for signature”.

### Atlassian (Jira)

- **SPD-42045** — linked from Rootly for this incident class (verify current status in Jira).

---

## Environment-to-Dataset Mapping

| Environment | BQ Project | Dataset (example) |
|-------------|------------|---------------------|
| Prod US | `spotdraft-prod` | `prod_usa_db` + `public_` table prefix |
| Prod IN | `spotdraft-prod` | `prod_india_db` |
| Prod EU | `spotdraft-prod` | `prod_eu_db` |
| QA US | `spotdraft-qa` | `qa_usa_public` (no `public_` prefix) |

---

## When to Use / Not Use

**Use when:**
- Customer uses **DocuSign** and the **execution date variable** in the DOCX is blank on the executed PDF.
- Support or CS pastes API / HAR data showing **`contract$execution_date` is null** in contract details / cfexporter-related payload while the template clearly includes that variable.
- Same template works for **Spotdraft Sign** but not DocuSign (Spotdraft Sign regenerates executed PDF; DocuSign does not).

**Do not use when:**
- Execution date is missing only from a **generated repository report** or column picker → **`execution-date-missing-from-report`**.
- Generic document generation timeout / Word / CF exporter errors without execution-date context → **`contract-signing-triage`** (may combine with this skill if DocuSign + null execution date is confirmed).

---

## Step-by-Step Investigation Workflow

1. **Confirm identifiers:** `{contract_id}` or contract URL, `{workspace_id}`, `{cluster}`, signing provider (DocuSign vs Spotdraft Sign).
2. **Confirm variable in template:** From customer or Draftmate — execution date slug is typically **`contract$execution_date`** (example: SiteRx Practice Services Agreement incident).
3. **Inspect payload:** From `contract_details` / support HAR — is `contract$execution_date` **null** before or after execution? If null pre-send, this skill applies; if populated in API but not on PDF, escalate (different path).
4. **Confirm signing method (BQ or admin):** DocuSign vs native sign drives whether the pre-populate flag path applies.
5. **Check feature flag:** Is **`DOCUSIGN_PRE_POPULATE_EXECUTION_DATE`** enabled for `{workspace_id}`? If not, null payload for unsigned contracts is expected without the workaround.
6. **Logs:** For a recent “prepare for sign” / pre-sign PDF run, search for **Pre-populating execution_date for DocuSign contract** and `contract_id`. Absence may mean flag off, non-DocuSign method, or task failure earlier in the chain.
7. **Post-deploy QA:** If the date appears but **format** looks wrong, verify **workspace `date_format` and timezone** (`format_datetime` in pre-populate path) — staging example from Mar 2026 incident thread.

---

## Code Paths (Reference)

| Step | Location |
|------|----------|
| Feature flag + formatted date | `contracts_v3.services.contract_v3_service.save_pre_sign_pdf_for_template_contract` — `FlagName.DOCUSIGN_PRE_POPULATE_EXECUTION_DATE`, `format_datetime` with universal config |
| Override blanks_json before CF exporter | `contracts_v3.services.contract_export_service.get_contract_docx_preview` — sets `cfexporter_payload["blanks_json"]["contract$execution_date"]` when `pre_populate_execution_date` is set |
| Unsigned payload merge / DB execution_date | `contracts_v3.services.cf_exporter_payload_service.CFExporterPayloadGenerator.convert_blanks_data_for_unsigned_contract` — `contract_variables["contract"]["execution_date"]` from DB merged into flattened blanks (null until executed) |
| Flag definition | `sd_organizations.constants.FlagName.DOCUSIGN_PRE_POPULATE_EXECUTION_DATE` |

---

## How to Confirm / Disprove Likely Causes

| Finding | Interpretation |
|---------|----------------|
| DocuSign + `contract$execution_date` null in API payload for unsigned / pre-sign flow | Expected without flag; enable **`DOCUSIGN_PRE_POPULATE_EXECUTION_DATE`** for workspace after customer accepts “sending date” semantics |
| Spotdraft Sign + same template shows execution date | Consistent with known product difference (regeneration path) |
| Flag on + log line shows pre-populate value but PDF still blank | Escalate — CF exporter, caching, or wrong contract version / template |
| Wrong date format with flag on | Check universal config timezone + date format for workspace |

---

## Mitigation / Workaround

1. **Customer communication:** The date shown with the flag is the **date the contract was sent to DocuSign** (or “today” at send time per implementation), **not** guaranteed to match the day all parties signed. If execution spans multiple days, dates can be “out of sync” with real execution — align expectations before enabling.
2. **Enable** workspace feature flag **`DOCUSIGN_PRE_POPULATE_EXECUTION_DATE`** (internal process per FeatureFlagService / admin — follow your org’s flag rollout procedure).
3. **Already executed contracts** without the date in the PDF cannot be fixed retroactively without re-signing or manual amendment — same as original incident comms.
4. **No engineering workaround** for true execution date on the DocuSign-executed PDF without changing signing product or legal/process acceptance of send-time date.

---

## Escalation Triggers

- Flag enabled, DocuSign confirmed, logs show pre-populate, but **field still blank** on PDF.
- **Document generation failure** (Rohit thread — staging contract) with execution-date work in progress — treat as possible **regression** or separate CF exporter error; gather `task_id` and stack traces.
- Customer requires **true execution date** on the PDF and rejects send-time semantics — product / legal discussion, not a quick flag toggle.

---

## Related Incidents & Links

| Reference | Notes |
|-----------|--------|
| **#incident-20260306-… SiteRx** (channel `C0AKV7G6N5N`) | Practice Services Agreement, WSID 31514, US; RCA in-thread; mitigation flag; resolved when fix reached prod (Mar 2026). |
| **#incident-20250211-high-storj-execution-date-not-populating-via-docusign** | Prior same theme — DocuSign + execution date. |
| **Rootly incident 2962** | Metadata and customer context. |
| **Jira SPD-42045** | Ticket linked from Rootly. |
| **Leads thread (2025)** | Historical product discussion on DocuSign + computed fields — linked from Hemil in incident thread. |

---

## Connections to Other Skills

- **`execution-date-missing-from-report`** — Report/export column visibility and `columns_filter`; not DocuSign payload null.
- **`contract-signing-triage`** — Pre-sign PDF failures, Celery limits, generic signing errors; use alongside this skill when symptoms overlap.
