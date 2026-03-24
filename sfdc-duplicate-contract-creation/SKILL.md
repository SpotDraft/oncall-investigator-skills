---
name: sfdc-duplicate-contract-creation
description: "Diagnose multiple SpotDraft contracts created from the Salesforce (SFDC) widget for the same Opportunity when the customer expected one. Use when support reports duplicate contracts per Salesforce opportunity, same template opened several times from SFDC, 'three contracts for one opportunity', or triage of ExternalIntegrationContractDetail rows. Trigger phrases: 'duplicate contract Salesforce', 'SFDC widget created two contracts', 'multiple contracts same opportunity', 'getExistingContractInSignState', 'allow_duplicate_execution_version', 'template_contracts duplicate', 'DRAFT duplicate SFDC'."
---

# Salesforce Widget: Duplicate Contract Creation (Same Opportunity)

## What This Skill Debugs

Reports that **more than one contract** was created from the SpotDraft **Salesforce embedded app** for the **same Opportunity** (and often the same template), when the customer believed a single contract should exist.

This is usually **not** a failed API call: each `POST` may **succeed**. The gap is **client-side deduplication**: the widget only warns about an **existing contract in SIGN display status** (awaiting signatures). Contracts in **DRAFT** or **INFO_COLLECTION** for the same opportunity are **not** treated as blockers, so each fresh widget session can call `createContract()` again.

**Earliest failure point to validate:** Confirm whether the customer (or automation) triggered **multiple successful creation requests** (distinct timestamps / HTTP requests), versus a single request that forked server-side (rare).

---

## When to Use / Not Use

| Use this skill | Do not confuse with |
|----------------|---------------------|
| Same Opportunity ID → multiple contracts from SFDC flow | `hubspot-sfdc-conditional-overpersistence` (hidden fields persisted incorrectly, activity-log mass cleanup) |
| User opened widget multiple times, clicked through questionnaire | `questionnaire-hidden-variable-data-loss` (missing fields after edit; may **co-occur** as motivation to retry — cross-check dates) |
| Need to pick which contract to keep and void extras | Generic `contract-lookup` alone (use this for SFDC-specific dedupe logic) |

---

## Code Paths (angular-frontend / `apps/sfdc`)

**Template selection → questionnaire (dedupe gate):** `TemplateContainerService#unmarkExistingTemplate()`

- If `allow_duplicate_execution_version` is **true** for the workspace/opportunity context → **skips** the existing-contract check and navigates straight to the questionnaire.
- If **false** → calls `getExistingContractInSignState(template)` and only shows the "unmark for execution" modal when a contract in **SIGN** state exists.

**Existing-contract search:** `SfdcExistingContractsService#getExistingContractInSignState()` builds a contract search with `display_status = SIGN` (among other filters). **DRAFT** and **INFO_COLLECTION** contracts for the same external metadata (Salesforce record / opportunity) are **not** returned by this query.

**Questionnaire submit:** On a fresh load, `contract$.id` may be unset → `createContract()` runs. Repeated widget opens can each create a new contract.

---

## Investigation Workflow

### 1. Capture identifiers

Collect:

- `{opportunity_id}` (Salesforce Opportunity ID)
- `{wsid}` / workspace id
- `{cluster}` (US / EU / IN / MEA)
- Contract IDs involved (`{contract_id}` list)
- Template name / template id if known
- Time range when duplicates appeared

### 2. Confirm duplicates are real and scoped

- In **admin** or **Metabase** (per org conventions), list rows for **ExternalIntegrationContractDetail** (or equivalent reporting) filtered by `{opportunity_id}` or external id — expect **one row per contract** linked to that opportunity.
- Compare **created** timestamps across contracts: **minutes apart** usually means **separate user or automation sessions**, not a double-submit race on one click.

**Example pattern (incident SPD-42524):** Three contracts for one opportunity, created ~5 minutes apart; one middle contract with **no** intake responses (abandoned attempt).

### 3. Separate “bug” from “working as designed today”

If logs show **multiple successful** `POST`s to contract creation / template-contract endpoints (see below) aligned with timestamps, engineering may classify this as **WAI under current product rules** while a **product change** (dedupe for non-terminal states, server-side idempotency) is tracked separately.

### 4. HTTP / GCP — prove distinct creation events

Use **HTTP load balancer** logs (prod project for the cluster, e.g. `spotdraft-prod` for US) around the reported window:

