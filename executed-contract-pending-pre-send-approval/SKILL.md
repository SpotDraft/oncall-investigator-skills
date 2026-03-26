---
name: executed-contract-pending-pre-send-approval
description: "Diagnose executed/completed contracts blocked from download because PRE_SEND approval remains pending (often assigned to deleted user). Use when: 'download disabled on executed contract', 'pending pre-send approval on completed contract', 'executed contract cannot be downloaded', 'approval assigned to deleted user', 'multiple contracts stuck after uploaded executed version', 'need to skip pending approvals and resync contracts'."
---

# Executed Contract Blocked by Pending PRE_SEND Approval

## What this skill debugs

This runbook covers incidents where a contract is already **Executed/Completed**, but download (or related restricted actions) is still blocked because a **PRE_SEND** required approval remains in a non-terminal state.

Most common production pattern:

- `ContractRequiredApprovalV2` still exists with `sent_for_approval=True` and active approval status not in `APPROVED/SKIPPED`
- Approver can be a deleted org user, leaving no practical path to complete the approval
- A new executed version may have been uploaded, but stale pending approval still gates action checks
- Blast radius can include many contracts in the same workspace

## When to use / not use

Use when:

- Customer says executed contracts cannot be downloaded
- Incident references pending pre-send approvals on completed contracts
- Rootly/Jira mentions deleted approver or no workflow associated

Do not use for:

- Stale assignee in contract search/manual review (`contract-search-stale-assignee-manual-review`)
- Approval reset after upload caused by V4 collect_approvals (`approvals-reset-on-upload-v4-triage`)

## Key identifiers to collect

- `{workspace_id}`
- `{cluster}`
- `{contract_id}` (sample impacted contract)
- `{org_user_email}` for audit attribution in mitigation script
- Optional: `{request_id}` for log correlation

## Investigation workflow

### 1) Confirm incident signature (Slack/Jira)

- Rootly/Jira/Slack should show:
  - contract status is executed/completed
  - pending pre-send approval still present
  - download disabled / restricted

Known example (for pattern only): Rootly incident 3005, Jira `SPD-42470`, workspace `179471`, sample contract `436143`.

### 2) Verify DB state in BigQuery

Run sequentially (one query at a time):

```sql
SELECT
  cra.id AS required_approval_id,
  cra.approval_name,
  cra.contract_id,
  c.status AS contract_status,
  c.execution_date,
  c.created_by_workspace_id,
  ca.id AS active_approval_id,
  ca.status AS active_approval_status
FROM `spotdraft-prod.prod_{region}_db.public_contracts_v3_contractrequiredapprovalv2` AS cra
JOIN `spotdraft-prod.prod_{region}_db.public_contracts_v3_contractv3` AS c
  ON c.id = cra.contract_id
LEFT JOIN `spotdraft-prod.prod_{region}_db.public_contracts_v3_contractapprovalv2` AS ca
  ON ca.id = cra.active_approval_id
WHERE c.created_by_workspace_id = {workspace_id}
  AND c.status = 'COMPLETED'
  AND c.execution_date IS NOT NULL
  AND cra.approval_type = 'PRE_SEND'
  AND cra.is_deleted = FALSE
  AND (ca.status IS NULL OR ca.status NOT IN ('APPROVED', 'SKIPPED'))
ORDER BY cra.contract_id, cra.id;
```

If rows are returned, the issue class is confirmed.

### 3) Validate restriction code-path in django-rest-api

- `ContractRequiredApprovalV2DBRepository.are_approvals_in_progress(...)` checks pending required approvals and returns true when non-terminal approvals exist
- `IsActionRestrictedDueToActiveApproval` maps download endpoints to `ApprovalRestrictedAction.DOWNLOAD`

Together, these can block download even when contract is completed, if approval state is still pending.

### 4) Check deletion guard gap for approvers

- Org user deletion/deactivation checks may not clear pending PRE_SEND approvals automatically
- If assignee was deleted, the pending approval can remain orphaned

### 5) Optional logs/telemetry

- Groundcover/GCP logs are usually low-signal here (often no runtime exception)
- Use logs mainly to correlate timeline, not to prove root cause

## Mitigation (customer unblock)

Use the mitigation script in `django-rest-api`:

- `scripts/oncall_mitigations/probo_skip_pending_approvals.py`

Runbook:

1. Set `WORKSPACE_ID` and `ORG_USER_EMAIL` in script
2. Dry run first:
   - `execute(dry_run=True)`
3. Apply:
   - `execute(dry_run=False)`
4. Script bulk-skips pending approvals and triggers partial contract search resync

Expected outcome:

- affected PRE_SEND approvals move to `SKIPPED`
- executed contracts are downloadable again

## Confirm / disprove matrix

| Observation | Interpretation |
|---|---|
| Completed contract + pending PRE_SEND in query | Confirms this issue class |
| Download blocked, no pending PRE_SEND rows | Use another skill (not this class) |
| Pending approvals skipped and download works | Mitigation successful |
| No contract version found for some approvals in script output | Data integrity edge case, escalate |

## Escalation triggers

- Mitigation script cannot apply because of missing contract versions or repository errors
- Recurs frequently across workspaces (needs product-level fix)
- Need durable behavior change (e.g., auto-resolve PRE_SEND after execution, or cleanup pending approvals on user deletion)

## Code paths to reference

- `contracts_v3/contract_approvals/data/db_repo.py` (`are_approvals_in_progress`)
- `sd_auth/permissions.py` (`IsActionRestrictedDueToActiveApproval`)
- `sd_organizations_v2/org_user/domain/org_user_can_be_deleted_or_deactivated_check_use_case.py`
- `scripts/oncall_mitigations/probo_skip_pending_approvals.py`

## Related incidents / tickets

- Jira: `SPD-42470`
- Rootly: incident 3005 (Probo)
- Slack: incident channel `C0AMYDFLLBA`

## Connections to other skills

- `contract-lookup` for baseline workspace/contract state
- `contract-search-stale-assignee-manual-review` for ES assignee drift after manual task reassignment (different issue class)
