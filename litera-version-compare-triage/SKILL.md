---
name: litera-version-compare-triage
description: "Triage failures when comparing contract versions in SpotDraft (Litera Compare / UI Compare): infinite loading, HTTP 422, misleading 'Compare Server 9.5.3 or later' UI message, or errors after swapping left/right versions. Use when: version compare broken, Litera UI Compare Failed, comparison document could not be created, DOCX vs PDF pair fails, vendor Litera or Kofax mentioned. Trigger phrases: cannot compare versions, Litera Compare Online, UI Compare, contract version compare, 422 compare, swap versions compare."
---

# Litera Contract Version Compare Triage

## What this skill debugs

SpotDraft’s **contract version comparison** feature calls **Litera Compare** (hosted Compare Server + UI Compare flow). Failures often surface as:

- **422** from Litera with a generic “corrupt or unsupported format” message
- **Browser/UI** showing *“This version of Litera Compare Online requires Litera Compare Server 9.5.3 or later”* — this can appear **even when** the backend server is on a **9.24.x** line; treat it as a **symptom to verify server health and the real API error**, not as proof of an old server version alone
- Compare **works once**, then **breaks after swapping** which version is “original” vs “modified” (customer may retry many times)
- **UICompare** path fails while **non-UI** Compare (e.g. RTF/DOCX output in Litera Swagger) **succeeds** for the same logical content

**Earliest failure point** is usually: **Litera rejecting one or both inputs** (often **PDF-related** or **conversion pipeline / Kofax**), not SpotDraft’s generic “compare” button.

---

## When to use / when not to use

**Use when:**

- Customer cannot compare two versions of `{contract_id}` in production
- Engineering sees `Litera UI Compare Failed` / `422` tied to `ContractVersionsCompare` or Celery `task_id`

**Do not use for:**

- Redlining inside OnlyOffice / editor-only issues → **pdf-upload-visual-artifacts** or editor pod
- Signing or execution failures → **contract-signing-triage**
- No Litera in path (internal diff only) — confirm product behavior first

---

## System flow (high level)

```
User selects two ContractVersions → Django creates ContractVersionsCompare
  → async task → Litera Compare Server (e.g. GCE VM) → UI Compare / Compare API
  → result or 422 / timeout / cancellation
```

**Identifiers to collect first:**

| Placeholder | Use |
|-------------|-----|
| `{contract_id}` | Customer contract |
| `{workspace_id}` | Workspace / cluster routing |
| `{cluster}` | `us` / `in` / `eu` — sets admin host and log project |
| `{comparison_id}` | `ContractVersionsCompare` PK (from admin or logs) |
| `{task_id}` | Celery task UUID from Django logs (if present) |

---

## Investigation workflow

### Step 1: Confirm the comparison record

1. Open Django admin (example US):  
   `https://api.us.spotdraft.com/admin/contracts_v3/contractversionscompare/?q={contract_id}`  
   (Adjust `api.{cluster}.spotdraft.com` for IN/EU.)

2. Open the **failed** row — note:
   - **Original** and **modified** `ContractVersion` IDs
   - **Status** / error fields if exposed
   - Timestamps vs customer report

3. Open each `ContractVersion` and note **`action`** (e.g. `UPLOAD_EDITABLE`, `UPLOADED_PDF`) — **DOCX vs PDF involvement** is a common discriminator.

### Step 2: Reproduce the matrix (Litera behavior)

From incident **SPD-41228** validation (same class of bug):

| Pair | Typical result |
|------|----------------|
| DOCX vs DOCX | **201** / success |
| DOCX vs PDF | **422** (failed in incident) |
| PDF vs PDF | **422** |
| PDF converted to DOCX, then DOCX vs self | **201** |

If your failure **only** involves **PDF** in the pair, prioritize **PDF/input pipeline** and **Litera UICompare** limitations — not “SpotDraft viewer.”

**Direction can matter:** In the same incident, **DOCX → PDF** comparison failed with the customer-visible error; **PDF → DOCX** did not show the same failure mode in internal testing — log **which version is original vs modified**.

### Step 3: GCP logs — Django (k8s)

Search for the comparison id and/or contract id:

- Filter ideas (adapt project/cluster to prod):
  - `jsonPayload.message` or text containing `{comparison_id}` or `Litera UI Compare Failed`
  - `jsonPayload.task_id:"{task_id}"` if known

**Known log line shape (example):**

- `Litera UI Compare Failed for version compare: {comparison_id}`
- `Error: 422 Client Error`

**Example GCP link pattern** (from incident thread — replace timestamps/IDs):  
`https://console.cloud.google.com/logs/query` with query on `k8s_container`, `{contract_id}`, `ERROR`, and `task_id` if available.