- Filter paths such as `/api/v3/contracts/v2/template_contracts/` (adjust version if your investigation uses v2 public contracts — align with live routes).
- Group by `httpRequest.remoteIp`, timestamp, and path to show **separate** successful creates.

> Preserve **anonymized** IPs in runbooks; only use full IPs under company policy.

**Example query shape (adapt project, time range, path):**

```
resource.type="http_load_balancer"
httpRequest.requestUrl:"/api/v3/contracts/v2/template_contracts/"
```

Narrow with `httpRequest.remoteIp` once a single actor is confirmed.

### 5. BigQuery — contract rows (if prod dataset access)

Map `{cluster}` → dataset (see `contract-lookup` / `contract-signing-triage` skills for `prod_*_db` naming). Example **Prod US**:

```sql
SELECT id, status, workflow_status, contract_kind, created, modified
FROM `spotdraft-prod.prod_usa_db.public_contracts_v3_contractv3`
WHERE workspace_id = {workspace_id}
  AND created >= TIMESTAMP('{start_ts}')
  AND created <= TIMESTAMP('{end_ts}')
ORDER BY created ASC
```

Cross-check multiple rows with same business context via integration tables (naming differs by env; use Metabase/admin if ORM table names are unclear).

### 6. Settings check

- **`allow_duplicate_execution_version`:** If **true**, the SFDC flow **intentionally** skips dedupe — duplicates are more likely. Verify in workspace/opportunity configuration via admin (exact navigation is product-internal).

### 7. Correlate with hidden-questionnaire regressions

If contract dates fall in a known **hidden integration-mapped questions** incident window (e.g. March 4–17, 2026 per internal comms), missing data on the **first** contract may have caused the user to **retry** the widget. That does not change the duplicate-creation mechanics but explains **why** multiple sessions occurred. Use `questionnaire-hidden-variable-data-loss` for the **data** issue.

---

## Mitigation (customer unblock)

1. **Agree** which single contract is canonical (often the **most complete** — questionnaire responses, documents — not necessarily the oldest).
2. **Void** duplicate contracts via **admin** change URLs for the environment, e.g. `https://api.{region}.spotdraft.com/admin/contracts_v3/contractv3/{contract_id}/change/` (replace region/host per cluster conventions).
3. Confirm **voided** contracts are excluded from reporting and customer-facing lists.
4. Document **`allow_duplicate_execution_version`** for the workspace if duplicates must not recur operationally before a product fix.

---

## Long-Term Fix Directions (engineering / product)

These options were discussed as **pod-level** work — track in Jira; not all may be scheduled:

1. Broaden “existing contract” checks beyond **SIGN** (e.g. any **non-terminal** state for same opportunity + template) and surface a **warning modal** before creating another.
2. On SFDC widget load, resolve **ExternalIntegrationContractDetail** (or backend equivalent) for active links to the opportunity and **redirect** to the existing contract instead of template selection.
3. **Server-side idempotency:** same `(workspace, opportunity_id, template_id)` while a contract is in a non-terminal state → return **existing** contract instead of creating a new one.

---

## Escalation Triggers

- Duplicates created **without** multiple HTTP creates (investigate race/idempotency at API layer).
- Duplicates across **different** opportunities or templates (likely different root cause).
- **Automation** in Salesforce firing widget URLs or integrations repeatedly — engage CS + customer SFDC admin.

---

## Related Incidents / Tracking

- **Jira:** [SPD-42524](https://spotdraft.atlassian.net/browse/SPD-42524) — Lighcast duplicate contract creation from Salesforce opportunity (incident thread March 2026).
- **Rootly:** incident `3016` (duplicate contract creation from Salesforce opportunity) — used for timeline / status.

**Known example identifiers (do not treat as live data — for pattern only):** Opportunity `0Q0RQ000004NdhV0AS`, WSID `259502`, contracts `1429870` / `1429872` / `1429873`, cluster US, Mar 14 2026.

---

## Connections to Other Skills

- `questionnaire-hidden-variable-data-loss` — missing fields motivating retries.
- `contract-lookup` — resolving workspace, cluster, and contract identity.
- `contract-signing-triage` — if one duplicate progressed to signing and others did not.
- `hubspot-sfdc-conditional-overpersistence` — **different** problem (conditional hidden field persistence / cleanup), not duplicate creation.
