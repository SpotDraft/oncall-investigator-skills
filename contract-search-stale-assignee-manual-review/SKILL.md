---
name: contract-search-stale-assignee-manual-review
description: "Diagnose why a reassigned contract review (LEGAL_REVIEW manual task) still appears for the previous assignee in task list or contract search while DB/activity log show the new assignee. Trigger on: 'reassigned review still in old user's tasks', 'wrong user sees review task after reassignment', 'Metabase correct but UI wrong', 'Elasticsearch assigned_to_list stale', 'TEMPLATE_{contract_id} LEGAL_REVIEW wrong org user', 're_sync_contract_search_object race', 'Retry 1 for get_pending_tasks_for_manual_task', 'personal task list not updating after reassignment'. Covers ES contracts index lag vs ManualTaskData versioning and mitigation via contract search resync."
---

# Contract search: stale assignee after manual review reassignment

## What this skill debugs

**Symptom:** After a user reassigns a **contract review** (manual / LEGAL_REVIEW-style task), the task **still shows up for the original assignee** in the **task list** or **search-driven views**, while **activity log** and **Metabase** already show the reassignment to the new user.

**Pattern:** **PostgreSQL is authoritative** (latest `ManualTaskData` has the new `assignee_org_user_id`), but the **`contracts` Elasticsearch index** still has the **old** assignee under `assigned_to_list` for `LEGAL_REVIEW` (or equivalent). User-facing lists that query ES therefore disagree with DB truth.

**Distinct from:** Approval visibility bugs (use approvals-specific skills), pure FE caching (refresh + ES check rules this out), and **transient** ES lag that self-heals seconds later — this pattern is specifically **a resync that ran and committed old state** because the **new `ManualTaskData` row did not exist yet** at resync time (ordering/race).

---

## When to use / not use

| Use this skill | Do not use |
|----------------|------------|
| Task list / search shows old assignee; DB shows new assignee | DB still shows old assignee → data or workflow bug, not ES staleness |
| Contract doc id `TEMPLATE_{contract_id}` has wrong `assigned_to_list` | Issue is only approvals v5 UI with no search involvement |
| Cluster + `contract_id` + `workspace_id` known | Pure process / training issue with no technical inconsistency |

---

## Available MCP tools

### Slack

- `slack_read_channel` / `slack_read_thread` — read incident context.
- `slack_search_public` — find similar past incidents (`#incident-*`, `reassign`, `LEGAL_REVIEW`, `assigned_to_list`).

### Jira (Atlassian MCP)

