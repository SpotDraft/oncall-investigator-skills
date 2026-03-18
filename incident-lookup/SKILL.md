---
name: incident-lookup
description: "Find past incidents similar to a current issue and surface their resolutions. Use this skill when someone reports a problem and you need to check if it has happened before, find a prior resolution, or identify if it's a known regression. Trigger on phrases like 'has this happened before', 'similar incident', 'known issue', 'past resolution', 'regression', 'prior fix', or when investigating any issue where historical context would help. Also use when someone asks about the status of a known bug or previously reported problem."
---

# Incident Lookup & Historical Resolution

The fastest way to resolve many support issues is to find a prior incident with the same symptoms. SpotDraft uses Rootly for incident management, which creates dedicated #incident-* channels in Slack.

## Available MCP Tools

### Slack Search (primary tool for incident lookup)
Use `slack_search_public` to find past incidents and their resolutions across #oncall and #incident-* channels.

### Jira Search (for bug tracking)
Use `searchJiraIssuesUsingJql` to find related bug reports and fix status. Get Cloud ID first via `getAccessibleAtlassianResources`.

### GCP Cloud Logging (validate if the same error pattern is currently occurring)

**✅ Prod logs are accessible** via `resourceNames: ["projects/spotdraft-prod"]`. Always check prod when the incident involves prod workspaces.

**⚠️ Key difference — Prod uses `jsonPayload`, QA uses `textPayload`:**
- QA: `textPayload:"{error_message}"`
- Prod: `jsonPayload.message:"{error_message}"` with `logName="projects/spotdraft-prod/logs/stderr"`

**Prod cluster names:** `prod-usa`, `prod-india` *(EU cluster name not yet confirmed)*

**Check if a specific error is occurring in prod RIGHT NOW:**
```
logName="projects/spotdraft-prod/logs/stderr"
resource.labels.cluster_name="prod-usa"
jsonPayload.message:"{error_message}"
severity >= ERROR
```

**Always add a time window for prod (very high volume):**
```
logName="projects/spotdraft-prod/logs/stderr"
resource.labels.cluster_name="prod-usa"
jsonPayload.message:"{error_message}"
timestamp >= "2026-03-03T02:00:00Z"
timestamp <= "2026-03-03T04:00:00Z"
```

**QA: Check if a specific error message is occurring now:**
```
resource.type="k8s_container" labels."k8s-pod/app"="spotdraft-qa-django-app" textPayload:("{error_message}") severity >= ERROR
```

**Check Cloud SQL for database-related issues matching past incidents:**
```
resource.type="cloudsql_database" logName="projects/spotdraft-qa/logs/cloudsql.googleapis.com%2Fpostgres.log" textPayload:("{error_keyword}") severity >= WARNING
```

**CRITICAL — Ask AI / VerifAI failing (indexing loop) — QA EU cluster:**

The key failure is NOT a 4xx on the access log. The `indexing_status` API returns HTTP 200 but internally is calling the metadata extraction service which 404s. Use `logName="projects/spotdraft-qa/logs/stderr"` (worker logs), not `stdout` (nginx access logs):

```
logName="projects/spotdraft-qa/logs/stderr"
resource.labels.cluster_name="qa"
resource.labels.namespace_name="qa"
jsonPayload.message=~"(Index not found|IndexingStatusPendingException|indexing_status|document_indexing_report)"
```

The exact log sequence to look for (all under the same `task_id`):
1. `CANNONICAL_LOG_LINE ... url=https://metadata-extraction-app-*.run.app/document_indexing_report?document_id={wsid}_{contract_id}_{version_id} response_code=404` — confirms the metadata extraction service has no index for this contract
2. `WARNING: Index not found for contract {X} and contract_version {Y} considering the indexing status as PENDING` — emitted by `openai_metadata_extraction_adapter`
3. `Contract version indexing status is PENDING for contract {X}` — task logic reports pending
4. `Trying to fetch the contract version indexing status {N}/3 times` — shows retry loop
5. `retry: Retry in Xs: IndexingStatusPendingException()` — backoff kicks in (3s → 12s → 51s)

