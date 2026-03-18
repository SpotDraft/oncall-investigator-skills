---
name: clickwrap-400-submission-triage
description: "Diagnose and resolve 400 Bad Request errors during clickwrap (clickthrough) contract submission via the SpotDraft SDK or public API. Use this skill when a customer reports intermittent or consistent 400 errors while calling the clickwrap execute/submit endpoint, 'invalid payload' errors, 'user_identifier missing or invalid', null field errors, 400 on agreement submission, clickwrap contract not being created, or SDK submit() returning a 400. Also trigger on: 'clickthrough 400', 'clickwrap API validation', 'SDK validation error', 'null user_identifier', 'invalid user_email', 'agreement ID mismatch', 'clickwrap submission fails', or 'TnC packet 400'."
---

# Clickwrap 400 Submission Error Triage

Covers 400 Bad Request errors raised when a customer's integration calls the clickwrap execute (submit) API — `POST /api/v2.1/clickwraps/{clickwrap_public_id}/consent/`. The most common root cause is null, empty, or invalid payload fields sent by the client integration.

---

## API Endpoint & Code Path

```
POST /api/v2.1/clickwraps/{clickwrap_public_id}/consent/
Auth: ExternalClickwrapAuthentication (API key in header)
```

**Server-side call chain:**

```
CreateClickwrapContractView.post()
  └── CreateClickwrapContractRequest(**request.data)   ← Pydantic validation (400 raised here for invalid fields)
  └── CreateClickwrapContractUseCase.execute()
        ├── _validate()                                 ← Feature flag check (raises 403, not 400)
        ├── _create_executed_contract()                 ← Creates ContractV3 of kind CLICKWRAP
        ├── _get_or_create_clickwrap_user()             ← Upsert ClickwrapUser by user_identifier + wsid
        ├── _create_clickwrap_consent()                 ← Creates ClickwrapConsent record
        ├── _create_clickwrap_consent_agreement_version_mapping()  ← Links consent to agreement versions
        └── peripherals task (async)                   ← Email, key pointers, audit trail
```

**Key files:**
- `public/clickwraps/clickwrap/presentation/create_clickwrap_contract_view.py`
- `clickwraps/clickwrap_consent/domain/domain_models.py` — `CreateClickwrapContractRequest`
- `clickwraps/clickwrap_consent/domain/use_cases/create_clickwrap_contract.py`

---

## Payload Field Rules

```python
class CreateClickwrapContractRequest:
    clickwrap_public_id: UUID          # REQUIRED — the UUID from the clickthrough URL
    user_identifier: str               # REQUIRED — non-null, non-empty string (Pydantic rejects null)
    agreements: List[{id, version_id, has_clicked}]  # REQUIRED — must reference mapped agreement versions
    first_name: Optional[str] = None   # Optional — send empty string fallback instead of null
    last_name: Optional[str] = None    # Optional — send empty string fallback instead of null
    user_email: Optional[str] = None   # Optional — OMIT if null or invalid; sending invalid email triggers 400
    additional_custom_information: Optional[dict] = None  # Optional — avoid null values within the dict
    key_pointer_information: Optional[dict] = None        # Optional — avoid null values within the dict
    external_metadata: Optional[dict] = None
```

**400 trigger matrix:**

| Field | Sent As | Result |
|-------|---------|--------|
| `user_identifier` | `null` / `None` | **400** — Pydantic rejects (field is required `str`) |
| `user_identifier` | `""` (empty string) | **400 or silent bug** — passes Pydantic but creates bad contract name; SDK should reject before API call |
| `user_email` | `"not-an-email"` or `null` | **400** if email validation is applied downstream |
| `agreements` | `[]` (empty list) | **400** — no agreement version mappings to create |
| `agreements[].version_id` | invalid/unmapped ID | **400** — foreign key constraint failure |
| Optional dict fields | `{"key": null}` | Potentially **400** — avoid null values inside dicts |
| `baseUrl` | trailing slash | SDK builds malformed endpoint URL → 400 or 404 |
| `submit()` called before `init()` | race condition | `clickwrap_public_id` / agreement IDs are wrong → 400 |

---

## Available MCP Tools for Investigation

### 1. Groundcover — API Request Logs

Check incoming request logs for the failed submissions. Prod India cluster is `prod-india`.

