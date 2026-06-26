# DCA Local Login Setup (GitHub Pages)

Static site for the **DCA local participant login** setup guide.

**Live site (after you publish):** `https://<your-github-username>.github.io/dca-local-login-docs/`

---

## What's in this repo

| File | Purpose |
|------|---------|
| `docs/index.html` | Full setup guide (steps, downloads, Cursor prompt + Copy button) |
| `docs/DCA-Local-Login-Prerequisites.md` | Prerequisites checklist |
| `docs/DCA-Local-Full-Login-Setup.md` | Detailed troubleshooting |
| `docs/.nojekyll` | Serves static files as-is (no Jekyll processing) |

---

## Publish to GitHub Pages (first time)

### 1. Create the GitHub repo

1. Go to [https://github.com/new](https://github.com/new)
2. Repository name: **`dca-local-login-docs`** (or any name — update the Pages URL below)
3. Visibility: **Private** (recommended for internal dev docs) or **Public** per team policy
4. Do **not** add README, .gitignore, or license (you already have files locally)
5. Click **Create repository**

### 2. Push this folder to GitHub

Open PowerShell in this folder:

```powershell
cd C:\Users\anpaul\source\repos\dca-local-login-docs

git init
git add .
git commit -m "Initial DCA local login docs site"
git branch -M main
git remote add origin https://github.com/<YOUR-GITHUB-USERNAME>/dca-local-login-docs.git
git push -u origin main
```

Replace `<YOUR-GITHUB-USERNAME>` with your GitHub username or org name.

### 3. Enable GitHub Pages

1. On GitHub, open the repo → **Settings** → **Pages**
2. **Build and deployment** → Source: **Deploy from a branch**
3. Branch: **`main`** · Folder: **`/docs`**
4. Click **Save**
5. Wait 1–3 minutes. GitHub shows the site URL, e.g.  
   `https://<your-github-username>.github.io/dca-local-login-docs/`

### 4. Test the site

Open the Pages URL in Edge or Chrome:

- Setup steps render as a normal web page
- **Preview** / **Download** links work for the `.md` files
- **Copy** button on the Cursor prompt works

---

## Link from SharePoint (optional entry page)

On your SharePoint site page, set **Quick links** → **Open setup guide** to your GitHub Pages URL:

```
https://<your-github-username>.github.io/dca-local-login-docs/
```

Developers: SharePoint page → one click → full guide on GitHub Pages.

---

## Update the guide later

1. Edit files in `docs/` locally (source of truth can stay in `NewportDeveloperSetup\docs` — copy changes here when ready)
2. Commit and push:

```powershell
cd C:\Users\anpaul\source\repos\dca-local-login-docs
git add .
git commit -m "Update setup guide"
git push
```

GitHub Pages redeploys in 1–3 minutes.

---

## Private repo note

GitHub Pages on a **private** repo requires **GitHub Pro**, **Team**, or **Enterprise**. On a free account, use a **public** repo or keep the guide on SharePoint only.

---

## Tell developers

```
DCA local login setup:
https://<your-github-username>.github.io/dca-local-login-docs/

Follow the setup steps on that page.
Use the Cursor prompt at the bottom only if manual setup fails.
```
