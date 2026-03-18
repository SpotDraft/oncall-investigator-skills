---
name: contract-computation-null-propagation
description: "Diagnose contract preview or view showing blank/missing values for computed fields (salary, totals, rates) when those fields depend on a Liquid template computation that includes a conditionally-shown questionnaire variable. Use when: a contract shows nothing (or the raw variable name) for a computed numeric field like salary or total; the field is calculated via a Liquid template computation using the plus/minus/times/divided_by filter; the contract was created via a workflow where the questionnaire has conditional variables (e.g., variable component, bonus, commission) that are only shown when a flag variable is true; the variable was not filled because the question was hidden at contract creation time. Trigger phrases: 'contract not showing salary', 'computed field blank in contract', 'sd_salary null', 'variable shows raw name in PDF', 'total not calculated', 'Liquid computation returning null', 'conditional variable missing from contract data', 'plus filter returning null'. Known example: BharatPe (Rootly #2977, WSID 40462, Prod India cluster, March 2026)."
---

# Contract Computation Null Propagation

## What This Skill Diagnoses

A Liquid template computation that chains a conditionally-shown questionnaire variable produces `null` — causing the final computed field (e.g., `sd_salary`) to display as blank or as the raw variable name in the contract preview/view.

**Root mechanism:** Liquid arithmetic filters (`plus`, `minus`, `times`, `divided_by`) return `null` if **either operand is null or missing**. When a questionnaire variable is conditionally shown (behind a flag like `sd_v_c`), and the flag is `false` at contract creation, the variable is never prompted and never stored in `ContractData`. When the computation runs at preview/view time, it evaluates "value + missing" → `null`, and the entire chain collapses.

**Example (BharatPe, March 2026):**
```
sd_total_a  = sd_fixed_a | plus: sd_variable_a
sd_salary   = sd_total_a
```
`sd_variable_a` is shown only when `sd_v_c` (Variable Component present) is `true`. For contracts where `sd_v_c = false`, `sd_variable_a` is never stored → `sd_total_a` and `sd_salary` are both null → contract shows blank.

**This is different from:**
- `questionnaire-hidden-variable-data-loss` — values were stored, then wiped by a platform edit
- `hubspot-sfdc-conditional-overpersistence` — values were incorrectly persisted, then later cleaned up

Here, the variable was **never stored to begin with** because the question was never shown. The problem surfaces only at preview/view computation time.

---

## Available MCP Tools

### BigQuery — Contract Data

> ⚠️ Run BQ queries **one at a time** — parallel `execute_sql` calls return 500 errors.

**Check ContractData to confirm missing variable (Prod India):**
```sql
SELECT id, source, is_completed, critical_update, created, modified
FROM `spotdraft-prod.prod_india_db.public_contracts_v3_contractdata`
WHERE contract_id = {contract_id}
ORDER BY created DESC
LIMIT 10
```

If `sd_variable_a` (or the equivalent conditional variable) is absent from all rows where `is_completed = true`, the variable was never stored.

**Check contract state:**
```sql
SELECT id, status, contract_kind, workflow_status, workspace_id, created, modified
FROM `spotdraft-prod.prod_india_db.public_contracts_v3_contractdata`
WHERE id = {contract_id}
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

### Admin & Metabase URLs

| Purpose | URL |
|---------|-----|
| Template computation (Metabase) | `https://metabase.in.spotdraft.com/question?objectId={template_id}` |
| Question conditions (Metabase) | `https://metabase.in.spotdraft.com/question?objectId={question_item_id}` |
| Contract data (Admin — India) | `https://api.in.spotdraft.com/admin/contracts_v3/contractdata/?contract_id={contract_id}` |
| Contract data (Admin — US) | `https://api.us.spotdraft.com/admin/contracts_v3/contractdata/?contract_id={contract_id}` |
| Questionnaire question items | `https://api.{cluster}.spotdraft.com/admin/questionnaire_v2/questionitemv2/?template_id={template_id}` |
| Contract link | `https://app.spotdraft.com/contracts/v2/{contract_id}?workspaceId={workspace_id}` |

---

## When to Use This Skill

**Use when:**
- A contract shows blank / raw variable name for a computed field (salary, total, rate, fee)
- The missing field is set by a Liquid computation using arithmetic filters (`plus`, `minus`, etc.)
- The computation depends on a variable that is only shown in the questionnaire under a condition
- The condition flag controlling visibility is `false` for the affected contract
- The issue is specific to certain contracts (not all contracts on the template)

**Do not use for:**
- All contracts on a template show blank → likely a template configuration error, not conditional null propagation
- Values were present before and then disappeared → use **questionnaire-hidden-variable-data-loss**
- Signing or document generation failure (error message, not blank field) → use **contract-signing-triage**

---

## Step-by-Step Investigation Workflow

### Step 1: Confirm the Symptom

