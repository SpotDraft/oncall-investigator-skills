---
name: email-sender-variable-business-user-fallback
description: "Diagnose and resolve issues where SpotDraft email template variables like {{current.sign.sender.sender_name}} or {{current.sign.sender.sender_email}} show the Business User's name/email instead of the actual user who triggered the action. Use this skill when a customer reports that their email signature or sender name is wrong, shows the wrong user, or defaults to the Business User when someone else sent the contract. Trigger on: 'wrong sender name in email', 'email shows Business User instead of actual user', 'sign.sender variable wrong', 'email signature defaults to BU', 'sender_name showing wrong name', or any complaint about incorrect sender identity in outbound emails."
---

# Email Sender Variable → Business User Fallback

Covers the class of bugs where `{{current.sign.sender.sender_name}}` or `{{current.sign.sender.sender_email}}` (and similar `sign.sender.*` variables) resolve to the Business User's details instead of the actual user who triggered the action.

## Root Cause

`GetSignContextUseCase._get_sender_details()` determines the sender for email template variable resolution. It reads `sent_for_signature_by_org_user` from `ContractSignatureSetup`.

**Problem:** For "Send to Counterparty" (pre-signing flow), the contract hasn't been formally sent for signature yet, so `ContractSignatureSetup` does not exist. The code falls back to the Business User.

The `current_logged_in_user` is **not** passed into `_get_sender_details()` in this flow, so there is no way for it to know who actually clicked "Send."

> **This is WAI for `sign.sender.*` variables** — they are designed to capture the "sent for signature" sender. The issue is that no variable currently exists for "current acting user." Adding one (e.g., `current.sender`) requires a **feature enhancement** (tracked as SPD-42062, currently "Mitigated").

## Trigger Conditions

All three must be true:
1. Workspace uses Email V4 templates with `{{current.sign.sender.sender_name}}` or `{{current.sign.sender.sender_email}}`
2. The email is sent via the "Send" button (counterparty review / pre-signing), not "Send for Signature"
3. The user who clicked Send is **not** the Business User

**Email V5 note:** In Email V5, the `sender_name` variable is resolved differently and works correctly. Check if the workspace is on Email V5 before spending time debugging.

## Diagnosis Steps

### Step 1 — Check Email V5 FF

First rule out Email V5. If the workspace has `EMAIL_V5` enabled and is actually using it, the bug should not apply.

```
Django admin → SD Organizations → Enhanced Flags
Filter: workspace_id={wsid}, flag=EMAIL_V5
```

Also check: `WORKFLOW_EMAIL_CUSTOMIZATION` — if enabled, the workspace is using email template customization.

### Step 2 — Inspect the Email Audit

Find the exact rendered email to confirm the wrong value was used:

```
api.{cluster}.spotdraft.com/admin/emails/emailaudit/?contract_id={contract_id}
```

Look at the **body** field — confirm the resolved `sender_name` is the Business User's name, not the sending user's name.

US example (Cocoon incident):
```
https://api.us.spotdraft.com/admin/emails/emailaudit/5b60fb85-9ffb-4afc-936c-7f0a8f2a2a58/change/?_changelist_filters=contract_id%3D1392453
```

### Step 3 — Confirm Email Template Uses `sign.sender` Variables

Check the workspace's email template to confirm it uses `{{current.sign.sender.sender_name}}` or `{{current.sign.sender.sender_email}}`:

```
api.{cluster}.spotdraft.com/admin/emails/emailtemplate/?workspace_id={wsid}
```

If the template uses these variables and the send happens before `ContractSignatureSetup` is created, the fallback to Business User is expected.

### Step 4 — Verify ContractSignatureSetup State (BigQuery)

Confirm no `ContractSignatureSetup` existed at the time of sending:

```sql
-- Prod US
SELECT
  css.id, css.contract_id, css.sent_for_signature,
  css.sent_for_signature_by_org_user_id, css.is_completed, css.is_deleted,
  css.created, css.modified
FROM `spotdraft-prod.prod_usa_db.public_contracts_v3_contractsignaturesetup` css
WHERE css.contract_id = {contract_id}
ORDER BY css.created DESC
LIMIT 5
```

| Environment | Table |
|-------------|-------|
| Prod US | `spotdraft-prod.prod_usa_db.public_contracts_v3_contractsignaturesetup` |
| Prod IN | `spotdraft-prod.prod_india_db.public_contracts_v3_contractsignaturesetup` |
| Prod EU | `spotdraft-prod.prod_eu_db.public_contracts_v3_contractsignaturesetup` |
| QA India | `spotdraft-qa.qa_india_public.contracts_v3_contractsignaturesetup` |

If no row exists at the time of the email send, the fallback to Business User is confirmed as the root cause.

## Mitigation / Workaround

**Immediate (no code change):** Update the workspace's email template to replace `{{current.sign.sender.sender_name}}` and `{{current.sign.sender.sender_email}}` with a static name or a different variable that represents the Business User explicitly (if that's the intended design) or remove the signature block entirely.

```
api.{cluster}.spotdraft.com/admin/emails/emailtemplate/{template_id}/change/
```

Replace:
```
{{current.sign.sender.sender_name}}
{{current.sign.sender.sender_email}}
```

With a static value or an alternative variable (e.g., the Business User's name if that's acceptable to the customer).

**Long-term fix:** A feature enhancement is required to add a `current.sender` (or equivalent) variable that captures the currently logged-in user at action time. Tracked in **SPD-42062** (status: Mitigated). Mark as resolved once the variable is shipped.

## Escalation Triggers

- Escalate to the email/signing pod (cc Anjali Swami or Hemil Shah) if:
  - The workspace IS on Email V5 and the issue still occurs — unexpected behavior
  - The template does NOT use `sign.sender.*` but the wrong name still appears — different bug
  - The issue persists after updating the email template

## Known Instances

| Customer | WSID | Cluster | Date | Jira | Incident |
|----------|------|---------|------|------|----------|
| Cocoon | 425092 | US | Mar 2026 | [SPD-42062](https://spotdraft.atlassian.net/browse/SPD-42062) | [#incident-20260307-medium-cocoon-email-signature-defaults-to-business-user-name-i](https://spotdraft.slack.com/archives/C0AKL21DMB3) |

> Anjali Swami noted in the incident thread: "We have similar issues reported where the Business User name is shown instead of the user who actually triggered the Send_for_signing action." — additional unreported instances likely exist.

## Related Skills

- `email-delivery-triage` — if the email is not being received at all (delivery failure, Postmark suppression)
- `contract-signing-triage` — if the signing flow itself is broken (not just wrong sender name in email)
- `contract-lookup` — to look up workspace feature flags and contract state
