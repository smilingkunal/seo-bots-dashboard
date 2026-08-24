# SEO Bot Dashboard

Live dashboard for the 12 SEO bots monitoring **weedistillery.com**.

🌐 **Live URL:** https://smilingkunal.github.io/seo-bots-dashboard/

## What This Is

A real-time dashboard showing data from 7 specialized SEO bots:

| Bot | Purpose | Status |
|---|---|---|
| **SEO Rank Tracker** | Daily keyword positions | 🟢 live |
| **GSC Reader** | Google Search Console data | 🟢 live |
| **GA4 Reader** | Google Analytics 4 data | 🟡 pending |
| **SEO Analyst** | Cross-correlates data into opportunities | 🔵 waiting |
| **GBP Analyzer** | Google Business Profile snapshot | 🟡 pending |
| **GBP Improver** | Drafts posts (approval-gated) | 🟡 pending |
| **Master Dashboard** | Weekly report + action items | 🔵 waiting |

## Auto-Refresh

The dashboard polls data every 30 seconds — open the URL in any browser, leave it open.

## File Structure

```
dashboard/
├── public/
│   ├── index.html              # The dashboard (single-file SPA)
│   └── data/                    # JSON data files (one per bot)
│       ├── bots-status.json     # Fleet status
│       ├── gsc-data.json        # GSC queries + pages
│       ├── ga4-data.json        # GA4 sessions + conversions
│       ├── ranks-data.json      # Keyword positions
│       ├── gbp-data.json        # GBP profile + reviews
│       ├── opportunities.json   # SEO analyst opportunities
│       ├── approvals.json       # Pending approvals queue
│       └── audit-log.json       # Every action ever taken
├── DEPLOYMENT.md                # How it works
└── README.md                    # This file
```

## How Bots Update It

Each bot calls `update_dashboard.py` after it collects its data:

```bash
# GSC bot
python -m seo_bots.update_dashboard gsc --source-data /path/to/gsc.json

# Rank tracker
python -m seo_bots.update_dashboard ranks --source-data /path/to/ranks.json

# GBP bot updates approvals
python -m seo_bots.update_dashboard approvals \
  --token apr_xxx \
  --action-id publish_post:gbp-post-draft-001 \
  --bot gbp-improver \
  --action-type publish_post \
  --preview "🌟 Summer sale: 20% off all services..."

# Audit log entry
echo '{"ts": ..., "action": "publish_post", ...}' | \
  python -m seo_bots.update_dashboard audit
```

The bot writes the JSON file. The dashboard polls the file every 30s. No server needed.

## GitHub Pages Deployment

The `public/` directory is the static site. To deploy:

1. Create a GitHub repo called `seo-bots-dashboard`
2. Push `public/` to the `main` branch (or `gh-pages`)
3. Settings → Pages → Source: `main` branch, `/public` folder
4. Visit https://YOUR_USERNAME.github.io/seo-bots-dashboard/

Every cron run that updates data → commits to git → auto-deploys.

## Local Testing

```bash
# Serve the dashboard locally
cd ~/.hermes/seo-bots/dashboard/public
python -m http.server 8080

# Open in browser
open http://localhost:8080
```

## How Updates Flow

```
┌─────────────────────────────────────────────────────────┐
│                    HERMES AGENT                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │gsc-reader│  │seo-tracker│ │gbp-improver│             │
│  └─────┬────┘  └─────┬────┘  └─────┬────┘               │
│        │             │             │                    │
│        └──────┬──────┴──────┬──────┘                    │
│               ▼             ▼                           │
│       ┌───────────────────────────┐                    │
│       │  update_dashboard.py      │                    │
│       │  writes JSON to public/data/│                  │
│       └─────────────┬─────────────┘                    │
└─────────────────────┼──────────────────────────────────┘
                      ▼
              ┌────────────────┐
              │  public/data/  │
              │  *.json files  │
              └───────┬────────┘
                      │ (every 30s)
                      ▼
              ┌────────────────┐
              │   GitHub       │
              │   Pages (CDN)  │──── users see dashboard
              └────────────────┘
```

## Safety Guarantees (unchanged from bot layer)

The dashboard is **read-only** — it only displays data. All write operations
still require approval tokens issued by the human approval system.