### Step 4: GCP logs — Litera (GCE)

Litera may run on a **GCE instance** (example from incident: `litera-usa-9-24`, zone `us-east4-a`).

- `resource.type="gce_instance"`
- `logName` containing `litera` (e.g. `projects/spotdraft-prod/logs/litera`)

Look for **cancellation token** / timeout / 422 correlation around the failure time.

> Confirm current instance name, zone, and log sink with **platform / pod-editor** owners — names can change between envs.

### Step 5: Vendor (Litera) escalation package

When opening or updating a Litera ticket, include:

- Redacted **DOCX + PDF** (as used in compare)
- **Exact API** tested (e.g. `/api/Compare`, UI Compare vs Swagger Compare)
- **422** body if available: *“A comparison document could not be created…”*
- **Litera server version** (example from incident correspondence: **9.24.14**)
- Reproduction: **UICompare fails** but **Compare with RTF/DOCX output in Swagger** works for “same” content → indicates **PDF / UICompare** path issue

**Vendor follow-ups seen in production incidents:**

- Litera may state the engine cannot process the document pair due to **“corrupted content”** and cite **Kofax conversion** — permanent remediation may require **pod-level** investigation (conversion pipeline), not only Litera ticket closure.

---

## Mitigation & customer communication

**Internal / deploy:** Incident **Rootly #2880** was **mitigated** after **fix deployed to production** (plus QA path). On-call can stop tracking once mitigated; **permanent fix** may remain with **product/engineering pod** (e.g. Kofax / conversion).

**Customer workaround (use carefully — incident feedback):**

- Litera suggested: use **original DOCX** if available, or recreate PDF with **Microsoft Print to PDF** and retry compare.
- Internal guidance in the same incident: avoid over-promising “hacky” workarounds; prefer **honest timeline** + **vendor engaged** + optional **convert PDF to Word and compare manually** as a **last-resort**, diplomatic CS message.

---

## How to confirm / rule out

| Finding | Confirms |
|---------|----------|
| 422 + message about comparison document not created + PDF in pair | Litera input/rendering path; align with Litera |
| DOCX-only compare works; any PDF fails | PDF-specific (format, conversion, UICompare) |
| Same error in Litera Swagger UICompare | Not SpotDraft-only |
| Compare works after customer replaces PDF (e.g. Print to PDF) | Source PDF / conversion quality |
| Only happens after swap | Log order of versions; state on server |

---

## Escalation triggers

- **Widespread** compare failure across many contracts / workspaces → **platform outage**, Litera VM capacity, or **bad deploy** — not one bad PDF.
- **Single contract**, matrix matches PDF failure → **vendor + document** path; loop in **pod** if Kofax/conversion suspected.
- Misleading **“9.5.3 or later”** banner with no actual server downgrade → still collect **422** and **Litera logs**; don’t close on UI string alone.

---

## Related incidents & Jira

| Reference | Notes |
|-----------|--------|
| **SPD-41228** | Red Ventures — cannot compare versions; **Mitigated**; WS **326328**, contract **983154** (US). |
| **Rootly #2880** | [incident channel](https://slack.com/app_redirect?channel=C0AFKD8MN2W&team=T23C8AV7T) — `#incident-20260217-medium-red-ventures-cannot-compare-contract-versions` |
| **Debugging notes (example)** | Google Doc linked from incident thread (initial engineering findings) — use as template for future write-ups. |
| **Known comparison id (example only)** | `17439` — **do not hardcode** in new investigations; use admin to find current id. |

---

## Connections to other skills

- **contract-lookup** — Resolve `{contract_id}` versions, `action`, `docx_version` / `pdf_version`, workspace.
- **pdf-upload-visual-artifacts** — If the underlying PDF is malformed (encoding, layers), compare may fail; cross-check PDF health in multiple viewers.

---

## BigQuery (optional)

If `ContractVersionsCompare` (or equivalent) is exposed in your warehouse, prefer **Django admin** first — schema and table names vary. Example pattern only (verify table exists before running):

```sql
-- VERIFY TABLE NAME IN YOUR ENV — placeholder example
SELECT id, contract_id, status, created, modified
FROM `spotdraft-prod.prod_usa_db.public_contracts_v3_contractversionscompare`
WHERE contract_id = {contract_id}
ORDER BY created DESC
LIMIT 20
```

> If the query fails with “table not found,” use admin + GCP logs only.

---

## Operational note

> Run multiple **BigQuery** probes **sequentially** if using BQ MCP — parallel queries can 500 (see repo README).
