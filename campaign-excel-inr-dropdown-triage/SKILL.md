---
name: campaign-excel-inr-dropdown-triage
description: "Triage campaign contract creation failures when bulk-upload Excel has dropdown fields whose options include the Indian rupee symbol (₹). Use when: customer cannot create campaign contracts; validation errors on ExpertFee or similar currency-looking dropdowns; values pasted or selected in Excel lose the ₹ symbol; 'dropdown value does not match' after upload. Trigger phrases: campaign upload failed, ExpertFee IncludingGST Excluding GST error, rupee symbol missing after dropdown select, Excel campaign data invalid option, bulk upload campaign IN cluster. Not for: integration questionnaire values cleared on edit (questionnaire-hidden-variable-data-loss), template Edit Questionnaire stale prefill (edit-template-questionnaire-contractdata-split), HubSpot mass metadata cleanup (hubspot-sfdc-conditional-overpersistence)."
---

# Campaign Excel: INR (₹) in Dropdown Options vs Excel Cell Formatting

## What This Skill Debugs

Customers **cannot create campaign contracts** from a **bulk-upload Excel** when **questionnaire dropdown** options include the **₹ (INR)** character in the **label/value** (e.g. `₹42.37`). **Microsoft Excel** often treats those cells as **currency** while editing: it **stores or displays only the numeric part**, so the **saved cell value no longer exactly matches** the allowed dropdown option string — campaign ingestion / validation then fails.

**Observed customer pattern (CRED, March 2026):** Required and optional **ExpertFee** dropdowns; user **copy-paste** from Excel and/or **pick from in-sheet data validation**; after selection, **re-opening the cell shows numbers without ₹** — consistent with Excel currency behavior. **Reproduced in dev:** ₹ omitted on select or paste when Excel applies currency formatting. **Google Sheets** in the same scenario **retained ₹** as string (per engineering thread).

**Known example (do not hardcode in new tickets):** WSID `9712`, cluster **IN**, campaign id `5565`, `CampaignDataUpload` admin id `10357` — [SPD-42501](https://spotdraft.atlassian.net/browse/SPD-42501), [Rootly #3011](https://rootly.com/account/incidents/3011-cred-unable-to-create-campaign), Slack `#incident-20260318-high-cred-unable-to-create-campaign` (`C0AMQCGF4EM`).

---

## When to Use / Not Use

| Use this skill | Use a different skill |
|----------------|------------------------|
| **Campaign** bulk upload; failure at **create / process** step; **dropdown** fields with **₹** in options | **questionnaire-hidden-variable-data-loss** — values lost after **questionnaire edit** on integration contracts |
| Excel-specific: symbol **drops** after edit/select; works in **Google Sheets** | **hubspot-sfdc-conditional-overpersistence** — wrong persistence / cleanup from HubSpot SFDC widget |
| **ExpertFee**-style **numeric + currency symbol** in **dropdown option text** | **edit-template-questionnaire-contractdata-split** — template **Edit Template Questionnaire** shows stale values while key card is correct |

---

## Expected Behavior vs Failure Mode

- **Questionnaire config:** Dropdown options are defined as **strings** (value + label can both include ₹ from product side).
- **Excel behavior:** If Excel interprets the column/cell as **Currency**, **editing** may **normalize** to numeric display/storage → **substring match to full option string fails**.
- **UI nuance (web):** Support observed that after choosing `₹42.37` from dropdown, **clicking the field again** showed **only the number** — treat as **signal** to check **Excel formatting** and **option strings**, not only “user pasted wrong.”

---

## Investigation Workflow

### 1. Confirm scope

- [ ] **Workspace** `{workspace_id}` / WSID and **cluster** `{cluster}` (example: IN).
- [ ] **Campaign** id `{campaign_id}` and link to app campaign page.
- [ ] **Failing fields** — names (e.g. ExpertFee Excluding GST, ExpertFee Including GST) and whether **required**.
- [ ] **Upload artifact** — customer Excel vs generated template; **Excel vs Google Sheets**.

### 2. Reproduce or validate the mismatch

- [ ] Open the **CampaignDataUpload** row in Django admin (filter by `campaign_id`):
  `https://api.{cluster}.spotdraft.com/admin/campaigns_v3/campaigndataupload/{campaigndataupload_id}/change/?_changelist_filters=campaign_id%3D{campaign_id}`
- [ ] Inspect **dropdown option strings** in **Questionnaire V2** for the template (same patterns as other skills): do options **include ₹**?
- [ ] In Excel: check **cell format** for affected columns — **Currency** vs **Text**. Select from **data validation** list, save, re-open cell — is ₹ **gone**?

### 3. Classify

| Finding | Likely classification |
|--------|----------------------|
| Options contain ₹; Excel column is Currency; numeric-only after edit | **Excel formatting / currency normalization** — apply mitigations below |
| Options are plain numbers; still failing | **Different root cause** — expand logs / BQ / Groundcover for campaign pipeline |
| Only fails after **questionnaire edit** on created contracts | Route to **questionnaire-hidden-variable-data-loss** if integration + hidden questions |

---

## Mitigation & Customer Communication

**Immediate unblock (customer-safe, from incident):**

1. **Unprotect** the sheet if protected (customer workflow-dependent).
2. Select affected cells → **Format → Format Cells → Text** (or equivalent).
3. **Re-select** values from the **dropdown** (do not rely on pasted currency-looking strings without re-picking).

**Alternative mitigations (trade-offs):**

- **Remove ₹** from **questionnaire dropdown option** text for new uploads — **may break** other **in-flight or historical** Excel files that still use the old option strings; coordinate with customer before bulk changing options.
- **Temporarily mark** the blocking field **not required** (if product/process allows) to allow campaign run — use only with Legal/Ops approval.

**Permanent fix direction (engineering):** Ensure **campaign Excel download** sets affected columns to **string/text** formatting (not currency) when the column is a **dropdown** whose options include currency symbols — so Excel does not coerce on edit. Track via [SPD-42501](https://spotdraft.atlassian.net/browse/SPD-42501) / pod backlog.

---

## Available MCP Tools

| Tool | Use |
|------|-----|
| **Slack** | Search incident channel or `slack_search_public` for “ExpertFee”, “campaign”, “rupee”, “dropdown” |
| **Jira** | [SPD-42501](https://spotdraft.atlassian.net/browse/SPD-42501); search duplicates for **campaign** + **dropdown** + **currency** |
| **BigQuery / Groundcover** | Campaign processing errors, upload validation — use standard workspace/campaign id filters when logs are available (no incident-specific query captured in-channel) |

---

## Escalation Triggers

- Mitigation steps **do not** unblock after **Text** format + re-select — collect **exact error string**, **row/column**, and **sample cell value** vs **canonical option list** from admin.
- Same symptom **without** ₹ in options — likely **different** validation bug; escalate with repro file.
- **Regression** after a deploy touching **campaign Excel generation** or **questionnaire dropdown** serialization — link deployment window.

---

## Related Links

| Artifact | Link |
|----------|------|
| Jira | [SPD-42501](https://spotdraft.atlassian.net/browse/SPD-42501) |
| Rootly | [Incident #3011](https://rootly.com/account/incidents/3011-cred-unable-to-create-campaign) |
| Slack incident | `C0AMQCGF4EM` |

---

## Connections to Other Skills

- **questionnaire-hidden-variable-data-loss** — Post-creation **questionnaire edit** data loss; not the same as **upload-time** string mismatch.
- **contract-lookup** — If investigation pivots to **contract** state after a campaign contract is partially created.
