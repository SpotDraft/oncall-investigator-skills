---
name: word-cross-reference-numbering-triage
description: "Triage wrong or stale clause/section reference numbers, REF/SEQ fields, or cross-references in template contracts in SpotDraft (OnlyOffice): numbers look wrong on first open but fix after F9 or 'update fields'; legal says template is correct; DOCX vs PDF signing mismatch. Use when: reference numbers not working, numbering wrong until refresh, Word field codes, cross-reference not updating, template agreement numbering. Trigger phrases: F9 fixes references, select all update fields, wrong reference number contract, Product Agreement numbering, workflow template refs."
---

# Word Cross-Reference & Numbering Field Triage

## What this skill debugs

Issues where **numbered clauses, section references, or Word field-based cross-references** appear **incorrect** in the contract editor (OnlyOffice) or in **generated outputs**, even when Legal believes the **template DOCX** is correct.

**Incident pattern (evidence-backed):** Numbers are wrong when the contract is **first opened**, but become **correct after** the user edits, selects all text, and presses **F9** (Word / OnlyOffice **Update Fields**). That strongly suggests **fields not refreshed in the client view** rather than bad data permanently stored — confirm against **downloaded DOCX / PDF** and backend artifacts.

**Related but different:** **DOCX → PDF** (e.g. for signing) can **change** how references resolve because conversion often **forces Word to update fields**, **reflow** layout, and refresh cross-references — numbers may **differ** between in-editor DOCX and exported PDF **without** a single “buggy” number being stored.

---

## When to use / when not to use

**Use when:**

- Customer reports **reference numbers** or **clause numbering** “wrong” in SpotDraft
- Support reproduces **fix after F9** or full-document **Update Fields**
- Discussion touches **template** `workflow_id`, Legal-updated numbering, **PDF vs DOCX** for send/sign

**Do not use for:**

- **OnlyOffice list numbering** glitches on **uploaded** PDF/TPP (no Word fields) → **pdf-upload-visual-artifacts**
- **Computation / metadata** driving display numbers → **contract-computation-null-propagation**
- **Litera compare** failures → **litera-version-compare-triage**

---

## System flow (high level)

```
Template DOCX (Word fields: SEQ, REF, STYLEREF, …)
  → cf-exporter / templater merges variables → Contract version DOCX
  → OnlyOffice loads DOCX → user sees numbers (fields may be stale until updated)
  → Optional: export / convert to PDF for email or signing → field update + reflow may change appearance
```

**Useful identifiers:**

| Placeholder | Use |
|-------------|-----|
| `{contract_id}` | Affected contract(s) |
| `{workflow_id}` | Template workflow (e.g. admin workflow change URL) |
| `{template_id}` | Contract template if distinct from workflow |
| `{cluster}` | `us` / `in` / `eu` for admin hosts |

---

## Investigation workflow

### Step 1: Reproduce the “F9” signal

1. Open the contract in the **editor** as the customer does.
2. Note **clause / REF** values **before** any bulk selection.
3. **Select all** → **F9** (or OnlyOffice equivalent **Update fields**).
4. If numbers **snap to correct** → treat as **field refresh / display pipeline**, not necessarily wrong merge data.

