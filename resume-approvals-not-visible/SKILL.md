---
name: resume-approvals-not-visible
description: "Diagnose why the 'Resume Approvals' button is not visible in the Manage Approvals sidesheet or contract page task card, despite the user having Admin, Editor, or Business User access. Use this skill when support reports: 'user can't see Resume Approvals', 'Resume Approvals option missing', 'approval breakpoint not resumable', 'user has access but can't resume approval', 'approval is paused but no resume button', 'resume button not showing', 'manual breakpoint not visible', or any report that the Resume button is absent even though the user has the required permissions. Also use for investigating FE/BE discrepancies in who sees the Resume button (e.g., some Admins see it, others don't)."
---

# Resume Approvals Button Not Visible

## What This Skill Diagnoses

The Resume Approvals button can be absent for two distinct, unrelated reasons:

1. **Sequential approval blocking the breakpoint (most common — WAI):** A manual pause breakpoint only reaches `PENDING` state after all sequentially ordered approvals _before_ it complete. If an upstream approval is still `PENDING` or `NOT_STARTED`, the breakpoint itself will be `NOT_STARTED`, and the Resume button will not render. This is correct behavior, not a bug.

2. **FE/BE inconsistency in who sees the button:** The Manage Approvals sidesheet shows the Resume button to _any_ user who receives the approvals list containing a `PENDING` breakpoint (no BU check in FE). The contract page task card is shown only to the Business User. This inconsistency can cause confusing support reports where some Admins see the button and others don't.

---

## Available MCP Tools for Investigation

### BigQuery — Approval State

**⚠️ Run BQ queries one at a time** — parallel `execute_sql` calls return 500 errors.

**Check all approvals for a contract (run this first):**
```sql
SELECT
  id, name, current_state, `order`,
  enforcement_point_type, breakpoint_type, approval_type,
  is_deleted
FROM `spotdraft-prod.prod_eu_db.public_approvals_v5_approvalv5`
WHERE linked_to_entity_id = {contract_id}
  AND linked_to_entity_type = 'CONTRACT'
  AND is_deleted = FALSE
ORDER BY `order`
```

Look for:
- Is there a row with `approval_type = 'MANUAL_BREAKPOINT'`? What is its `current_state`?
- Are there rows with lower `order` values that are still `PENDING` or `NOT_STARTED`?
- If a sequential approval at a lower order is `PENDING`, the manual breakpoint downstream will be `NOT_STARTED` → Resume button will not show → **this is WAI**.

**Check the Business User for a contract (Prod EU example):**
```sql
SELECT
  c.id AS contract_id,
  c.business_user_id,
  ou.user_email AS business_user_email,
  ou.id AS org_user_id
FROM `spotdraft-prod.prod_eu_db.public_contracts_v3_contractv3` c
LEFT JOIN `spotdraft-prod.prod_eu_db.public_sd_organizations_organizationuser` ou
  ON ou.id = c.business_user_id
WHERE c.id = {contract_id}
```

**Check who is in the workspace and whether a user has is_dummy profile:**
```sql
SELECT
  cp.id AS workspace_id,
  cp.name AS workspace_name,
  ou.id AS org_user_id,
  ou.user_email,
  ou.user_id AS auth_user_id,
  p.is_dummy AS user_profile_is_dummy,
  ou.is_active,
  ou.is_deleted,
  cp.is_enabled,
  cp.is_active_workspace
FROM sd_organizations_companyprofile cp
JOIN sd_organizations_organizationuser ou ON ou.organization_id = cp.owner_id
LEFT JOIN sd_organizations_profile p ON p.user_id = ou.user_id
WHERE ou.user_email = '{user_email}'
  AND (ou.is_deleted = false OR ou.is_deleted IS NULL)
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

### GCP Cloud Logging

**⚠️ Prod uses `jsonPayload`; QA uses `textPayload`.**

**Inspect the approvals API call for a specific request (Prod EU):**
```
resource.type="k8s_container"
resource.labels.project_id="spotdraft-prod"
resource.labels.location="europe-west4-a"
resource.labels.cluster_name="prod-eu"
resource.labels.namespace_name="prod"
--"/api/v5/approvals/contract/{contract_id}"
jsonPayload.request_id="{request_id}"
```

**Find the approvals API call for a contract in Prod EU (no request_id):**
```
resource.type="k8s_container"
resource.labels.project_id="spotdraft-prod"
resource.labels.cluster_name="prod-eu"
resource.labels.namespace_name="prod"
jsonPayload.message:"approvals/contract/{contract_id}"
```

**Check for non-BU policy filtering warning (expected for non-BU users, not an error):**
```
jsonPayload.message:"user does not have access to filtering all policies"
jsonPayload.message:"{contract_id}"
```

### Admin URLs

| Action | URL |
|--------|-----|
| View approvals for a contract | `https://api.{cluster}.spotdraft.com/admin/approvals_v5/approvalv5/?linked_to_entity_id={contract_id}` |
| Resume approval from admin | `https://app.spotdraft.com/contracts/v2/{contract_id}?blocked=true&clusterId={CLUSTER}&workspaceId={wsid}` |
| Contract detail | `https://api.{cluster}.spotdraft.com/admin/core/contract/{contract_id}/change/` |

