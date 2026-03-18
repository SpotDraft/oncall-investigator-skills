---
name: questionnaire-hidden-variable-data-loss
description: "Diagnose and remediate questionnaire field values being cleared or zeroed out in contracts created via integration workflows (HubSpot, Salesforce). Use this skill when a customer reports that contract fields populated from an integration are missing, zeroed out, or reset after the questionnaire is edited in SpotDraft. Trigger on phrases like: 'questionnaire values reset', 'HubSpot data missing from contract', 'fields showing 0', 'integration data cleared', 'SFDC values wiped', 'data disappeared after editing', or any report of contract fields not matching what was pushed from HubSpot/Salesforce. Also use if investigating the March 4–17 2026 regression that affected 15+ customers with hidden integration-mapped questions."
---

# Questionnaire Hidden Variable Data Loss

## What This Skill Diagnoses

Integration-sourced questionnaire variable data is wiped when a contract's questionnaire is edited in SpotDraft, if the associated question items are hidden behind an **always-false visibility condition** (e.g., `sd_show = false`). The initial data push from the integration bypasses visibility checks, but any subsequent questionnaire edit triggers a re-evaluation that clears values for hidden questions.

**March 4–17, 2026 regression:** A platform change deployed on March 4 (for BIC) that stopped persisting questionnaire responses for hidden questions made this behavior systemic. It broke an existing Legal Ops workaround used across 15+ customer workspaces. Fix deployed March 17 via feature flag. **Contracts created March 4–17 on affected workflows may have missing field values — there is no backfill path.**

---

## Available MCP Tools

### BigQuery — Contract Data & Questionnaire State

**Check contract data entries to confirm missing values (Prod US):**
```sql
SELECT id, source, is_completed, critical_update, created, modified
FROM `spotdraft-prod.prod_usa_db.public_contracts_v3_contractdata`
WHERE contract_id = {contract_id}
ORDER BY created DESC
LIMIT 10
```

Look for the pattern:
- `source: Info-Collection`, `is_completed: False`, `critical_update: True` — **values present** (initial HubSpot push)
- `source: partial-save` or `source: Info-Collection` with `is_completed: True` — **values absent/zeroed** (result of questionnaire edit)

**Check contract state (Prod US):**
```sql
SELECT id, status, contract_kind, workflow_status, created, modified
FROM `spotdraft-prod.prod_usa_db.public_contracts_v3_contractv3`
WHERE id = {contract_id}
```

**Check contract version history:**
```sql
SELECT id, version_number, sub_version_number, action, is_current, created
FROM `spotdraft-prod.prod_usa_db.public_contracts_v3_contractversion`
WHERE contract_id = {contract_id}
ORDER BY version_number DESC, sub_version_number DESC
LIMIT 10
```

> ⚠️ **Run BQ queries one at a time** — parallel `execute_sql` calls return 500 errors.

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

### GCP Cloud Logging — Questionnaire Edit Activity

**Search for questionnaire save events for a contract (Prod US):**
```
logName="projects/spotdraft-prod/logs/stderr"
resource.labels.cluster_name="prod-usa"
jsonPayload.message:"{contract_id}"
severity >= ERROR
```

**QA equivalent:**
```
resource.type="k8s_container" labels."k8s-pod/app"="spotdraft-qa-django-app"
textPayload:("{contract_id}") textPayload:("questionnaire" OR "question_item" OR "contract_data")
```

### Admin Panel URLs

| Cluster | Contract Data | Frozen Question Variable | Workflow Admin |
|---------|--------------|--------------------------|----------------|
| US | `https://api.us.spotdraft.com/admin/contracts_v3/contractdata/?contract_id={contract_id}` | `https://api.us.spotdraft.com/admin/questionnaire_v2/frozenquestionvariable/?template_id={template_id}` | `https://api.us.spotdraft.com/admin/workflow_v1/workflow/{workflow_id}/change/` |
| IN | `https://api.in.spotdraft.com/admin/contracts_v3/contractdata/?contract_id={contract_id}` | — | `https://api.in.spotdraft.com/admin/workflow_v1/workflow/{workflow_id}/change/` |
| EU | `https://api.eu.spotdraft.com/admin/contracts_v3/contractdata/?contract_id={contract_id}` | — | — |

---

## When to Use This Skill

**Use when:**
- Customer reports integration-sourced field values (HubSpot/Salesforce) missing from a contract that was created via a template workflow
- Customer says "fields that used to show correct values now show 0 / are blank"
- Issue appears after the customer or SpotDraft team edited the contract questionnaire
- Customer is on a workflow where implementation hid certain fields from the UI using a dummy condition variable (commonly `sd_show`)
- Customer reports the issue started around March 4–17, 2026, or after any deployment that changed questionnaire visibility behavior

