---
name: onlyoffice-preview-left-clipping
description: "Diagnose OnlyOffice preview issues where the left side of a DOCX is clipped or offset in SpotDraft preview, while downloaded DOCX/PDF renders correctly. Use when contract preview or workflow template preview is broken only inside OnlyOffice, especially for specific templates that become readable after a DOCX -> PDF -> DOCX round trip. Trigger phrases: OnlyOffice left section not visible, preview clipped, downloaded document looks fine, WOPI viewer issue, template preview broken, round-trip docx fix."
---

# OnlyOffice Preview Left Clipping

## What this skill debugs

This skill covers **OnlyOffice rendering / viewport issues** where:

- the **left side of the document is clipped or invisible** in SpotDraft preview
- the same file looks correct when **downloaded**
- the issue reproduces both in **contract preview** and **workflow template preview**
- a **round-trip conversion** (`DOCX -> PDF -> DOCX`) fixes the rendering

This is usually not a “document corruption” issue in the normal Word sense. The file can be readable and downloadable while OnlyOffice renders it badly inside the embedded viewer.

---

## When to use / when not to use

**Use when:**

- Customer reports preview is broken in OnlyOffice but the downloaded file looks correct
- The issue reproduces on the workflow template preview as well as on contracts created from that template
- Re-saving / round-tripping the DOCX fixes the preview

**Do not use for:**

- Generic DOCX upload rejection (`Invalid DOCX file`) before contract creation
- PDF viewer issues
- Inline editing failures where the file itself is not opening at all

---

## First identifiers to collect

- `{workspace_id}`
- `{cluster}`
- `{contract_id}`
- `{contract_version_id}`
- affected `{workflow_id}` / template ids
- whether the problem appears in:
  - workflow template preview
  - contract preview
  - both

If the download is correct but OnlyOffice preview is wrong, this skill is the right entry point.

---

## Relevant code paths

### Contract preview path

- `angular-frontend/apps/spotdraft-fe/src/app/contract-render/contract-detail-v2/contract-detail-v2.component.html`
- `angular-frontend/apps/spotdraft-fe/src/app/contract-render/contract-detail-v2/contract-detail-v2.component.ts`
- `angular-frontend/apps/spotdraft-fe/src/app/contract-render/only-office-viewer/only-office-viewer.component.ts`

The contract page chooses `WOPI_OO_VIEW` and mounts `app-only-office-viewer`.

### Workflow template preview path

- `angular-frontend/apps/spotdraft-fe/src/app/docx-template/docx-template-preview/docx-template-preview.component.ts`
- `angular-frontend/apps/spotdraft-fe/src/app/docx-template/docx-template-preview/docx-template-preview.component.html`
- `angular-frontend/apps/spotdraft-fe/src/app/onlyoffice/onlyoffice-editor/onlyoffice-editor.component.ts`

The template preview explicitly uses `OnlyOfficeZoomLevel.FIT_TO_WIDTH`.

### Important implementation nuance

In the March 2026 Amagi incident, the investigation pointed to a likely mismatch between:

- contract preview viewer config / embedded WOPI behavior
- template preview editor config with explicit zoom handling

This makes **zoom / viewport parity** a useful debugging angle, even when there is no backend error.

---

## Investigation workflow

### Step 1: Prove it is preview-only

Confirm all three:

1. OnlyOffice preview is broken
2. Downloaded DOCX or PDF looks correct
3. The issue reproduces on the workflow template as well as affected contracts

If all three are true, prioritize **OnlyOffice rendering / normalization**, not backend contract data.

### Step 2: Check whether the issue is template-family-wide

Ask whether contracts created from the same workflow/template show the same left-clipping behavior.

If yes:

- treat the **base template** as the primary artifact to fix
- expect existing contracts created from that template family to need backfill or re-upload of their latest versions

### Step 3: Use the round-trip mitigation

The proven mitigation from the Amagi incident was:

1. upload the DOCX to Google Docs or Word
2. export / convert to PDF
3. convert back to DOCX
4. upload the round-tripped DOCX as the new base template

This normalizes the document into a form OnlyOffice renders correctly.

If the workflow preview is fixed after this round trip, the root cause is strongly tied to **OnlyOffice compatibility with the original DOCX structure**, not business data.

### Step 4: Backfill affected active contracts

After fixing the base workflow/template, identify already-created contracts from the affected workflow that have not reached terminal / excluded states and update their latest versions with the normalized document.

Operational notes from the incident:

- scope contracts by workspace + affected workflow ids
- exclude contracts already in signing / completed states if the mitigation plan says so
- update the latest `ContractVersion` docx/pdf to the round-tripped files

The incident thread used a bulk script pattern to:

- enumerate affected contracts
- download the latest DOCX
- convert `DOCX -> PDF -> DOCX`
- upload the normalized assets back to storage
- repoint the current `ContractVersion`

Treat any bulk script as high-risk:

- dry-run first
- verify contract kind
- confirm whether editable / WOPI-backed documents could desync if only `ContractVersion` is updated

### Step 5: Check zoom / viewer parity if mitigation must become productized

If engineering is turning this into a permanent fix, compare:

- contract preview `app-only-office-viewer`
- template preview `app-onlyoffice-editor`
- `initialZoom`
- `FIT_TO_WIDTH`
- other OnlyOffice customization / editor config differences

This is where rendering parity bugs tend to hide.

---

## How to confirm / disprove

| Finding | Interpretation |
|---------|----------------|
| Downloaded file is correct, OnlyOffice preview is clipped | Viewer / renderer issue, not core document content loss |
| Same issue on workflow preview and contracts from that workflow | Base template family is affected |
| Round-tripped DOCX renders correctly | Original DOCX shape is incompatible with OnlyOffice rendering path |
| No backend / telemetry error but visible UI issue persists | Still treat as real; this can be UI-only |
| PDF path looks fine while DOCX preview is wrong | Narrow further to OnlyOffice / DOCX embedding path |

---

## Mitigation / workaround

### Immediate unblock

- use the round-trip normalization flow (`DOCX -> PDF -> DOCX`)
- replace the base workflow/template DOCX with the normalized version
- if needed, update latest versions for already-created affected contracts

### Business continuity fallback

If the contract can be consumed via a correct download while engineering works on a permanent fix, use that as the temporary customer-facing workaround. The incident thread also treated PDF / download correctness as a safe fallback while OnlyOffice rendering remained the problem.

---

## Escalation triggers

- The issue reproduces across multiple customers / templates without a shared source DOCX pattern
- Round-trip normalization does **not** fix the problem
- The same contract family breaks again after template replacement
- Backfill would require touching many live contracts and contract kinds are mixed / unsafe for bulk update

Escalate to the owning pod when the mitigation needs code-level normalization or viewer-config parity work.

---

## Related incident

| Reference | Notes |
|-----------|--------|
| `#incident-20260325-medium-amagi-contract-preview-issue-in-onlyoffice-left-section-not-visible` | Source incident for this runbook |
| `SPD-42771` | Jira linked in incident |
| `SPD-42173` | Related OnlyOffice / view-divergence theme referenced during investigation |
| `SPD-37333` | Historical OnlyOffice blank-screen instability reference |

---

## Permanent fix directions

- unify or review zoom / customization parity between contract preview and template preview OnlyOffice integrations
- add a QA fixture for “preview clipped on left side” DOCX layouts
- consider a document-normalization step for templates that OnlyOffice renders incorrectly but Word/download handles fine
- add safer tooling / scripting guidance for backfilling already-created contracts from a bad template family
