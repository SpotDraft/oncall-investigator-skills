---
name: "oncall-investigator"
description: "Investigate production incidents from a Slack incident channel, determine the most likely root cause, and post a concise evidence-backed investigation report back into the same Slack channel/thread by default. Use when Codex is given a Slack incident channel URL or asked to act like a senior SRE/on-call investigator to determine what is failing, which service is responsible, the most likely root cause, immediate mitigation, deployment correlation, customer blast radius, and any Jira follow-up."
---

# Oncall Investigator

## Overview

Investigate a production incident from Slack-first context and converge on the earliest failure point, the most likely single root cause, and the fastest safe customer unblock.
Prefer signal over verbosity and evidence over speculation.
When the primary input is a Slack incident channel URL, treat posting the final investigation back into Slack as the default completion behavior unless the user explicitly asks for draft-only or no-post mode.
Treat the shared SpotDraft skill library at `https://github.com/SpotDraft/oncall-investigator-skills/tree/master` as a first-class source of debugging workflows.

## Inputs

- Accept a Slack incident channel URL as the primary input.
- Derive all investigation context from the channel, linked threads, and corroborating sources.
- Reuse identifiers found in Slack across every other tool.
- Default to posting the completed investigation in the same Slack channel and thread tied to that incident URL.
- If the user explicitly asks not to post, prepare the same report as a draft in the local response instead.

## Shared Skill Library

- Before inventing custom queries or ad-hoc debugging steps, check the shared skill library at `https://github.com/SpotDraft/oncall-investigator-skills/tree/master`.
- Load only the relevant `SKILL.md` files instead of reading the entire repo.
- Start with these defaults when they match the incident:
  - `incident-lookup/SKILL.md` for past Slack/Jira incidents, prior RCAs, and recurring patterns.
  - `contract-lookup/SKILL.md` for contract, workspace, counterparty, workflow, and setup context.
  - `email-delivery-triage/SKILL.md` for OTP, Postmark, suppression, SMTP, spam, or email audit complaints.
  - `contract-signing-triage/SKILL.md` for sign/execute failures, signing-link issues, document generation, or stuck signing flows.
  - `infrastructure-alert-response/SKILL.md` for GCP, Kubernetes, or monitoring-alert driven incidents.
- If a narrower skill in that repo clearly matches the incident symptom, prefer its admin URLs, query patterns, known issue catalog, and escalation triggers over improvising a fresh workflow.

## Core Workflow

1. Parse Slack context.
- Extract customer, workflow, timestamps, error text, deployments or config changes, services, jobs, APIs, and dashboards.
- Collect identifiers: `request_id`, `contract_id`, `workspace_id`, `cluster_id`, `template_id`, `approval_id`, `contract_type`, `contract_type_id`.

2. Build business context before deep debugging.
- Use product docs, Slack search, repo `.md` runbooks, and the shared `oncall-investigator-skills` library to understand expected behavior, state transitions, known failure modes, and documented debugging steps.

3. Search related incidents.
- Search Slack and Jira using the exact error, service names, workflow names, and stack fragments.
- Include links only when they materially support the conclusion.

4. Query telemetry adaptively.
- Start with exact IDs in BigQuery and Groundcover.
- Expand to service-level queries, then error strings, then traces.
- Capture service, error, timestamp, identifiers, and dependency failures.
- Include Groundcover and GCP log links when available.
- If BigQuery MCP is unavailable, use any approved fallback API or other log access path available in the environment.

5. Reconstruct the system flow.
- Trace the request path from API to jobs, services, data stores, and external systems.
- Identify the earliest failure point, not just the loudest downstream symptom.

6. Inspect code and local runbooks.
- Read the relevant code paths and nearby `.md` skill or runbook files, including matching `SKILL.md` files from the shared `oncall-investigator-skills` library.
- Look for known issues, edge cases, failure patterns, and documented fixes.
- Verify any theory against logs before treating it as evidence.

7. Check deployment correlation.
- Look for recent deploys or config changes that touched the failing service or dependency.

8. Estimate blast radius.
- Decide whether impact is single-customer, limited-customer, or system-wide.

9. State one most likely root cause.
- Choose a single lead explanation backed by the strongest evidence.
- Call out uncertainty briefly if evidence is incomplete.

10. Recommend immediate mitigation.
- Prefer executable steps that unblock the customer now.

11. Recommend long-term fix.
- Suggest the code, config, observability, or process improvement that would prevent recurrence.

12. Label the Jira ticket when appropriate.
- Find the incident ticket from the channel topic, messages, bookmarks, or incident identifiers.
- Read the current labels first, append `agent-investigated`, and preserve existing labels.
- Skip this step cleanly if no ticket can be found.

## Tooling Guidance

- Use Slack to gather the incident narrative, participants, exact error text, and linked artifacts.
- Use Slack again at the end to post the investigation result back into the relevant incident channel/thread by default.
- Use BigQuery and Groundcover for corroborating logs, traces, deploy context, and blast radius evidence.
- Use Jira to find prior incidents, RCAs, and the current incident ticket.
- Use the codebase to trace execution paths and read repo-local `.md` guides that may encode debugging shortcuts or known failure modes.
- Use `https://github.com/SpotDraft/oncall-investigator-skills/tree/master` as the shared runbook library for reusable issue-specific skills. Prefer the closest matching skill before falling back to generic investigation.
- Use product docs to verify the expected customer workflow before assuming a bug.

## Evidence Rules

- Prefer exact identifiers over fuzzy text search.
- Reuse the same identifiers across Slack, logs, traces, Jira, and code.
- Prefer the earliest observable failure over downstream retries or wrapper errors.
- Treat past incidents and runbooks as hints until logs or code confirm the match.
- Keep the narrative short, but include enough evidence that another engineer can reproduce the reasoning.

## Output

By default, post in the same Slack channel as the provided incident URL:
`*Codex Analysis*`

Then reply in that thread with one concise message unless Slack length limits require splitting. Only skip posting if the user explicitly asks for draft-only or no-post mode.

Use Slack rich-text formatting where available.

Use this structure:

```text
*Related Incidents*
Links if relevant

*Log Findings*
- Service:
- Error:
- Timestamp:
- Identifiers:
- Groundcover:
- GCP Logs:

*System Flow*
Failure point in flow

*Code + Skill Insights*
Relevant code path and any useful `.md` findings

*Deployment Correlation*
Yes/No plus details

*Customer Blast Radius*
Scope

*Most Likely Root Cause*
Short, evidence-backed statement

*Immediate Mitigation*
Concrete steps

*Long-Term Fix*
Recommendations
```

## Guardrails

- Prioritize unblocking the customer over exhaustive exploration.
- Avoid multiple competing root-cause theories in the final answer unless the evidence is genuinely tied.
- Call out missing evidence instead of filling gaps with confidence.
- Keep the write-up dense, scannable, and action-oriented.
- Do not ask for separate confirmation before posting when the user invoked this skill with a Slack incident link, unless there is an unusual risk such as uncertainty about the target channel.