**Do not use for:**
- Integration connection errors (HubSpot/Salesforce not syncing at all)
- Contract PDF generation issues
- Email or signing workflow failures

---

## Platform Behavior: The Root Cause

### Normal flow (working correctly)
1. Contract created via HubSpot/Salesforce integration → integration pushes variable values directly into `ContractData`
2. The initial data push **bypasses questionnaire visibility rules** (it writes to variables, not question items)
3. Fields appear correctly in the contract document

### Break point: questionnaire edit on SpotDraft
When a user edits the questionnaire via the SpotDraft UI (or when the platform re-evaluates it):
1. The visibility rule engine evaluates each question's `condition_as_json_logic` or `condition_as_complex_expression`
2. Questions that evaluate as **hidden** have their values cleared and not written back to `ContractData`
3. Result: integration-sourced values are wiped, replaced with defaults (often 0 or empty)

### The always-false hidden question pattern (workaround used by Legal Ops)
Implementation teams sometimes hide integration-mapped questions from the UI by placing them behind a condition that always evaluates to `false` — typically a dummy variable like `sd_show` (which has no questionnaire field and no default, so it is never set to `true`). This achieves the goal of hiding the field from sellers, but creates a data-loss risk on any questionnaire edit.

**Technical form of the condition:**
- Simple: `condition_as_json_logic` contains `{...sd_show = false...}`
- Advanced (conditional library): `condition_as_complex_expression` = `Equals(intake.sd_show, False)` with `condition_as_json_logic = null`

---

## Step-by-Step Investigation Workflow

### Step 1: Confirm the Symptom

Ask the support person for:
- **Contract ID** and link (`https://app.spotdraft.com/contracts/v2/{contract_id}`)
- **Workspace ID (WSID)** and **Cluster** (US/IN/EU/ME)
- **Specific fields showing wrong values** (e.g., PEPM rate, billing cadence, employee count)
- **When did the issue start?** Was it after the questionnaire was edited?

Check contract data in Django Admin:
```
https://api.{cluster}.spotdraft.com/admin/contracts_v3/contractdata/?contract_id={contract_id}
```

Look for multiple `ContractData` rows:
- First row (initial HubSpot/Salesforce push) → values **present**
- Later rows (after questionnaire edit) → same fields **missing or 0**

### Step 2: Identify the Workflow Configuration

From the contract admin page or BQ, find the `contract_template_id` and `workflow_id`.

Check the workflow admin for hidden questions:
```
https://api.us.spotdraft.com/admin/workflow_v1/workflow/{workflow_id}/change/
```

Look for `FrozenQuestionVariable` entries referencing a dummy visibility variable (e.g., `sd_show`) in:
```
https://api.us.spotdraft.com/admin/questionnaire_v2/frozenquestionvariable/?template_id={template_id}
```

**Confirm the pattern:** A `FrozenQuestionVariable` for `sd_show` (or similar) that:
- Has no questionnaire question linked to it (cannot be modified by users)
- Has no default value
- Other question items are conditionally visible on `sd_show = false`

### Step 3: Check if the March 4 Regression Applies

**Trigger condition:** The issue started between March 4–17, 2026 AND the workspace uses the always-false hidden question pattern.

**Feature flag check:** As of March 17, the BIC behavior change is behind a feature flag scoped only to BIC. All other customers should be on the restored behavior (hidden question data flows into contracts). If a new customer is still experiencing this, check whether the feature flag has been incorrectly enabled for their workspace.

### Step 4: Check for the Specific Variables

The affected integration variables for Cocoon (WSID 425092, US cluster — known example) were:
- `sd_pepm_rate`
- `sd_number_of_covered_employees`
- `sd_annual_subscription_fee`
- `sd_billing_cadence`

For other customers, the affected variables will differ — look for any variable whose question item is behind an always-false condition.

### Step 5: Identify Scope (How Many Workflows/Workspaces Affected)

Parth Srivastava's script (March 17) identified affected workspaces by querying for integration-linked questions with always-false conditions across all clusters. The confirmed affected workspace IDs from March 2026 were:

**US Cluster:** 425092, 127495, 147532, 49805, 259502, 121073, 353009, 128311, 258584, 123225, 257980, 288797  
**IN Cluster:** 3409, 5193, 546494, 239272  
**EU Cluster:** None identified

> These are known examples from the March 2026 scan. Re-run discovery if investigating a new incident.

---

## Mitigation Options

Two options exist for workspaces with the always-false hidden question pattern. Confirm with the customer which they prefer.

### Option A: Make the Hidden Questions Visible (Recommended)

Remove the always-false condition on the question items so they appear on the SpotDraft questionnaire UI. The integration mapping continues to work. The caveat: users can now edit the integration-sourced values directly.

This requires no code change — update the workflow conditions in the admin panel and **republish the workflow** for changes to take effect.

