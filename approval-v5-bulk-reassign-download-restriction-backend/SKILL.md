---
name: approval-v5-bulk-reassign-download-restriction-backend
description: "Backend / support playbook for Approvals V5 when workflow settings were changed in Workflow Manager but in-flight contracts still need pending approvers updated, download allowed during pending approval, or both—operations not fully achievable via product UI. Use for: bulk reassign pending approval to new org user, Restrict Actions DOWNLOAD exemption for a user mid-approval, customer contracts before a cutoff date, 'cannot update approval setup for existing contracts', workflow vs contract frozen state. Related: frozen workflow context in approvals-reset-on-upload-v4-triage; contract-lookup for IDs. Not for: Resume Approvals button (resume-approvals-not-visible), upload-triggered reset (approvals-reset-on-upload-v4-triage)."
---

# Approvals V5 — Bulk approver reassignment & download restriction (backend)

## What this skill covers

**Problem class:** The customer (or CSM) updates **approval configuration** on a workspace workflow, but **contracts already in progress** still have **pending** approvals with the **old** approver mapping, old restrict-action behavior, or both. They need **data fixes on existing approval rows**, not only the live workflow template.

**Typical symptoms**

- Pending approvals still show the previous approver after the workflow was republished or approvers were edited in Workflow Manager.
- Support asks whether reassignment can be done in bulk from the UI; only a handful of approvals may be affected, but UI/workflows are **workflow-definition** oriented—investigators confirmed confusion between **workflow-level** setup and **per-contract** approval instances (`#incident-20260319-high-crunchbase-request-to-update-approver-and-permissions-for`).
- Customer wants a specific user to **download** while approvals are **in progress**, under **Restrict Actions → Download**—requires aligning `ApprovalV5RestrictAction` and related **restriction** `ApprovalV5ActorMapping` rows.

**Product / safety note:** Engineering may flag that bulk backend changes **are not first-class product features** and can have **unwanted side effects**—get **explicit scope** (approval names, workflows, date cutoffs, which contracts) and **dry-run** before write paths. See escalation below.

---

## When to use / not use

| Use this skill | Use a different skill |
|----------------|------------------------|
| Bulk **reassign** pending V5 **APPROVER** mappings to a new org user; optional **contract audit** + ES resync | **Resume** button missing → `resume-approvals-not-visible` |
| Adjust **DOWNLOAD** restrict action + add **restriction** actor mappings for exemptions | Upload/version reset behavior → `approvals-reset-on-upload-v4-triage` |
| Confirm **which** pending approvals match names/workflows/`{wsid}` | Base IDs and tables → `contract-lookup` |

---

## Identifiers to collect first

- `{workspace_id}` (tenant workspace)
- `{cluster}` → pick **BigQuery** `project` / `dataset` / `prefix` (see `contract-lookup` / `resume-approvals-not-visible` mapping; **US** → `spotdraft-prod.prod_usa_db` + `public_` prefix)
- **Exact pending approval step names** (e.g. `Legal Approval - CS`) — customer quotes may omit punctuation/spacing; verify in DB
- **Workflow scope** — live workflow IDs from Workflow Manager vs **frozen** workflow on contracts (see `approvals-reset-on-upload-v4-triage`)
- **Time cutoff** if the request is only for contracts created before `{cutoff_date}` — confirm with CSM; add `created` filter in listing queries
- `{new_approver_email}`, `{workspace_user_email}` (support actor for audits / `created_by`)

---

## Evidence & tooling

| Source | Role |
|--------|------|
| **Slack** | Scope clarification (e.g. all pending with listed names vs subset), execution confirmation |
| **Metabase** | Pre-built questions joining `approvals_v5_*`, workflow frozen mapping, filters by name/state/workspace (SpotDraft internal Metabase) |
| **BigQuery** | Authoritative rows for approvals, mappings, contracts |
| **Django shell (PROD)** | Controlled script: dry run → transactional apply (see workflow below) |
| **Jira** | Example tracker: `SPD-42585` (Crunchbase approver + download; Mar 2026). Link new work similarly. |

**⚠️ Run BQ queries one at a time** when using BigQuery MCP (parallel calls may 500).

---

## BigQuery — list candidate pending approvals (pattern)

Adjust `{prefix}` and table names to match the dataset (`public_approvals_v5_approvalv5` on most prod regions).

**Approvals for one contract (fast sanity check):**

```sql
SELECT
  id,
  name,
  current_state,
  `order`,
  linked_to_entity_id AS contract_id,
  tenant_workspace_id
FROM `{project}.{dataset}.{prefix}approvals_v5_approvalv5`
WHERE linked_to_entity_id = {contract_id}
  AND linked_to_entity_type = 'CONTRACT'
  AND is_deleted = FALSE
ORDER BY `order`;
```

**List pending approvals by workspace + names (join contracts for a `created` cutoff if needed):**

Metabase filters in the reference incident used `current_state` **not** `APPROVED`/`SKIPPED`; confirm exact enum values in the environment (`PENDING`, `NOT_STARTED`, etc.).

```sql
SELECT
  a.id AS approval_id,
  a.name,
  a.current_state,
  a.linked_to_entity_id AS contract_id,
  a.tenant_workspace_id,
  c.created AS contract_created
FROM `{project}.{dataset}.{prefix}approvals_v5_approvalv5` a
JOIN `{project}.{dataset}.{prefix}contracts_v3_contractv3` c
  ON c.id = a.linked_to_entity_id
WHERE a.linked_to_entity_type = 'CONTRACT'
  AND a.is_deleted = FALSE
  AND c.is_deleted = FALSE
  AND a.tenant_workspace_id = {workspace_id}
  AND a.current_state NOT IN ('APPROVED', 'SKIPPED')
  AND a.name IN ('{approval_name_1}', '{approval_name_2}')
  -- AND c.created < TIMESTAMP('{cutoff_datetime}')  -- optional scope from incident playbooks
ORDER BY c.created DESC;
```

