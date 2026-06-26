# DCA local login — before you run the Cursor prompt

**Goal:** Login → OTP → dashboard at `http://dev.newportgroup.com/Participant/Dashboard`

Do these **once** before pasting the setup prompt into Cursor. Takes ~30–60 minutes the first time.

---

## 1. Open Cursor correctly

| Setting | Value |
|---------|--------|
| **Workspace** | Parent folder that contains **all four** repos as siblings (your path may differ): |
| | `DCA\` |
| | `UnifiedLogin\` |
| | `EnterpriseUserManagement\` |
| | `NewportDeveloperSetup\` |
| **Examples** | `C:\AscensusGit` · `C:\Users\<you>\source\repos` · `D:\dev\ascensus` |
| **Mode** | **Agent** (not Ask) |
| **OS** | Windows 10/11 |

---

## 2. One-time machine setup (Administrator PowerShell)

Run NewportDeveloperSetup menu **once** if you never did:

```powershell
cd <your-root>\NewportDeveloperSetup\Scripts
.\DeveloperMachineSetup.ps1
```

Run at least: **IIS config**, **Connection Strings**, **Public apps**, **Services**, **Host file entries**, **SSO certificate** (menu item 8).

If IIS still uses the **old single-site** layout (no `dev.pdxweb.com`), run:

```powershell
cd <your-root>\DCA\.localSetup\Scripts
.\Migrate-IIS-PdxwebYarp.ps1
```

---

## 3. Network & SQL (required for login submit)

| Requirement | Why |
|-------------|-----|
| **VPN** (if your team uses it) | Redis + SQL |
| **SQL reachable** | Login validates against EUM database |

Ask your team for the correct SQL server for **your** machine. Examples from past setups:

- Hostname: `devsql`
- Or IP: `10.7.20.49` (worked for one dev when `devsql` timed out)

Test (use what your team gives you):

```powershell
sqlcmd -S devsql -d EnterpriseUserManagement -E -Q "SELECT 1"
# or
sqlcmd -S 10.7.20.49 -d EnterpriseUserManagement -E -Q "SELECT 1"
```

Must return `1`. If not, fix VPN/SQL **before** the Cursor prompt.

---

## 4. App pool passwords (very common blocker)

Local IIS app pools often use **your AD account** (`RSD\<username>`). After password change or `iisreset`, pools show **Stopped** / **503**.

**Before the prompt**, run once as **Administrator**:

```powershell
cd <your-root>\DCA\.localSetup\Scripts
.\ResetAppPoolCredentials.ps1
# Enter your current Windows password when prompted
```

---

## 5. Pull latest docs & scripts

Ensure `NewportDeveloperSetup` has:

- `docs\DCA-Local-Full-Login-Setup.md`
- `Scripts\Fix-LocalSsoCertAndRestart.ps1`
- `Scripts\Start-DcaYarp.ps1`

If missing → `git pull` on `NewportDeveloperSetup`.

---

## 6. Test credentials (dev only)

| Field | Example |
|-------|---------|
| URL | `http://dev.newportgroup.com/login/participant` |
| User | `Demo40001` |
| Password | Ask team / dev creds doc |
| Captcha bypass (optional) | `?dc=0c4c30f07ec84f88ab6122605449cc41` |

Always use **`http://`**, not `https://`.

---

## 7. Keep an elevated PowerShell ready

Cursor will fix configs and builds, but **you** must run (as Admin):

- `iisreset /restart`
- `ResetAppPoolCredentials.ps1`
- `Migrate-IIS-PdxwebYarp.ps1`
- `Fix-LocalSsoCertAndRestart.ps1`
- `icacls` on project folders

When Cursor prints **ACTION REQUIRED**, run the commands and reply **done**.

---

## Known challenges (quick reference)

Use this if something breaks. Full detail is in `DCA-Local-Full-Login-Setup.md`.

| # | Symptom | Usual cause | Quick fix |
|---|---------|-------------|-----------|
| 1 | Straight to dashboard, no login | `AutoLogonPersonId` still on | `_AutoLogonPersonId` in PDXWeb `DeveloperAppSettings.json` |
| 2 | Redirect to QA login | QA URL in `Web.config` | Override `external-login-participant-url` in `DeveloperAppSettings.json` |
| 3 | 500.19 on dev.newportgroup | IIS can't read your repo folder | Admin: `icacls` on Yarp + UnifiedLogin paths |
| 4 | `/login/Error` | Frontend not built | `npm install` + `npm run build` in UnifiedLogin.FrontEnd.Mvc |
| 5 | Login submit generic error | WCF on devsvcs down | Build WCF projects + `ResetAppPoolCredentials.ps1` + `iisreset` |
| 6 | **503** on login or devsvcs | App pool stopped / bad AD password | `ResetAppPoolCredentials.ps1` + start pools |
| 7 | **504** on login | YARP timeout + slow Redis | YARP OutOfProcess + increase timeout; wait 15–20s first load |
| 8 | Login page "Error Encountered" | Redis config wrong shape | UnifiedLogin `DeveloperAppSettings.json` keys under `"AppSettings": { }` |
| 9 | Empty red error banner | EUM API can't reach SQL | VPN + correct SQL server in connection strings |
| 10 | "Incorrect credentials" but password is right | `EnterpriseWeb` SQL login denied | EUM API/WCF: use Integrated Security (`-E`) on dev |
| 11 | `dev.pdxweb.com` apps 503 | IIS not migrated to YARP layout | `Migrate-IIS-PdxwebYarp.ps1` |
| 12 | No OTP | EUM WcfService 500.31 | Kill orphan process, rebuild, recycle pool |
| 13 | Error after OTP | Missing SSO cert | `SetupLocalDevelopmentCertificate.ps1` or `Fix-LocalSsoCertAndRestart.ps1` |
| 14 | `?error=ssoFail` | SAML issuer/cert mismatch | EUM rule 4: cert **177**, issuer `https://dev.newportgroup.com/login` |
| 15 | 500.0 / 503 after OTP | YARP not built / pool down | Build DCA.Yarp + `Start-DcaYarp.ps1` |
| 16 | `devsql` in hosts → 127.0.0.1 | Wrong — breaks SQL | Remove `127.0.0.1 devsql` from hosts if present |

### IIS app pool names (devsvcs)

| IIS pool name | Project folder |
|---------------|----------------|
| `NGUnifiedLogin.WcfService` | `UnifiedLogin\Source\UnifiedLogin.WcfService` |
| `EnterpriseUserManagement.WcfService` | `EnterpriseUserManagement\Source\...WcfService` |
| `SSOIntegration.WcfService` | `UnifiedLogin\Source\SSOIntegration.WcfService` |
| `UnifiedLogin` | Login + UserProfile front ends |
| `DCA.Yarp` | YARP proxy |

---

## Then run the prompt

Only if the manual setup steps did not work:

1. On the **SharePoint page** → scroll to **Cursor prompt** → click **Copy** (or Ctrl+A in the box) → paste into Cursor Agent → reply **go**.

Optional: download `DCA-Local-Full-Login-Setup.md` from SharePoint and **@ attach** in Cursor.
One approval at the start; Cursor audits, fixes what it can, and gives you **one short list** of Admin steps at the end.