**Find 400 errors for a specific workspace (Groundcover log query):**
```
service = "spotdraft-django-app"
AND cluster = "{cluster}"           -- e.g. prod-india
AND http.status_code = 400
AND http.path CONTAINS "/clickwraps/"
AND workspace_id = "{wsid}"
```

**Find the exact validation error message:**
```
service = "spotdraft-django-app"
AND cluster = "{cluster}"
AND http.path CONTAINS "/clickwraps/"
AND http.path CONTAINS "/consent/"
AND http.status_code = 400
```

### 2. GCP Cloud Logging — Django Application Logs

**⚠️ Prod uses `jsonPayload`, QA uses `textPayload`.**

**Find 400 errors for clickwrap consent endpoint in prod (narrow time window required — high volume):**
```
logName="projects/spotdraft-prod/logs/stderr"
resource.labels.cluster_name="prod-india"
jsonPayload.message:"clickwrap"
jsonPayload.message:"400"
timestamp >= "{start_ts}"
timestamp <= "{end_ts}"
```

**Find `CreateClickwrapContractUseCase` log entries for a specific user_identifier:**
```
logName="projects/spotdraft-prod/logs/stdout"
resource.labels.cluster_name="prod-india"
jsonPayload.user_identifier="{user_identifier}"
jsonPayload.message:"CreateClickwrapContractUseCase called"
```

**QA equivalent:**
```
resource.type="k8s_container"
labels."k8s-pod/app"="spotdraft-qa-django-app"
textPayload:"CreateClickwrapContractUseCase"
textPayload:"{user_identifier}"
```

### 3. BigQuery — Clickwrap State

**⚠️ Prod India uses `spotdraft-prod` project, dataset `prod_india_db`, table prefix `public_`.**

**Environment-to-dataset mapping:**

| Cluster | BQ Project | Dataset | Table Prefix |
|---------|-----------|---------|--------------|
| Prod India (IN) | `spotdraft-prod` | `prod_india_db` | `public_` |
| Prod USA | `spotdraft-prod` | `prod_usa_db` | `public_` |
| Prod EU | `spotdraft-prod` | `prod_eu_db` | `public_` |
| QA India | `spotdraft-qa` | `qa_india_public` | *(none)* |

**Look up clickwrap by public_id (from the clickthrough URL):**
```sql
SELECT
  cw.id, cw.name, cw.public_id, cw.contract_type_id,
  cw.is_deleted, cw.tenant_workspace_id, cw.created, cw.modified
FROM `spotdraft-prod.prod_india_db.public_clickwraps_clickwrap` cw
WHERE cw.public_id = '{clickwrap_public_id}'
  AND cw.tenant_workspace_id = {wsid}
```

**Check agreements mapped to a clickwrap:**
```sql
SELECT
  cam.id, cam.clickwrap_id, cam.clickwrap_agreement_id,
  ca.name as agreement_name, ca.is_deleted as agreement_deleted,
  cav.id as version_id, cav.name as version_name,
  cav.is_published, cav.is_deleted as version_deleted
FROM `spotdraft-prod.prod_india_db.public_clickwraps_clickwrapagreementmapping` cam
JOIN `spotdraft-prod.prod_india_db.public_clickwraps_clickwrapagreement` ca
  ON ca.id = cam.clickwrap_agreement_id
JOIN `spotdraft-prod.prod_india_db.public_clickwraps_clickwrapagreementversion` cav
  ON cav.clickwrap_agreement_id = ca.id AND cav.is_deleted = FALSE
WHERE cam.clickwrap_id = {clickwrap_id}
  AND cam.is_deleted = FALSE
ORDER BY cav.is_published DESC, cav.id DESC
```

**Check consent records created for a user_identifier (to see which submissions succeeded):**
```sql
SELECT
  cu.user_identifier, cu.user_email, cu.first_name, cu.last_name,
  cu.tenant_workspace_id,
  cc.id as consent_id, cc.contract_id,
  cc.clickwrap_id, cc.consented_at, cc.ip,
  c.workflow_status, c.execution_date
FROM `spotdraft-prod.prod_india_db.public_clickwraps_clickwrapuser` cu
JOIN `spotdraft-prod.prod_india_db.public_clickwraps_clickwrapconsent` cc
  ON cc.clickwrap_user_id = cu.id
JOIN `spotdraft-prod.prod_india_db.public_contracts_v3_contractv3` c
  ON c.id = cc.contract_id
WHERE cu.user_identifier = '{user_identifier}'
  AND cu.tenant_workspace_id = {wsid}
ORDER BY cc.consented_at DESC
LIMIT 20
```