Identify from the Slack channel:
- **Contract ID** and contract link (`https://app.spotdraft.com/contracts/v2/{contract_id}`)
- **Workspace ID** and **Cluster**
- **Which field is blank** (e.g., `sd_salary`, `sd_total_a`)
- **Is it blank for all contracts on this template, or only some?**

Open the contract in the app and confirm the field is blank/showing raw variable name.

### Step 2: Trace the Computation

Find the template computation in Django Admin or Metabase:

```
https://api.{cluster}.spotdraft.com/admin/questionnaire_v2/
```

Or search via Metabase using the `template_id`. Identify the full computation chain for the blank field:
- What variables does it depend on? (e.g., `sd_fixed_a | plus: sd_variable_a`)
- Which of those dependency variables are conditionally shown in the questionnaire?

### Step 3: Check Contract Data for Missing Variables

Open Django Admin → ContractData for the affected contract:
```
https://api.{cluster}.spotdraft.com/admin/contracts_v3/contractdata/?contract_id={contract_id}
```

Or use BigQuery (if QA). For each `ContractData` row, check if the conditionally-shown variable (`sd_variable_a` in the BharatPe case) is present or absent.

**Confirmation pattern:** The variable is absent from all stored `ContractData` rows.

### Step 4: Identify the Condition Flag

Find the question item for the missing variable. Look at its visibility condition:
```
https://api.{cluster}.spotdraft.com/admin/questionnaire_v2/questionitemv2/?template_id={template_id}
```

Identify the condition variable (e.g., `sd_v_c` = "Variable Component present"). Check this flag's value in the contract data.

**Confirmation:** The condition flag is `false` (or absent), which means the question for the missing variable was never displayed, never answered, and therefore never stored.

### Step 5: Confirm Null Propagation

With the above confirmed, the root cause is:

```
condition flag = false
  → conditional question not shown at contract creation
  → variable never stored in ContractData
  → computation: (stored_value | plus: missing_variable) = null
  → computed field = null
  → contract preview shows blank
```

---

## Fix: Defensive Computation Default

The fix is in the **template computation**, not the platform code. The computation must handle the case where the conditional variable is absent.

**Pattern:**

Add an explicit default assignment for the conditional variable when its condition is false:

```
if sd_v_c == false → assign sd_variable_a = 0
```

This ensures `sd_variable_a` always has a numeric value before the arithmetic filter runs, preventing null propagation.

**Who applies the fix:** The engineer or implementation team responsible for the template (Sanya Bali in the BharatPe case), not the platform/backend team.

**After applying:**
- Test on the workflow preview stage
- Test on the sent-for-signing stage
- Confirm customer can re-preview the affected contract and sees the correct value

> ⚠️ **Note:** Existing contracts created before the fix that already have null `sd_salary` stored will not automatically update — the fix only affects new contract creations and subsequent previews that recompute.

---

## Confirming vs. Disproving the Root Cause

| Evidence | Confirms | Disproves |
|----------|---------|-----------|
| Conditional variable absent from all ContractData rows | Variable was never stored | — |
| Condition flag (`sd_v_c`) is `false` in contract data | Hidden question explains missing variable | — |
| Template computation uses `plus`/`minus` with the missing variable | Null propagation path confirmed | — |
| All contracts on the template show blank, even those where condition is `true` | Different root cause (template misconfiguration) | This pattern |
| Variable is present in ContractData but field still blank | Computation references wrong variable name | This pattern |

---

## Escalation Triggers

- Multiple customers/templates reporting blank computed fields → check if a platform change affected how computation-time variables are resolved
- Fix applied but old contracts still show blank → expected behavior (no auto-backfill); customer must re-trigger a preview or re-create the contract
- Variable is present in ContractData but computation still returns null → escalate to engineering (may be a Liquid filter edge case or type mismatch)

---

## Related Incidents & Jira

| Reference | Details |
|-----------|---------|
| **Rootly Incident #2977** | BharatPe — Contract preview not showing `sd_salary` (March 10–18, 2026) |
| **Slack channel** | `#incident-20260310-high-bharatpe-contract-preview` (C0AKKJ8F6RZ) |
| **Template ID** | 41998 (BharatPe, Prod India) |
| **Workspace ID** | 40462 (BharatPe, Prod India cluster) |
| **Fix contract (tested)** | `https://app.spotdraft.com/contracts/v2/1829265?workspaceId=40462` |

---

## Connections to Other Skills

- **questionnaire-hidden-variable-data-loss** — Related but distinct: values were stored and then wiped on questionnaire edit. Use if values were present before and disappeared.
- **contract-lookup** — Use to retrieve `template_id`, `workspace_id`, and `ContractData` rows before diagnosing.
- **contract-signing-triage** — Use if the blank field is causing a document generation or signing error (as opposed to a silent blank in the rendered contract).
