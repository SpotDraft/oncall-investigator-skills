---
name: questionnaire-dropdown-global-metadata-mismatch
description: "Diagnose Workflow Manager / questionnaire failures when updating a dropdown linked to global metadata: customer changed option labels in Metadata Manager but saves fail with errors about choices not present in global metadata, or mismatch between questionnaire option values and KeyPointer. Use when: dropdown metadata field; Metadata Manager vs Workflow Manager inconsistency; only labels changed; 'not present in the global metadata'; validate_dropdown_options. Trigger phrases: selected values not present in global metadata, global metadata mismatch dropdown, Fee Installment Schedule style metadata, KeyPointer value label out of sync. Related: campaign Excel ₹ issues (campaign-excel-inr-dropdown-triage), Edit Template Questionnaire stale prefill (edit-template-questionnaire-contractdata-split). Jira: SPD-42511."
---

# Questionnaire Dropdown vs Global Metadata (KeyPointer) Value Mismatch

## What This Skill Debugs

For **dropdown** (and **multi-dropdown**) questionnaire variables that map to **global metadata** (`KeyPointer`), the backend requires every **stored choice value** on the questionnaire side to exist as an option **value** on the global metadata `KeyPointer` — not merely the same **label**.

**Product constraint (called out in incident):** In Metadata Manager, editing a dropdown option today can effectively change **labels** while leaving older **values** in place. **Workflow Manager** then validates the full questionnaire against global metadata and fails when `QuestionVariable` / `FrozenQuestionVariable` **choices** (values) are not a subset of **`KeyPointer.format_option.options[].value`**.

**Confirmed backend error shape** (API / validation):

```text
The following choices are not present in the global metadata with name {label}: [...]
```

(Also: `Global metadata with name {label} does not have any choices present` if options are empty.)

**Code (source of truth for the rule):**

- `questionnaire_v2/utils/question_variable_and_global_metadata_validations.py` — `validate_dropdown_options_for_question_variable` compares `question_variable.choices` to the **set of `option["value"]`** from global metadata `format_option.options`.
- `questionnaire_v2/question_variable/domain/use_cases/validate_question_variable_with_global_metadata_use_case.py` — wires validation for `DROPDOWN` / `MULTI_DROPDOWN`.

**Not the same as:**

- **Campaign Excel / ₹ in options** — see [campaign-excel-inr-dropdown-triage](../campaign-excel-inr-dropdown-triage/SKILL.md).
- **Edit Template Questionnaire stale prefill** — see [edit-template-questionnaire-contractdata-split](../edit-template-questionnaire-contractdata-split/SKILL.md).
- **Hidden variable data loss on edit** — see [questionnaire-hidden-variable-data-loss](../questionnaire-hidden-variable-data-loss/SKILL.md).

---

## When to Use / Not Use

**Use when:**

- Customer updated dropdown options in **Metadata Manager** but **Workflow Manager** (questionnaire/template) save fails with global-metadata validation messaging.
- You can see **label** vs **value** drift in Django admin: `KeyPointer.format_option` JSON shows **labels** matching new copy while **values** still hold legacy strings.
- Engineering needs a **data fix** path: align `KeyPointer`, **`ContractKeyPointer`** rows still pointing at old **values**, and **`QuestionVariable` / `FrozenQuestionVariable`** `format_option` + `choices`, then resync affected contracts.

**Do not use when:**

- The failure is Excel/campaign ingestion or currency-symbol stripping — use the campaign skill above.
- No global metadata linkage — different triage.

---

## Investigation Workflow

1. **Capture identifiers** from the ticket or admin:
   - `{workspace_id}`, cluster (`api.{us|in|eu}.spotdraft.com`), workflow / template / questionnaire links if provided.
   - Global metadata: **`KeyPointer`** id or search by label / `template_field_name` (question variable name).

2. **Inspect global metadata (KeyPointer)** — Django admin (replace cluster):
   - List: `https://api.{cluster}.spotdraft.com/admin/historic_contracts/keypointer/?created_by_workspace_id={workspace_id}&q={search_term}`
   - Change: `https://api.{cluster}.spotdraft.com/admin/historic_contracts/keypointer/{key_pointer_id}/change/`
   - In **`format_option`**, confirm each option: for dropdowns, **`value`** is what validation uses; compare to the **labels** the customer believes they “renamed.”

