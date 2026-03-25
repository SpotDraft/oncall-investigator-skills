---
name: tpp-workflow-stale-entity-mapping
description: "Diagnose missing creator-party (or other) entities in TPP/third-party-paper workflow conditional logic while Contract Type settings list all expected entities. Use when dropdowns show fewer entities than configured, 'only one selectable entity', stale workflow entity mapping, entity added after workflow was created, or post–Sep 2025 sync behavior questions. Trigger phrases: 'missing entity conditional logic', 'TPP workflow entity dropdown', 'Ocrolus-style entity missing', 'workflow clone to fix entity', 'SPD-42665', 'SPD-35479', 'WorkflowToOwnerEntityMapping', creator-party conditional."
---

# TPP / Workflow Stale Entity Mapping (Conditional Logic)

## What This Skill Debugs

**Symptom:** For a **TPP** (third-party paper / upload) **workflow**, **conditional logic** that depends on **creator-party entities** (or similar) shows **fewer selectable entities in the UI** than the **Contract Type / workspace settings** show — e.g. only one entity in the dropdown when two are configured.

**Nature:** This is usually **data consistency**, not a live 5xx or flaky dependency. Logs/traces often show nothing useful unless a failed **admin/API** call was captured.

**Historical context (engineering, incident thread):** Through **2025-09-14**, when a **new organization entity** was created **after** an existing **workflow** was already created, that entity was **not** automatically linked to that workflow’s entity mapping set. A **prod fix** landed **2025-09-15** so **new** entities are synced to relevant workflows going forward. **Older workflows** can still have **stale mappings** until **clone** or **backend remap/resync**.

