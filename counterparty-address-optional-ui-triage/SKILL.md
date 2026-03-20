---
name: counterparty-address-optional-ui-triage
description: "Triage reports that counterparty address fields show mandatory indicators (asterisks) during contract creation or add-counterparty flows, but users can skip or save with partial address. Use when: customer says address step looks required but is skippable; Creator Party Questionnaire has Counterparty Address set to Optional but UI still shows asterisks; confusion between workflow address settings and add-new-counterparty modal; 'Prompt Health style' mandatory-but-skippable address. Trigger phrases: 'asterisk but optional', 'counterparty address mandatory mismatch', 'can skip address with stars', 'Party Information counterparty address', 'optional address still shows required'. Not for: integration questionnaire values disappearing (use questionnaire-hidden-variable-data-loss) or email delivery to counterparty."
---

# Counterparty Address: Optional Config vs Mandatory-Looking UI

## What This Skill Debugs

Customers report that **counterparty address** during contract creation (especially after **Add new counterparty**) shows **required-field markers (asterisks)**, while **validation still allows skipping** the step or saving with incomplete data when the **Creator Party Questionnaire** marks **Party Information → Counterparty Address** as **Optional**.

This is primarily a **product/UX consistency** issue, not a backend outage. Engineering discussion in the Dec 2025–Jan 2026 incident concluded that some behavior was **intentional** (required markers apply to *partial address entry rules*), but the **add-new-counterparty flow** cannot mirror the clearer **“Add address”** pattern used elsewhere—creating confusion and a **platform-level UX backlog** to align copy, banners, and when asterisks appear.