If you see this pattern, the root cause is **`ON_DEMAND_CONTRACT_INDEXING` feature flag is OFF for the workspace**. Resolution: enable it in admin at `api.eu.qa.spotdraft.com/admin/sd_organizations/enhancedflag/?q=ON_DEMAND`.

### BigQuery Django Models (validate contract state for a specific incident)

**✅ MCP has access to both `spotdraft-qa` AND `spotdraft-prod` projects.** Earlier documentation saying prod BQ is inaccessible was incorrect.

> **⚠️ Run BQ queries one at a time** — parallel `execute_sql` calls in the same message all return 500 errors.

**Check contract state for a QA contract involved in an incident:**
```sql
SELECT
  c.id, c.status, c.contract_kind, c.workflow_status,
  c.created_by_workspace_id, c.created, c.modified
FROM `spotdraft-qa.qa_india_public.contracts_v3_contractv3` c
WHERE c.id = {contract_id}
```

**Check version history for version mismatch incidents:**
```sql
SELECT cv.version_number, cv.sub_version_number, cv.action, cv.is_current, cv.is_deleted, cv.created
FROM `spotdraft-qa.qa_india_public.contracts_v3_contractversion` cv
WHERE cv.contract_id = {contract_id}
ORDER BY cv.version_number DESC, cv.sub_version_number DESC
LIMIT 15
```

### BigQuery Operational Logs (validate error frequency/patterns)

Use `projectId: "spotdraft-qa"`.

**Check if a specific task is failing (matching a past DLQ-related incident):**
```sql
SELECT task_name, COUNT(*) as failures, MIN(created) as first, MAX(created) as last
FROM `spotdraft-qa.dlq_qa_india.dlq`
WHERE created >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND task_name LIKE '%{task_keyword}%'
GROUP BY task_name
ORDER BY failures DESC
```

**Check if a specific API endpoint is erroring:**
```sql
SELECT request_path, response_status, COUNT(*) as count
FROM `spotdraft-qa.request_response_logs_qa_india.logs`
WHERE inserted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND request_path LIKE '%{endpoint_keyword}%'
  AND response_status >= 400
GROUP BY request_path, response_status
ORDER BY count DESC
```

### Environment-to-Dataset Mapping

| Environment | BQ Project | Dataset | MCP Access |
|-------------|-----------|---------|------------|
| QA India | `spotdraft-qa` | `qa_india_public` | ✅ Yes |
| QA EU | `spotdraft-qa` | `qa_eu_public` | ✅ Yes |
| QA USA | `spotdraft-qa` | `qa_usa_public` | ✅ Yes |
| Dev India | `spotdraft-qa` | `dev_india_public` | ✅ Yes |
| **Prod India** | `spotdraft-prod` | `prod_india_db` | ✅ Yes |
| **Prod EU** | `spotdraft-prod` | `prod_eu_db` | ✅ Yes |
| **Prod USA** | `spotdraft-prod` | `prod_usa_db` | ✅ Yes |
| **Prod MEA** | `spotdraft-prod` | `prod_mea_db` | ✅ Yes |

> Prod tables have a `public_` prefix, e.g. `prod_usa_db.public_contracts_v3_contractv3`. QA/Dev have no prefix.

## How to Search for Past Incidents

### Step 1: Search Slack by Symptoms

Search #oncall and #incident-* channels for key symptoms:

**Search strategies:**
- Exact error message (e.g., "Word server raised an error")
- Affected feature area (e.g., "signing OTP", "approval reset", "signature fields")
- Customer name if it's a recurring issue
- Contract type or workflow if relevant

**Useful Slack search patterns:**
- `error message in:#oncall` — find discussions of the error
- `customer_name in:#oncall` — find all issues for a customer
- `feature_keyword after:2025-12-01` — find recent incidents
- Look in `#incident-*` channels — Rootly creates these with descriptive names

### Step 2: Validate Current Occurrence in GCP/BigQuery

