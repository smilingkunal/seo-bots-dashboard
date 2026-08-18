# Dashboard Deployment Guide

## Files Generated

```
~/.hermes/seo-bots/dashboard/
├── public/
│   ├── index.html               # The dashboard (42 KB, single-file SPA)
│   └── data/                     # JSON data files
│       ├── bots-status.json      # 1.6 KB — fleet health
│       ├── gsc-data.json         # 2.2 KB — top queries, pages, opportunities
│       ├── ranks-data.json       # 1.9 KB — keyword positions + SERP
│       ├── ga4-data.json         # 318 B — placeholder
│       ├── gbp-data.json         # 311 B — placeholder
│       ├── opportunities.json    # 2.4 KB — 3 prioritized actions
│       ├── approvals.json        # 348 B — empty queue
│       └── audit-log.json        # 1.9 KB — 7 historical actions
├── README.md
└── DEPLOYMENT.md
```

## Quick Test Locally

```bash
cd ~/.hermes/seo-bots/dashboard/public
python -m http.server 8080
# Open: http://localhost:8080
```

You should see all 8 tabs populated with the real GSC + DataForSEO data we already captured.

## GitHub Pages Deployment (5 minutes)

### Step 1 — Create the repo

Go to https://github.com/new and create a **public** repo:
- Name: `seo-bots-dashboard`
- Description: "Live SEO dashboard for weedistillery.com"
- Public
- **Do NOT** initialize with README

### Step 2 — Push from your machine

Open a terminal and run these commands (replace `YOUR_GITHUB_USERNAME` with your actual GitHub username):

```bash
cd ~/.hermes/seo-bots/dashboard
git init
git config user.email "kunal@weedistillery.com"
git config user.name "SEO Bot Dashboard"

# Only push the public/ folder as the site root
git add public/
git commit -m "Initial dashboard deployment"

git branch -M main
git remote add origin https://github.com/YOUR_GITHUB_USERNAME/seo-bots-dashboard.git
git push -u origin main
```

### Step 3 — Enable Pages

1. Go to your repo → **Settings** → **Pages**
2. Source: **Deploy from a branch**
3. Branch: **main**, folder: **/public**
4. Click **Save**
5. Wait 1-2 minutes for first deploy
6. Visit: `https://YOUR_GITHUB_USERNAME.github.io/seo-bots-dashboard/`

### Step 4 — Set up auto-deploy (optional)

To have the dashboard update automatically when cron jobs run new data:

```bash
# Add a personal access token (PAT) for cron to use
# GitHub → Settings → Developer settings → Personal access tokens
# Scope: repo (or just public_repo)

# Save token in .env
echo 'GITHUB_DEPLOY_TOKEN=ghp_your_token_here' >> ~/.hermes/.env
echo 'GITHUB_DEPLOY_REPO=kunalprime/seo-bots-dashboard' >> ~/.hermes/.env

# Add a deploy script that runs after every cron job (covered in update_dashboard.py)
```

## Updating Data After Deployment

Each bot writes JSON files. To deploy updates to GitHub Pages:

### Option A — Manual push (after every cron run)

```bash
cd ~/.hermes/seo-bots/dashboard
git add public/data/
git commit -m "Auto-update: GSC + ranks"
git push
```

### Option B — Auto-push from cron (recommended)

Add a wrapper script that runs after each bot's update:

```bash
# Save as ~/.hermes/seo-bots/bin/deploy_dashboard.sh
#!/usr/bin/env bash
cd ~/.hermes/seo-bots/dashboard
git add public/data/ 2>/dev/null
if git diff --cached --quiet; then
  exit 0
fi
git commit -m "Auto-update dashboard: $(date +%Y-%m-%d\ %H:%M)"
git push origin main
```

Make executable: `chmod +x ~/.hermes/seo-bots/bin/deploy_dashboard.sh`

Then add this to each bot's cron job (or add a new wrapper cron):

```bash
# Add to crontab (or use hermes cron add)
hermes cron add "*/15 * * * *" --name "deploy-dashboard" --prompt \
  "Run ~/.hermes/seo-bots/bin/deploy_dashboard.sh"
```

## What You'll See

### Tabs (8 total)
1. **Overview** — bot fleet status, headline metrics, recent audit
2. **Opportunities** — 3 prioritized actions (currently from real GSC + DataForSEO data)
3. **Approvals** — empty queue, will populate when GBP bot runs
4. **SEO Rank Tracker** — keyword positions + SERP competitors
5. **GSC Reader** — top queries, pages, opportunities
6. **GA4 Reader** — placeholder (waiting for data)
7. **SEO Analyst** — waiting for 3+ bots with data
8. **GBP Analyzer + Improver** — pending (quota + location ID)
9. **Audit Log** — every action ever taken

### Real Data Already in Dashboard
- 10 GSC queries with positions + CTR
- 1 SERP competitor analysis (you rank #1 for "weedistillery" in Canada)
- 3 SEO opportunities with revenue estimates
- 7 audit log entries (from our OAuth tests)

## Troubleshooting

**Dashboard shows "Loading…" forever:**
- Open browser DevTools → Console → check errors
- Most likely cause: CORS issue if opening `file://` directly
- **Fix:** Always serve via `python -m http.server` or GitHub Pages

**JSON files not updating:**
- Check cron jobs ran: `hermes cron runs`
- Verify data files exist: `ls ~/.hermes/seo-bots/dashboard/public/data/`
- Test manual update: `python -m seo_bots.update_dashboard bots --bot gsc-reader --status live`

**GitHub Pages shows 404:**
- Make sure repo Settings → Pages → folder is `/public`, not root
- Check the gh-pages branch isn't being used (use `main` only)

## Cost

**Hosting:** $0 (GitHub Pages is free for public repos)
**Bandwidth:** Effectively unlimited (GitHub serves static files globally)