---

## When to Use This Skill

**Use when:**
- User reports the Resume Approvals button is not visible
- User has Admin, Editor, or BU access but still can't see the button
- Some users in the same workspace see the button, others don't
- Approvals are reportedly "paused" but no resume option is showing

**Do not use for:**
- User can see the button but the Resume action fails on click → different issue
- Approvals not triggering at all or approval reset issues → use `contract-signing-triage` or check for workflow state bugs
- Approval reset issues → unrelated flow

---

## System Architecture: How the Resume Button Works

### Two locations the Resume button appears

**1. Contract page — task card**
- The backend assigns a "paused approval" task only to the **Business User (BU)**.
- Only the BU has this task card in their contract view.
- Non-BU users never see this card regardless of their access level.

**2. Manage Approvals sidesheet**
- Populated by `GET /api/v5/approvals/contract/{contract_id}/`
- Backend: `ListApprovalDetailsUseCase` returns the full approvals list, including the manual breakpoint row.
- Non-BU users receive the same response as BU (with the `"user does not have access to filtering all policies"` warning — this is non-fatal and expected).
- Frontend: `ApprovalBreakpointItemInfoComponent` renders the breakpoint row and Resume button when `breakpoint.status === ApprovalStatus.PENDING`.
- **⚠️ FE bug (unresolved):** There is no BU check in this component. Any user who receives a `PENDING` breakpoint in the response will see the Resume button — even non-BU Admins. This contradicts BE-intended behavior.

### FE render logic

File: `angular-frontend/libs/approvals/ui/src/lib/approval-breakpoint-item-info/approval-breakpoint-item-info.component.ts`

```typescript
get isResumeVisible() {
  return this.breakpoint?.status === ApprovalStatus.PENDING;
}
```

`isPendingOnCurrentUser()` controls only **button style** (primary vs secondary), NOT visibility. This function is misleadingly named — it does not restrict who can see or click the button.

### Sequential approval flow (critical to understand)

When a contract has **sequential approvals** (ordered steps), each approval is triggered in order. A manual pause breakpoint at order N will only reach `PENDING` state after all approvals at orders < N are completed. While any prior approval is `PENDING` or `NOT_STARTED`, the breakpoint remains `NOT_STARTED`, and `isResumeVisible` returns `false`.

**Approval states in `current_state` field:**
- `PENDING` — currently active, awaiting action
- `NOT_STARTED` — not yet triggered (upstream steps not complete)
- `APPROVED` / `SKIPPED` / `REJECTED` — completed states

---

## Step-by-Step Investigation Workflow

### Step 1: Gather identifiers

From the support ticket, get:
- **Contract ID** and link
- **Workspace ID (WSID)** and **Cluster** (IN / EU / US / ME)
- **User email** of the user who can't see the button
- Screenshot of the Manage Approvals sidesheet (if available)

### Step 2: Check the approval state (BQ — run first)

Run the approvals query for the contract:

```sql
SELECT
  id, name, current_state, `order`,
  enforcement_point_type, breakpoint_type, approval_type,
  is_deleted
FROM `spotdraft-prod.{dataset}.{prefix}approvals_v5_approvalv5`
WHERE linked_to_entity_id = {contract_id}
  AND linked_to_entity_type = 'CONTRACT'
  AND is_deleted = FALSE
ORDER BY `order`
```

**Decision tree:**

| What you see | Conclusion | Action |
|---|---|---|
| `MANUAL_BREAKPOINT` row has `current_state = NOT_STARTED` AND a lower-order approval is `PENDING` | Sequential approval is blocking the breakpoint — **WAI** | Explain to customer: the earlier approval must complete before resume becomes available |
| `MANUAL_BREAKPOINT` row has `current_state = PENDING` AND the user still can't see the button | FE issue — could be stale state/cache or a rendering condition | Ask user to hard-refresh; check GCP logs for the user's request |
| No `MANUAL_BREAKPOINT` row | Approvals haven't been instated or the breakpoint was never configured | Check the workflow configuration |
| `MANUAL_BREAKPOINT` row has `current_state = APPROVED` | Approval was already resumed (possibly by admin) | Confirm with support team — it's already resolved |

### Step 3: Check who the Business User is

Run the BU query to confirm whether the reporting user is or isn't the BU:

```sql
SELECT c.id, c.business_user_id, ou.user_email
FROM `spotdraft-prod.{dataset}.{prefix}contracts_v3_contractv3` c
LEFT JOIN `spotdraft-prod.{dataset}.{prefix}sd_organizations_organizationuser` ou
  ON ou.id = c.business_user_id
WHERE c.id = {contract_id}
```

