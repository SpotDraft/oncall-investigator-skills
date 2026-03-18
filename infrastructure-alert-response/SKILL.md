---
name: infrastructure-alert-response
description: "Respond to infrastructure alerts from GCP Cloud Monitoring and related systems. Use this skill when automated alerts fire in #oncall for: Cloud SQL CPU >85%, postgres oldest transaction age >1000, django-app-worker-kill events, GKE node issues, nginx worker kills, high memory usage, or any Google Cloud Monitoring alert. Also trigger on phrases like 'Cloud SQL alert', 'CPU spike', 'database alert', 'worker killed', 'GKE node', 'postgres transaction', 'infra alert', or any automated monitoring notification."
---

# Infrastructure Alert Response

These alerts come from GCP Cloud Monitoring and typically appear as automated messages in #oncall. Most have runbooks — the key is identifying which runbook to follow and whether the alert requires immediate action.

## Available MCP Tools for Investigation

You have direct access to GCP Cloud Logging and BigQuery for the `spotdraft-qa` project. Use these programmatically BEFORE telling people to go check dashboards manually.

### GCP Cloud Logging (MCP tool: `list_log_entries`)

Use `list_log_entries` with `resourceNames: ["projects/spotdraft-qa"]` and these filters:

**Cloud SQL Postgres Errors:**
```
resource.type="cloudsql_database" logName="projects/spotdraft-qa/logs/cloudsql.googleapis.com%2Fpostgres.log" severity >= WARNING
```

**Django App Errors (500s, exceptions):**
```
resource.type="k8s_container" labels."k8s-pod/app"="spotdraft-qa-django-app" severity >= ERROR
```

**Worker kills / SIGTERM / OOM:**
```
resource.type="k8s_container" labels."k8s-pod/app"="spotdraft-qa-django-app" textPayload:("OOMKilled" OR "worker killed" OR "SIGTERM" OR "MemoryError")
```

**Nginx upstream timeouts / 502s:**
```
resource.type="k8s_container" resource.labels.namespace_name="kube-system" textPayload:("upstream timed out" OR "502")
```

**GKE cluster events (pod evictions, node issues):**
```
logName="projects/spotdraft-qa/logs/events" resource.type="k8s_cluster"
```

**Cloud SQL audit activity (connection failures, instance state):**
```
resource.type="cloudsql_database" logName="projects/spotdraft-qa/logs/cloudaudit.googleapis.com%2Factivity" severity >= ERROR
```

Always set `orderBy: "timestamp desc"` and `pageSize: 10` to get recent entries first.

### BigQuery (MCP tool: `execute_sql`)

Use `projectId: "spotdraft-qa"` for all queries. **MCP only has access to `spotdraft-qa`; prod data is not queryable.**

**Check DLQ for failed tasks (potential cause of CPU spikes):**
```sql
SELECT task_name, COUNT(*) as failure_count, MIN(created) as first_failure, MAX(created) as last_failure
FROM `spotdraft-qa.dlq_qa_india.dlq`
WHERE created >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)
GROUP BY task_name
ORDER BY failure_count DESC
LIMIT 20
```