3. **Inspect questionnaire variables** for the same logical field:
   - `QuestionVariable` / `FrozenQuestionVariable` by workspace + **name** (`template_field_name` / variable name). Check **`choices`** list vs KeyPointer option **values**. Mismatch confirms the failure mode.

4. **Check contracts using old values** (if any): `ContractKeyPointer` rows for that `key_pointer_id` with **`value`** equal to legacy option strings — those must be migrated when option **values** change, or saved answers will reference stale tokens.

5. **Optional — BigQuery (Prod US pattern)** — inspect key pointer JSON (run one query at a time; adjust dataset for cluster):

```sql
SELECT id, label, template_field_name, format_option
FROM `spotdraft-prod.prod_usa_db.public_historic_contracts_keypointer`
WHERE created_by_workspace_id = {workspace_id}
  AND id = {key_pointer_id}
```

Use `prod_india_db` / `prod_eu_db` / `prod_mea_db` per region as in other skills.

---

## Mitigation Pattern (Engineering / On-Call Script)

Incidents of this class have been **mitigated with a reviewed Django shell script** (dry-run first), not a self-serve product button. Exact steps from a resolved incident:

1. Define a **`VALUE_FIXES`** map: `{old_value: new_value}` for options where the **value** must change to match new labels or canonical strings.
2. **`KeyPointer`**: deep-update `format_option` so each affected option’s **`value`** (and usually **`label`**) matches the intended pairings.
3. **`ContractKeyPointer`**: `UPDATE` rows where `value` is still an **old** token to the **new** value for the same `key_pointer_id` / workspace; collect **`contract_id`**s touched.
4. **`QuestionVariable`** and **`FrozenQuestionVariable`**: set **`format_option`** (full options list) and **`choices`** (list of values) so questionnaire definitions match KeyPointer.
5. **Async resync**: enqueue `partially_resync_contracts_task` (or equivalent) for affected **`contract_id`**s and relevant fields (e.g. `key_pointer_list`) so downstream views stay consistent.

**Operational notes:**

- Always **`dry_run=True`** first; use a transaction with rollback for dry runs if that is your team’s standard.
- Get **two-person review** for production scripts (as in incident workflow).
- **Permanent fix** is product-side: allow safe editing of option **values** or keep labels/values in sync in Metadata Manager — track with product/engineering backlog; this skill covers **operational recovery**.

---

## Environment-to-Dataset Mapping

Same as [edit-template-questionnaire-contractdata-split](../edit-template-questionnaire-contractdata-split/SKILL.md): `spotdraft-prod` + `prod_usa_db` / `prod_india_db` / `prod_eu_db` / `prod_mea_db` with `public_` table prefix where applicable.

---

## Escalation Triggers

- Unclear which **variable name** maps to which **KeyPointer** across many templates — need eng to trace `template_field_name` and frozen questionnaire IDs.
- Large blast radius (many workspaces or contracts) — script review + change window.
- Suspected **platform bug** (validation wrong, or UI not persisting `choices`) — file/attach logs and link a new ticket; use **Groundcover** / request logs around the failed save if available.

---

## Related Incidents / Tickets

| Item | Link / ref |
|------|------------|
| Jira (this pattern) | [SPD-42511](https://spotdraft.atlassian.net/browse/SPD-42511) — *The Global Poverty Project: unable to update the dropdown options in the questionnaire* (Mitigated) |
| Slack incident channel | `C0AMG6UV4J0` — March 2026; mitigation: backend alignment script across KeyPointer, ContractKeyPointer, QuestionVariable/FrozenQuestionVariable + resync |
| Similar prior handling | Internal thread referenced in incident: Slack `C0AKZ05S19V` (same script-style mitigation per engineers) |

---

## Known Example (Illustrative Only)

**Do not treat as live config.** From SPD-42511 / incident channel:

- Workspace **564833**, US cluster, metadata field **“Fee Installment Schedule”**, example **KeyPointer** id **97109**, variable name like **`sd_fee_installment_schedule`**.
- Observed drift: **labels** updated to strings such as `Monthly installments during the Term` while **values** remained e.g. `Equal monthly installments during the term` — validation failed until **values** and dependent rows were aligned.

---

## Connections to Other Skills

- [execution-date-missing-from-report](../execution-date-missing-from-report/SKILL.md) — KeyPointer admin URLs and BQ table naming for `historic_contracts_keypointer`.
- [fullstory-bulk-delete-metadata-value](../fullstory-bulk-delete-metadata-value/SKILL.md) — `ContractKeyPointer` model context when bulk-touching metadata per contract.
