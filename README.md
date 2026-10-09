# Insta-AGENT
THIS IS AN AGENT THAT WILL AUTOMATICALLY POST ON YOUR INSTA ACCOUNT....
# Instagram Autopilot (n8n)

**BOX → AGENT → INSTAGRAM.** Drop images and captions into a Google Sheet. An n8n workflow publishes them to Instagram on your schedule, then pings you on Telegram before the queue runs dry.

Your caption goes from the sheet to Instagram character for character. No LLM touches it, so there is nothing to hallucinate and nothing to run out of credits on.

## How it works

1. **Trigger:** runs every 15 minutes and checks whether it's a posting slot.
2. **Config node:** the only node you edit. Set `post_times`, `active_days`, `blackout_dates`, `jitter_minutes`, `paused` and `dry_run`.
3. **Claim and lock:** picks the oldest `pending` row and marks it `processing` so it can never double-post.
4. **Image validation (pure code):** checks JPEG format, aspect ratio (0.8 to 1.91), minimum width, file size and duplicate hash before any API call.
5. **Publish:** two-step Instagram Graph API flow (create container, poll until `FINISHED`, publish). A row is marked `posted` only when Instagram returns a real media ID.
6. **Watchdog:** recounts the queue after every post and alerts on Telegram when it drops to your threshold or hits zero.
7. **Error handler:** names the failed node in an alert, and stuck rows self-release after 2 hours.

A separate **Token Keeper** workflow checks the long-lived token daily and sends a fresh one 14 days before it expires.

## Features

- Fully editable schedule: multiple daily slots, weekday filters, blackout dates, optional human-like jitter
- Dry-run mode: runs the whole pipeline without publishing
- Retry cap: 2 attempts per post, then dead-lettered to `failed`
- Duplicate-image protection
- Secrets live in env vars, never in the workflow JSON
- Runs at ₹0/month on a self-hosted free tier

## Stack

n8n (self-hosted) · Google Sheets (queue) · Instagram Graph API · Telegram Bot API · public image hosting (Catbox or Drive)

## Setup

1. Run n8n on an always-on machine (Docker).
2. Create the queue sheet from `queue-sheet-template.csv`.
3. Create a Meta app and get a long-lived token plus your IG user ID. Set `IG_USER_ID`, `IG_ACCESS_TOKEN`, `META_APP_ID` and `META_APP_SECRET` as env vars.
4. Import both workflow JSONs and attach your Google Sheets and Telegram credentials.
5. Fill in the Config node, test with `dry_run: true`, then go live.

Full walkthrough: [`SETUP-GUIDE.md`](SETUP-GUIDE.md)

## Requirements

- Instagram **Business or Creator** account linked to a Facebook Page
- Images must be **JPEG**
- Instagram's limit of 25 published posts per rolling 24 hours applies

## What it deliberately does not do

No auto-DM, auto-comment, auto-follow, scraping or unofficial endpoints. It only uses the official publishing API.

## Repo contents

| File | Purpose |
|---|---|
| `instagram-autopilot-workflow.json` | Main publishing workflow |
| `instagram-token-keeper-workflow.json` | Token expiry monitor |
| `queue-sheet-template.csv` | Sheet schema |
| `SETUP-GUIDE.md` | Setup guide |
| `n8n-instagram-autopilot/` | Reusable skill for rebuilding the pipeline |
