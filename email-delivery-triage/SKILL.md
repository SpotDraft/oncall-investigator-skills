---
name: email-delivery-triage
description: "Diagnose and resolve email and OTP delivery failures for SpotDraft customers. Use this skill whenever a support person reports that a counterparty is not receiving OTP, signing links, email notifications, or any SpotDraft-sent emails. Also trigger on phrases like 'email not received', 'OTP not triggered', 'signing link not delivered', 'blank email', 'email suppressed', 'Postmark', 'domain whitelisting', or any mention of counterparty email delivery problems."
---

# Email & OTP Delivery Triage

This is the most common issue type in #oncall and #incident channels. Follow this systematic diagnostic flow.

## Available MCP Tools for Investigation

You have direct access to GCP Cloud Logging and BigQuery for `spotdraft-qa`. Use these to investigate email delivery failures programmatically.

### BigQuery Django Models — Contract Context

For QA contracts, look up counterparty and contract state directly from the synced model tables. **MCP only has access to `spotdraft-qa` project.**

**Look up contract details for context (QA India):**
```sql
SELECT
  c.id, c.status, c.contract_kind, c.created_by_workspace_id, c.contract_type_id
FROM `spotdraft-qa.qa_india_public.contracts_v3_contractv3` c
WHERE c.id = {contract_id}
```

**Check signing recipients and their status (for OTP/signing email issues):**
```sql
SELECT
  ss.id as sig_setup_id, ss.is_completed, ss.sent_for_signature,
  ss.signing_method, ss.is_deleted
FROM `spotdraft-qa.qa_india_public.contracts_v3_contractsignaturesetup` ss
WHERE ss.contract_id = {contract_id}
  AND ss.is_deleted = FALSE
```

### Environment-to-Dataset Mapping

| Environment | BQ Project | Dataset | MCP Access |
|-------------|-----------|---------|------------|
| QA India | `spotdraft-qa` | `qa_india_public` | ✅ Yes |
| QA EU | `spotdraft-qa` | `qa_eu_public` | ✅ Yes |
| QA USA | `spotdraft-qa` | `qa_usa_public` | ✅ Yes |
| **Prod IN/EU/US** | `spotdraft-prod` | `prod_{region}_db` | ❌ Use admin panel |
| **Prod MEA** | `spotdraft-prod` | `prod_mea_db` | ❌ Use admin panel |

### BigQuery — DLQ Analysis for Email Failures

The DLQ tables contain failed email-related tasks. Query these FIRST for diagnosing email issues.

**Find all email-related task failures in the last 7 days:**
```sql
SELECT task_name, COUNT(*) as failures, MIN(created) as first, MAX(created) as last
FROM `spotdraft-qa.dlq_qa_india.dlq`
WHERE created >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND (task_name LIKE '%email%' OR task_name LIKE '%postmark%' OR task_name LIKE '%otp%'
       OR task_name LIKE '%notification%' OR task_name LIKE '%djcelery%')
GROUP BY task_name
ORDER BY failures DESC
LIMIT 20
```

**Known email-related DLQ task names (from QA data):**
- `integrations_v2.slack...send_contract_postmark_event_slack_message_use_case` — Postmark event delivery notifications (3,424 failures/3 days seen in QA)
- `bounced_email_audit_use_case` — Bounced email processing
- `djcelery_email_send_multiple` — Bulk email sending via Celery (4,164 failures/3 days — HIGH ALERT if spiking)

**Check email failures for a specific workspace:**
```sql
SELECT task_name, created, signature
FROM `spotdraft-qa.dlq_qa_india.dlq`
WHERE created >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND workspace_id = {wsid}
  AND (task_name LIKE '%email%' OR task_name LIKE '%postmark%' OR task_name LIKE '%otp%'
       OR task_name LIKE '%djcelery%')
ORDER BY created DESC
LIMIT 20
```

### BigQuery — Access Control Logs for Auth/Email Failures