- Search: `SPD-41912` or JQL for `manual task` + `search` + `elastic` (example ticket from incident [SPD-41912](https://spotdraft.atlassian.net/browse/SPD-41912)).

### Groundcover

- **Logs** (adjust `start`/`end` to reassignment window, `{cluster}` e.g. `prod-eu`):

  - Filter tokens: `ReSync`, `contract`, `search`, `object`, `{contract_id}`.
  - Confirm Celery path: messages containing `ReSync contract search object to elastic successful for contracts: [{contract_id}]`.
  - Look for **`Retry 1 for get_pending_tasks_for_manual_task`** near the same timestamp as resync (secondary signal for read/retry race; not sufficient alone).

**Confirmed example (Touch ’n Go, Rootly [2937](https://rootly.com/account/incidents/2937-touch-n-go-reassigned-review-task-still-visible-in-original-user-s-task-list)):**  
`prod-eu`, contract `360819`, window `2026-02-25T08:50:44Z`–`2026-02-25T09:10:44Z`, filters included `ReSync`, `contract`, `search`, `object`, `360819`.

### BigQuery (PostgreSQL mirrors)

**Run BQ queries one at a time** — parallel `execute_sql` calls often return 500 errors.

**Latest manual task data rows for a contract’s review task (adjust table if schema differs):**

```sql
SELECT
  mtd.id,
  mtd.created,
  mtd.modified,
  mtd.assignee_org_user_id,
  mtd.parent_manual_task_id,
  mtd.is_latest,
  mtd.type,
  mtd.status,
  mtd.previous_data_version_id
FROM `spotdraft-prod.prod_{region}_db.public_contracts_v3_manualtaskdata` mtd
WHERE mtd.contract_id = {contract_id}
  AND mtd.created_by_workspace_id = {workspace_id}
ORDER BY mtd.id DESC
LIMIT 20
```

Replace `{region}` with `eu`, `usa`, `india`, or `mea` matching the incident cluster.

**Contract workspace sanity check:**

```sql
SELECT id, created_by_workspace_id, business_user_id, status
FROM `spotdraft-prod.prod_{region}_db.public_contracts_v3_contractv3`
WHERE id = {contract_id}
```

### GCP Cloud Logging (optional)

**Prod uses `jsonPayload`; QA often uses `textPayload`.**

```
resource.type="k8s_container"
resource.labels.project_id="spotdraft-prod"
resource.labels.cluster_name="prod-{region}"
resource.labels.namespace_name="prod"
jsonPayload.message:"ReSync contract search object"
jsonPayload.message:"{contract_id}"
```

Narrow with a **short time range** around reassignment.

### Elasticsearch (direct / admin tooling)

- **Index:** `contracts` (alias may include version suffix in some envs — confirm in cluster docs).
- **Document id:** `TEMPLATE_{contract_id}` (historic contracts may use `HISTORIC_{contract_id}` — see `BulkUpdateUseCase` delete paths in django-rest-api).
- **Field:** `assigned_to_list` — entries include role/type (e.g. `LEGAL_REVIEW` vs `approval-v5-*`). Compare org user ids to latest `ManualTaskData.assignee_org_user_id`.

---

## Environment-to-dataset mapping

| Environment | BQ project | Dataset | Table prefix |
|-------------|------------|---------|--------------|
| Prod EU | `spotdraft-prod` | `prod_eu_db` | `public_` |
| Prod USA | `spotdraft-prod` | `prod_usa_db` | `public_` |
| Prod India | `spotdraft-prod` | `prod_india_db` | `public_` |
| Prod MEA | `spotdraft-prod` | `prod_mea_db` | *(verify)* |
| QA | `spotdraft-qa` | `qa_*_public` | *(varies)* |

---

## Step-by-step investigation workflow

1. **Capture identifiers:** `{workspace_id}`, `{contract_id}`, `{cluster}`, affected `{org_user_id}` (old and new), `{parent_manual_task_id}` if known.

2. **Confirm DB truth:**  
   - Latest `ManualTaskData` for the review task: `is_latest = true`, `assignee_org_user_id` = **new** user.  
   - Note **`created`** timestamp on the **new** row (critical for race proof).

3. **Confirm approvals not blaming:**  
   - Ensure no pending approval still assigned to old user if symptom mixes review + approvals.

4. **Inspect ES document `TEMPLATE_{contract_id}`:**  
   - If `assigned_to_list` still shows old user for **LEGAL_REVIEW** while step 2 is correct → **stale index**.

5. **Correlate timing (Groundcover + BQ):**  
   - Find `re_sync_contract_search_object` / bulk update **success** log time for `{contract_id}`.  
   - Compare to `ManualTaskData.created` for the **new** row.  
   - **Confirmed failure mode:** resync **completed before** the new row’s `created` time → worker indexed prior version (e.g. previous `ManualTaskData` id still pointing at old assignee).

6. **Rule out:**  
   - **Replica lag only** — less likely if `created` is tens of seconds **after** resync end; timestamp ordering is the strong proof.  
   - **Client cache** — hard refresh / incognito; if ES still wrong, not cache alone.

---

## Architectural flow (django-rest-api)

| Step | Component |
|------|-----------|
| Manual task reassignment | New `ManualTaskData` row; `is_latest` moves forward on task chain. |
| Contract save / signals | `re_sync_contract_search_object.delay(contract_id=...)` from `contracts_v3.signals` on `ContractV3` post_save. |
| Build search document | `BulkUpdateUseCase.execute` → `build_search_object_use_case` from contract domain model. |
| Pending manual tasks in document | `ManualTaskService.get_pending_tasks_for_manual_task` (retries logged as `Retry N for get_pending_tasks_for_manual_task`). |
| Index write | `contract_search_v2_repo.bulk_update` → Elasticsearch `contracts` index. |

**Key files:** `contracts_v3/tasks.py` (`re_sync_contract_search_object`), `contracts_v3/search_v2/domain/use_cases/bulk_update_use_case.py`, `contracts_v3/services/manual_task_service.py` (`get_pending_tasks_for_manual_task`), `contracts_v3/signals.py`, `contracts_v3/search_v2/data/elastic_repo.py`, `contracts_v3/contract_assigned_to/domain/use_cases/compute_contract_assigned_to.py`.

---

## How to confirm or disprove the hypothesis

| If true | Evidence |
|---------|----------|
| **ES stale after race** | Latest `ManualTaskData` correct; ES `assigned_to_list` wrong; resync log timestamp **before** new row `created`. |
| **Not this** | ES matches DB; or DB still wrong. |
| **Different issue** | No resync in window; then look for failed tasks, different contract id, or workspace/cluster mismatch. |

---

## Mitigation and workaround

1. **Operational:** Trigger **`re_sync_contract_search_object`** for `{contract_id}` (Celery / runbook used in production).  
2. **Customer:** Ask user to **refresh** task list after index updates.  
3. **Verify:** Re-read ES document `TEMPLATE_{contract_id}` — `LEGAL_REVIEW` assignee matches latest `ManualTaskData`.

**Does not require** changing DB rows if DB is already correct.

---

## Escalation triggers

- Repeated occurrences on same workspace → product fix: ensure resync runs **after** `ManualTaskData` commit (transaction hook, post-commit task, or debounced coalescing).  
- `BulkIndexError` or resync failures in logs → infra / ES cluster issue, not this race pattern.

---

## Related incidents and links

- **Slack:** `https://spotdraft.slack.com/archives/C0AJJEU7G84` (incident channel; login required)  
- **Rootly:** [2937 — Touch ’n Go reassigned review task still visible](https://rootly.com/account/incidents/2937-touch-n-go-reassigned-review-task-still-visible-in-original-user-s-task-list)  
- **Jira:** [SPD-41912](https://spotdraft.atlassian.net/browse/SPD-41912)

**Confirmed example values (for regression testing only):**  
`workspace_id=178717`, `contract_id=360819`, `parent_manual_task_id=50950`, `ManualTaskData` ids `134526` → `134527`, org users `88664` (Akhir) → `88631` (Chen), EU cluster.

---

## Connections to other skills

- **`expiry-report-missing-contracts`** — different symptom, but shared theme of **Elasticsearch lag** vs reporting; mitigation mindset similar (resync / wait for index).  
- **`incident-lookup`** — use Slack search to find other `#incident-*` threads on reassignment + search.  
- **`fullstory-bulk-delete-metadata-value`** — mentions Elasticsearch resync via `partially_resync_contracts_task` for metadata-driven fields; same **index freshness** family.
