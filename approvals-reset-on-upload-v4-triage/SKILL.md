---
name: approvals-reset-on-upload-v4-triage
description: "Diagnose reports that contract approvals were reset or re-collected after uploading a new document version, especially when workspace workflow shows Approvals V5 but behavior looks like a bug. Use when support says: 'approvals reset on upload', 're-triggered after upload', 'automatic reset disabled but approvals restarted', 'upload caused approvals to start again', 'collect approvals on version', or when the customer compares workspace-level reset settings to actual behavior. Also use when Workflow Manager ID on the contract page differs from the workflow ID mentioned in errors or recent publishes (frozen workflow vs live workflow)."
---

# Approvals Reset on Upload — V4 vs V5 / Frozen Workflow Triage

## What This Skill Debugs

Support and customers often interpret **approval steps restarting after a document upload** as a platform bug tied to workspace “reset approvals on upload” settings. In production, a frequent explanation is:

1. **The contract still runs on Approvals V4** (pinned to a historical workflow snapshot), while **Workflow Manager shows the current V5 configuration** — so settings and IDs **will not match** what the contract actually uses.
2. **On V4 contracts, the upload flow can explicitly ask the user which approvals to re-collect** (`collect_approvals`). If the user selects approvals in that UI, the system will reset/recollect those steps — **this is working as designed for V4**, not the same rules as V5 (where workflow configuration governs behavior).

A **second-order confusion**: the **activity log** may show approval events in an order that **does not match** the true sequence. Engineering has called this an **activity-log bug** (not fixed as of the Noon incident). Do not use the activity log alone to disprove upload-time recollection when logs/FullStory show `collect_approvals`.

---

## When to Use / Not Use

**Use this skill when:**

- Approvals appear reset immediately after **upload new version** / **replace document**.
- Customer insists workspace-level auto-reset is off — but contract may be **V4** with per-upload picker.
- **Workflow ID mismatch**: contract links to WF `{old_workflow_id}` while errors or recent publishes reference WF `{new_workflow_id}`.
- You need to confirm whether behavior is **user-driven (collect_approvals)** vs **true unexpected reset**.

**Do not use for:**

- **Resume Approvals** button missing → `resume-approvals-not-visible`.
- Signing, execution, PDF generation failures → `contract-signing-triage`.
- Generic approval state without an upload event → start with `contract-lookup` and approval queries.

---

## Available MCP Tools & Data Sources

| Source | Role |
|--------|------|
| **BigQuery** (`spotdraft-prod` + correct regional dataset) | Confirm whether the contract has **V5** rows; workflow / contract creation timing. |
| **GCP Cloud Logging** | Find `update_required_approvals_on_version_update_use_case` and upload pipeline for `{contract_id}` + `request_id`. **Prod: `jsonPayload`.** |
| **FullStory / session replay** (if available) | Upload request payload may show **`collect_approvals`** when log text search does not. |
| **Jira** | Link customer-facing ticket to engineering tracker. |
| **Slack** | Prior incident context (e.g. Noon — channel `#incident-20260226-medium-noon-approvals-reset-on-upload`). |

**⚠️ Run BQ queries one at a time** when using MCP (parallel runs can fail with 500s).

---

## Environment-to-Dataset Mapping (Prod)

Align with `contract-lookup` / `resume-approvals-not-visible`:

| Cluster | BQ project | Dataset | Table prefix |
|---------|------------|---------|--------------|
| EU | `spotdraft-prod` | `prod_eu_db` | `public_` |
| IN | `spotdraft-prod` | `prod_india_db` | `public_` |
| US | `spotdraft-prod` | `prod_usa_db` | `public_` |
| **ME** | `spotdraft-prod` | **`prod_mea_db`** | *(none — no `public_` prefix)* |

Replace `{project}`, `{dataset}`, `{prefix}` in snippets accordingly.

---

## Expected Product Behavior (Condensed)

- **Workspace migrates to Approvals V5** for **new** contracts; **existing** contracts often **remain on V4** until a defined migration exists (there may be **no general migration path** for old contracts — confirm with product/engineering for current policy).
- Each **workflow publish** creates **frozen state** for in-flight contracts. The **UI** can show the **latest** workflow while the **contract** still references an **older** snapshot — large divergence over time is **expected**.
- **V4 upload UX**: user can choose which approvals to re-collect; **V5** is described as **respecting workflow configuration** for that behavior (per incident discussion — validate in current product if customer challenges).

---

## Step-by-Step Investigation Workflow

### 1. Collect identifiers

- `{contract_id}`, `{wsid}`, `{cluster}`, approximate **upload timestamp** (customer timezone OK).
- **User email** who performed the upload (from ticket).
- **Workflow IDs** from contract page vs from error/support narrative.

### 2. Confirm V4 vs V5 for this contract (BQ)

**If the contract has no V5 approval rows (or no V5 contract properties), treat it as V4 for behavior purposes.**

Example pattern (adjust dataset/prefix for region):

```sql
SELECT id, name, current_state, `order`, approval_type, is_deleted
FROM `spotdraft-prod.{dataset}.{prefix}approvals_v5_approvalv5`
WHERE linked_to_entity_id = {contract_id}
  AND linked_to_entity_type = 'CONTRACT'
  AND is_deleted = FALSE
ORDER BY `order`;
```