**Check for email-based authentication failures:**
```sql
SELECT event_name, event_outcome, COUNT(*) as count, MAX(datetime) as last_occurrence
FROM `spotdraft-qa.access_control_logs_qa_india.logs`
WHERE datetime >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND event_name = 'email'
  AND event_outcome != 'success'
GROUP BY event_name, event_outcome
ORDER BY count DESC
```
Note: QA data shows ~107 email failures in a typical week.

### GCP Cloud Logging — Email-Related Errors

Use `list_log_entries` with `resourceNames: ["projects/spotdraft-qa"]` and `orderBy: "timestamp desc"`:

**Search for email sending errors in Django app:**
```
resource.type="k8s_container" labels."k8s-pod/app"="spotdraft-qa-django-app" textPayload:("email" OR "postmark" OR "smtp" OR "OTP" OR "otp_code") severity >= ERROR
```

**Search for email errors related to a specific contract:**
```
resource.type="k8s_container" labels."k8s-pod/app"="spotdraft-qa-django-app" textPayload:("{contract_id}") textPayload:("email" OR "notification" OR "OTP")
```

**Search deferred tasks for email task failures:**
```
resource.type="k8s_container" labels."k8s-pod/app"="spotdraft-qa-django-app-deffered-tasks" textPayload:("email" OR "postmark" OR "smtp") severity >= ERROR
```

### BigQuery Datasets by Cluster
- QA India: `spotdraft-qa.dlq_qa_india.dlq`, `spotdraft-qa.access_control_logs_qa_india.logs`
- QA Europe: `spotdraft-qa.dlq_qa_europe.dlq`, `spotdraft-qa.access_control_logs_qa_europe.logs`
- QA USA: `spotdraft-qa.dlq_qa_usa.dlq`, `spotdraft-qa.access_control_logs_qa_usa.logs`

## Diagnostic Flow

### Step 1: Gather Context

Ask the support person for:
- **Customer name and Workspace ID (WSID)**
- **Cluster** (IN, US, ME, EU)
- **Counterparty email address** that isn't receiving emails
- **What was expected** — OTP for signing, signing link, notification email?
- **Email client** the counterparty uses (Outlook/Exchange is a known problem area)

### Step 2: Run Automated DLQ Analysis

Immediately query the DLQ for the workspace to check for email task failures. This often reveals the root cause faster than manual dashboard checks.

For QA workspaces: also query `qa_india_public.contracts_v3_contractv3` to confirm contract and workspace context.

### Step 3: Check SpotDraft Email Audit Logs

Look up email delivery records in the SpotDraft admin panel:
```
https://api.{cluster}.spotdraft.com/admin/emails/emailaudit/?contract_id={contract_id}
```

Was the email sent? What error was recorded?

Important nuance from repeated production incidents: **absence of a fresh EmailAudit event does not rule out Postmark suppression**. If the recipient hard-bounced earlier, Postmark can silently suppress later sends before SpotDraft surfaces a meaningful delivery event for the new attempt.

### Step 3.5: Search Postmark Alert Channels First

Before logging into Postmark directly, search the Postmark alert channel used by your team (for example `#postmark-bot` or `#postmark-alerts`) for the recipient email address. This is the fastest way to spot prior hard bounces and suppression events.

**Search query to run:**
- `in:#postmark-bot {recipient_email}` or `in:#postmark-alerts {recipient_email}` — look for bounce, spam, or suppression events

If results show bounce/spam entries matching the recipient, skip to Step 4 (suppression confirmed via Postmark).

### Step 4: Check Postmark for Suppression

Postmark is SpotDraft's email delivery provider. Check if the recipient email is **suppressed**.

Common suppression reasons:
- Previous hard bounce
- Previous spam complaint
- Email on suppression list

**Key diagnostic signal:** If the SpotDraft email audit shows emails were "sent successfully" but events are **on and off** (intermittent delivery records), this is a strong indicator that Postmark has suppressed the recipient — emails leave SpotDraft but Postmark silently drops them.