> ⚠️ **Callout to customer before republishing:** The workflow will need to be republished, and there may be unsaved changes if the customer has edited it. Confirm before proceeding.

### Option B: Soft-Delete the Hidden Questions from Django Admin

Removes the question from the UI entirely. The integration mapping remains intact (mappings are linked to variables, not question items). Users cannot edit those values. The limitation: if they ever want to modify these variables in the future, they must contact SpotDraft.

**Procedure (only steps 1 and 2 are required):**

**Step 1 — Soft delete the question in Questionnaire V2 → Question Item V2:**
- Open the question by ID: `https://api.{cluster}.spotdraft.com/admin/questionnaire_v2/questionitemv2/{question_item_id}/change/`
- Set `is_deleted = True`
- Set `deleted_at = <current timestamp>`
- Save

**Step 2 — Remove entity dependency mappings in Core → Entity Dependency Mappings:**
- Filter: `child_entity_id = {question_item_id}`, `child_entity_type = QUESTION_ITEM`, `tenant_workspace = {workspace_id}`
- Delete all matching rows

**Do NOT need:**
- Step 3 (version record) — optional audit trail only
- Step 4 (workflow default currency) — only if the variable is a dropdown linked to currency settings
- Step 5/6 (frozen questionnaire, express templates) — known limitations, acceptable for one-off manual deletion

> ⚠️ **Test on an internal workspace first** before applying to the customer's production workspace. Confirm the integration still works by creating a test contract from HubSpot/Salesforce after the soft-delete.

---

## How to Find Hidden Questions with Always-False Conditions (Script Pattern)

Parth Srivastava's approach (March 17) for identifying affected question items across all workflows:

1. Query `QuestionItemV2` for all question items associated with the workspace's templates
2. Filter for items where `condition_as_json_logic` evaluates to always `false` (typically a reference to a variable like `sd_show`)
3. Cross-check with integration mappings to confirm those question items are mapped to HubSpot/Salesforce fields
4. Handle the advanced conditional library case: if `condition_as_json_logic` is `null`, parse `condition_as_complex_expression` instead (format: `Equals(intake.sd_show, False)`)

> ⚠️ Must handle both `condition_as_json_logic` (simple conditions) and `condition_as_complex_expression` (advanced/conditional library mode). When in advanced mode, `condition_as_json_logic` is `null` and `condition_as_complex_expression` is populated.

---

## Confirming the Root Cause

| Evidence | Confirms | Disproves |
|----------|---------|-----------|
| `ContractData` has values in first row (initial push), missing in later rows | Data was wiped during questionnaire edit | — |
| `FrozenQuestionVariable` for `sd_show` (or similar dummy) with no default, no question | Always-false hidden question pattern present | — |
| Questions conditionally visible on `sd_show = false` | Root cause confirmed | — |
| No always-false conditions found | Different root cause | This pattern |
| Issue occurred March 4–17 | Likely the BIC regression | — |
| Issue occurred before March 4 or after March 17 | Likely an implementation-specific hidden question pattern | The March 4 regression |

---

## Escalation Triggers

- Multiple customers reporting simultaneously → check if a new platform change has reintroduced the always-false visibility clearing behavior
- Issue persists after March 17 for a workspace not on BIC feature flag → escalate to engineering to verify feature flag scoping
- Customer requests data backfill for contracts affected during March 4–17 → **no automated backfill path exists**; manual re-creation of contracts is the only remediation

---

## Related Incidents & Jira

| Reference | Details |
|-----------|---------|
| **Rootly Incident #2972** | Cocoon — Questionnaire Responses Updated After Contract Signing (March 9–17, 2026) |
| **Jira SPD-42118** | Original bug report (Cocoon, US cluster, WSID 425092) — In Progress |
| **Jira SPD-42427** | Fix task (feature flag scoping, cherry-picked to prod March 17) — Verified |
| **Slack incident channel** | `#incident-20260309-critical-cocoon-questionnaire-responses-updated-after-contract` (C0AK3099ALX) |
| **Affected workflow (Cocoon)** | `https://api.us.spotdraft.com/admin/workflow_v1/workflow/4462/change/` |
| **March 4 FE PR (BIC change)** | `https://github.com/SpotDraft/angular-frontend/pull/14423/changes` |

**Known affected customers (March 4–17 regression):** Crunchbase, Notable, Lightcast, Zerocater Inc, Cocoon, Obsidian Security Inc, OneMarketData, Cardinal Health, Metergy Solutions, Grand River Solutions, Oyster HR, Ascend Fundraising Solutions, Sword Health Inc, Chargebee, Wingify.

---

## Connections to Other Skills

- **contract-lookup** — Use to retrieve the contract's `contract_template_id`, `workflow_id`, and version history before diagnosing this issue
- **incident-lookup** — Use to search for prior incidents with the same customer or similar symptoms
