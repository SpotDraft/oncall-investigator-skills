---
name: hubspot-sfdc-conditional-overpersistence
description: "Diagnose and remediate contracts where HubSpot or Salesforce (SFDC) integration incorrectly persisted hidden conditional field values at creation time, causing destructive mass-cleanup of metadata and activity log spam (70+ field changes) on subsequent UI edits. Use when: HubSpot/SFDC-initiated contracts show unexpected mass field deletions in activity log, metadata linked to questionnaire variables gets silently wiped when a user edits any field, contract shows 'X fields modified / Y metadata fields deleted' from a trivial single-field edit, or HubSpot-initiated contracts contain incorrect default values for fields that should be empty. Trigger phrases: 'incorrect data flowing into contract from HubSpot', 'metadata deleted after editing', 'activity log shows 70+ changes', 'conditional fields saved incorrectly', 'HubSpot contract data wrong', 'allValues vs allValuesExcludingHiddenQuestions', 'sfdc-questionnaire-container overpersisting', 'stale HubSpot deployment'."
---

# HubSpot/SFDC Conditional Field Overpersistence

## What This Skill Diagnoses

When the SFDC/HubSpot integration app is running on a **stale version** of the code that does not include the fix to exclude hidden conditional fields from intermediate saves, every contract initiated from HubSpot or Salesforce is created with **incorrect default values** for fields whose visibility conditions were not met at creation time.

**Two-stage failure pattern:**