**Count submission successes vs failures by checking consent records for a workspace:**
```sql
SELECT
  DATE(cc.consented_at) as date,
  COUNT(*) as successful_submissions,
  cw.name as clickwrap_name
FROM `spotdraft-prod.prod_india_db.public_clickwraps_clickwrapconsent` cc
JOIN `spotdraft-prod.prod_india_db.public_clickwraps_clickwrap` cw
  ON cw.id = cc.clickwrap_id
WHERE cc.tenant_workspace_id = {wsid}
  AND cc.consented_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 14 DAY)
GROUP BY date, cw.name
ORDER BY date DESC
```

**Check clickwrap feature flag for a workspace:**
```sql
SELECT
  cs.type, cs.is_enabled, cs.tenant_workspace_id,
  cs.created, cs.modified
FROM `spotdraft-prod.prod_india_db.public_clickwraps_clickwrapsettings` cs
WHERE cs.tenant_workspace_id = {wsid}
```

---

## Step-by-Step Investigation Workflow

### Step 1 — Gather Identifiers

Collect from the support ticket or incident channel:
- `wsid` (workspace ID) and `cluster` (e.g. IN, USA, EU)
- Clickthrough URL or `clickwrap_public_id` (UUID from the URL: `app.spotdraft.com/clickthrough/{id}/settings`)
- `user_identifier`, `user_email` used in the failing submission
- Any error message / HTTP response body from the client
- Code snippet or payload structure if the customer shared one
- Time window of failures

### Step 2 — Confirm the 400 is Payload-Driven (Not Server-Side)

Check GCP logs or Groundcover for server-side error logs during the failure window. If you see:
- **Pydantic validation error** (`value_error`, `none is not an allowed value`, `str type expected`) → payload issue, confirm which field
- **`CLICKWRAP_FEATURE_FLAG_NOT_ENABLED`** → Feature flag off for workspace (see Step 5)
- **DB IntegrityError / ForeignKeyViolation** → Agreement version IDs sent do not exist or are not mapped
- **No server-side error at all** → Request may not be reaching SpotDraft (DNS/network issue on client side, or wrong base URL)

### Step 3 — Validate the Clickwrap Configuration

```sql
-- Get clickwrap by public_id
SELECT id, name, public_id, is_deleted, tenant_workspace_id
FROM `spotdraft-prod.prod_india_db.public_clickwraps_clickwrap`
WHERE public_id = '{clickwrap_public_id}'
```

Check:
- `is_deleted = FALSE` — if deleted, submissions will 404 not 400
- `tenant_workspace_id` matches the expected WSID
- Agreements are mapped and have published versions (run the agreement mapping query above)

### Step 4 — Validate the Client Payload

Ask the customer or check the code snippet they shared. Verify:

1. **`user_identifier`** — Is it possible to be null/empty/undefined for some users?
   - Common case: identifier derived from a session token or login field that may not be populated for all users
   - Fix: Always use a non-empty fallback (e.g. `user.id || user.email || 'anonymous'`)

2. **`user_email`** — Is it validated before sending?
   - Fix: Omit the field entirely when null or when format is invalid; don't send `null`

3. **`first_name` / `last_name`** — Sent as `null`?
   - Fix: Use `""` (empty string) as fallback instead of `null` or omitting

4. **`agreements` array** — Populated from SDK `init()` response?
   - Fix: Only call `submit()` after the `init()` promise resolves; confirm `agreements[].id` and `agreements[].version_id` come from the `init()` response, not hardcoded values

5. **Base URL** — Does it have a trailing slash?
   - Fix: Use `https://api.spotdraft.com` not `https://api.spotdraft.com/`

### Step 5 — Rule Out Feature Flag Issue

If logs show `ClickwrapFeatureFlagNotEnabledException`, check the feature flag:

```sql
SELECT type, is_enabled, tenant_workspace_id
FROM `spotdraft-prod.prod_india_db.public_clickwraps_clickwrapsettings`
WHERE tenant_workspace_id = {wsid}
```

If the clickwrap feature flag (`CLICKWRAP`) is disabled → escalate to eng to enable for the workspace.

### Step 6 — Verify Successful Submissions Exist