- **No rows** → strong signal the contract is **not** on the V5 approvals model for that entity (incident pattern: team confirmed “contract is on V4”).
- Cross-check any internal **ApprovalV5ContractProperties**-style tables your runbooks use (naming may vary by env).

**Known example (Noon, Feb–Mar 2026):** Contract `76848`, WSID `10`, cluster **ME** — no V5 rows; workflow manager showed V5 while contract remained V4.

### 3. Logs — version update / approvals update use case (GCP)

Search around the upload time for the contract.

**Useful message fragment (from incident):**

```text
update_required_approvals_on_version_update_use_case called for the contract id = {contract_id}
```

**Prod Django app log filter shape (adapt `cluster_name` for ME/EU/US/IN):**

```text
resource.type="k8s_container"
resource.labels.project_id="spotdraft-prod"
resource.labels.namespace_name="prod"
labels.k8s-pod/app="sd-apps-django-app"
labels.k8s-pod/release="sd-apps"
labels.k8s-pod/tier="app"
labels.k8s-pod/track="prod"
```

Add:

```text
jsonPayload.message:("{contract_id}" OR "update_required_approvals_on_version_update_use_case")
```

If you have a single **`request_id`** from access logs or support, narrow with:

```text
jsonPayload.request_id="{request_id}"
```

**Incident note:** One investigation **did not** find the string `collect_approvals` in GCP for the upload, but **FullStory** showed the upload request **did** include `collect_approvals`. If logs are inconclusive, **session replay** is a valid next step.

### 4. Confirm user selected re-collection (signal for WAI on V4)

Evidence hierarchy:

| Evidence | Interpretation |
|----------|----------------|
| Upload payload / replay shows **`collect_approvals`** with approval IDs | User explicitly requested re-collection → **WAI on V4** |
| `sent_for_approval` (or equivalent) updated on objects inside **`collect_approvals`** in that request | Same — aligns with user action |
| No payload evidence + no log line + approvals still reset | **Escalate** — true unexpected reset |

**Known example (Noon):** Engineer listed approval IDs **`2513511, 2513580, 2513599, 2513970`** as those included in `collect_approvals` for the upload; reset was attributed to user **`jsethia@noon.com`** selecting **four** approvals to re-collect.

### 5. Rule out misleading activity log ordering

If the customer shows the **activity log** with “approval requested **before** upload”:

- Do **not** treat that alone as proof against upload-driven recollection.
- **Incident resolution:** engineer stated the **activity log behavior is a known bug** and should not override request/log/replay evidence.

### 6. Customer communication (mitigation)

- Explain **V4 vs V5** and **frozen workflow** vs **current Workflow Manager** so they understand why IDs and settings differ.
- If **`collect_approvals`** was used: confirm **who** uploaded and that the **V4 upload UI** allows choosing approvals to re-collect; advise process/training to avoid accidental selection.
- For **new** contracts after migration, behavior should follow **V5** rules — **old** contracts may stay on V4 **without** an automatic migration path (confirm current product stance with PM/eng if renewal or compliance depends on it).

---

## How to Confirm / Disprove Likely Causes

| Hypothesis | Confirm | Disprove |
|------------|---------|----------|
| Contract is V4 | No / empty V5 approval rows; historical creation before workspace V5 migration | Rows present and consistent with V5 model |
| User chose re-collection on upload | `collect_approvals` in payload/replay; approval IDs match | Absent across replay, API logs, and backend traces |
| Workflow “mismatch” bug | N/A — usually **expected** frozen snapshot vs live WF | Show contract was recreated on new WF and still mislinked (rare — escalate) |
| Settings should force no reset | Workspace V5 setting does not govern **V4** contract upload picker | Evidence contract is V5 and reset still wrong |

---

## Escalation Triggers

- **V5 contract** (confirmed via BQ/model) with unexplained approval reset on upload — **platform bug** candidate.
- No **`collect_approvals`** / user-action evidence anywhere, but approvals clearly reset — **eng investigation** (trace full upload use case).
- Customer requires **migrating** specific legacy contracts from V4 to V5 — product/eng decision (may not exist as self-serve).

---

## Related Incidents, Jira, Links

| Reference | Notes |
|-----------|--------|
| **Jira [SPD-41629](https://spotdraft.atlassian.net/browse/SPD-41629)** | Noon — Approvals reset on upload (incident driver). |
| **Rootly** | [Incident 2915 — Noon approvals reset on upload](https://rootly.com/account/incidents/2915-noon-approvals-reset-on-upload) |
| **Slack** | `#incident-20260226-medium-noon-approvals-reset-on-upload` (`C0AH7LZMPGT`) |
| **Contract (example)** | `https://app.spotdraft.com/contracts/v2/76848` — WSID `10`, cluster **ME** (illustrative only; use placeholders in new incidents). |

---

## Connections to Other Skills

- **contract-lookup** — Contract row, versions, workflow IDs, BQ dataset selection.
- **resume-approvals-not-visible** — Different symptom (missing Resume button); explicitly not for upload reset.
- **contract-signing-triage** — Signing/execution failures, not approval recollection on upload.
- **incident-lookup** — Search for repeat patterns per customer or workspace.
