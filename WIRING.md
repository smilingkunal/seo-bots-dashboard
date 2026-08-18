# Auto-Deploy Wiring — How It Works

## The Flow

Every cron job runs through this pipeline:

```
┌─────────────────────────────────────────────────────────────────────┐
│ CRON TRIGGER (e.g. 06:00 UTC)                                       │
└──────────────────────┬──────────────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 1. Bot runs (via hermes --profile <bot>)                           │
│    - GSC reader pulls search analytics                              │
│    - Rank tracker checks positions                                  │
│    - SEO analyst correlates everything                               │
│    - Bot writes its JSON to /tmp/<bot>-latest.json                   │
└──────────────────────┬──────────────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 2. Wrapper publishes to dashboard                                    │
│    - update_dashboard.py reads /tmp/<bot>-latest.json                │
│    - Writes to dashboard/public/data/<bot>.json                      │
│    - Marks bot as "live" in bots-status.json                        │
└──────────────────────┬──────────────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 3. Wrapper deploys to GitHub Pages                                  │
│    - deploy_dashboard.sh runs only if GITHUB_DEPLOY_TOKEN is set    │
│    - git add public/data/ + public/index.html                       │
│    - git commit with timestamp                                      │
│    - git push origin main (to GitHub Pages)                          │
│    - GitHub auto-deploys in ~30 seconds                              │
└──────────────────────┬──────────────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 4. Audit log entry                                                  │
│    - {ts, action: "bot_run", bot, duration_s, exit, data_written}  │
│    - Appended to ~/.hermes/seo-bots/logs/audit.jsonl                │
│    - Mirrored to dashboard/public/data/audit-log.json                │
└─────────────────────────────────────────────────────────────────────┘
```

## Cron Schedule (UTC)

| Bot | Schedule | Job ID | Output file |
|---|---|---|---|
| gsc-reader | `0 6 * * *` | `1d4fe3655d48` | `gsc-data.json` |
| ga4-reader | `5 6 * * *` | `f1ca551f408e` | `ga4-data.json` |
| gbp-analyzer | `10 6 * * *` | `b627fd52da59` | `gbp-data.json` |
| seo-tracker | `15 6 * * *` | `3d49753f50c1` | `ranks-data.json` |
| seo-analyst | `30 6 * * *` | `c2b684fa91ce` | `opportunities.json` |
| gbp-improver | `0 8 * * 1` (Mon) | `2ce6af30b490` | `approvals.json` |
| dashboard | `0 9 * * 1` (Mon) | `fed9aa7a98d8` | `bots-status.json` (summary) |

## Files Involved

| File | Purpose |
|---|---|
| `~/.hermes/seo-bots/bin/run_bot.sh` | Wrapper that runs bot + publishes + deploys |
| `~/.hermes/seo-bots/bin/deploy_dashboard.sh` | Git push step (called by wrapper) |
| `~/.hermes/seo-bots/bin/cron_jobs.sh` | One-shot script to register all 7 cron jobs |
| `~/.hermes/seo-bots/seo_bots/update_dashboard.py` | Python module: bot data → dashboard JSON |
| `~/.hermes/seo-bots/dashboard/public/index.html` | Dashboard UI (auto-refreshes every 30s) |
| `~/.hermes/seo-bots/dashboard/public/data/*.json` | Live data files |

## Per-Bot Data Schemas

### gsc-reader → gsc-data.json
```json
{
  "period": "last_7_days",
  "totals": {"clicks": 6, "impressions": 56, "ctr": 10.7, "position": 12.4},
  "top_queries": [{"query": "...", "clicks": N, "impressions": N, "ctr": 0.0-1.0, "position": 1-100}],
  "top_pages": [{"page": "https://...", "clicks": N, "impressions": N, "ctr": 0.0-1.0, "position": 1-100}],
  "opportunities": [{"query": "...", "position": N, "impressions": N, "issue": "...", "potential_clicks_per_month": N, "action": "...", "effort": "low|medium|high", "priority": "low|medium|high"}]
}
```

