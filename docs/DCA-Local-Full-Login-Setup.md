# DCA Local Full Login Setup Guide

This guide walks through setting up the **full participant login flow** on a local Windows dev machine:

**Login screen → username/password → OTP/MFA → SAML SSO → DCA dashboard**

Use **Dev local** (`dev.newportgroup.com`), not QA. Always browse with **`http://`**, not `https://` — there is no local HTTPS binding for these sites.

---

## Table of contents

1. [What you are building](#what-you-are-building)
2. [Prerequisites](#prerequisites)
3. [Architecture overview](#architecture-overview)
4. [Setup steps](#setup-steps)
5. [Verify the flow](#verify-the-flow)
6. [Helper scripts](#helper-scripts)
7. [Important notes for the team](#important-notes-for-the-team)
8. [Troubleshooting — errors and fixes](#troubleshooting--errors-and-fixes)

---

## What you are building

By default, local DCA **skips login** using developer shortcuts (`AutoLogonPersonId`, `LocalUserSession`). QA does not use those, so login there looks different.

This guide turns those shortcuts **off** and wires up every layer needed for a real login:

| Step | What happens |
|------|----------------|
| 1 | User visits DCA → redirected to Unified Login |
| 2 | User enters username/password |
| 3 | Unified Login calls backend WCF services on `devsvcs` |
| 4 | OTP/MFA screen appears (may use `/userprofile`) |
| 5 | After OTP, SSO Integration generates a SAML assertion |
| 6 | Browser posts SAML to `dev.newportgroup.com/common/login/participant` |
| 7 | YARP proxies to PDXWeb on `dev.pdxweb.com` |
| 8 | PDXWeb validates SAML and creates the session → **Dashboard** |

**Success looks like:** `http://dev.newportgroup.com/Participant/Dashboard`

---

## Prerequisites

- Windows 10/11 with **IIS** enabled
- **Administrative PowerShell** for IIS, hosts file, and certificate steps
- Repos cloned locally (adjust paths to your machine):
  - `DCA`
  - `UnifiedLogin`
  - `EnterpriseUserManagement`
  - `NewportDeveloperSetup`
- **.NET SDK** (8.x for YARP; 6.x for some WCF services)
- **Node.js 18 or 20** for Unified Login frontend build (Node 22 may work but is not officially supported)
- Network access to **dev SQL** (`devsql`) for EUM database
- VPN / corp network if required for Redis and shared dev services

### Confluence references

- [Unified Login - Developer Setup](https://confluence.ascensus.com/display/GLOBAL/Unified+Login+-+Developer+Setup)
- [How to run UnifiedLogin and EUM locally](https://confluence.ascensus.com/display/GLOBAL/How+to+run+UnifiedLogin+and+EUM+locally)
- [DCA - Developer Setup](https://confluence.ascensus.com/display/GLOBAL/DCA+-+Developer+Setup)
- [How to Generate a Certificate for SSO](https://confluence.ascensus.com/display/GLOBAL/How+to+Generate+a+Certificate+for+SSO)

---

## Architecture overview

```
dev.newportgroup.com  (127.0.0.1:80)   ← browser entry point
├── /              → DCA.Yarp           (app pool: DCA.Yarp)
├── /login         → UnifiedLogin       (app pool: UnifiedLogin)
└── /userprofile   → EUM User Profile   (app pool: UnifiedLogin)  ← MFA steps

dev.pdxweb.com  (127.0.0.2:80)         ← YARP backend (actual DCA apps)
├── /              → PDXWeb             (app pool: DCA)
├── /login         → UnifiedLogin
├── /userprofile   → EUM User Profile
├── /app           → DCAAngular
├── /legacy        → LegacyPD
└── /reporting     → PDXReportingWeb

devsvcs.newportgroup.com  (127.0.0.1)  ← backend WCF/API services
├── NGUnifiedLogin.WcfService
├── EnterpriseUserManagement.WcfService
├── SSOIntegration.WcfService
└── EnterpriseUserManagement.Api
```

YARP forwards most traffic from `dev.newportgroup.com` to `http://dev.pdxweb.com` (configured in `DCA\MFE\DCA.Yarp\appsettings.json`).

### Hosts file (required)

| IP | Hostname |
|----|----------|
| `127.0.0.1` | `dev.newportgroup.com`, `devsvcs.newportgroup.com`, brand subdomains |
| `127.0.0.2` | `dev.pdxweb.com` |

Run `NewportDeveloperSetup\Scripts\SetupHostFileEntries.ps1` for the standard entries. Ensure `dev.pdxweb.com` → `127.0.0.2` exists (added by `DCA\.localSetup\Scripts\SetupDCA.ps1` or `Migrate-IIS-PdxwebYarp.ps1`).

---

## Setup steps

### Step 1 — Run base machine setup (once)

Open **Administrator PowerShell**:

```powershell
cd C:\Users\<you>\source\repos\NewportDeveloperSetup\Scripts
.\DeveloperMachineSetup.ps1
```

From the menu, run at minimum:

1. Setup IIS Server Configuration
2. Setup Global Connection Strings
3. Setup Public Facing WebApps (DCA, Unified Login)
4. Setup Services (EUM, Unified Login Services)
5. Setup Host File Entries
6. **Install Local Development SSO Certificate** (menu item 8)

If you already have the old single-site IIS layout, migrate first:

```powershell
cd C:\Users\<you>\source\repos\DCA\.localSetup\Scripts
.\Migrate-IIS-PdxwebYarp.ps1
```

### Step 2 — Confirm IIS layout

**`dev.newportgroup.com`** must have:

| Path | Physical path | App pool |
|------|---------------|----------|
| `/` | `...\DCA\MFE\DCA.Yarp` | `DCA.Yarp` |
| `/login` | `...\UnifiedLogin\Source\UnifiedLogin.FrontEnd.Mvc` | `UnifiedLogin` |
| `/userprofile` | `...\EnterpriseUserManagement\Source\EnterpriseUserManagement.UserProfile.Mvc` | `UnifiedLogin` |

**`dev.pdxweb.com`** must have PDXWeb at `/` plus `/login`, `/userprofile`, `/app`, `/legacy`, `/reporting`.

If `/userprofile` is missing on `dev.newportgroup.com`, add it (same as `SetupUnifiedLogin.ps1`):

```powershell
# Example — adjust paths to your clone location
New-WebApplication -Name "userprofile" `
  -Site "dev.newportgroup.com" `
  -PhysicalPath "C:\Users\<you>\source\repos\EnterpriseUserManagement\Source\EnterpriseUserManagement.UserProfile.Mvc" `
  -ApplicationPool "UnifiedLogin" -Force
```

### Step 3 — Grant IIS read access to your source folders

IIS app pools must read projects under your user profile. Run **as Administrator**:

```powershell
icacls "C:\Users\<you>\source\repos\DCA\MFE\DCA.Yarp" /grant "IIS_IUSRS:(OI)(CI)RX" /T
icacls "C:\Users\<you>\source\repos\UnifiedLogin\Source\UnifiedLogin.FrontEnd.Mvc" /grant "IIS_IUSRS:(OI)(CI)RX" /T
icacls "C:\Users\<you>\source\repos\EnterpriseUserManagement\Source\EnterpriseUserManagement.UserProfile.Mvc" /grant "IIS_IUSRS:(OI)(CI)RX" /T
```

### Step 4 — Disable auto-login bypasses

**PDXWeb** — edit `DCA\PDX\PDXWeb\DeveloperAppSettings.json`:

Disable `AutoLogonPersonId` by renaming it with a leading underscore (JSON does not support comments on the same key):

```json
{
  "AppSettings": {
    "_AutoLogonPersonId": "10885513",
    "_AutoLogonSponsorUserId": "8319002"
  }
}
```

**Optional — MFEs only:** If testing MFE routes directly, set `LocalUserSession.Enabled` to `false` in each MFE's `appsettings.Development.json`. Not required for dashboard-only login testing.

### Step 5 — Point PDXWeb at local Unified Login (not QA)

`Web.config` defaults to QA URLs. Override in `DeveloperAppSettings.json`:

```json
{
  "AppSettings": {
    "_AutoLogonPersonId": "",
    "external-login-participant-url": "http://dev.newportgroup.com/login/Participant",
    "external-login-sponsor-url": "http://dev.newportgroup.com/login/Sponsor"
  }
}
```

After disabling auto-login, visiting DCA should redirect to `http://dev.newportgroup.com/login/participant` — **not** `qa.newportgroup.com`.

### Step 6 — Build Unified Login frontend

```powershell
cd C:\Users\<you>\source\repos\UnifiedLogin\Source\UnifiedLogin.FrontEnd.Mvc
npm install
npm run build
```

Confirm `wwwroot\icon-sprite.svg` exists after the build.

### Step 7 — Build backend WCF services

```powershell
dotnet build C:\Users\<you>\source\repos\UnifiedLogin\Source\UnifiedLogin.WcfService\UnifiedLogin.WcfService.csproj -c Debug
dotnet build C:\Users\<you>\source\repos\EnterpriseUserManagement\Source\EnterpriseUserManagement.WcfService\EnterpriseUserManagement.WcfService.csproj -c Debug
dotnet build C:\Users\<you>\source\repos\UnifiedLogin\Source\SSOIntegration.WcfService\SSOIntegration.WcfService.csproj -c Debug
```

### Step 8 — Build DCA.Yarp

```powershell
dotnet build C:\Users\<you>\source\repos\DCA\MFE\DCA.Yarp\DCA.Yarp.csproj -c Debug
```

Confirm this file exists:

`DCA\MFE\DCA.Yarp\bin\Debug\net8.0\DCA.Yarp.exe`

YARP `web.config` should use **OutOfProcess** hosting (already set in repo):

```xml
<aspNetCore processPath=".\bin\Debug\net8.0\DCA.Yarp.exe" hostingModel="OutOfProcess" ... />
```

### Step 9 — Install local SSO certificate

**As Administrator:**

```powershell
cd C:\Users\<you>\source\repos\NewportDeveloperSetup\Scripts
.\SetupLocalDevelopmentCertificate.ps1
```

This installs `Support\dev-local.newportgroup.com.pfx` to `LocalMachine\My` and grants `IIS_IUSRS` read access to the private key.

**Certificate details:**

| Field | Value |
|-------|-------|
| Subject | `CN=dev-local.newportgroup.com` |
| CertificateId (EUM DB) | **177** |
| Thumbprint | `F688CF58860E7DA30EA70620EE2700B2C610DCA1` |

### Step 10 — Configure EUM SSO database rules (shared devsql)

> **Team warning:** These updates are on the **shared** `EnterpriseUserManagement` database on `devsql`. Coordinate with your team. Other devs doing local SSO need the same values; revert if switching back to QA issuer/certs.

Run on `devsql` (Windows auth):

```sql
-- UL → DCA SAML generation (EnterpriseSSOGenerationProcessingId = 4)
UPDATE EnterpriseSSOGenerationProcessing
SET SignAssertionCertificateId = 177,
    EncryptAssertionCertificateId = 177,
    IssuerId = 'https://dev.newportgroup.com/login'
WHERE EnterpriseSSOGenerationProcessingId = 4;
```

**Why both cert and issuer matter:**

- **Cert 177** — SSO Integration signs SAML with `dev-local.newportgroup.com`
- **Dev issuer** — PDXWeb looks up validation rule **129** (`Unified Login (Local Development)`) which expects cert 177, not QA certs 204/190

Validation rule 129 already exists in EUM:

```
Incoming issuer: https://dev.newportgroup.com/login
Verify cert:     177 (dev-local)
Decrypt cert:    177 (dev-local)
```

### Step 11 — Restart IIS

**As Administrator:**

```powershell
iisreset /restart
```

Or use the all-in-one script (Step 12).

### Step 12 — Run the all-in-one fix script (recommended)

After initial setup — or whenever SSO/YARP breaks — run **as Administrator**:

```powershell
& "C:\Users\<you>\source\repos\NewportDeveloperSetup\Scripts\Fix-LocalSsoCertAndRestart.ps1"
```

This script:

1. Installs/grants access to the dev-local SSO cert
2. Updates EUM SSO generation rule (cert 177 + dev issuer)
3. Builds DCA.Yarp
4. Recycles `DCA.Yarp`, `SSOIntegration.WcfService`, `UnifiedLogin`, `NGUnifiedLogin.WcfService` app pools
5. Runs `iisreset /restart`

For **YARP-only** issues:

```powershell
& "C:\Users\<you>\source\repos\NewportDeveloperSetup\Scripts\Start-DcaYarp.ps1"
```

---

## Verify the flow

### Quick URL checks

| URL | Expected |
|-----|----------|
| `http://dev.newportgroup.com/login/participant` | Login page (200) |
| `http://dev.newportgroup.com/common/login/participant` | 302 redirect (via YARP → PDXWeb) |
| `http://dev.pdxweb.com/common/login/participant` | 302 redirect (PDXWeb direct) |
| `http://devsvcs.newportgroup.com/EnterpriseUserManagement.Api/` | 200 |

### Full login test

1. Open `http://dev.newportgroup.com/Participant/Dashboard` (or any protected DCA page)
2. You should land on **Participant Login** at `dev.newportgroup.com/login/participant`
3. Enter dev test credentials (see your team's dev user list; e.g. `Demo40001`)
4. Complete OTP/MFA
5. Land on **Participant Dashboard**

### Optional — bypass captcha during dev testing

Append the disable-captcha query key to the login URL (dev only):

```
http://dev.newportgroup.com/login/participant?dc=0c4c30f07ec84f88ab6122605449cc41
```

### Activity logs (when things fail)

| Log folder | Used for |
|------------|----------|
| `C:\ActivityLog\UnifiedLogin\` | Login, OTP, SSO generation |
| `C:\ActivityLog\Services\` | WCF service calls |
| `C:\ActivityLog\PdxWeb\` | SAML consumption on DCA (look for `PD_SSO_CLIENTSSO_DENIALREASON`) |

---

## Helper scripts

| Script | Purpose |
|--------|---------|
| `NewportDeveloperSetup\Scripts\DeveloperMachineSetup.ps1` | Main setup menu |
| `NewportDeveloperSetup\Scripts\SetupHostFileEntries.ps1` | Hosts file entries |
| `NewportDeveloperSetup\Scripts\SetupLocalDevelopmentCertificate.ps1` | Install dev-local SSO cert |
| `NewportDeveloperSetup\Scripts\Fix-LocalSsoCertAndRestart.ps1` | **All-in-one** cert + EUM + YARP + IIS restart |
| `NewportDeveloperSetup\Scripts\Start-DcaYarp.ps1` | Build YARP + IIS restart (503 fixes) |
| `DCA\.localSetup\Scripts\Migrate-IIS-PdxwebYarp.ps1` | Migrate old IIS layout to YARP |

---

## Important notes for the team

1. **Always use `http://`** — `https://dev.newportgroup.com` will fail (no local HTTPS binding).
2. **Shared EUM DB** — the SQL update in Step 10 affects all devs on `devsql`. Document when you change it.
3. **Rebuild after pulls** — frontend (`npm run build`), WCF services, and YARP may need rebuilding after code changes.
4. **Recycle IIS** after builds — especially `devsvcs`, `UnifiedLogin`, `DCA.Yarp`, and `DCA` app pools.
5. **Auto-login** — re-enabling `_AutoLogonPersonId` skips the login screen entirely.
6. **Node version** — prefer Node 18 or 20 for Unified Login frontend builds.

---

## Troubleshooting — errors and fixes

Use this section when something breaks. Find your symptom, apply the fix, then retry the login flow.

---

### Login is skipped — I go straight to the dashboard

**Symptom:** No login screen; dashboard loads immediately.

**Cause:** `AutoLogonPersonId` is active in `DCA\PDX\PDXWeb\DeveloperAppSettings.json`.

**Fix:** Rename to `_AutoLogonPersonId` (or clear the value). Recycle the `DCA` app pool.

---

### Redirect goes to QA instead of local login

**Symptom:** Browser opens `https://qa.newportgroup.com/login/...` instead of `dev.newportgroup.com`.

**Cause:** `external-login-participant-url` in `Web.config` points to QA; not overridden in `DeveloperAppSettings.json`.

**Fix:** Add to `DeveloperAppSettings.json`:

```json
"external-login-participant-url": "http://dev.newportgroup.com/login/Participant"
```

---

### HTTP 500.19 — Cannot read configuration file (0x80070005)

**Symptom:** IIS error page: *HTTP Error 500.19 — Configuration file is not well-formed* or access denied when visiting `dev.newportgroup.com`.

**Cause:** IIS app pool identity cannot read `DCA.Yarp\web.config` (or other apps) under your user folder.

**Fix:** Run `icacls` grants from Step 3 (as Administrator). Recycle the `DCA.Yarp` app pool.

---

### HTTP 500.0 — ASP.NET Core IIS hosting failure (in-process)

**Symptom:** After OTP, URL like `dev.newportgroup.com/common/login/participant` shows *HTTP Error 500.0 - ASP.NET Core IIS hosting failure (in-process)*.

**Cause:** `DCA.Yarp.exe` missing — project not built.

**Event log:**

```
Executable was not found at '...\DCA.Yarp\bin\Debug\net8.0\DCA.Yarp.exe'
```

**Fix:**

```powershell
dotnet build C:\Users\<you>\source\repos\DCA\MFE\DCA.Yarp\DCA.Yarp.csproj -c Debug
iisreset /restart   # as Administrator
```

---

### HTTP 503 — Service Unavailable / Application Shutting Down

**Symptom:** `dev.newportgroup.com` returns 503; browser may show *This page isn't working* / *HTTP ERROR 503*.

**Cause:** `DCA.Yarp` app pool stopped after repeated startup failures (rapid-fail protection), often after the missing-exe issue above.

**Fix:** Run `Start-DcaYarp.ps1` as Administrator (builds YARP, `iisreset`, starts app pool and site). Confirm YARP uses **OutOfProcess** in `web.config`.

**Verify:**

```powershell
curl.exe -s -o NUL -w "%{http_code}" http://dev.newportgroup.com/common/login/participant
# Expect: 302
```

---

### `/login/Error` or missing page after switching app pool

**Symptom:** Unified Login redirects to `/login/Error` immediately.

**Cause:** `wwwroot` frontend assets not built.

**Event log:**

```
DirectoryNotFoundException: ...\UnifiedLogin.FrontEnd.Mvc\wwwroot\icon-sprite.svg
```

**Fix:**

```powershell
cd UnifiedLogin\Source\UnifiedLogin.FrontEnd.Mvc
npm install
npm run build
```

---

### Login page loads but submit fails — "An error occurred while processing your request"

**Symptom:** Username/password submit shows a generic error on the login form.

**Cause:** Backend WCF services on `devsvcs` not built or not running.

**Services involved:**

| Service | Typical failure |
|---------|-----------------|
| `NGUnifiedLogin.WcfService` | 500 — missing DLLs in `bin` |
| `EnterpriseUserManagement.WcfService` | 500 — not built |
| `SSOIntegration.WcfService` | 500 — not built |

**Fix:** Build all WCF projects (Step 7), then:

```powershell
iisreset /restart   # as Administrator
```

---

### HTTP 500.31 — Failed to load ASP.NET Core runtime (EUM WcfService)

**Symptom:** `EnterpriseUserManagement.WcfService` returns 500.31; OTP screen never appears.

**Cause:** `EnterpriseUserManagement.WcfService.runtimeconfig.json` missing from build output (clean build, failed build, or file lock).

**Fix:**

1. Stop orphan processes:

```powershell
Get-Process -Name "EnterpriseUserManagement.WcfService" -ErrorAction SilentlyContinue | Stop-Process -Force
```

2. Rebuild:

```powershell
dotnet build EnterpriseUserManagement\Source\EnterpriseUserManagement.WcfService\EnterpriseUserManagement.WcfService.csproj -c Debug
```

3. `iisreset /restart`

---

### OTP screen never appears (login fails at username/password)

**Symptom:** Credentials rejected or generic error before MFA/OTP.

**Cause:** Usually `EnterpriseUserManagement.WcfService` down (see 500.31 above) or EUM database connectivity.

**Fix:** Fix EUM WcfService first. Confirm `http://devsvcs.newportgroup.com/EnterpriseUserManagement.Api/` returns 200.

---

### `/userprofile` returns 500

**Symptom:** MFA/profile steps fail; `/userprofile` errors.

**Cause:** Missing IIS app on `dev.newportgroup.com`, or EUM User Profile app pool/permissions issue.

**Fix:**

1. Add `/userprofile` application under `dev.newportgroup.com` (Step 2)
2. Grant `icacls` on UserProfile.Mvc folder
3. Ensure app pool is `UnifiedLogin`

---

### After OTP — generic "Error Encountered" page

**Symptom:** OTP accepted, then a generic Unified Login error page (not `ssoFail`).

**Cause:** SSO Integration failed to **generate** SAML — missing signing certificate.

**Activity log (`C:\ActivityLog\UnifiedLogin\`):**

```
could not find signing certificate with thumbprint (2b34a13beca0af3a27c6b22271cdaf40b38c6447)
```

The EUM rule was pointing at QA cert **204** (expired), not dev-local cert **177**.

**Fix:**

1. Run `SetupLocalDevelopmentCertificate.ps1` or `Fix-LocalSsoCertAndRestart.ps1`
2. Confirm EUM DB rule 4 uses cert **177**:

```sql
SELECT SignAssertionCertificateId, EncryptAssertionCertificateId, IssuerId
FROM EnterpriseSSOGenerationProcessing
WHERE EnterpriseSSOGenerationProcessingId = 4;
-- Expect: 177, 177, https://dev.newportgroup.com/login
```

3. Recycle `SSOIntegration.WcfService` app pool

---

### After OTP — `?error=ssoFail` on login page

**Symptom:** Redirected to `dev.newportgroup.com/login/participant?error=ssoFail` with message **"Single Sign On attempt failed."**

**Cause:** SAML was generated but **PDXWeb rejected** it — signature verification failed.

**Activity log (`C:\ActivityLog\PdxWeb\`):**

```
PD_SSO_CLIENTSSO_DENIALREASON: Invalid assertion - Signature Not Verified
... certificates used: *.qa.newportgroup.com (thumbprint 2b34a13b..., 28172a6c...)
```

SAML was signed with **dev-local cert 177**, but the SAML **Issuer** was still `https://qa.newportgroup.com/login`, so PDXWeb used QA validation certs instead of dev-local rule 129.

**Fix:** Update issuer in EUM (Step 10):

```sql
UPDATE EnterpriseSSOGenerationProcessing
SET IssuerId = 'https://dev.newportgroup.com/login'
WHERE EnterpriseSSOGenerationProcessingId = 4;
```

Run `Fix-LocalSsoCertAndRestart.ps1` to apply cert + issuer + recycle IIS.

**Verify in PDXWeb log:** Issuer in SAML should be `https://dev.newportgroup.com/login` and denial reason should be gone.

---

### HTTPS URL fails (404 or connection error)

**Symptom:** Browser uses `https://dev.newportgroup.com/...` and fails.

**Cause:** No HTTPS binding configured for local dev sites.

**Fix:** Use **`http://`** explicitly. SSO postbacks and redirects expect HTTP locally.

---

### Forms authentication expired ticket warnings

**Symptom:** Event log shows `Forms authentication failed ... ticket supplied has expired` on `dev.pdxweb.com`.

**Cause:** Stale auth cookies from prior sessions (usually harmless during setup).

**Fix:** Clear browser cookies for `dev.newportgroup.com` and `dev.pdxweb.com`, or use a private/incognito window.

---

### YARP works for `/login` but not `/common/login/participant`

**Symptom:** `/login/participant` returns 200; `/common/login/participant` returns 503/500.

**Cause:** YARP app pool down or not built; post-OTP SAML posts to `/common/login/participant` which hits YARP root.

**Fix:** Build YARP + run `Start-DcaYarp.ps1` or `Fix-LocalSsoCertAndRestart.ps1`.

---

### EUM WcfService rebuild blocked — file in use

**Symptom:** `dotnet build` fails; cannot overwrite `EnterpriseUserManagement.WcfService.exe`.

**Cause:** Orphan `w3wp` or standalone process holding the file.

**Fix:**

```powershell
Get-Process -Name "EnterpriseUserManagement.WcfService","w3wp" -ErrorAction SilentlyContinue | Stop-Process -Force
# Then rebuild and iisreset
```

---

### Node/npm build warnings or failures

**Symptom:** `npm run build` fails or produces incomplete `wwwroot`.

**Cause:** Wrong Node version (project expects 18 or 20).

**Fix:** Install Node 18 or 20 via nvm-windows, then re-run `npm install` and `npm run build`.

---

### Still stuck?

1. Check the **newest** file in `C:\ActivityLog\PdxWeb\` and `C:\ActivityLog\UnifiedLogin\`
2. Check **Windows Event Viewer → Application** for `IIS AspNetCore Module` or `ASP.NET` errors
3. Run `Fix-LocalSsoCertAndRestart.ps1` as Administrator (fixes most post-OTP issues)
4. Confirm all URL checks in [Verify the flow](#verify-the-flow) pass before attempting login

---

## Quick recovery checklist

When login breaks after a reboot, pull, or IIS change, run through this list:

- [ ] `dev.newportgroup.com` and `dev.pdxweb.com` resolve in hosts file
- [ ] `_AutoLogonPersonId` is disabled in PDXWeb `DeveloperAppSettings.json`
- [ ] `external-login-participant-url` points to `dev.newportgroup.com`
- [ ] Unified Login `wwwroot` exists (`npm run build`)
- [ ] WCF services built (`NGUnifiedLogin`, `EUM.WcfService`, `SSOIntegration`)
- [ ] `DCA.Yarp.exe` exists in `bin\Debug\net8.0\`
- [ ] Dev-local cert installed; EUM rule 4 uses cert 177 + dev issuer
- [ ] `iisreset /restart` (Administrator)
- [ ] Test: `http://dev.newportgroup.com/login/participant` → login page
- [ ] Test: `http://dev.newportgroup.com/common/login/participant` → 302

---

*Document created from the local DCA full-login setup session (June 2026). Update this file when the setup process changes.*
