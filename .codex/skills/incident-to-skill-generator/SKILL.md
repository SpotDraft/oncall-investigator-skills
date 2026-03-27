---
name: "incident-to-skill-generator"
description: "Convert a resolved Slack incident into a reusable on-call debugging skill for the `oncall-investigator-skills` repo. Use when Codex is given a resolved incident channel URL and should decide whether to create a new skill, update an existing skill, or skip because the issue is too one-off; gather evidence from Slack, Jira, BigQuery, Groundcover, product docs, code paths, and existing repo skills; then write the full `SKILL.md`, publish it, and notify the same incident channel unless the user asks for draft-only mode."
---

# Incident To Skill Generator

Convert a resolved incident into a reusable debugging skill for future on-call investigators.
Produce a complete `SKILL.md`, not an RCA recap, and prefer future reuse over incident narration.

## Use When

- The main input is a resolved Slack incident channel URL.
- The user wants a new or updated skill in `/Users/adityagupta/Projects/oncall-investigator-skills`.
- The goal is to preserve a repeatable debugging workflow, not just summarize what happened.

## Do Not Use When

- The incident is still live and needs active triage rather than documentation.
- The issue is purely procedural, customer-specific, or otherwise too one-off to help a future engineer.
- The user only wants an RCA, status update, or bug summary.

## Required Inputs

- A resolved Slack incident channel URL.
- Optional hints: customer, workspace, cluster, Jira key, suspected skill overlap, or whether to run in draft-only mode.

## Source Of Truth

Read `/Users/adityagupta/Projects/oncall-investigator-skills/.cursor/commands/skill-generator.md` first and treat it as the behavioral contract for this skill.

Also inspect:

- Existing `*/SKILL.md` files in `/Users/adityagupta/Projects/oncall-investigator-skills`
- Relevant code paths in the local codebase
- Product docs at `https://help.spotdraft.com/`

Match the style, density, and operational usefulness of the existing incident skills in that repo.

## Available Tools

- Slack MCP:
  - Read the incident channel and all relevant threads.
  - Search for similar past incidents and related discussions.
  - Post the completion message back to the same incident channel unless the user asked for draft-only mode.
- Jira MCP:
  - Find bugs, fixes, regressions, RCAs, and related implementation tickets.
- BigQuery MCP:
  - Validate state transitions, request history, DLQ evidence, and other structured debugging signals.
  - Run queries sequentially when using the SpotDraft BigQuery setup; parallel `execute_sql` calls are known to fail.
- Groundcover MCP:
  - Inspect logs, traces, workloads, infra events, and telemetry correlations.
- Codebase:
  - Read the repo skill prompt, existing skills, and relevant application code.
- Product docs:
  - Confirm expected user workflow before assuming a failure mode.

## Core Workflow

1. Parse the incident channel.
- Read the full Slack channel and material threads, not just the opener.
- Extract customer, workspace, cluster, affected workflow, timestamps, identifiers, dashboards, deploy/config references, mitigation, and final resolution.
- Keep the earliest confirmed failure point separate from downstream noise.

2. Understand expected behavior before generalizing.
- Use product docs, code, and existing repo skills to understand the intended workflow, state transitions, dependencies, and likely breakpoints.
- Identify which parts of the incident are product behavior versus environment quirks or operator mistakes.

3. Generalize the incident into a reusable issue class.
- Define the triggering symptoms, likely customer-facing behavior, primary systems, and the first identifiers that unlock investigation.
- Replace incident-specific values with placeholders such as `{contract_id}`, `{workspace_id}`, `{request_id}`, `{approval_id}`, `{template_id}`, `{cluster}`, and `{error_message}`.
- Keep broadly reusable operational quirks, including environment naming, dataset differences, query gotchas, log-format differences, and exact error strings that strongly identify the issue class.

4. Compare against existing skills before writing.
- Read the existing skill set in `/Users/adityagupta/Projects/oncall-investigator-skills`.
- Decide one of:
  - `NEW_SKILL`: a distinct reusable workflow not already covered
  - `UPDATE_EXISTING_SKILL`: an existing skill already covers most of the issue, but this incident adds a valuable new failure mode, query, workaround, Jira reference, or environment quirk
  - `NO_SKILL`: the incident is too one-off or process-specific to justify a reusable skill
- Prefer updating an existing skill over creating a near-duplicate.

5. Gather supporting evidence across systems.
- Slack: search for prior incidents with matching symptoms.
- Jira: check historical bugs, regressions, fixes, and RCA notes.
- BigQuery: confirm state and request-level evidence.
- Groundcover: confirm logs, traces, and service-level symptoms.
- Codebase: find the relevant code path and likely failure point.
- Do not invent queries, root causes, or operational facts that are not supported by evidence.

6. Build the final skill artifact.
- Write a complete `SKILL.md` in the target skill directory.
- Keep the body dense, actionable, and operational.
- Include only sections that materially help future debugging, such as:
  - what the skill debugs
  - when to use and when not to use it
  - environment-to-dataset mapping when relevant
  - investigation workflow
  - exact or adaptable BigQuery, log, or Groundcover queries
  - code paths and architectural flow
  - known issue patterns with examples clearly labeled as examples
  - confirmation and disproof criteria
  - mitigation and workaround guidance
  - escalation triggers
  - related incidents, Jira references, or links to other skills

7. Apply authoring rules.
- Use YAML frontmatter with only `name` and `description`.
- Make `description` invocation-friendly and specific enough that another agent would know when to use the skill.
- Use placeholders in reusable queries and examples.
- Label incident-specific values as examples, not defaults.
- Preserve exact admin URLs, table names, filters, and code paths when they are reusable.

8. Quality check before publishing.
- Confirm the result is reusable for future investigations rather than tied to a single incident chronology.
- Confirm the decision is correct: new skill, update existing, or no skill.
- Confirm the workflow is evidence-backed.
- Confirm the skill does not duplicate existing coverage without a good reason.
- Confirm the file reads like the existing repo skills: concise, dense, and practical under pressure.

## Publishing Workflow

By default, finish the full workflow unless the user explicitly asks for draft-only or no-post mode.

1. Write or update the target `SKILL.md` in `/Users/adityagupta/Projects/oncall-investigator-skills`.
2. Stage only the changed skill file.
3. Commit with:
- `skill: add <skill-name>` for new skills
- `skill: update <skill-name>` for updates
4. Push to `origin master`.
5. Post one Slack message to the same incident channel containing:
- `Skill Created:` or `Skill Updated:`
- the GitHub link to the skill file
- 3-6 short bullets summarizing what the skill covers or what changed
- concise visual polish with scan-friendly formatting

If the user asks for draft-only mode, still produce the same artifact and summary locally, but skip commit, push, and Slack posting.

## Expected Output

Use this structure when returning the result locally:

1. `Decision`
- `NEW_SKILL`, `UPDATE_EXISTING_SKILL`, or `NO_SKILL`

2. `Why`
- Explain the reusable pattern and note any overlapping existing skill.

3. `Skill Scope`
- Summarize issue class, trigger symptoms, primary systems, and the best first checks.

4. `Final Artifact`
- Include the complete final `SKILL.md`.

5. `Published`
- Commit hash
- GitHub link
- Slack notification confirmation

If the decision is `NO_SKILL`, omit publication steps that were intentionally skipped and explain why.

## Guardrails

- Future reuse over chronology
- Signal over noise
- Evidence over assumptions
- Specific operational details over generic advice
- Update existing skills instead of fragmenting coverage
- Never hardcode incident-specific identifiers as if they were reusable defaults
