---
name: edit-template-questionnaire-contractdata-split
description: "Diagnose template contracts where Intake Form Responses and the contract document show updated questionnaire answers but 'Edit Template Questionnaire' opens with stale, default, or empty values until the user saves again. Use when: template contract; key card and document match; edit modal does not; customer re-saved via edit and then everything aligned. Trigger phrases: 'Edit Template Questionnaire wrong values', 'questionnaire not syncing', 'intake shows correct but edit shows old', 'template questionnaire stale', 'prefill wrong after submit'. Related: partial-save vs full submit for key card. Confirmed class: ContractData vs QuestionnaireResponse divergence on first submit (Jira SPD-42290, SPD-2306)."
---

# Edit Template Questionnaire vs ContractData / Intake Key Card

## What This Skill Debugs

On **template contracts**, questionnaire answers can be written to **`ContractData`** (driving the **Intake Form Responses** key card and contract document) while **"Edit Template Questionnaire"** pre-fills from **`QuestionnaireResponse`** via a separate read path. If the initial submission flow creates or updates **completed** `ContractData` but does **not** create or upsert a matching **`QuestionnaireResponse`** row for that contract + frozen questionnaire mapping, the edit modal shows **stale, empty, or template-default** values even though the rest of the product shows the correct answers.

**This fails silently** — no error logs; the GET path simply returns missing or old `QuestionnaireResponse` data.

**Secondary pattern (partial save):** The Intake Form Responses key card is driven by the latest **fully completed** `ContractData` (`is_completed = true`). **Partial saves** (`source` such as `partial-save`, `is_completed = false`) may not appear on the key card until the user **fully completes and submits** the questionnaire. Do not confuse that with the ContractData vs `QuestionnaireResponse` split — confirm which case applies using `ContractData` ordering and completion flags.

**Not the same as** integration hidden-question data loss on edit ([questionnaire-hidden-variable-data-loss](../questionnaire-hidden-variable-data-loss/SKILL.md)), which wipes `ContractData` after a questionnaire edit due to visibility rules.

---

## Available MCP Tools

### BigQuery — `ContractData` rows (primary signal)

> Run BQ queries **one at a time** — parallel `execute_sql` calls often return 500 errors.

**Inspect recent `ContractData` for the contract (Prod US example):**
```sql
SELECT id, source, is_completed, critical_update, created, modified, data
FROM `spotdraft-prod.prod_usa_db.public_contracts_v3_contractdata`
WHERE contract_id = {contract_id}
ORDER BY created DESC
LIMIT 20
```

**What to look for:**
- Latest row is **`partial-save`** / **`is_completed = false`** → key card may lag until a **completed** submission exists; explain expected behavior to support.
- Latest **completed** row has correct `data`, but customer still sees wrong edit prefill → suspect **missing or stale `QuestionnaireResponse`** for the edit path (see code paths below).

### BigQuery — Request path audit (optional)

If your environment can query request/response logs (pattern from incident investigation):
```sql
SELECT start_timestamp, request_method, request_path, response_status, time_taken
FROM `spotdraft-prod.request_response_logs_prod_usa.logs`
WHERE workspace_id = '{workspace_id}'
  AND TIMESTAMP(inserted_at) BETWEEN TIMESTAMP('{start_ts}') AND TIMESTAMP('{end_ts}')
  AND (request_path LIKE '%questionnaire%' OR request_path LIKE '%{contract_id}%')
ORDER BY start_timestamp ASC
LIMIT 50
```

Adjust dataset/table for cluster (e.g. `request_response_logs_prod_in` patterns if documented elsewhere).

### GCP Cloud Logging — contract-scoped search (US prod example)

Silent bug: use **INFO** and broad `SEARCH`, not only `severity >= ERROR`.

```
resource.type="k8s_container"
resource.labels.cluster_name="prod-us"
SEARCH("{contract_id}")
```

Narrow to questionnaire / contract-data handling if needed by adding terms from your standard logging query runbooks.

### Environment-to-Dataset Mapping

| Environment | BQ Project | Dataset | Table prefix |
|-------------|------------|---------|--------------|
| QA India | `spotdraft-qa` | `qa_india_public` | *(none)* |
| QA EU | `spotdraft-qa` | `qa_eu_public` | *(none)* |
| QA USA | `spotdraft-qa` | `qa_usa_public` | *(none)* |
| **Prod India** | `spotdraft-prod` | `prod_india_db` | `public_` |
| **Prod EU** | `spotdraft-prod` | `prod_eu_db` | `public_` |
| **Prod USA** | `spotdraft-prod` | `prod_usa_db` | `public_` |
| **Prod MEA** | `spotdraft-prod` | `prod_mea_db` | *(none)* |

### Admin panel

| Cluster | Contract data list |
|---------|-------------------|
| US | `https://api.us.spotdraft.com/admin/contracts_v3/contractdata/?contract_id={contract_id}` |
| IN | `https://api.in.spotdraft.com/admin/contracts_v3/contractdata/?contract_id={contract_id}` |
| EU | `https://api.eu.spotdraft.com/admin/contracts_v3/contractdata/?contract_id={contract_id}` |

Use Django admin for `QuestionnaireResponse` / frozen questionnaire mappings only if you already know the mapping IDs — prefer BQ or engineering confirmation before bulk querying `questionnaire_v2` tables in unfamiliar joins.

---

## When to Use / Not Use

**Use when:**
- **Template** contract (not TPP / upload-and-sign-only confusion — template flows are **ContractData**-centric; `QuestionnaireResponse` may only appear after certain API paths).
- Symptom: **Intake Form Responses** + document **correct**; **Edit Template Questionnaire** **wrong** until user edits and saves again.
- Investigating whether the latest `ContractData` is **completed** vs **partial-save**.

