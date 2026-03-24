---
name: workspace-date-format-contract-title-email-mismatch
description: "Diagnose when workspace Profile and Entity date format (e.g. DD-MMM-YYYY) does not match dates shown in contract-related emails—especially review and signing emails—while the in-product Activity Log shows the correct format. Trigger on: '{CREATION_DATE} wrong format in email', 'date format not reflecting in emails', 'MM-DD-YYYY in email but DD-MMM in UI', 'contract title date wrong in notification', 'Internal Review email date format', 'Signing email subject date', NIUM-style reports, or any mismatch between configured date_format_config and email-rendered contract title dates."
---

# Workspace Date Format vs. Contract Title in Emails

## What This Skill Debugs

Customers configure **Date Format** at workspace and entity level (`SettingKey.date_format_config`). Some surfaces respect that config; others historically did not. A common pattern:

- **Emails** (review, signing, notifications that include **contract title**) show dates in **MM-DD-YYYY** (or another wrong pattern).
- **Activity Log / audit trail** shows the **correct** workspace format (e.g. **DD-MMM-YYYY**).

That split is a strong signal that the problem is **not** “email delivery” and **not** “user misconfigured the workspace,” but **how the contract title (or other email context) was built** vs. **how audits format dates**.

**Primary historical cause (contract title):** `{CREATION_DATE}` in the contract title template was substituted using a **hardcoded** `strftime("%m-%d-%Y")` in `BuildContractTitleFromRegexUseCase`, so the **stored contract name** contained the wrong date format. Every email that echoes **contract title** inherited that baked-in string.

**Additional fixes (email body / key pointers / datetime context):** Separate changes ensure `executed_at` and DATE key pointers in email context use `GetUniversalConfigUseCase` where applicable (see Related PRs).

---

## When to Use / Not Use

| Use this skill | Do not use this skill |
|----------------|----------------------|
| Wrong **date format in email content** (subject/body) while UI/audit is correct | Email **never arrives** (bounces, suppression, OTP) → use **email-delivery-triage** |
| Reports involving **`{CREATION_DATE}`** or contract **title** in notifications | Generic “date wrong in report export” without email contract title → see **execution-date-missing-from-report** or other report skills |
| **Same contract**, activity log date format OK, email wrong | **All** surfaces wrong → verify workspace/entity settings first in admin |

---

## Available MCP Tools and Queries

### BigQuery — Contract Title (stored name)

**⚠️ Run BQ queries one at a time** — parallel `execute_sql` calls may return 500 errors.

Confirm the **persisted** contract name for a suspected contract (adjust project/dataset to cluster):

```sql
SELECT
  id,
  contract_name,
  created_by_workspace_id,
  created
FROM `spotdraft-prod.prod_eu_db.public_contracts_v3_contractv3`
WHERE id = {contract_id}
```

If `contract_name` contains a date segment matching **MM-DD-YYYY** while the workspace expects **DD-MMM-YYYY**, that aligns with **title substitution at creation** having used the default format.

**Workspace sanity check (example pattern — table names vary by env):** resolve `{workspace_id}` and cluster before querying settings-backed tables; universal config is resolved at runtime via `GetUniversalConfigUseCase`, not always a single denormalized column in BQ.

### Environment-to-Dataset Mapping

| Environment | BQ Project | Dataset | Table Prefix |
|-------------|-----------|---------|--------------|
| QA India | `spotdraft-qa` | `qa_india_public` | *(varies)* |
| QA EU | `spotdraft-qa` | `qa_eu_public` | *(varies)* |
| QA USA | `spotdraft-qa` | `qa_usa_public` | *(varies)* |
| **Prod India** | `spotdraft-prod` | `prod_india_db` | `public_` |
| **Prod EU** | `spotdraft-prod` | `prod_eu_db` | `public_` |
| **Prod USA** | `spotdraft-prod` | `prod_usa_db` | `public_` |
| **Prod MEA** | `spotdraft-prod` | `prod_mea_db` | *(check conventions)* |

### GCP Cloud Logging

**⚠️ Prod uses `jsonPayload`; QA often uses `textPayload`.**

After deploy, **CREATION_DATE** formatting logs from `BuildContractTitleFromRegexUseCase`:

```
resource.type="k8s_container"
resource.labels.project_id="spotdraft-prod"
resource.labels.cluster_name="prod-eu"
resource.labels.namespace_name="prod"
jsonPayload.message:"Formatting CREATION_DATE using workspace date format"
```

Fallback / pre-fix behavior:

```
jsonPayload.message:"No date format provided, using default MM-DD-YYYY"
```

Narrow by `jsonPayload.request_id="{request_id}"` when available.

