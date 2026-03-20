---
name: google-oauth-redirect-uri-mismatch
description: "Diagnose and resolve Google OAuth login failures caused by redirect_uri_mismatch errors in SpotDraft. Use this skill when users cannot log in or sign up via Google OAuth and see 'Error 400: redirect_uri_mismatch' or 'This app doesn't comply with Google's OAuth 2.0 policy'. Also trigger on: double slash in redirect_uri (https://api.spotdraft.com//api/v1/...), 'google login not working', 'SSO login broken', 'OAuth callback rejected', 'redirect_uri not registered', Pydantic HttpUrl trailing-slash normalization side-effects, or any global Google login outage. Covers login, signup, and trial OAuth flows proxied through Oogway."
---

# Google OAuth redirect_uri_mismatch Triage

This skill covers global Google OAuth login failures where all users are blocked from logging in or signing up via Google. The failure is always at the Google OAuth layer — Google rejects the `redirect_uri` before the request ever reaches SpotDraft's backend.

## System Flow

```
User → Browser → Oogway (proxy) → Google OAuth (redirect_uri validation) → FAIL
                                           ↓ (on success)
                              Oogway oauth callback → Django cluster → App
```

Oogway acts as the OAuth proxy. It constructs the `redirect_uri` at:
- **Login**: `oogway/app/api/endpoints/proxies/oauth.py` line 34 (`/login`) and line 55 (`/callback`)
- **Signup**: `oogway/app/api/endpoints/proxies/oauth.py` line 108 (`/signup`) and line 129 (`/signup/callback`)

All four flows use:
```python
redirect_uri = f"{settings.OAUTH_REDIRECT_BASE_URL}/{path}"
```

`OAUTH_REDIRECT_BASE_URL` is defined in `oogway/app/core/config.py`:
```python
OAUTH_REDIRECT_BASE_URL: HttpUrl | None = None
```

## The Pydantic HttpUrl Trailing-Slash Trap

**Pydantic v2 changed `HttpUrl` behavior**: It now automatically appends a trailing slash to bare host URLs (e.g. `https://api.spotdraft.com` → `https://api.spotdraft.com/`).

When this happens, the f-string concatenation produces a **double slash**:
```
"https://api.spotdraft.com/" + "/" + "api/v1/sd_auth/oauth/callback"
= "https://api.spotdraft.com//api/v1/sd_auth/oauth/callback"   ← WRONG
```

Google's OAuth 2.0 only accepts URIs registered in Google Cloud Console. Only `https://api.spotdraft.com/api/v1/sd_auth/oauth/callback` (single slash) is registered, so Google rejects the request with `redirect_uri_mismatch`.

**When this gets triggered**: Any oogway deployment that upgrades Pydantic from v1 to v2 (or changes Pydantic v2 settings behavior) risks triggering this. The culprit is not the config value itself — the env var can be set correctly without a trailing slash and Pydantic v2 will still add one.

## Symptoms

- `Error 400: redirect_uri_mismatch` in the browser
- Error text: "You can't sign in to this app because it doesn't comply with Google's OAuth 2.0 policy"
- `redirect_uri=https://api.spotdraft.com//api/v1/sd_auth/oauth/callback` (double slash visible in Google error page)
- **Scope**: Global — affects ALL users attempting Google login or signup simultaneously
- **Not environment-specific**: Triggered only in the environment where oogway was deployed with the breaking config/version change

## Step-by-Step Investigation

### Step 1: Confirm the Error

Ask the support person for a screenshot or copy of the error. Confirm:
1. Is the error `redirect_uri_mismatch` from Google (not a SpotDraft 500)?
2. Is the `redirect_uri` in the error URL showing `//` (double slash)?
3. Is this affecting **all** Google login attempts or just a subset of users?

If all users are blocked globally → this is a Sev1/Critical.

### Step 2: Check Oogway Logs for the Malformed URI