**Do not use when:**
- Values **wrong everywhere** → broader questionnaire / template / variable bug.
- Values **wiped after an edit** on integration-mapped hidden questions → [questionnaire-hidden-variable-data-loss](../questionnaire-hidden-variable-data-loss/SKILL.md).
- Signing, PDF, or email issues only.

---

## Architectural Flow (Evidence-Backed)

1. **Contract summary / template questionnaire overview**  
   For questionnaire v2 template contracts, `GetQuestionnaireV2ResponseForTemplateContractUseCase` loads **`ContractDataService.get_latest_completed_contract_data_for_contract`** — aligns with key card behavior when `use_questionnaire_v2` is true.  
   Code: `contracts_v3/contract/domain/use_cases/get_questionnaire_v2_response_for_template_contract_use_case.py`

2. **GET pre-fill for "Edit Template Questionnaire" (per frozen questionnaire)**  
   `ContractQuestionnaireResponseGetPostView` GET uses **`GetQuestionnaireResponseFromFrozenQuestionnaireForContractUseCase`** → **`GetQuestionnaireResponseUseCase`** → **`QuestionnaireResponse`** by frozen questionnaire + **CONTRACT** entity mapping — **not** the latest `ContractData` blob.  
   Code: `contracts_v3/contract/presentation/views.py` (`ContractQuestionnaireResponseGetPostView`), `contracts_v3/contract/domain/use_cases/get_questionnaire_response_from_frozen_questionnaire_for_contract_use_case.py`

3. **POST from edit modal**  
   **`EditQuestionnaireResponseForContractUseCase`** can create/update **`QuestionnaireResponse`** via **`AddQuestionnaireResponseForContractUseCase`** — which is why **re-saving through Edit Template Questionnaire** can **heal** the divergence.  
   Code: `questionnaire_v2/questionnaire_response/domain/use_cases/edit_questionnaire_response_for_contract_use_case.py`, `contracts_v3/contract/domain/use_cases/add_questionnaire_response_for_contract_use_case.py`

**Engineering direction for a durable fix (from incident resolution notes):** upsert **`QuestionnaireResponse`** on initial template submit **or** change the edit GET path to pre-fill from **`ContractData`** for template contracts — product/engineering decision.

---

## Step-by-Step Investigation Workflow

1. **Confirm cluster, `workspace_id`, `contract_id`, and that the contract is a template workflow.**  
   Example from incident: WSID **274685**, US, contract **1425891** (Prompt Therapy, March 2026).

2. **Reproduce the reported split:** document + key card vs edit modal. Capture whether the user only **partial-saved** before complaining.

3. **Inspect `ContractData` in admin or BQ** (query above).  
   - If the newest relevant row is **incomplete** → customer guidance: **finish and submit** the questionnaire; refresh.  
   - If the newest **completed** row is correct but edit is wrong → treat as **ContractData vs `QuestionnaireResponse`** gap.

4. **Optional:** Pull request logs around the report time for questionnaire endpoints and the contract id.

5. **Mitigate for the customer:** Ask them to open **Edit Template Questionnaire**, enter or confirm answers, and **submit/update** so a **`QuestionnaireResponse`** is written — incident **2990** was resolved this way.

6. **Track product fix** via Jira (see below); link **SPD-2306** if discussing historical duplicate symptom class.

---

## How to Confirm / Disprove

| Evidence | Interpretation |
|----------|----------------|
| Latest **completed** `ContractData` has correct `data`; edit modal still wrong | Supports **`QuestionnaireResponse`** pre-fill gap |
| Only **partial-save** / `is_completed = false` after user's changes | Key card **expected** to lag; not necessarily QR divergence |
| After one **Edit Template Questionnaire** save, all views match | Consistent with **healing write** to **`QuestionnaireResponse`** |
| First `ContractData` row missing values, later rows correct | Different issue — trace submission / integration / visibility |

---

## Mitigation / Workaround

- **Customer unblock:** Complete flow through **Edit Template Questionnaire** and save — creates/syncs **`QuestionnaireResponse`** so pre-fill matches **`ContractData`**-backed views (confirmed in Rootly resolution for incident **2990**).
- **Comms:** If partial save only, set expectation that the **Intake Form Responses** card shows the latest **fully submitted** snapshot, not in-progress drafts.

---

## Escalation Triggers

- Many workspaces hit at once after a deploy → platform regression; involve engineering with deploy window.
- Data correctness issues **after** edit save → not this split; deeper questionnaire or auth bug.
- Request for **backfill** of `QuestionnaireResponse` from `ContractData` → engineering / data — no standard self-serve path documented in incident.

---

## Related Incidents, Jira, Links

| Reference | Notes |
|-----------|--------|
| **Rootly** | [Incident 2990](https://rootly.com/account/incidents/2990-prompt-therapy-questionnaire-values-not-syncing-correctly-in-edit-template-questionnaire-view) — Prompt Therapy, March 2026 |
| **Slack** | `#incident-20260312-medium-prompt-therapy-questionnaire-values-not-syncing-correct` (archived) |
| **Jira SPD-42290** | This incident's ticket |
| **Jira SPD-2306** | Older ticket, same symptom class (stale edit vs key card / document) — verify current status before promising fix timeline |
| **Example contracts (incident)** | `1425891`, `1398913` (US) — **examples only**; use `{contract_id}` in new investigations |

---

## Connections to Other Skills

- **questionnaire-hidden-variable-data-loss** — integration + hidden question **clearing** on edit; different mechanism.
- **contract-lookup** — resolve template, workflow, and cluster before deep diving.
- **incident-lookup** — find prior reports for the same customer or workspace.