- If the reporting user **is** the BU → they should see the task card AND the sidesheet button when the breakpoint is `PENDING`. If they can't see it, the breakpoint is likely `NOT_STARTED` (Step 2 above).
- If the reporting user **is not** the BU → they should only see the button in the Manage Approvals sidesheet when the breakpoint is `PENDING`. If they can't, either the breakpoint is `NOT_STARTED` (WAI) or there is a FE rendering issue.

### Step 4: Check GCP logs for the user's approvals API call

Inspect the actual API response for the affected user to confirm what the backend returned. Find the relevant `request_id` from the GCP log and verify:
- Did the API return 200?
- Does the response include the breakpoint row?
- Is the `"user does not have access to filtering all policies"` warning present? (Expected for non-BU users — not an error.)

### Step 5: Confirm or rule out FE inconsistency

If two users with the same role (e.g., both Admins, neither the BU) see different behavior:
- Check if they accessed the sidesheet at the same time (approval state could have changed between accesses)
- Check if one user's browser has stale cached state
- Check if both received the same approvals list from the backend (compare GCP logs)

---

## Common Patterns

### Pattern 1: Sequential approval blocking downstream breakpoint (most common)

**Symptom:** User reports "can't see Resume Approvals" for a contract with a manual pause breakpoint.

**Diagnosis:**
- BQ: The `MANUAL_BREAKPOINT` row has `current_state = NOT_STARTED`
- BQ: A lower-order approval (e.g., "Information Security" approval at order 1) has `current_state = PENDING`
- GCP: The API returns 200 with the approvals list; no error in backend

**Conclusion:** WAI. The manual breakpoint at step N cannot be resumed because step N-1 (or earlier) hasn't completed. The FE correctly renders no button.

**Known example (Dott, Mar 2026):** Contract 370498 (WS 99755, EU). The Information Security approval at step 1 was `PENDING`. The manual breakpoint at step 2 was `NOT_STARTED`. Cleo Baker (Admin, non-BU) couldn't see the Resume button — correctly. Gourav resumed the approval from SpotDraft admin to unblock the customer before the root cause was confirmed. Jira: SPD-42215.

**Customer communication:** Ask them to wait for the upstream approval (step 1) to complete; the Resume option will appear once the breakpoint becomes active. Or offer to check who the assigned approver is for the upstream step and remind them.

### Pattern 2: Non-BU Admin can resume (FE inconsistency — potential future bug)

**Symptom:** A non-BU Admin user can see and potentially click the Resume button in the Manage Approvals sidesheet.

**Diagnosis:**
- BQ: Breakpoint `current_state = PENDING`
- The user is NOT the BU
- BE intent: only the BU should resume; FE has no such restriction

**Conclusion:** FE bug — `isResumeVisible` does not check BU status. Any user with the approvals list containing a `PENDING` breakpoint can see (and click) Resume. If a customer reports an unexpected resume by a non-BU user, this is the likely cause.

**Related:** Jira SPD-42215 (In Progress, unresolved FE alignment task).

---

## Immediate Mitigation

If the customer needs to be unblocked immediately (e.g., approvals paused but stakeholder can't resume):

1. **Ask the BU to resume:** Identify the BU (BQ query above) and ask them to open the Manage Approvals sidesheet and click Resume.
2. **Update the BU on the contract:** If a different person should be the BU, update the contract's `business_user_id` in Django Admin. Confirm with customer/CS team first.
3. **Admin resume:** SpotDraft support staff can resume from the admin contract view:
   ```
   https://app.spotdraft.com/contracts/v2/{contract_id}?blocked=true&clusterId={CLUSTER}&workspaceId={wsid}
   ```
4. **Direct approval state update (last resort):** Via Django Admin, transition the approval state.

---

## Escalation Triggers

- User is the confirmed BU and the breakpoint is confirmed `PENDING`, but Resume button is still not showing → FE rendering bug, escalate to FE team with GCP log `request_id` evidence
- Non-BU user has resumed an approval that only the BU should be able to resume → FE bug (SPD-42215), escalate to FE team
- Multiple customers reporting Resume button missing on contracts with `PENDING` breakpoints → potential regression

---

## Related Incidents & Jira

| Reference | Details |
|-----------|---------|
| **Rootly Incident #2980** | Dott — Unable to View "Resume Approvals" Option Despite Having Required Access (Mar 11, 2026) |
| **Jira SPD-42215** | Original bug report (Dott, EU cluster, WS 99755) — In Progress |
| **Slack incident channel** | `#incident-20260311-medium-dott-unable-to-view-resume-approvals-option-despite-hav` (C0AKX9UETA6) |
| **FE component** | `angular-frontend/libs/approvals/ui/src/lib/approval-breakpoint-item-info/approval-breakpoint-item-info.component.ts` |

---

## Connections to Other Skills

- **contract-lookup** — Use to get the contract's `business_user_id`, `workflow_status`, and version context before diagnosing
- **incident-lookup** — Use to search for prior approval-related incidents with the same customer or similar symptoms
- **contract-signing-triage** — Use if the issue involves approval reset or approvals blocking a signing/execution flow
