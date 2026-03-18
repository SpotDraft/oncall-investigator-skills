# OnCall Investigator Skills

A collection of [Cursor Agent Skills](https://docs.cursor.com) for SpotDraft's autonomous on-call incident investigation workflow. These skills give the Cursor AI agent deep, domain-specific knowledge to investigate production incidents, triage support issues, and respond to infrastructure alerts — without requiring a human to manually check dashboards.

## Overview

The repository is structured around a central **OnCall Investigator** command (`.cursor/commands/oncall-investigator.md`) that orchestrates an autonomous, 11-step investigation workflow. The individual skill files act as specialized runbooks the agent loads on demand to handle specific issue categories.

```
oncall-investigator-skills/
├── .cursor/
│   └── commands/
│       └── oncall-investigator.md       # Main agent prompt & investigation workflow
├── contract-lookup/
│   └── SKILL.md                         # Contract & workspace context lookup
├── contract-signing-triage/
│   └── SKILL.md                         # Signing, execution & document generation errors
├── email-delivery-triage/
│   └── SKILL.md                         # Email & OTP delivery failures
├── incident-lookup/
│   └── SKILL.md                         # Past incident search & resolution lookup
└── infrastructure-alert-response/
    └── SKILL.md                         # GCP Cloud Monitoring alert response
```

## How It Works

The **OnCall Investigator** is triggered with a Slack incident channel URL. It then autonomously:

1. Parses the Slack channel for customer, error, and identifier context
2. Looks up business domain knowledge (help docs, internal runbooks)
3. Searches past incidents in Slack and Jira
4. Queries BigQuery and GCP Cloud Logging for log evidence
5. Reconstructs the system flow to find the earliest failure
6. Correlates with recent deployments
7. Determines the blast radius
8. Outputs a structured root cause analysis with mitigation steps — posted directly to the Slack thread

### MCP Tools Used

| Tool | Purpose |
|------|---------|
| **Slack MCP** | Read incident channels, search for past incidents |
| **BigQuery MCP** | Query GCP application logs, Django model tables, DLQ, and request/response logs |
| **Groundcover MCP** | Access service logs, traces, and infrastructure telemetry |
| **Atlassian MCP (Jira)** | Search past incident RCAs and bug reports |

---

## Skills

### `contract-lookup`
Look up contract details, workspace configuration, version history, signature setup state, and approval state. This is typically the **first skill invoked** in any triage workflow — other skills depend on it for baseline context.

- Queries BigQuery Django model tables (`contracts_v3_contractv3`, `contracts_v3_contractversion`, etc.)
- Covers all environments: QA India/EU/USA, Dev India, Prod India/EU/USA/MEA
- Includes ready-to-use SQL templates for common lookups
- Documents the CLICKWRAP + Tray `download_link` race condition (SPD-39109, Mar 2026)

### `contract-signing-triage`
Diagnose signing, execution, and document generation failures.

**Triggers:** "Word server raised an error", signature fields retained, contract stuck in signing, unable to execute, preview failure, version mismatch, ConvertAPI 500.

- Queries contract state, version history, signature setup, and approvals from BigQuery
- Searches GCP logs for Word server, ConvertAPI, signing, and execution errors
- Checks DLQ for failed async tasks (`merge_documents_task`, `document_conversion_task`)
- Includes a correlation table mapping BQ/log/DLQ signals to likely root causes
- Documents known regressions: signature fields retained (Feb 2026), ConvertAPI 500 (SPD-33571)

### `email-delivery-triage`
Diagnose email and OTP delivery failures for counterparties.

**Triggers:** "email not received", "OTP not triggered", "signing link not delivered", email suppressed, Postmark issues, Outlook/Exchange problems.

- Queries DLQ for email-related task failures (`djcelery_email_send_multiple`, `bounced_email_audit_use_case`)
- Checks access control logs for auth/email failures
- Covers the full diagnostic flow: email audit logs → Postmark suppression check → domain whitelisting
- Documents OTP emergency workaround (retrieve from Django shell via Tray.ai)
- References past incidents: BIAL (Jan 2026), Ather Energy (Feb 2026), Visit app (Dec 2025)

### `incident-lookup`
Find past incidents matching current symptoms and surface their resolutions.

**Triggers:** "has this happened before", "known issue", "regression", "prior fix", investigating any issue where historical context helps.

- Searches Slack `#oncall` and `#incident-*` channels by error message, feature area, or customer name
- Validates current occurrence using GCP logs and BigQuery
- Searches Jira (SPD project) for related bug reports and fix status
- Includes a resolution table of 15+ known issue patterns with links to past incidents
- Documents the Ask AI / VerifAI indexing loop pattern and `ON_DEMAND_CONTRACT_INDEXING` feature flag fix

### `infrastructure-alert-response`
Respond to automated GCP Cloud Monitoring alerts.

**Triggers:** Cloud SQL CPU >85%, postgres oldest transaction age >1000, django-app-worker-kill, GKE node issues, nginx worker kills, 502 errors.

- Provides ready-to-use GCP log filter queries for each alert type (Cloud SQL, OOM, nginx, GKE)
- Queries DLQ for task storms driving DB load and request/response logs for slow endpoints
- Documents known QA background noise (e.g., `core.tasks.send_frontend_amplitude_event` — 30K+ DLQ failures/3 days is normal)
- Covers escalation thresholds for each alert type
- Includes Django shell access instructions via Tray.ai for deeper DB investigation

---

## Environment Reference

| Environment | BQ Project | Dataset | Table Prefix |
|-------------|-----------|---------|--------------|
| QA India | `spotdraft-qa` | `qa_india_public` | *(none)* |
| QA EU | `spotdraft-qa` | `qa_eu_public` | *(none)* |
| QA USA | `spotdraft-qa` | `qa_usa_public` | *(none)* |
| Dev India | `spotdraft-qa` | `dev_india_public` | *(none)* |
| Prod India | `spotdraft-prod` | `prod_india_db` | `public_` |
| Prod EU | `spotdraft-prod` | `prod_eu_db` | `public_` |
| Prod USA | `spotdraft-prod` | `prod_usa_db` | `public_` |
| Prod MEA | `spotdraft-prod` | `prod_mea_db` | *(none)* |

> **Note:** Prod tables have a `public_` prefix (e.g. `prod_usa_db.public_contracts_v3_contractv3`). QA/Dev tables have no prefix.

> **BigQuery parallel query bug:** Do NOT fire multiple `execute_sql` calls in the same message — all return 500 errors. Always run BQ queries sequentially.

---

## Adding New Skills

1. Create a new directory named after the skill (e.g., `approval-triage/`)
2. Add a `SKILL.md` file with a YAML front matter block (`name`, `description`) followed by the runbook content
3. The `description` field is used by the agent to decide when to invoke the skill — make it specific and include trigger phrases
4. Reference BigQuery queries, GCP log filters, and admin panel URLs relevant to the skill
5. Document any known issue patterns, past incidents, and escalation triggers