---

## Step-by-Step Investigation

1. **Capture identifiers:** `{workspace_id}`, `{contract_id}`, cluster, example email type (review vs signing), and whether **contract title** includes **`{CREATION_DATE}`** (or creation-date variable).
2. **Confirm the split:** Is **only** email (or title in email) wrong while **Activity Log** / audit shows the correct format? If **both** wrong, validate workspace and entity **Date Format** in admin first.
3. **Inspect stored contract name** (BQ query above). If the title already contains **MM-DD-YYYY**, the failure is likely **at contract creation / title build**, not at send time.
4. **Code paths (read order):**
   - **Title substitution:** `contracts_v3/contract/domain/use_cases/build_contract_title_from_regex_use_case.py` — `CREATION_DATE` should use `format_datetime` with `date_format` and `timezone_str` from `GetUniversalConfigUseCase` when provided; otherwise falls back to `strftime("%m-%d-%Y")`.
   - **Callers:** `build_template_contract_title_use_case`, `build_contract_title_for_third_party_paper_workflows_use_case` — pass universal config into the regex use case.
   - **Parallel path:** `contracts_v3/services/contract_service.py` — `get_contract_name_from_settings` also resolves `CREATION_DATE` via `GetUniversalConfigUseCase` with fallback logging on failure.
   - **Email context:** `emails/email/domain/use_cases/get_contract_context_use_case.py` — workspace-aware formatting for relevant datetime fields (see PR below).
   - **Audits (correct behavior):** formatting via universal config / settings manager (`GetUniversalConfigUseCase`, `SettingKey.date_format_config`) — explains why **Activity Log** can differ from **legacy title string**.
5. **Rule out delivery:** If the issue is **format only**, skip DLQ/deep Postmark triage unless there is also a **send failure**.

---

## Known Issue Patterns

### Example A — NIUM (Mar 2026, confirmed)

- **Symptom:** DD-MMM-YYYY set at workspace and entity; **review/signing emails** showed **MM-DD-YYYY** in title; **Activity Log** correct.
- **Workspace (example):** NIUM QA, `workspace_id` **16423**, cluster **EU**.
- **Root cause class:** Hardcoded MM-DD-YYYY in contract title build for `{CREATION_DATE}`; downstream emails used stored `contract_name`.
- **Mitigation:** Customer renamed affected contracts; advised to **avoid `{CREATION_DATE}` in title** until fix deployed.
- **Tracking:** [SPD-42387](https://spotdraft.atlassian.net/browse/SPD-42387).

---

## Mitigation and Workaround

**Until fixes are deployed everywhere:**

- **Manual rename** contracts whose titles contain the wrong date literal.
- **Remove `{CREATION_DATE}` from contract title templates** temporarily if the customer cannot tolerate MM-DD-YYYY in titles.

**After deploy:** new contracts should get workspace-aware substitution when callers pass config; still verify in QA per cluster before closing with customer.

---

## Related PRs and Regressions

| PR | Scope |
|----|--------|
| [django-rest-api#12386](https://github.com/SpotDraft/django-rest-api/pull/12386) | Email context / `executed_at`, DATE key pointers (`DateType.to_string` / `GetContractKeyPointersContextUseCase`) |
| [django-rest-api#12479](https://github.com/SpotDraft/django-rest-api/pull/12479) | **Short-term:** `BuildContractTitleFromRegexUseCase` + template / third-party paper callers — `{CREATION_DATE}` uses workspace `date_format` / timezone |

If a **new** report matches this pattern after both PRs are in production, treat as **regression** and compare logs for `"No date format provided"` vs config resolution failures.

---

## Escalation Triggers

- **Regression** after PRs deployed: file bug with contract ID, workspace ID, cluster, and sample email headers.
- **Config resolution errors** in logs (`GetUniversalConfigUseCase` / settings failures) causing fallback to MM-DD-YYYY — escalate to platform/contracts owning universal config.
- **Only one email type wrong** — verify whether that template uses **contract title** vs other variables; may need separate template/key-pointer check.

---

## Connections to Other Skills

- **email-delivery-triage** — delivery and OTP; use when mail never arrives.
- **contract-lookup** — general contract/workspace resolution across tools.

---

## Related Incidents / Jira

- [SPD-42387](https://spotdraft.atlassian.net/browse/SPD-42387) — NIUM workspace date format not reflecting in emails (mitigated; incident channel archived workflow).

**Example Slack incident channel (internal):** `#incident-20260316-high-nium-workspace-date-format-configuration-not-reflecting-d` — Rootly and Cursor RCA captured the analysis and PR links above.