**Another strong signal:** if support reports **no fresh email audit events**, **no SMTP / Django errors**, and the issue is isolated to **one recipient email**, check for a **prior hard bounce** in the Postmark alert channel. After the initial bounce, Postmark may suppress future sends without producing a new webhook or a useful application-side error.

**⚠️ Postmark server access is restricted by region.** Not all engineers have access to the Postmark US transactional server. Before attempting to check Postmark directly, confirm you have access. If you don't, escalate to someone who does (US server access is limited to specific team members — check with your team lead or DevOps).

- US cluster → Postmark US transactional server (restricted access — ask team lead)
- IN cluster → Postmark India transactional server
- Check in Postmark under Suppressions / Bounces for the recipient email

**Resolution if suppressed:** Reactivate the email on Postmark, then inform the customer/CSM to follow up with the end user that future emails will now be delivered. Note: reactivation only affects *future* emails — it does not resend previously missed ones.

If the customer is blocked on signing while reactivation is being handled, offer the **Get Link** workaround so they can proceed immediately while email delivery is being restored.

### Step 5: Check Customer Email Configuration

**Outlook/Exchange — Known Issue Pattern:**
Outlook and Exchange servers frequently block or delay SpotDraft emails. This is the #1 root cause after confirming emails were sent.

**Resolution:** Ask customer's IT team to **whitelist SpotDraft's email domain** (`spotdraft.com` and sending domains).

**Check spam/junk folders** — resolves ~20% of cases.

### Step 6: OTP-Specific Handling

OTPs have special behavior:
- OTP is **generated when the signing link is opened** (not before)
- OTP is **valid for 10 minutes**
- If counterparty opens the link but doesn't receive the OTP email, the OTP still exists in the system

**Emergency workaround (use sparingly):**
1. Ask counterparty to open the signing link
2. Have them inform support **immediately** once on the signing page
3. An engineer can retrieve and share the OTP — valid for 10 minutes
4. Requires Django shell access via Tray.ai

**Permanent fix:** Always push for domain whitelisting.

## Escalation Triggers
- Emails confirmed sent AND not suppressed AND customer has whitelisted → escalate to engineering
- Multiple customers report same issue simultaneously → likely SpotDraft-side email service problem, escalate as P1
- DLQ shows spike in `djcelery_email_send_multiple` failures → systemic email sending issue, escalate immediately

## Past Incidents to Reference
- **BIAL OTP issue** (Jan 2026): Outlook client, resolved by domain whitelisting
- **Ather Energy OTP** (Feb 2026): Same pattern — Outlook, domain whitelisting needed
- **Visit app email suppression** (Dec 2025): Email suppressed on Postmark, reactivated user
- **Doral — user not receiving emails** (Mar 2026, WSID 186781, Cluster US): User `cwalker@doral-llc.com` not receiving contract execution emails. Email audit showed emails sent but events "on and off". Postmark bot channel had alerts for the address. Root cause: user suppressed in Postmark (hard bounce / spam complaint). Fix: reactivated in Postmark. Took ~3 days due to restricted access to Postmark US server. Jira: [SPD-42048](https://spotdraft.atlassian.net/browse/SPD-42048). Incident: [#incident-20260306-medium-doral-user-not-receiving-emails](https://spotdraft.slack.com/archives/C0AK28S600J)
- **Fulcrum Digital — signatory not receiving email** (Mar 2026, WSID 184914, Cluster IN): Recipient `sathish@fulcrumdigital.com` had a historical hard bounce in Postmark alerts on Jan 23, 2026. During the incident there were no useful SMTP / app errors and no fresh delivery signal, but the recipient was still suppressed. Fix: reactivate in Postmark and ask the customer to resend; immediate unblock available via **Get Link**. Related tickets: SPD-22750 and Rootly #3028 / [#incident-20260323-medium-fulcrum-issue-in-receiving-email-for-signatory](https://spotdraft.slack.com/archives/C0AN4NW37B7)
