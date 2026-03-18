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