Use Groundcover or GCP Cloud Logging to confirm oogway is generating the double-slash URI.

**GCP Cloud Logging — Prod oogway logs:**
```
logName="projects/spotdraft-prod/logs/stderr"
resource.labels.cluster_name="{cluster}"
labels."k8s-pod/app"="sd-apps-oogway"
jsonPayload.message:("redirect_uri" OR "oauth" OR "callback")
timestamp >= "{incident_start_time}"
```

**QA oogway logs:**
```
resource.type="k8s_container"
labels."k8s-pod/app"="spotdraft-qa-oogway"
textPayload:("redirect_uri" OR "oauth")
```

> Note: The error itself surfaces at Google's OAuth layer, not in oogway logs. Oogway logs will show normal traffic routing; the double-slash URI is what Google rejects before any callback reaches oogway.

### Step 3: Correlate with Recent Oogway Deployments

The double-slash bug is only introduced when oogway is **deployed** with a Pydantic v2 `HttpUrl` config type for `OAUTH_REDIRECT_BASE_URL`. Always check:

1. **When was oogway last deployed to prod?** Check the oogway deployment Slack channel (e.g. `#deployments` or `#oogway-releases`) or argo-manifests git history.
2. **What changed in that deployment?** Look at the argo-manifests commit. Specifically check if `oogway/app/core/config.py` or Pydantic version was changed.
3. **Did the outage start ~immediately after the deployment?** In the 2026-03-20 incident, oogway was deployed at 1:05 PM IST; first failure reports came at ~3:06 PM IST (some users noticed later).

**Check argo-manifests for recent oogway config changes:**
```bash
git log --oneline -- overlays/prod/oogway/
git show {commit_hash} -- overlays/prod/oogway/
```

Look for changes to:
- Pydantic package version in oogway's requirements
- `OAUTH_REDIRECT_BASE_URL` value in the argo config
- Any "version upgrade" commit touching oogway

### Step 4: Immediate Mitigation

**Option A (fastest): Revert the oogway argo-manifests commit**

1. Identify the culprit commit in `SpotDraft/argo-manifests` (the most recent oogway deployment commit)
2. Create a revert PR targeting that commit
3. Get it approved and merged
4. Verify Google login works after rollout (~2-5 minutes for argo to sync)

```bash
git revert {culprit_commit_hash}
# create PR and merge
```

**Option B (temporary bandaid): Add the double-slash URI to Google Cloud Console**

This unblocks users immediately while the proper fix is prepared, but must be cleaned up afterward:
1. Go to Google Cloud Console → APIs & Services → Credentials → OAuth 2.0 Client IDs
2. Find the SpotDraft app's client ID (`1064216506250-ofrj84fhk7bf640fqcqimdujaqn8v69t`)
3. Add `https://api.spotdraft.com//api/v1/sd_auth/oauth/callback` (double slash) to Authorized Redirect URIs
4. **Remove this after the proper fix is deployed**