Once you find a matching past incident, use GCP Cloud Logging and BigQuery to verify:
1. Is the same error pattern currently occurring?
2. How frequently is it happening?
3. Which workspaces/contracts are affected?
4. For QA contracts: query `qa_india_public` Django model tables to check current state

This gives a much more informed answer than just "we've seen this before."

### Step 3: Check Jira for Bug Reports

Search Jira for related bug reports:
- Use error message or feature area as search terms
- Check SPD project for open bugs
- Look at recently resolved tickets

**Key Jira fields:** Status, Fix version, Comments (often contain workarounds).

### Step 4: Cross-Reference with Known Regressions

**Signature fields retained after editing (Feb 2026 regression)**
- Affected: Allen, Chargebee, Stedi, Rain, ADP, Amagi, others
- Workaround: Manually remove signature fields during editing

**Contract stuck in signing — ConvertAPI 500 (Jun 2025)**
- Seen in QA: contract 413213, WS 8498
- Symptom: version history shows alternating EXECUTION_PDF/UPLOADED_TPP_PDF with 100+ retry rows
- Jira: SPD-33571

**Approval reset issues**
- Often caused by workflow manager version mismatch

**Cloud SQL CPU spikes (recurring)**
- Very frequent in prod-eu-india and prod-us-east4
- Usually self-resolving; check Query Insights if sustained >15 min
- Feb 2026: prod-eu-india had multiple CPU spikes (0.85-0.97) on same day
- Correlate with DLQ task storms: `execute_auto_follow_contract_on_domain_event_use_case` and `automations` tasks are high-volume and can cause spikes

## Presenting Results

When you find a matching past incident:
1. **Link to the incident channel** for full context
2. **Summarize the root cause**
3. **Share the resolution** — what fixed it last time
4. **Show current GCP/BQ data** — confirm whether the same pattern is happening now (use `qa_india_public` Django model tables for QA contract state)
5. **Link to the Jira ticket** with current status

If no matching incident found:
1. Show GCP/BQ data for the current error to help characterize it
2. Suggest this may be a new issue
3. Recommend creating a Rootly incident if severity warrants it
4. Suggest next diagnostic steps using other skills

## Common Resolution Patterns

| Symptom | Likely Resolution | Past Incident |
|---------|------------------|---------------|
| OTP not received (Outlook) | Domain whitelisting | BIAL Jan 2026, Ather Feb 2026 |
| Email suppressed | Reactivate on Postmark | Visit app Dec 2025 |
| Signature fields retained | Manual removal workaround | Allen Feb 2026 (regression) |
| Word server error | Re-upload clean document | Multiple incidents |
| Approval reset | Check WF manager version | Noon Feb 2026 |
| ConvertAPI 500 on execute | Retry or check specific document | QA 413213, SPD-33571 |
| Cloud SQL CPU spike | Check Query Insights, check DLQ for task storms | Recurring (Feb 2026 prod-eu-india) |
| Questionnaire date field error | Check computation formula | SpotDraft Core WS-2633 Feb 2026 |
| Obligations extraction failed | Check SD_OBLIGATIONS feature flag | Sunsure Energy Feb 2026 |
| Duplicate key pointer warning | Check metadata field labels | ETRO Feb 2026, WS 301084 |
| Ask AI loads indefinitely → "Technical hiccup" error | Enable `ON_DEMAND_CONTRACT_INDEXING` FF for the workspace | NIUM QA Feb 2026 (Incident #2908) |
| VerifAI guide-runs returns 404 in DD RUM, no backend error logs | Same as above — legacy Firebase path triggered because `ON_DEMAND_CONTRACT_INDEXING` is OFF | NIUM QA Feb 2026 |
| Tray "Get contract download link" → 404 / "Current contract version not found" on CLICKWRAP contracts | Race condition — `ContractVersion.is_current` not yet `TRUE` when Tray webhook fires. Not credentials. Workaround: retry. Fix: delay webhook or add retry in `download_link`. See SPD-39109. | Abnormal Security US (WSID 147206) + Cumulocity EU (WSID 139789), Mar 2026 |