1. **At creation (HubSpot/SFDC save):** Hidden conditional fields (fields whose conditions aren't met at the time the HubSpot widget is filled) are saved to `ContractData` with their default values. The main UI flow uses `allValuesExcludingHiddenQuestions$` and handles this correctly — only the SFDC/HubSpot flow has this bug (uses `allValues$` instead).

2. **On first UI edit:** When the contract is later edited in the SpotDraft UI — even editing an unrelated field — the conditional cleanup logic runs correctly, strips out all the incorrectly-saved hidden field values, and syncs back. This generates **70+ field change entries** in the Activity Log and **silently deletes 10+ metadata fields**.

> This is the **opposite pattern** from `questionnaire-hidden-variable-data-loss`. That skill covers integration values being _wiped_ (platform clears correct values). This skill covers hidden values being _incorrectly persisted_ (integration saves incorrect values, platform later cleans up correctly).

**Affected code path:**
- Repo: `SpotDraft/angular-frontend`
- File: `sfdc-questionnaire-container.component.ts`, method: `getDataUpdateRequest()`, ~line 1701
- Bug: Uses `this.questionFormService.allValues$` instead of `this.questionFormService.allValuesExcludingHiddenQuestions$`
- Fix branch: `cursor/prompt-therapy-conditional-logic-03ae`

---

## Available MCP Tools

### BigQuery — Contract Data & Scope Identification

> ⚠️ Run BQ queries **one at a time** — parallel `execute_sql` calls return 500 errors.

**Check ContractData entries for a contract (Prod US):**
```sql
SELECT id, source, is_completed, critical_update, created, modified,
       JSON_EXTRACT(data, '$') as data_snapshot
FROM `spotdraft-prod.prod_usa_db.public_contracts_v3_contractdata`
WHERE contract_id = {contract_id}
ORDER BY created ASC
LIMIT 20
```

Look for:
- **Early rows** (`source: Info-Collection`, `is_completed: False`, `critical_update: True`) — hidden fields incorrectly populated with defaults
- **Later rows** (after UI edit) — same fields absent/wiped (cleanup fired correctly)

**Check contract status and kind:**
```sql
SELECT id, status, contract_kind, workflow_status, created, modified
FROM `spotdraft-prod.prod_usa_db.public_contracts_v3_contractv3`
WHERE id = {contract_id}
```

**Identify HubSpot-initiated contracts for a workspace in a date range:**
```sql
SELECT id, status, contract_kind, workflow_status, created, modified
FROM `spotdraft-prod.prod_usa_db.public_contracts_v3_contractv3`
WHERE workspace_id = {workspace_id}
  AND created >= '{start_date}'
  AND created <= '{end_date}'
ORDER BY created DESC
LIMIT 200
```
> Cross-reference with the contract data source to confirm `Info-Collection` (HubSpot) origin.

**Check metadata linked to questionnaire variables (to see what was deleted):**
```sql
SELECT id, key, value, source, created, modified, is_deleted
FROM `spotdraft-prod.prod_usa_db.public_metadata_v2_metadatafield`
WHERE contract_id = {contract_id}
ORDER BY modified DESC
LIMIT 50
```

### Environment-to-Dataset Mapping

| Environment | BQ Project | Dataset | Table Prefix |
|-------------|-----------|---------|--------------|
| QA India | `spotdraft-qa` | `qa_india_public` | *(none)* |
| QA USA | `spotdraft-qa` | `qa_usa_public` | *(none)* |
| **Prod India** | `spotdraft-prod` | `prod_india_db` | `public_` |
| **Prod EU** | `spotdraft-prod` | `prod_eu_db` | `public_` |
| **Prod USA** | `spotdraft-prod` | `prod_usa_db` | `public_` |
| **Prod MEA** | `spotdraft-prod` | `prod_mea_db` | *(none)* |

### Admin Panel URLs

| Purpose | URL |
|---------|-----|
| Contract data history (US) | `https://api.us.spotdraft.com/admin/contracts_v3/contractdata/?contract_id={contract_id}` |
| Contract data detail (US) | `https://api.us.spotdraft.com/admin/contracts_v3/contractdata/{contractdata_id}/change/?_changelist_filters=contract_id%3D{contract_id}` |
| Workflow admin (US) | `https://api.us.spotdraft.com/admin/workflow_v1/workflow/{workflow_id}/change/` |
| Contract detail (US) | `https://api.us.spotdraft.com/admin/contracts_v3/contractv3/{contract_id}/change/` |

---

## When to Use This Skill

**Use when:**
- HubSpot or Salesforce-initiated contracts show mass field deletions in Activity Log triggered by a single minor edit
- Customer reports "editing one field deleted 10+ metadata fields" or "70+ fields changed at once"
- HubSpot-initiated contracts have unexpected default values in conditional fields that should be empty
- Customer reports metadata linked to questionnaire fields is suddenly missing after a contract edit
- Investigation points to the SFDC/HubSpot app running on stale code (check HubSpot deployment version vs. main app)

**Do NOT use for:**
- Integration connection errors (HubSpot/SFDC not syncing at all) → use email-delivery-triage or infrastructure-alert-response
- Values wiped because of always-false hidden conditions (`sd_show = false`) → use **questionnaire-hidden-variable-data-loss**
- Non-HubSpot/SFDC contract creation paths
- Questionnaire conditional logic slowness or UI rendering bugs

---

## Step-by-Step Investigation Workflow

### Step 1: Confirm the Pattern

Get the contract ID, workspace ID, and cluster from support. Open the contract data in Django Admin:
```
https://api.{cluster}.spotdraft.com/admin/contracts_v3/contractdata/?contract_id={contract_id}
```

Look for **multiple ContractData rows**:
- First rows: `source = Info-Collection`, `is_completed = False` — hidden fields **incorrectly present** with defaults
- Later rows: same fields **missing** (cleanup triggered by UI edit)

The Activity Log should show 70+ field changes from a single edit. Check `https://app.spotdraft.com/contracts/v2/{contract_id}` → Activity Log tab.

### Step 2: Confirm the Integration Path

Ask: was this contract created via the SpotDraft widget on HubSpot, or through a HubSpot workflow automation? The bug affects contracts where the SFDC/HubSpot frontend app handled the creation-time questionnaire save.

Cross-check with feature flags:
- `CONDITIONAL_QUESTIONNAIRE` — if **disabled** for the workspace, BE cannot evaluate conditional visibility; only FE does. This matters for cleanup scope.
- `USE_QUESTION_VISIBILITY_GRAPH` — used for on-the-fly FE condition evaluation; ruled out as root cause in this incident.

### Step 3: Identify the Scope (How Many Contracts Affected)

The scope window = time between when the buggy HubSpot/SFDC code was deployed to prod and when the fix was deployed.

**Run the CDV (Contract Data Validation) endpoint approach:**
- Use the internal CDV validation endpoint (exposed by CFExporter) to evaluate conditional visibility for each contract's data
- The endpoint strips hidden fields from contract data — used to identify which contracts have overpersisted values

Parth Srivastava's script approach (March 2026, Prompt Therapy):
1. Fetch list of HubSpot-initiated contracts for the workspace in the date range
2. Call the CDV validation endpoint for each contract
3. Contracts where the CDV response removes fields → affected
4. Log both contract IDs with hidden fields and those with validation errors separately

**Confirmed scope for Prompt Therapy (known example):**
- Workspace: 274685, US cluster
- Date window: ~2 weeks (based on PR merge date of `SpotDraft/argo-manifests#982`)
- 167 contracts in scope → **100 contained hidden fields** → 91 with validation errors, 5 with incomplete data (DRAFT status)
- Contract stages affected: `INVITE_CLIENT`, `INFO_COLLECTION`, `COMPLETED`, `DRAFT`, `SIGN`, `REVIEW`

### Step 4: Determine Immediate Mitigation

**If the fix has NOT been deployed yet:**
1. **Disable metadata sync** for the affected workspace temporarily (prevents incorrect metadata from propagating back to HubSpot/CRM)
2. Escalate to deploy the latest SFDC/HubSpot app to production:
   - In the March 2026 incident: `release-20260122`, qa tag `qa_20260122_RC103`, prod tag `prod_20260122_RC14`
   - HubSpot app is **not automatically deployed** with main app — requires separate manual deployment
3. After deployment, verify on an internal PROD workspace that new HubSpot-initiated contracts no longer have overpersisted hidden fields

**If the fix IS deployed:**
- No new contracts will be affected; proceed to cleanup of existing impacted contracts

### Step 5: Run the Cleanup Script

Parth Srivastava's script (March 2026) is the reference implementation. Key behaviors:
- **Use the CDV endpoint** to evaluate which fields should be removed from each contract's data
- **For TEMPLATE contract kind** in `INVITE_CLIENT` or `INFO_COLLECTION` stage: call `_generate_template_contract_version_files` to regenerate the docx/pdf with corrected data
- **Delete metadata** linked to hidden questionnaire fields (no activity log entry is generated by this operation)
- **Skip COMPLETED contracts** (only process active contracts)
- Handle **TPP (Third-Party Party) contracts** separately if they also use the SFDC/HubSpot questionnaire flow
- Run in **batches with transactions** — never bulk-update without batching

> The script link from the incident: https://spotdraft.slack.com/archives/C0AK5N3CT7C/p1773229577398619?thread_ts=1773218093.387869&cid=C0AK5N3CT7C

**Global cleanup (across all workspaces):**
Remove the workspace filter and run for all HubSpot/SFDC workspaces that may have been on the stale deployment. Confirmed by the team (March 17, 2026) that the same script works globally — just remove the `wsid` filter and run with batching.

---

## Code Path & Architecture

```
HubSpot Widget (SpotDraft embedded) 
  → sfdc-questionnaire-container.component.ts
      → getDataUpdateRequest()
          → BUG: allValues$  (includes hidden fields with defaults)
          → FIX: allValuesExcludingHiddenQuestions$
  → Saves ContractData with incorrect hidden field values

Later: User edits via SpotDraft main UI
  → questionnaire-container.component.ts
      → Uses allValuesExcludingHiddenQuestions$ (correct)
      → Conditional cleanup fires
      → Strips all hidden-field values from ContractData
      → Syncs metadata → 70+ metadata field deletions
      → Activity log records all changes
```

The main UI has **always** used the correct observable. The SFDC/HubSpot component diverged at some point and was not kept in sync with the main app's fix.

---

## Confirming vs. Disproving the Root Cause

| Evidence | Confirms | Disproves |
|----------|----------|-----------|
| Multiple ContractData rows; early rows have hidden field values, later rows don't | Overpersistence pattern | — |
| Activity log shows 70+ changes from a single field edit | Cleanup fired on first UI edit after overpersistence | — |
| Contract created from HubSpot/SFDC widget (not main UI or API) | SFDC flow involved | — |
| HubSpot app was on stale version at time of contract creation | Root cause confirmed | — |
| Affected fields are conditional (not always visible in template) | Conditional overpersistence | — |
| Issue only on new contracts (post-fix HubSpot contracts look clean) | Deployment fix worked | — |
| `sd_show = false` always-false condition present | Points to `questionnaire-hidden-variable-data-loss` instead | This skill |
| Issue on contracts NOT initiated from HubSpot/SFDC | Different root cause | This skill |

---

## Known Issue Patterns

### Pattern 1: Mass Activity Log Spam on Edit
**Symptom:** Customer edits a single paragraph field (e.g., `custom_discount_details`) with zero conditional dependencies; contract activity log records 77 fields modified and 10+ metadata deletions.  
**Cause:** Hidden conditional fields were overpersisted at HubSpot creation time; this was the first UI edit post-creation, triggering full conditional cleanup.

### Pattern 2: Incorrect Metadata Values in HubSpot CRM
**Symptom:** HubSpot CRM shows incorrect or default field values in metadata-linked HubSpot properties.  
**Cause:** Metadata sync ran after contract creation, pushing the incorrectly-saved hidden field values back to HubSpot. These get wiped when the first UI edit occurs.  
**Mitigation:** Disable metadata sync temporarily until cleanup is complete.

### Pattern 3: Stale HubSpot Deployment Not Auto-Updated
**Root cause:** The HubSpot integration app is a **separately deployed service** — it does not automatically receive main app deploys. Engineers must explicitly deploy it.  
**Check:** Compare the HubSpot app prod tag against the main app's current release. In this incident, HubSpot was behind by the fix for `allValuesExcludingHiddenQuestions$`.

---

## Escalation Triggers

- Multiple workspaces reporting the same pattern simultaneously → the HubSpot/SFDC deployment globally fell behind; run cleanup globally and deploy latest to all
- Customer has already discovered mass metadata deletions before cleanup → escalate to customer success with a communication plan immediately; do not wait for cleanup
- Cleanup script fails on contracts in `SIGN` or `REVIEW` state → consult engineering before touching — these contracts are in active signing flows
- `CONDITIONAL_QUESTIONNAIRE` flag is ON for a workspace → use the CDV endpoint with the BE path; the FE-only cleanup script may not apply

---

## Related Incidents, Jira & Code

| Reference | Details |
|-----------|---------|
| **Rootly Incident #2965** | Prompt Therapy — Incorrect data flowing into contract from HubSpot (March 7–14, 2026) |
| **Jira SPD-42059** | Prompt Therapy: Incorrect data flowing into Contract when Contract is initiated from HubSpot — Mitigated |
| **Slack incident channel** | `#incident-20260307-critical-prompt-therapy-incorrect-data-flowing-into-contract-w` (C0AK5N3CT7C) |
| **Fix branch** | `SpotDraft/angular-frontend` → `cursor/prompt-therapy-conditional-logic-03ae` |
| **Introducing PR** | `SpotDraft/argo-manifests#982` — merged ~2 weeks before fix deployment |
| **HubSpot prod tag at fix** | `prod_20260122_RC14` (qa tag: `qa_20260122_RC103`, release: `release-20260122`) |
| **Sample contract (Prompt Therapy)** | Contract 1406703, WS 274685, US cluster |
| **Admin URL (sample)** | `https://api.us.spotdraft.com/admin/contracts_v3/contractdata/482172/change/?_changelist_filters=contract_id%3D1406703` |

---

## Connections to Other Skills

- **questionnaire-hidden-variable-data-loss** — The inverse pattern: integration values are _lost_ because always-false conditions cause platform to clear them on edit. Use if fields are behind `sd_show = false` or similar dummy condition.
- **contract-lookup** — Use to retrieve `contract_template_id`, `workflow_id`, and ContractData rows before diagnosing.
- **incident-lookup** — Use to find prior incidents for the same workspace (Prompt Therapy has had multiple HubSpot integration incidents in March 2026).
