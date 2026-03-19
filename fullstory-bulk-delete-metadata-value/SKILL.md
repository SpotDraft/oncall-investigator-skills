---
name: bulk-delete-contract-metadata-values
description: "Diagnose and run bulk delete of contract metadata (key pointer) values. Use when customer or support requests bulk removal of metadata values from multiple contracts, delete SFDC Opp ID / Salesforce Account ID / key pointer values from contracts, or incident involves Fullstory/metadata value deletion. Trigger phrases: bulk delete metadata, delete metadata value from contracts, remove key pointer values, SFDC opportunity ID deletion."
---

# Bulk delete contract metadata (key pointer) values

## What this skill debugs

Customer or support requests to **bulk delete metadata (key pointer) values** from a list of contracts. Typical drivers:

- Remove a specific metadata field value (e.g. Salesforce Opportunity ID, Salesforce Account ID) from many contracts.
- Correct data that was set in error or must be cleared for compliance/privacy.
- One-off cleanup when the product does not expose bulk delete for that field.

This skill covers how to validate the request, extract contract IDs from the provided sheet, run the mitigation script safely, and confirm resolution.

## When to use this skill

- Incident or ticket: "bulk delete metadata value" / "remove metadata from multiple contracts".
- Support provides an Excel/CSV with Contract IDs (and optionally metadata field name or column for reference).
- Customer: Fullstory or any workspace; cluster: US or EU (script runs in that cluster’s Django env).
- You have workspace ID (`wsid`), contract ID list or sheet (e.g. Column K = Contract IDs), and optionally the key pointer name (e.g. "Salesforce Account ID").

## When NOT to use

- Deleting a **single** contract’s metadata → use product UI or single-record flows.
- Deleting **all** metadata for a workspace or template → different tooling/runbook.
- Request is to **update** (not delete) metadata values → use bulk update or import runbooks.
- You do not have a clear list of contract IDs and workspace ID from support/customer.

## Key identifiers

| Identifier   | Source / example                          |
|-------------|--------------------------------------------|
| `{workspace_id}` | Support/customer or Rootly (e.g. 41811)   |
| `{contract_ids}` | Excel Column K or equivalent; must be integers |
| `{cluster}`  | US / EU — determines which Django/DB to run against |
| Metadata field | Key pointer name, e.g. "Salesforce Account ID", "Salesforce Opportunity ID" |

## Step-by-step investigation and mitigation

### 1. Parse the request

- Confirm **workspace ID**, **cluster**, and that the attachment is the **final** sheet (support may send an updated sheet; use the one they explicitly say to use).
- Note which column has Contract IDs (e.g. **Column K**) and whether a second column (e.g. **Column AH** for SF Opportunity ID) is for reference only.
- Confirm the **metadata field** to remove (key pointer name). If the script deletes *all* key pointers for the listed contracts, ensure that matches intent.

### 2. Get contract ID list

- Use the project script to extract contract IDs from the Excel:
  - Script: `scripts/oncall_mitigations/extract_contract_ids_from_excel.py`
  - Config at top: `EXCEL_PATH`, `CONTRACT_ID_COLUMN` (e.g. `"K"`), `HEADER_ROW`, optional `SF_OPPORTUNITY_ID_COLUMN` (e.g. `"AH"`).
  - Run from **project root**:  
    `python scripts/oncall_mitigations/extract_contract_ids_from_excel.py`
  - Copy the printed `CONTRACT_IDS` list (or use generated `contract_ids_output.py`) into the bulk-delete script.

### 3. Run bulk delete script (mitigation)

- Script: `scripts/oncall_mitigations/full_story_bulk_delete_metadata_values.py`
- **Set at top**: `CONTRACT_IDS`, `WORKSPACE_ID`. Ensure `settings.CELERY_TASK_ALWAYS_EAGER = False` so resync runs via Celery (not inline).
- **Dry run first** (no DB writes):
  - In Django shell or script run: `execute()` or `execute(dry_run=True)`.
  - Check batch count, "Would delete X key pointers", "Would resync Y contracts".
- **Live run**: `execute(dry_run=False)`.
- Script behavior:
  - Deletes `ContractKeyPointer` rows for `contract_id__in=CONTRACT_IDS` and `created_by_workspace={WORKSPACE_ID}`.
  - Batches (default `BATCH_SIZE = 100`) with **5 second delay** between batches to avoid overload.
  - After each batch, triggers Elasticsearch resync: `partially_resync_contracts_task.delay(contract_ids=..., fields=KEY_POINTER_DERIVED_SEARCH_FIELDS)`.

### 4. Verify

- Confirm in DB (same workspace) that key pointers for sample contract IDs are gone.
- Optionally confirm search/reporting reflects the change after resync.
- If support provided a "before" sheet, spot-check a few contracts that should no longer have the value.

### 5. Respond and close

- Reply in the incident channel that the script was run (dry run then live), batch size and delay used, and that metadata values were deleted (and ES resync triggered).
- Share the script or a sanitized summary for visibility. Close the incident after verification.

## Code paths and scripts

| What | Where |
|------|--------|
| Extract contract IDs from Excel | `scripts/oncall_mitigations/extract_contract_ids_from_excel.py` |
| Bulk delete key pointers + resync | `scripts/oncall_mitigations/full_story_bulk_delete_metadata_values.py` |
| Model | `contracts_v3.models.ContractKeyPointer` (contract, key_pointer, created_by_workspace, value, …) |
| Resync task | `contracts_v3.search_v2.tasks.partially_resync_contracts_task` |
| Resync fields | `contracts_v3.search_v2.domain.constants.KEY_POINTER_DERIVED_SEARCH_FIELDS` |

## Operational notes

- **Always run dry run first** to avoid accidental bulk deletes.
- **5 second delay** between batches is recommended to avoid overloading DB and Celery.
- **CELERY_TASK_ALWAYS_EAGER = False**: ensure Celery runs the resync task asynchronously; otherwise resync runs inline and can be slow for large batches.
- **Sheet version**: Support may send an updated sheet (e.g. "ignore previous sheet, use Column K for Contract IDs, Column AH for Salesforce Opportunity IDs"). Always use the sheet and columns they specify in the channel.
- **Filtering by key pointer**: The reference script deletes *all* key pointers for the given contracts in the workspace. If the request is to delete only one metadata *field* (e.g. only "Salesforce Account ID"), the script must filter by `key_pointer` (e.g. join to `historic_contracts.KeyPointer` and filter by name/slug). The Fullstory incident used the "all key pointers for these contracts" approach.

## Environment and cluster

- Run the script in the **correct cluster** (US vs EU) for the workspace.
- Use the Django environment (shell or one-off command) that has access to the workspace’s database and Celery broker.

## Escalation

- If the script fails with FK/constraint errors, check that `contract_id` and `created_by_workspace` are correct and that no other code holds references that block deletes.
- If resync never completes, check Celery workers and `partially_resync_contracts_task` logs for the given `contract_ids`.

## Related incidents and references

- **Incident (example)**: Fullstory – bulk delete metadata value. Customer: Fullstory, WS ID: 41811, Cluster: US. Rootly incident 3007, Jira SPD-42475.
- **Similar incident**: Gourav shared a similar Rootly link in the same channel for reference.
- **Mitigation**: Script with 5s delay and `CELERY_TASK_ALWAYS_EAGER = False`; dry run then live run; verify and close.

## Connections to other skills

- **Contract lookup**: Use when you need to verify contract existence or workspace before or after the bulk delete.
- **Search/Elasticsearch**: If reports or search are stale after delete, ensure `partially_resync_contracts_task` completed for the affected `contract_ids`.