**Known customer example (do not hardcode in new tickets):** Prompt Health, WSID `274685`, cluster US — see [SPD-39313](https://spotdraft.atlassian.net/browse/SPD-39313), [Rootly #2699](https://rootly.com/account/incidents/2699-prompt-health-counterparty-address-marked-as-mandatory-but-skippable), Slack `#incident-20251217-low-prompt-health-counterparty-address-marked-as-mandatory-but` (`C0A466WLML5`).

---

## When to Use / Not Use

| Use this skill | Use a different skill |
|----------------|------------------------|
| UX mismatch: asterisks vs ability to skip / partial save on **counterparty address** | **questionnaire-hidden-variable-data-loss** — integration-sourced questionnaire values cleared on edit |
| Clarifying **two different settings** (workflow vs modal / questionnaire) | **hubspot-sfdc-conditional-overpersistence** — wrong persistence of hidden conditional fields at creation |
| Customer **not blocked** but confused; need talking points and Jira context | **email-delivery-triage** — counterparty not receiving email |

---

## Configuration Surfaces (Separate Behaviors)

On-call should **not assume** one toggle controls everything.

1. **Creator Party Questionnaire → Party Information → Counterparty Address**  
   Controls whether the **counterparty address step** in the flow is **optional vs mandatory** (as described in support tickets and [SPD-39313](https://spotdraft.atlassian.net/browse/SPD-39313)).

2. **Workflow / contract-creation settings for “address on the contract”**  
   Engineers called out that **workflow-manager address options** concern **whether counterparty address is needed on the contract**, which is **not the same** as validation inside the **add new counterparty** modal flow. Customers may change one setting and expect the other UI to change — explain the distinction before filing duplicate bugs.

**Support-side check:** Confirm with the customer **which screen** (add CP modal after name/email vs main questionnaire vs workflow settings) and **current Optional/Mandatory** for **Counterparty Address** in the Creator Party Questionnaire.

---

## Expected Product Behavior (From Engineering Thread)

Summarized from incident discussion (treat as **behavioral context**, not a spec):

- **Address component rule:** If the user **chooses to enter an address**, at least **three fields** (e.g. country, street, city) are expected complete; **state** and **zip** may be optional for jurisdictions that do not use them.
- **When address is optional at workflow/questionnaire level:** The user may **skip** the entire address step — so **Save/Continue** cannot be disabled solely because address is empty.
- **Gap:** The UI can still show **asterisks** on fields as if the whole step were mandatory, which **conflicts** with “optional step” mental model.
- **Add-new-CP flow nuance:** After **Add new CP → Next**, user lands on the address page with **optional** address; **save cannot be disabled** for empty address, but **partially filled** address may still be savable — flagged as a **product/design** decision (thread with FE/Design, Jan 2026).

---

## Investigation Workflow

### 1. Confirm the report

- [ ] **Workspace** `{workspace_id}` / WSID and **cluster** `{cluster}`.
- [ ] **Repro path:** Add new counterparty vs existing CP vs questionnaire-only.
- [ ] **Screenshot or recording:** Do asterisks appear on **all** fields or only when user starts typing?

### 2. Confirm configuration

- [ ] In **Creator Party Questionnaire**, **Party Information → Counterparty Address**: **Optional** vs **Mandatory**.
- [ ] If customer changed **workflow** address-related settings, explain they may **not** change add-CP modal behavior (see “Configuration surfaces” above).

### 3. Classify the issue

| Observation | Likely classification |
|-------------|----------------------|
| Optional in questionnaire, can skip entirely, asterisks still show | **Known UX inconsistency** — track product/UX fix; mitigate with explanation |
| Mandatory in questionnaire but user can skip | **Possible bug** — escalate with repro; dev repro worth doing |
| Customer expects workflow toggle to change add-CP modal | **Configuration misunderstanding** — education, not engineering |

### 4. Data integrity (only if customer claims wrong CP records)

If the claim is **“counterparty saved without address and that broke something else”**, use **contract-lookup** (and related DB/admin tools there) to verify **contract** and **counterparty** records for `{contract_id}` — not covered step-by-step here.

---

## Available MCP Tools

| Tool | Use |
|------|-----|
| **Slack** | Search this incident channel or `slack_search_public` for “counterparty address”, “asterisk”, “optional” |
| **Jira** | `SPD-39313` pattern; search for duplicates with same symptom |
| **BigQuery / Groundcover** | **No primary signal** for this UX class unless investigating a **separate** data bug (use **contract-lookup**) |

**Admin URLs (when you already have `template_id`):**  
Question items for a template are reachable under Django admin questionnaire v2 (same pattern as other skills), e.g. `https://api.{cluster}.spotdraft.com/admin/questionnaire_v2/questionitemv2/?template_id={template_id}` — use only when debugging questionnaire structure, not as first step for “asterisk” complaints.

---

## Mitigation & Customer Communication

- **Severity:** Typically **low** for incident purposes — user can complete flow; issue is **trust/clarity** of UI ([Rootly](https://rootly.com/account/incidents/2699-prompt-health-counterparty-address-marked-as-mandatory-but-skippable) mitigated on that basis).
- **Explain:** Optional step means **skipping is allowed**; asterisks may reflect **“if you fill address, these lines must be complete”** rather than “you must open this step.”
- **Product follow-up:** UX improvements discussed in-channel: **inline info banner** near modal footer when optional, **copy change** for questionnaire dropdown (e.g. clarifying **optional + hidden** vs “Hidden” alone), **uniform platform** change (may take longer — see Jan 2026 eng update in incident thread).

---

## Escalation Triggers

- **Mandatory** questionnaire setting but users **reproducibly** skip with no validation → **file/escalate as functional bug** with WSID, cluster, template/workflow IDs, recording.
- Customer **blocked** on contract creation (cannot proceed at all) → treat as **higher severity**; verify network errors, permissions, or a different root cause outside this skill.

---

## Related Links

| Artifact | Link |
|----------|------|
| Jira | [SPD-39313](https://spotdraft.atlassian.net/browse/SPD-39313) (example; status was **Mitigated** as of skill authoring) |
| Rootly | [Incident #2699](https://rootly.com/account/incidents/2699-prompt-health-counterparty-address-marked-as-mandatory-but-skippable) |
| Slack incident | `C0A466WLML5` |

---

## Connections to Other Skills

- **contract-lookup** — Resolve `{workspace_id}`, `{contract_id}`, counterparty records when the issue involves **data** not just **UI**.
- **questionnaire-hidden-variable-data-loss** — Questionnaire **values wiped** on edit; different mechanism.
- **incident-lookup** — If routing a new incident, search for duplicate “counterparty address optional” threads.