If the customer says "some succeed, some fail," confirm this by counting consent records:

```sql
SELECT DATE(consented_at), COUNT(*) as successes
FROM `spotdraft-prod.prod_india_db.public_clickwraps_clickwrapconsent`
WHERE tenant_workspace_id = {wsid}
  AND clickwrap_id = {clickwrap_id}
  AND consented_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
GROUP BY 1 ORDER BY 1 DESC
```

Intermittent successes alongside 400s = strong signal that payload is conditionally invalid (e.g. `user_identifier` is null for some users but not others).

---

## How to Confirm Root Cause

| Symptom | Likely Root Cause |
|---------|-------------------|
| 400 only for some users, others succeed | `user_identifier` or `user_email` conditionally null/invalid |
| 400 always, no server-side error log | Request not reaching API (wrong base URL, network issue) |
| 400 with Pydantic `none is not allowed` in log | `user_identifier` sent as null |
| 400 with ForeignKey / IntegrityError | `agreements[].version_id` invalid or not mapped to clickwrap |
| 400 immediately after deploy | Regression — check if `agreements` structure changed in SDK update |
| 400 starts after `init()` error | `submit()` called before `init()` resolved; agreement IDs are undefined |

---

## Mitigation / Fix for Customer

Send the customer these recommendations (confirmed by engineering in SPD-42278):

1. **`user_identifier`** — Required. Must be a non-null, non-empty string. Use a reliable fallback if the primary identifier may be absent.

2. **`user_email`** — Optional. Validate format before sending. **Omit the field entirely** if null or invalid — do not send `null`.

3. **`first_name` / `last_name`** — Optional. Send empty string `""` as fallback instead of `null`.

4. **`additional_custom_information` / `key_pointer_information`** — If sending, ensure all values within the dict are valid types; avoid `null` values inside the object.

5. **Call order** — Only call `submit()` after the `init()` promise has resolved. The `agreements` array passed to `submit()` must come from the `init()` response.

6. **Base URL** — Use `https://api.spotdraft.com` without trailing slash.

7. **Omit optional fields** entirely when they have no valid value, rather than sending `null`.

---

## Known SDK Improvements Planned (Tracked in SPD-42278)

The following are known engineering improvements not yet shipped as of the incident date:
- Validate `user_identifier` is a non-empty string in the SDK before making the API call (throw `EXECUTION_REQUIRED_FIELD_MISSING` or `INVALID_USER_IDENTIFIER`)
- Validate payload in `execute-clickwrap.usecase` before API call
- Map 400/403/404 responses to SDK-level error codes (currently raw HTTP error)
- Normalize `baseUrl` (strip trailing slash) during SDK configuration
- Update SDK documentation to explicitly state `user_identifier` must be non-empty and `init()` must complete before `submit()`

These are not customer-facing yet — the customer must still implement client-side validation.

---

## Escalation Triggers

Escalate to engineering if:
- Server-side error (IntegrityError, unhandled exception) causes the 400 — not a client payload problem
- The clickwrap feature flag is incorrectly disabled for the workspace
- Agreement version IDs are valid and correctly mapped but still causing FK errors (DB state inconsistency)
- The customer's payload is valid per all rules above and 400s persist

---

## Admin Links

- Clickthrough settings: `https://app.spotdraft.com/clickthrough/{clickwrap_short_id}/settings`
- Django admin clickwrap consent: `/admin/clickwraps/clickwrapconsent/` (filter by `tenant_workspace_id`)
- Django admin clickwrap user: `/admin/clickwraps/clickwrapuser/` (filter by `user_identifier`)

---

## Related Incidents & Jira

- **SPD-42278** — Centricity: Intermittent 400 Error While Submitting Clickthrough Agreement (Status: Permanently Fixed — customer implemented client-side validation)
  - WSID: 594511 | Cluster: IN | Clickthrough ID: `44df08dd-a076-4722-85a5-af15ae4a8259`
  - Root cause: `user_identifier`, `first_name`/`last_name`, and `user_email` sent as null by customer integration
  - Incident channel: `#incident-20260312-medium-centricity-intermittent-400-error-while-submitting-clic`

---

## Related Skills

- **contract-lookup** — Look up the created clickwrap contract by `contract_id` to verify it was created and its workflow status
- **incident-lookup** — Search for similar past clickwrap 400 incidents in Slack or Jira