> The 2026-03-20 incident used Option A (revert of argo-manifests PR #1190) as primary mitigation. Option B was briefly considered as a bandaid.

### Step 5: Verify Fix

After the revert/fix is deployed, ask affected users to retry Google login. Check:
- Login flow works end-to-end
- Signup flow works
- Trial signup flow works
- Microsoft OAuth (if affected too) — the same config pattern applies to `MICROSOFT_OAUTH_CLIENT_ID`

## Long-Term / Permanent Fix

The root cause is `OAUTH_REDIRECT_BASE_URL: HttpUrl | None = None` in `oogway/app/core/config.py`. Pydantic v2 mutates the URL by appending a trailing slash.

**Fix 1 — Change type to `str` (simplest):**
```python
# oogway/app/core/config.py
OAUTH_REDIRECT_BASE_URL: str | None = None  # was: HttpUrl | None = None
```
This stops Pydantic from normalizing the URL. The f-string concatenation then produces a single slash correctly.

**Fix 2 — Defensive URL construction (belt-and-suspenders):**
```python
# oogway/app/api/endpoints/proxies/oauth.py
base = str(settings.OAUTH_REDIRECT_BASE_URL).rstrip("/")
redirect_uri = f"{base}/{ROUTER_OAUTH_LOGIN_CALLBACK_PATH}"
```

**Fix 3 — Pydantic v2 path operator (idiomatic):**
```python
redirect_uri = str(settings.OAUTH_REDIRECT_BASE_URL / path)
```
Pydantic v2's `HttpUrl` supports `/` for path joining (like `pathlib.Path`), which handles trailing slashes correctly.

**Recommendation**: Apply Fix 1 in `config.py` (type change to `str`) AND Fix 2 (defensive `rstrip`) in `oauth.py` for defense in depth. PR: https://github.com/SpotDraft/oogway/pull/131

## Process Gap: Oogway Deployments Skip QA Regression

The 2026-03-20 incident revealed that oogway deployments were going directly to prod without QA regression testing — unlike Django deployments. QAs were not running regression on oogway. Google OAuth login is in the regression suite, but was not caught because oogway changes were not in scope.

**Corrective action agreed**: Oogway to follow the same release structure as Django (staged QA → prod deployment with regression sign-off).

## Known Code Paths

| Flow | File | Line | Path constant |
|------|------|------|---------------|
| Login redirect | `oogway/app/api/endpoints/proxies/oauth.py` | 34 | `ROUTER_OAUTH_LOGIN_CALLBACK_PATH = "api/v1/sd_auth/oauth/callback"` |
| Login callback | `oogway/app/api/endpoints/proxies/oauth.py` | 55 | same |
| Signup redirect | `oogway/app/api/endpoints/proxies/oauth.py` | 108 | `ROUTER_OAUTH_SIGNUP_CALLBACK_PATH = "api/v1/sd_auth/oauth/signup/callback"` |
| Signup callback | `oogway/app/api/endpoints/proxies/oauth.py` | 129 | same |
| Config | `oogway/app/core/config.py` | 45 | `OAUTH_REDIRECT_BASE_URL: HttpUrl | None = None` |

## Confirming vs. Ruling Out

| Signal | Confirms this issue | Rules it out |
|--------|--------------------|-|
| Double slash in `redirect_uri` in Google error page | ✅ | |
| `redirect_uri_mismatch` error from Google | Consistent with | Could also be wrong domain/path altogether |
| Recent oogway deployment | ✅ Strong signal | |
| Error affects only specific users/workspace | | ❌ (this bug is global) |
| Error affects only Microsoft OAuth, not Google | | ❌ |
| Error is a SpotDraft 500 (not a Google 400) | | ❌ Different issue |
| oogway logs show `500` or exception | | ❌ This bug never reaches oogway |

## Escalation Triggers

- Global Google OAuth outage (all users affected) → Sev1/Critical immediately
- Fix not deployed within 15 minutes → escalate to Engineering Lead + CTO
- Both Option A and Option B fail to restore login → escalate to Google Cloud Console admin access

## Related Incidents / Jira

- **2026-03-20**: Global Google login outage. Oogway deployment at 1:05 PM IST included Pydantic v2 version upgrade changes. Outage from ~15:06 to ~15:48 IST. Customers confirmed affected: Phonepe, Pratilipi (global blast radius). Mitigated by reverting argo-manifests commit `6fa8925a87de822ff84d6ea0e8044e356b313875` via PR #1190. Jira: [SPD-42603](https://spotdraft.atlassian.net/browse/SPD-42603). Long-term fix PR: [oogway#131](https://github.com/SpotDraft/oogway/pull/131).

## Related Skills

- `infrastructure-alert-response` — if the OAuth failure is accompanied by infra alerts (pod crashes, high CPU) rather than a config/code change
- `incident-lookup` — if symptoms don't match the double-slash pattern, search Slack/Jira for prior incidents with similar error messages