### seo-tracker → ranks-data.json
```json
{
  "location": "Canada",
  "device": "desktop",
  "keywords": [{"keyword": "...", "your_rank": 1-100|null, "search_volume": N|null, "top_competitors": [...], "note": "..."}],
  "summary": {"keywords_tracked": N, "keywords_in_top_3": N, "keywords_in_top_10": N, "keywords_not_ranking": N}
}
```

### ga4-reader → ga4-data.json
```json
{
  "period": "last_7_days",
  "totals": {"sessions": N, "users": N, "conversions": N},
  "channels": [{"channel": "...", "sessions": N, "users": N, "conversions": N}],
  "top_pages": [{"page": "/...", "sessions": N, "users": N, "conversions": N}]
}
```

### gbp-analyzer → gbp-data.json
```json
{
  "profile": {"name": "...", "address": "...", "phone": "...", "completeness": 0-100},
  "reviews": {"total": N, "avg_rating": 0.0-5.0, "by_rating": {...}, "unanswered": N, "response_rate_pct": 0-100},
  "photos": N,
  "posts": {"count": N, "last_post_days_ago": N},
  "opportunities": [{"type": "...", "details": "...", "impact": "low|medium|high"}]
}
```

### seo-analyst → opportunities.json
```json
{
  "summary": {"high_impact": N, "medium_impact": N, "low_impact": N, "total_opportunities": N},
  "opportunities": [{"id": "opp-NNN", "priority": "...", "impact": "...", "category": "...", "title": "...", "context": "...", "estimated_clicks_per_month": N, "estimated_revenue_impact": "$0-1K/mo", "action": "...", "effort": "...", "reversible": true, "approval_required": true, "suggested_by": "seo-analyst", "date": "YYYY-MM-DD"}]
}
```

### gbp-improver → approvals.json
```json
{
  "pending": [{"token": "apr_...", "action_id": "publish_post:gbp-post-draft-XXX", "bot": "gbp-improver", "action_type": "publish_post", "target": "GBP", "preview": "...", "issued_at": "...", "expires_at": "..."}],
  "approved": [],
  "rejected": [],
  "reverted": [],
  "summary": {"total_pending": N, "total_approved": 0, "total_executed": 0, "total_reverted": 0}
}
```

### dashboard → bots-status.json (summary)
```json
{
  "fleet_health": {"live": N, "pending": N, "waiting": N, "errors": N},
  "key_metrics": {"total_clicks_7d": N, "total_opportunities": N, "pending_approvals": N},
  "highlights": ["..."],
  "next_actions": ["..."]
}
```

## Manual Testing

```bash
# Run one bot manually (uses wrapper — runs bot, updates dashboard, attempts deploy)
~/.hermes/seo-bots/bin/run_bot.sh gsc-reader "Test run"

# Force publish data manually
python -m seo_bots.update_dashboard gsc --source-data /path/to/gsc.json

# Force deploy manually
~/.hermes/seo-bots/bin/deploy_dashboard.sh

# Check cron status
hermes cron list
hermes cron runs  # history
```

## GitHub Pages Setup (One-Time)

1. Create a **public** repo on GitHub named `seo-bots-dashboard`
2. Get a Personal Access Token (PAT) with `public_repo` scope
3. Add to `~/.hermes/.env`:
   ```
   GITHUB_DEPLOY_TOKEN=ghp_your_pat_here
   GITHUB_DEPLOY_REPO=YOUR_USERNAME/seo-bots-dashboard
   ```
4. Run `~/.hermes/seo-bots/bin/deploy_dashboard.sh` once manually to bootstrap
5. Enable Pages on the repo: Settings → Pages → Source: `main`, Folder: `/public`

After that, every cron run will automatically deploy updates to GitHub Pages.

## What Happens If GitHub Deploy Fails?

The wrapper is **non-blocking** — if GitHub push fails:
- Bot run is still logged as successful
- Dashboard local data is still updated
- Audit log records the failure
- Cron will retry on next run

This means even without GitHub Pages deployed, you can always check the dashboard by serving `dashboard/public/` locally.