**Known high-volume DLQ tasks in QA (normal background noise — don't over-alarm on these):**
- `core.tasks.send_frontend_amplitude_event` — up to 30K+ failures/3 days (analytics events, non-critical)
- `contracts_v3.search_v2.tasks.partially_resync_contracts_task` — 14K+ failures/3 days (search sync backlog)
- `contracts_v3.process_metrics...` — 12K+ failures/3 days (metrics webhooks)
- `integrations_v2.tasks.native_integration_reverse_sync_task` — 10K+ failures/3 days (integration sync)
- `automations.automation...execute_automation_manager_use_case` — 10K+ failures/3 days
- `djcelery_email_send_multiple` — 4K+ failures/3 days (escalate if spiking abnormally)

**Check request/response logs for slow endpoints contributing to load:**
```sql
SELECT request_path, response_status, COUNT(*) as count, AVG(time_taken) as avg_time_ms
FROM `spotdraft-qa.request_response_logs_qa_india.logs`
WHERE inserted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)
  AND time_taken > 5000
GROUP BY request_path, response_status
ORDER BY count DESC
LIMIT 20
```

**Check DLQ for a specific workspace driving load:**
```sql
SELECT task_name, COUNT(*) as failures, MIN(created) as first_failure, MAX(created) as last_failure
FROM `spotdraft-qa.dlq_qa_india.dlq`
WHERE created >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)
  AND workspace_id = {wsid}
GROUP BY task_name
ORDER BY failures DESC
LIMIT 15
```

**DLQ datasets by cluster:**
- QA India: `spotdraft-qa.dlq_qa_india.dlq`
- QA Europe: `spotdraft-qa.dlq_qa_europe.dlq`
- QA USA: `spotdraft-qa.dlq_qa_usa.dlq`
- Dev India: `spotdraft-qa.dlq_dev_india.dlq`

### BigQuery Django Models — Contract State During Incidents

For infra alerts that correlate with specific contracts or workspaces, query the synced Django model tables:

**Check contract state for a workspace (e.g., to understand what operations were running during a CPU spike):**
```sql
SELECT c.id, c.status, c.contract_kind, c.workflow_status, c.modified
FROM `spotdraft-qa.qa_india_public.contracts_v3_contractv3` c
WHERE c.created_by_workspace_id = {wsid}
  AND c.modified >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 2 HOUR)
ORDER BY c.modified DESC
LIMIT 20
```

### Environment-to-Dataset Mapping

| Environment | BQ Project | Dataset | MCP Access |
|-------------|-----------|---------|------------|
| QA India | `spotdraft-qa` | `qa_india_public` | ✅ Yes |
| QA EU | `spotdraft-qa` | `qa_eu_public` | ✅ Yes |
| QA USA | `spotdraft-qa` | `qa_usa_public` | ✅ Yes |
| Dev India | `spotdraft-qa` | `dev_india_public` | ✅ Yes |
| **Prod IN/EU/US** | `spotdraft-prod` | `prod_{region}_db` | ❌ No — prod not accessible |
| **Prod MEA** | `spotdraft-prod` | `prod_mea_db` | ❌ No — prod not accessible |

## Alert Types & Response

### Cloud SQL CPU >85%

**What it means:** Database CPU utilization has exceeded 85% threshold. This is the most common infra alert.

**Automated investigation steps (run these immediately):**

1. **Check Cloud SQL postgres logs for recent errors:**
   Use `list_log_entries` with filter:
   ```
   resource.type="cloudsql_database" logName="projects/spotdraft-qa/logs/cloudsql.googleapis.com%2Fpostgres.log" severity >= WARNING
   ```
   Look for: long-running queries, lock contention, `could not register pglogical worker` (background worker slots exhausted), deadlocks.

2. **Check DLQ for task storms that might be hammering the DB:**
   ```sql
   SELECT task_name, COUNT(*) as failures, MIN(created) as first_failure, MAX(created) as last_failure
   FROM `spotdraft-qa.dlq_qa_india.dlq`
   WHERE created >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 MINUTE)
   GROUP BY task_name
   ORDER BY failures DESC
   LIMIT 10
   ```

3. **Check for slow API endpoints overwhelming the DB:**
   ```sql
   SELECT request_path, COUNT(*) as requests, AVG(time_taken) as avg_ms, MAX(time_taken) as max_ms
   FROM `spotdraft-qa.request_response_logs_qa_india.logs`
   WHERE inserted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 MINUTE)
     AND time_taken > 2000
   GROUP BY request_path
   ORDER BY requests DESC
   LIMIT 10
   ```

4. **Check for connection failures (instance not running):**
   Use `list_log_entries` with filter:
   ```
   resource.type="cloudsql_database" logName="projects/spotdraft-qa/logs/cloudaudit.googleapis.com%2Factivity" protoPayload.status.code!=0
   ```
   Look for: "Instance is not in EXPECTED_RUNNING state", authentication failures.

**Known patterns from QA:**
- `dev--india` instance frequently shows `could not register pglogical worker: all background worker slots are already used` — this indicates replication slot exhaustion, not necessarily a customer-facing issue.
- `qa-psql-us-east4` sometimes shows "Instance not in EXPECTED_RUNNING state" — this is a stopped QA instance, not an alert to act on.
- Duplicate key violations (e.g., `full_txt_metadata_uniq_per_contract_ver`) indicate race conditions in background tasks.
- High DLQ volume for `execute_auto_follow_contract_on_domain_event_use_case` (422K+ for a single QA workspace) can drive DB load.

**When to escalate:**
- CPU stays >90% for more than 15 minutes
- Query Insights shows queries from core product flows (not batch jobs)
- Multiple Cloud SQL instances affected simultaneously

### Postgres Oldest Transaction Age >1000

**What it means:** A database transaction has been open for an unusually long time. Long-running transactions prevent vacuum from reclaiming space and can cause performance degradation.

**Automated investigation:**

1. **Check postgres logs for long-running queries:**
   Use `list_log_entries` with filter:
   ```
   resource.type="cloudsql_database" logName="projects/spotdraft-qa/logs/cloudsql.googleapis.com%2Fpostgres.log" textPayload:("duration" OR "idle in transaction" OR "deadlock")
   ```

2. **If Django shell is available, run:**
   ```sql
   SELECT pid, now() - xact_start AS duration, query, state
   FROM pg_stat_activity
   WHERE state != 'idle'
   ORDER BY xact_start ASC;
   ```

**When to kill:**
- Transaction running >30 minutes with no clear legitimate purpose
- Transaction in "idle in transaction" state
- Database performance degrading

**When to escalate:**
- Transaction from a migration or deployment — coordinate with deploying engineer
- Multiple long transactions appearing simultaneously

### Django App Worker Kill

**What it means:** A Django application worker process was killed, usually due to OOM or timeout.

**Automated investigation:**

1. **Search for SIGTERM/OOM events:**
   Use `list_log_entries` with filter:
   ```
   resource.type="k8s_container" labels."k8s-pod/app"="spotdraft-qa-django-app" textPayload:("SIGTERM" OR "OOMKilled" OR "MemoryError" OR "worker killed")
   ```

2. **Check for correlated nginx upstream timeouts (502s):**
   Use `list_log_entries` with filter:
   ```
   resource.type="k8s_container" resource.labels.namespace_name="kube-system" textPayload:"upstream timed out"
   ```
   Parse the upstream timeout logs — they contain the exact API endpoint that timed out, e.g.:
   `request: "OPTIONS /api/v3/contracts/830593/verifai/guides/results`

3. **Cross-reference with slow requests in BigQuery:**
   ```sql
   SELECT request_path, response_status, time_taken, workspace_id, inserted_at
   FROM `spotdraft-qa.request_response_logs_qa_india.logs`
   WHERE inserted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)
     AND response_status >= 500
   ORDER BY inserted_at DESC
   LIMIT 20
   ```

**Common causes:**
- Large file uploads/exports consuming too much memory
- Memory leak in specific code path (check recent deployments)
- Bulk operations without proper batching

**When to escalate:**
- Workers being killed repeatedly (every few minutes)
- Multiple workers killed across different pods
- Correlates with customer-facing errors (502s, timeouts)

### GKE Node Issues

**What it means:** GKE node problems — node not ready, pod evictions, resource pressure.

**Automated investigation:**

1. **Check GKE events:**
   Use `list_log_entries` with filter:
   ```
   logName="projects/spotdraft-qa/logs/events" resource.type="k8s_cluster"
   ```

2. **Check for cluster autoscaler activity:**
   Use `list_log_entries` with filter:
   ```
   logName="projects/spotdraft-qa/logs/container.googleapis.com%2Fcluster-autoscaler-visibility"
   ```

3. **Check kubelet logs for node issues:**
   Use `list_log_entries` with filter:
   ```
   logName="projects/spotdraft-qa/logs/kubelet" severity >= WARNING
   ```

**When to escalate:**
- Nodes not recovering after 10 minutes
- Pod evictions affecting customer-facing services
- Cluster autoscaler unable to provision new nodes

### Nginx Worker Kills / 502 Errors

**What it means:** Nginx returning 502 Bad Gateway, indicating upstream Django workers are unavailable.

**Automated investigation:**

1. **Find which endpoints are timing out:**
   Use `list_log_entries` with filter:
   ```
   resource.type="k8s_container" resource.labels.namespace_name="kube-system" textPayload:"upstream timed out"
   ```
   The log entries contain the full request path — extract it to identify which API endpoints are affected.

2. **Check if Django workers are restarting:**
   Use `list_log_entries` with filter:
   ```
   resource.type="k8s_container" labels."k8s-pod/app"="spotdraft-qa-django-app" textPayload:("SIGTERM" OR "Booting worker" OR "Starting gunicorn")
   ```

## General Alert Response Protocol

1. **Run automated queries** — Use the GCP and BigQuery tools above to gather data BEFORE asking humans to check dashboards
2. **Summarize findings** — Present the query results: what errors are occurring, which endpoints are affected, what tasks are failing
3. **Assess severity** — Is this affecting customers? Cross-reference with error reports in #oncall
4. **Recommend action** — Based on the patterns found, suggest specific next steps
5. **Document** — Share the queries run and results in the thread

## Key GCP Resources in spotdraft-qa

| Resource | Log Name Pattern |
|----------|-----------------|
| Django App | `logs/stderr`, `logs/stdout` with label `k8s-pod/app=spotdraft-qa-django-app` |
| Cloud SQL (postgres) | `logs/cloudsql.googleapis.com%2Fpostgres.log` |
| Cloud SQL (audit) | `logs/cloudaudit.googleapis.com%2Factivity` |
| Nginx Ingress | `logs/stderr` in namespace `kube-system` |
| GKE Events | `logs/events` |
| Kubelet | `logs/kubelet` |
| Cluster Autoscaler | `logs/container.googleapis.com%2Fcluster-autoscaler-visibility` |

## GKE Clusters in spotdraft-qa

- `qa-india` (asia-south1-a) — primary QA cluster
- Pod naming: `spotdraft-qa-django-app-{hash}`, `spotdraft-qa-django-app-deffered-tasks-{hash}`

## Django Shell Access

For alerts requiring database investigation beyond what logs provide:
1. Request Django shell access via Tray.ai bot in #oncall
2. Access is time-limited (typically 30 minutes)
3. Always document commands run and results
4. Revoke access when done