**Restrict actions for those approvals:**

Table name pattern: `{prefix}approvals_v5_approvalv5restrictaction` (confirm in dataset).

```sql
SELECT *
FROM `{project}.{dataset}.{prefix}approvals_v5_approvalv5restrictaction`
WHERE approval_id IN ({approval_id_list})
  AND restrict_action_type = 'DOWNLOAD'
  AND tenant_workspace_id = {workspace_id};
```

**Approver mappings (APPROVER relation):**

Table pattern: `{prefix}approvals_v5_approvalv5actormapping` — verify in BigQuery `INFORMATION_SCHEMA`.

```sql
SELECT *
FROM `{project}.{dataset}.{prefix}approvals_v5_approvalv5actormapping`
WHERE approval_id IN ({approval_id_list})
  AND tenant_workspace_id = {workspace_id}
  AND actor_relation_type = 'APPROVER';
```

---

## Architectural notes (codebase)

- **`ApprovalV5RestrictAction`** — `restrict_action_type` includes `DOWNLOAD`; `restrict_action_value` includes `NO_ONE`, `ANY_ONE`, `SPECIFIC_USERS_AND_TEAMS` (`approvals_v5/domain/domain_models.py`, `approvals_v5/models.py`).
- **`ApprovalV5ActorMapping`** — `actor_relation_type` **`APPROVER`** vs **`RESTRICTION`** (restriction rows tie to `actor_relation_id` = restrict action id).
- After changing approver-related data, search indexing may need **`partially_resync_contracts_task`** with `fields=["assignees"]` for affected `contract_ids` (`contracts_v3.search_v2.tasks`).
- **Audits:** `ContractAuditTaskType.APPROVAL_V5_REASSIGNED` via `BulkCreateContractAuditsUseCase` preserves support accountability.

---

## Step-by-step support workflow

1. **Freeze scope in writing** — approval **names**, **workspace**, optional **contract created** cutoff, target **emails**, whether **download** exemption is **additive** (exempt user) vs changing global restrict value.
2. **List approvals** — Metabase or BQ; export `approval_id` list; reconcile count with customer (example: **6** approvals in Crunchbase incident).
3. **Resolve org users** — `OrganizationUserService.get_org_user_by_email_through_workspace` for `{new_approver_email}` and acting workspace user.
4. **Dry-run script** in Django shell — log planned `ApprovalV5ActorMapping` updates (APPROVER rows), planned restrict-action updates, new RESTRICTION mappings.
5. **Execute** inside `transaction.atomic()`:
   - `bulk_update` approver mappings (`actor_id`, `actor_type`).
   - `BulkCreateContractAuditsUseCase` for reassignment audit payload (match internal playbook for `ApprovalV5ReassignedAuditDataDomainModel` serialization).
   - For each **DOWNLOAD** restrict where value is **`ANY_ONE`** and customer needs named exemption: set to **`SPECIFIC_USERS_AND_TEAMS`** if appropriate, **`bulk_create`** `ApprovalV5ActorMapping` rows with `actor_relation_type=RESTRICTION` and `actor_relation_id=<restrict_action_pk>`.
   - **Skip** restrict rows already allowing broad download if no change needed (see incident script branch).
6. **Enqueue** `partially_resync_contracts_task` for distinct `contract_id`s.
7. **Verify** in app + ask customer to confirm; update incident / Rootly.

**Known example (Mar 2026):** Workspace `{example_ws_id}` **147532**, US, workflows **Order Form** / **Order Form (Changes)** — pending steps renamed consistently with Legal Approval variants; reassignment to **one** org user + download exemptions. Treat as **historical example** only; always re-query prod.

---

## Confirm / rule out

| Hypothesis | Confirm |
|------------|---------|
| Only live workflow changed | Compare contract frozen workflow vs `workflow_v1_workflow`; approvals on old snapshot won't match UI |
| UI could handle it | Often **no** for historical contracts; small count still may require backend if bindings are workflow-scoped |
| Download already open to everyone | If `restrict_action_value = 'ANY_ONE'`, adding a user is redundant—confirm customer intent |

---

## Mitigation / workaround

- **Short term:** Scoped Django shell script (dry run → prod run) with audits + resync — as executed in `#incident-20260319-high-crunchbase-request-to-update-approver-and-permissions-for`.
- **Medium term:** Internal **mancerie** / repeatable script (per engineer note in thread) for “selective approval id list” updates.

---

## Escalation

- **PM / engineering** if customer asks for behavior that conflicts with **restrict** semantics or broad bulk rewrite across workspaces.
- **Legal / CS** if audit trail or customer comms require explicit approval.

---

## Related incidents & links

- Slack: `#incident-20260319-high-crunchbase-request-to-update-approver-and-permissions-for` (`C0AN50E7P25`)
- Jira: [SPD-42585](https://spotdraft.atlassian.net/browse/SPD-42585)
- Rootly: incident **3020** (Crunchbase approver + permissions) — use for timeline / status page context

## Connections to other skills

- `approvals-reset-on-upload-v4-triage` — frozen workflow vs Workflow Manager expectations
- `contract-lookup` — contract/workspace queries and dataset map
- `resume-approvals-not-visible` — sequential approval visibility (unrelated root cause)