Document in ticket: Loom or screenshots **before/after** (incident used [Loom](https://www.loom.com/share/b51525b7c2814219ac8be97e41d0f995) — use as format example only).

### Step 2: Compare artifacts (backend vs UI)

1. **Download** current contract **DOCX** from SpotDraft (admin or product flow).
2. Open in **desktop Word** → **File → Info → Edit links** / right-click fields → **Update field** if needed.
3. Compare to **what OnlyOffice showed** before F9.
4. If Word-on-desktop matches **post-F9** view → UI refresh issue; if Word also wrong → template or merge issue.

> Rootly mitigation for one production incident stated **references are correct in the backend** with a **frontend display / field-refresh** angle — **verify** per contract before asserting.

### Step 3: PDF / signing path (Sanket’s note from incident thread)

When customer uses **PDF** for signing or distribution:

- **DOCX → PDF** conversion often causes Word to **update fields**, **reflow** text, change **margins**, and refresh **cross-references** — reference **numbers can change** vs in-editor DOCX.
- Customer may **refuse PDF** (e.g. partners **export PDF → Word**); CS should align with **Legal** and **product** on **tradeoffs** — there is **no** silent “always identical numbering” guarantee across DOCX and PDF when fields are involved.

### Step 4: Template / Legal ops

1. Confirm **which template version** is live (`workflow_id` / republish).
2. Legal Ops may need to **simplify numbering** (e.g. reduce fragile REF chains) or adjust how **SEQ/REF** are used so refresh order matters less.
3. Action item from incident: check whether **references can be refreshed** inside Word at template authoring time; **connect Legal Ops** for template mitigation.

### Step 5: Logs & BQ (optional)

- This class of issue is **rarely** confirmed via application **error logs** alone.
- Use **contract-lookup** / BQ on `contracts_v3_contractversion` to confirm **version**, `docx_version`, timestamps — not to prove field staleness.

---

## Mitigation options (pick with customer + Legal)

| Option | Tradeoff |
|--------|----------|
| User **Update fields** (F9) after open | Training / UX friction |
| **PDF** path for signing / external share | Numbering may **differ** from DOCX; customer may object to PDF |
| **Template change** (Legal) | Time; may need workflow **republish** |
| **Engineering fix** (display refresh) | Pod **editor / OnlyOffice** — if confirmed stale fields on load |

**CS note from thread:** Avoid promising **hacky** workarounds; prefer clear **vendor/editor behavior** + **Legal template** path when PDF is rejected.

---

## Confirm / disprove

| Observation | Interpretation |
|-------------|----------------|
| F9 fixes in editor | Stale / not-auto-updated Word fields in viewer |
| Desktop Word matches post-F9 | Same; not random corruption |
| PDF differs from DOCX | Expected with field update + reflow (not always a bug) |
| Wrong in PDF and Word after manual update | Template or merge bug → escalate engineering |
| Only one workspace / one workflow | Template-specific |

---

## Escalation triggers

- **Many** workflows broken after a **release** → regression in exporter or OnlyOffice integration → **#pod-editor** (or owning pod).
- **Single customer**, F9 pattern, backend file correct → **mitigation + template**; permanent **auto-refresh** is a **feature** request.
- Customer **blocked** on PDF policy → PM / CS + Legal, not L2 alone.

---

## Related incidents & Jira

| Reference | Notes |
|-----------|--------|
| **SPD-41146** | Reference numbers not working as expected — **Mitigated** / Done. |
| **Rootly #2870** | [incident channel](https://slack.com/app_redirect?channel=C0AF7SH52UV&team=T23C8AV7T) — `#incident-20260216-medium-reference-numbers-in-the-contract-is-not-working-as-exp` |
| **Customer (Jira)** | **EBG** (`customfield_10032` on SPD-41146) |
| **Example IDs** (do not hardcode in new tickets) | Workflow **156** (Product Agreement), contracts **1354997**, **1380771**, **1380744** |
| **Sanket context** | [Shared note](https://share.google/aimode/piiO8BM0vDvgpc5gl) on why **docx templater** struggles with this class of problem — use when explaining limits to stakeholders |

---

## Connections to other skills

- **contract-lookup** — `{contract_id}` → versions, template, workspace.
- **pdf-upload-visual-artifacts** — Numbering **visual** issues on **uploaded** PDFs / TPP / Chrome Print — different entry point.
- **contract-signing-triage** — If failure is **cannot generate PDF** or sign task errors, not wrong clause numbers.

---

## Operational note

> Run **BigQuery** queries **one at a time** if using BQ MCP (repo README — parallel calls can 500).
