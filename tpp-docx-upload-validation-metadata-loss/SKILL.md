---
name: tpp-docx-upload-validation-metadata-loss
description: "Diagnose Third-Party Paper request uploads that fail with 'Invalid DOCX file' and then wipe already-entered metadata when the user replaces the file. Use when support reports: DOCX opens fine in Word but SpotDraft rejects it, cleaned file workaround succeeds, Google Docs or multi-editor DOCX history is suspected, or TPP upload retries clear all metadata fields. Trigger phrases: invalid docx file, TPP upload failure, metadata lost after re-upload, customXml, ConvertAPI 5001, file cleaning workaround."
---

# TPP DOCX Upload Validation + Metadata Loss

## What this skill debugs

This skill covers a paired issue class in the **Third-Party Paper (TPP) request flow**:

1. SpotDraft rejects a `.docx` with a generic **`Invalid DOCX file`** error even though the document opens in Microsoft Word.
2. After that failure, when the user replaces the file and retries, the TPP form loses all previously entered metadata.

These often appear together and should be investigated as **two separate bugs**:

- **Backend validation strictness** on non-standard DOCX XML structure
- **Frontend state loss** when the file entry is replaced after an upload error

---

## When to use / when not to use

**Use when:**

- The customer is in a **TPP / upload-to-create-request** flow
- The UI shows **`Invalid DOCX file`**
- The same file opens normally in Word or LibreOffice
- Support says the internal “cleaned file” workaround succeeds
- Replacing the failed file causes previously entered metadata to disappear

**Do not use for:**

- Password-protected or section-protected DOCX files where the validator already returns a specific error
- Generic preview / execution failures after the contract is already created
- PDF upload issues that do not involve DOCX validation

---

## Confirm the two issue branches

### Branch A: DOCX validation failure

The upload path validates the file before contract creation:

```text
TPP upload
  -> POST /api/v3/contracts/v2/uploaded_contracts/
  -> core.views.ValidateDocxContent / upload validation path
  -> core.validators.validate_docx_content()
  -> WordServerClient.validate_and_cleanup_docx()
  -> Word Server / ConvertAPI
  -> InvalidDocxFileException
```

**Known backend references:**

- `django-rest-api/core/validators.py`
- `django-rest-api/core/views.py`
- `core.validators.validate_docx_content`
- `WordServerClient.validate_and_cleanup_docx`

### Branch B: metadata loss after file replacement

The TPP upload modal stores form state per uploaded file GUID. If a failed file is replaced, the new file can be treated as a fresh upload instead of a replacement, so the user loses previously entered form data.

**Known frontend references:**

- `angular-frontend/libs/features/new-contract-upload/components/new-contract-upload/src/lib/clean-architecture/contract-review-upload-modal.store.ts`
- `ContractReviewUploadModalStore`
- `collectedInfo`
- `rawFilesChange`

The key state update to inspect is `updateUploadedContracts`, which rebuilds `collectedInfo` from uploaded files and matches entries by `uploadedFileGuid`.

---

## Most useful first checks

Collect these first:

- `{workspace_id}`
- `{cluster}`
- failing filename
- whether the file was edited in **Google Docs**, Word, or multiple editors
- whether support already produced a **cleaned** copy that uploads successfully
- whether the customer lost metadata after replacing the file

Then confirm:

1. Does the raw file fail with **`Invalid DOCX file`**?
2. Does a cleaned / re-saved copy succeed?
3. Does the metadata disappear only after the file is replaced?

If the answers are **yes / yes / yes**, this skill applies directly.

---

## Investigation workflow

### Step 1: Verify the validator path

Look for the backend exception:

- `core.exceptions.InvalidDocxFileException: Invalid docx file.`

The key validator is:

- `validate_docx_content()` in `django-rest-api/core/validators.py`

This calls `WordServerClient.validate_and_cleanup_docx()` and maps unhandled Word Server failures to the generic invalid-DOCX error.

### Step 2: Treat “opens in Word” as insufficient proof

A DOCX can open normally in Word and still fail SpotDraft validation if the OOXML package contains structurally unusual content that the Word Server / ConvertAPI path rejects.

Common patterns from recurring incidents:

- non-standard `customXml/` artifacts
- malformed `itemProps.xml`
- excessive or stale namespace declarations
- files edited across multiple tools such as **Google Docs -> Word -> other editors**

This is often a **strict-parser failure**, not a visibly corrupt file.

### Step 3: Check whether cleaning the file fixes it

If support can “repair” the file by re-saving / cleaning it and the cleaned file uploads, that strongly confirms the issue is in the document structure, not in customer permissions or request payload shape.

Known mitigation path:

- open in Word or LibreOffice
- save a cleaned copy
- share the cleaned file back to the customer

If the cleaned file succeeds, preserve the failing original for pod-level debugging.

### Step 4: Confirm the frontend metadata-loss bug separately

If the customer replaces the failed file and all metadata disappears, inspect the upload-modal state flow:

- `ContractReviewUploadModalStore`
- `rawFilesChange`
- `collectedInfo`

The failure mode is that the replacement file gets a fresh GUID and the store rebuild treats it as a new upload entry, so the previously entered metadata is dropped.

This is a **frontend persistence bug**, not a validator bug.

### Step 5: Separate mitigation from permanent fixes

Use this incident pattern to split follow-up cleanly:

- **Create / document pod**: make DOCX validation more tolerant or preserve the actual parser failure details
- **Frontend pod**: preserve metadata when a file is replaced after upload failure

If both problems are present, file both tracks explicitly.

---

## Known root-cause pattern

### DOCX upload failure

The file contains non-standard XML artifacts that the Word Server / ConvertAPI path rejects. In the March 2026 Dream11 recurrence, the suspected failure mode was equivalent to **ConvertAPI / parser error code `5001`** tied to malformed OOXML internals, while the platform surfaced only **`Invalid DOCX file`**.

### Metadata loss

The TPP request form binds metadata to the uploaded file identity. Replacing the failing file creates a new upload identity, and the previous metadata does not carry forward.

---

## How to confirm / disprove

| Finding | Interpretation |
|---------|----------------|
| Raw DOCX fails, cleaned DOCX succeeds | File structure / XML artifact issue |
| File opens normally in Word, but SpotDraft rejects it | Strict validation path, not necessarily actual corruption |
| Customer loses metadata only after replacing the file | Frontend state-loss bug |
| Specific password / section protection error appears | Use the dedicated validator path instead of this skill |
| Same customer has repeated incidents with similar files | Likely recurring document-generation source problem, not a one-off typo |

---

## Mitigation / workaround

### Immediate unblock

1. Obtain the failing DOCX from support / customer.
2. Use the internal file-cleaning process to generate a cleaned copy.
3. Share the cleaned copy back so the customer can continue the TPP request.

### Reduce customer pain from metadata loss

Until the frontend fix lands:

- tell the customer to keep metadata values ready outside the form before retrying
- if support is walking them through the retry, have them re-enter metadata only after the cleaned file is accepted

This is not a real fix, but it shortens the retry loop.

---

## Escalation triggers

- Multiple failures from the same customer or workflow using different DOCX files
- Repeated need for manual “file cleaning” by support
- Generic invalid-DOCX errors with no surfaced parser detail
- Confirmed metadata loss after file replacement in TPP flow

Escalate both pods if both halves reproduce.

---

## Related incidents / references

| Reference | Notes |
|-----------|--------|
| `#incident-20260319-high-invalid-docx-upload-error-metadata-loss-on-tpp-request-flow` | Dream11 recurrence; paired DOCX-validation + metadata-loss incident |
| `SPD-42560` | Jira from the March 2026 incident |
| `#incident-20250728-low-dream11-unable-to-upload-the-document` | Earlier predecessor incident with the same customer and same invalid-DOCX theme |

---

## Suggested permanent fixes

### Backend / validation

- attempt a tolerant preprocessing / cleanup path before rejecting the DOCX outright
- preserve and surface the actual Word Server / parser error code instead of only `Invalid DOCX file`
- document the internal cleaning workaround until platform tolerance improves

### Frontend / state management

- detect “replace failed file” as distinct from “add new file”
- carry forward existing `collectedInfo` when the file slot is replaced
- add regression coverage around `rawFilesChange` and GUID replacement behavior in the TPP upload modal