**Know example (for calibration only — use placeholders in new incidents):** Rootly incident “[Missing entity in TPP workflow conditional logic](https://rootly.com/account/incidents/3027-missing-entity-in-tpp-workflow-conditional-logic)”, Jira [SPD-42665](https://spotdraft.atlassian.net/browse/SPD-42665), related class [SPD-35479](https://spotdraft.atlassian.net/browse/SPD-35479) (child contract type / draft workflow entity visibility).

---

## When to Use / Not Use

| Use this skill | Prefer another skill |
|----------------|---------------------|
| Entity **missing only inside workflow / conditional UI** while settings look correct | **HubSpot/SFDC conditional overpersistence** — wrong hidden values persisted on integration contracts |
| **Workflow created before** entity; entity missing in **TPP** branching | **Questionnaire hidden variable** data loss — different symptom (values cleared) |
| Suspect **Workflow ↔ contract-type / group mapping** drift | Generic “bug” with **no** workflow/entity angle |

---

## Available MCP Tools

### Slack

- `slack_read_channel` / `slack_read_thread` — read `#incident-*` context.
- `slack_search_public` — find prior “entity missing workflow conditional” threads.

### Jira (Atlassian MCP)

- `searchJiraIssuesUsingJql` — `SPD-42665`, `SPD-35479`, labels like `agent-investigated`.

### BigQuery

> Run queries **one at a time** (parallel `execute_sql` often returns 500s).

**Workflows for a workspace (adjust project/dataset for env — QA example):**
```sql
SELECT
  w.id AS workflow_id,
  w.title,
  w.type,
  w.status,
  w.created,
  w.tenant_workspace_id
FROM `spotdraft-qa.qa_india_public.workflow_v1_workflow` w
WHERE w.tenant_workspace_id = {workspace_id}
  AND w.is_deleted = FALSE
ORDER BY w.id;
```

**Workflow → owner entity mappings (QA table names; prod uses `spotdraft-prod`, dataset e.g. `prod_usa_db`, `public_` prefix):**
```sql
SELECT
  m.id,
  m.workflow_id,
  m.entity_id,
  m.entity_type,
  m.is_primary_source,
  m.is_deleted,
  m.created
FROM `spotdraft-qa.qa_india_public.workflow_v1_workflowtoownerentitymapping` m
WHERE m.workflow_id = {workflow_id}
  AND m.tenant_workspace_id = {workspace_id}
  AND m.is_deleted = FALSE
ORDER BY m.id;
```

**Contract settings entities (org entities allowed for a workspace contract type) — QA:**
```sql
SELECT
  cse.id,
  cse.organization_entity_id,
  cse.is_deleted,
  wct.id AS workspace_contract_type_id,
  wct.contract_type_id,
  wct.workspace_id
FROM `spotdraft-qa.qa_india_public.contracts_v3_contractsettingsentity` cse
JOIN `spotdraft-qa.qa_india_public.contracts_v3_workspacecontracttype` wct
  ON wct.id = cse.workspace_contract_type_id
WHERE wct.workspace_id = {workspace_id}
  AND wct.contract_type_id = {contract_type_id}
ORDER BY cse.organization_entity_id;
```

**Environment-to-dataset mapping:** See `contract-lookup` / `incident-lookup` skills (QA `*_public` tables; prod `spotdraft-prod` + `public_*` prefix).

### Metabase

Engineers used a **Metabase** question joining **Workflow V1** `Workflowtoownerentitymapping` with **Contract Type** / **Contract Settings Entity** rows to compare **workflow-linked** vs **settings-allowed** entities (see thread on Rootly #3027 / `#incident-20260323-medium-missing-entity-in-tpp-workflow-conditional-logic`). Prefer recreating that join in Metabase against the correct env DB if BQ lag or joins are easier there.

### GCP / Groundcover

- **Not first-line** for this class: no stable `request_id` unless reproducing an API failure.
- Use only if the UI calls a failing endpoint (then correlate as usual).

---

## Step-by-Step Investigation

1. **Confirm symptom**  
   - Which **workflow** (name / `workflow_id`)?  
   - Which **contract type**?  
   - How many **creator-party** (or relevant) entities in **settings** vs **conditional UI**?

2. **Gather ordered timestamps**  
   - **`workflow_v1_workflow.created`** vs **`organization_entity` (or entity-as-seen-in-settings) created**.  
   - If **entity created after workflow** and workflow was created **before 2025-09-15**, stale mapping is **likely** (per engineering explanation in incident thread).

3. **Compare mappings in data**  
   - List **`workflow_v1_workflowtoownerentitymapping`** for `workflow_id` + `tenant_workspace_id`.  
   - List **`contracts_v3_contractsettingsentity`** for the workspace + contract type.  
   - Reconcile with Metabase-style join if needed.

4. **Rule out**  
   - Wrong **workspace** / **contract type** / **workflow** (duplicate names).  
   - **Feature misunderstanding** (different questionnaire path than TPP conditional).  
   - Pure **integration** conditional bugs (see `hubspot-sfdc-conditional-overpersistence`).

5. **Confirm blast radius**  
   - Other **workflows** created **before** the same entity was added may lack the same mapping (example from incident: multiple `workflow_id` values listed for one new entity).

---

## Code Paths / Architecture (High Level)

- **Workflow ownership mappings** live in **`workflow_v1_workflowtoownerentitymapping`** (`WorkflowToOwnerEntityMapping` in `django-rest-api`).
- **Contract type ↔ allowed org entities** for a workspace: **`contracts_v3_contractsettingsentity`** (`ContractSettingsEntity`).
- **Workflow creation** links contract type **group** (and children) via `CreateWorkflowUseCase._create_and_get_workflow_to_owner_mappings_for_contract_type` (`django-rest-api/workflow_v1/workflow/domain/use_cases/create_workflow_use_case.py`).
- **Owner entities for workflow** resolution: `GetOwnerEntitiesForWorkflowUseCase` (`django-rest-api/workflow_v1/workflow_to_owner_entity_mapping/domain/use_cases/get_owner_entities_for_workflow_use_case.py`).
- **Contract settings API:** `contract_settings_service.update_contract_settings_entity` (`django-rest-api/contracts_v3/services/contract_settings_service.py`) — incident mitigation invoked this from a **Django shell**; align with owning team for **current** recommended bulk remediation beyond shell one-offs.

---

## Known Patterns

| Pattern | Notes |
|--------|--------|
| **Entity created after workflow (pre–2025-09-15)** | Classic stale mapping; fix in prod **2025-09-15** prevents **new** gaps but does not always **backfill** old workflows. |
| **Clone workflow** | Historically the **self-serve** way to refresh mappings; still valid mitigation per incident discussion. |
| **Backend resync / script** | Engineering ran targeted **shell** sync; get **`org_user_id`**, **`organization_entity_id`**, **`contract_type_id`**, **`workspace_id`** from admin/Jira — **verify** IDs (thread had a `workspace_id` note vs script; always confirm from ticket). |

---

## Mitigation / Workaround

1. **Customer-fast path:** **Clone the workflow** (or recreate) so mappings are rebuilt — confirm with PM/CS if side effects matter (published state, automations, links).

2. **Engineering path (example from incident — placeholders required):** Django shell pattern used on **production** under controlled change:

```python
import json
from contracts_v3.services import contract_settings_service
from contracts_v3.type.contract_settings_types import ContractSettingsEntityCreateRequest

def run() -> None:
    workspace_id = {workspace_id}
    contract_type_id = {contract_type_id}
    org_user_id = {org_user_id}
    organization_entity_id = {organization_entity_id}
    enable = True
    try:
        create_request = ContractSettingsEntityCreateRequest(
            organization_entity_id=organization_entity_id,
            enable=enable,
        )
        print(
            f"running for workspace {workspace_id}, contract type {contract_type_id}, "
            f"org user {org_user_id}, organization entity {organization_entity_id}, enable {enable}"
        )
        response = contract_settings_service.update_contract_settings_entity(
            workspace_id=workspace_id,
            contract_type_id=contract_type_id,
            request=create_request,
            org_user_id=org_user_id,
        )
        print(f"Successfully resynced contract type {contract_type_id} for workspace {workspace_id}")
        print(f"Response: {json.dumps(response.dict(), indent=2, default=str)}")
    except Exception as e:
        print("Error while executing the function", e)
```

- **Do not** bulk-run for all workflows without **listing affected** `workflow_id`s and **customer sign-off** (incident follow-up called for running for all workflows **before fixing date** after confirmation).

3. **Longer-term:** One-time **backfill** for pre-sync-era workflows; optional **product** signal when workflow entity set **lags** contract-type entity set (per incident discussion).

---

## Escalation Triggers

- Shell/resync **does not** restore UI after verified IDs — possible **different root** (frontend, wrong workflow revision, approvals v5 snapshot, etc.); escalate to **workflow + questionnaire** owners.
- Need **many** workspaces fixed — track as **engineering initiative** / migration, not ad hoc tickets only.

---

## Related

| Link | Role |
|------|------|
| [SPD-42665](https://spotdraft.atlassian.net/browse/SPD-42665) | Primary bug / incident ticket |
| [SPD-35479](https://spotdraft.atlassian.net/browse/SPD-35479) | Same class (entities not showing for draft workflow / child type) |
| [Rootly #3027](https://rootly.com/account/incidents/3027-missing-entity-in-tpp-workflow-conditional-logic) | Timeline, Metabase reference |
| `contract-lookup` | BQ table names, workspace/workflow queries |
| `incident-lookup` | Broader incident search patterns |
| `hubspot-sfdc-conditional-overpersistence` | Integration conditional **value** bugs (different) |

---

## Incident Channel Reference

- Slack: `#incident-20260323-medium-missing-entity-in-tpp-workflow-conditional-logic` (`C0AP2BVUC9W`).
