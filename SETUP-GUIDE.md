# Instagram Autopilot — Setup Guide

**BOX ➜ AGENT ➜ INSTAGRAM.** You drop posts in a Google Sheet. The workflow publishes them on your schedule and pings you before the box runs dry. Your caption is copied character-for-character — nothing rewrites it.

Total setup time: about 90 minutes, most of it waiting on Meta.

---

## What you need before you start

| Thing | Cost | Notes |
|---|---|---|
| Instagram **Business** or **Creator** account | Free | Personal accounts cannot use the publishing API. Convert in the app: Settings → Account type. |
| A Facebook **Page** linked to that account | Free | Required by Meta. It can be empty and unpublished. |
| Meta Developer app | Free | developers.facebook.com |
| Google account | Free | For the Sheet + Drive |
| Telegram account | Free | For notifications |
| A machine running n8n 24/7 | Free–₹500/mo | See Part 1 |

---

## Part 1 — Get n8n running

### Option A: Oracle Cloud Always Free (recommended)

Permanently free ARM VM, runs 24/7, never sleeps.

1. Sign up at `cloud.oracle.com`. A card is needed for identity verification only — pick **Always Free** resources and you are not charged.
2. Create a **VM.Standard.A1.Flex** instance (Ubuntu 22.04, 2 OCPU / 12 GB RAM is plenty and still inside the free allowance).
3. Open port 5678 in the security list *and* in the VM firewall.
4. SSH in and run:

```bash
sudo apt update && sudo apt install -y docker.io docker-compose
sudo usermod -aG docker $USER && newgrp docker
mkdir -p ~/n8n && cd ~/n8n
```

Create `docker-compose.yml`:

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - GENERIC_TIMEZONE=Asia/Kolkata
      - TZ=Asia/Kolkata
      - N8N_HOST=YOUR_SERVER_IP
      - WEBHOOK_URL=http://YOUR_SERVER_IP:5678/
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=CHANGE_THIS
      # --- Instagram secrets (filled in Part 3) ---
      - IG_USER_ID=
      - IG_ACCESS_TOKEN=
      - META_APP_ID=
      - META_APP_SECRET=
    volumes:
      - ./data:/home/node/.n8n
```

```bash
docker compose up -d
```

Open `http://YOUR_SERVER_IP:5678`.

> Once you have a domain, put Caddy or nginx in front for HTTPS. Not required for this workflow — it makes no inbound webhook calls.

### Option B: Your own PC

Same `docker compose up -d`, minus the cloud. **The machine must be powered on at your posting times.** If your laptop is shut at 9pm, the 9pm slot is skipped silently.

### Option C: n8n Cloud

Works, zero server admin, roughly €20–24/month. Set the env vars under **Settings → Variables** instead of docker-compose. Everything else is identical.

---

## Part 2 — Build the BOX (Google Sheet + Drive)

1. Create a Google Sheet named **Instagram Queue**. Rename the first tab to `Queue`.
2. Import `queue-sheet-template.csv` (File → Import → Replace current sheet). Row 1 must be the header — the workflow matches on column names.
3. Create a Drive folder for your images. Upload your JPEGs. For each: Share → **Anyone with the link** → Viewer.
4. Convert each share link into a direct link:

```
Share link:  https://drive.google.com/file/d/1AbC...XyZ/view?usp=sharing
Use this:    https://drive.google.com/uc?export=download&id=1AbC...XyZ
```

The `id` is the chunk between `/d/` and `/view`.

> Drive occasionally serves an interstitial page for large files. If a post fails validation with "not JPEG", host that image somewhere with a clean direct URL instead — Cloudinary, imgbb, or an S3/R2 bucket all work and all have free tiers.

### The columns

| Column | You fill it? | What it does |
|---|---|---|
| `id` | ✅ | Unique. `p001`, `p002`… This is the matching key — never reuse one. |
| `image_url` | ✅ | Direct link to a **JPEG** |
| `caption` | ✅ | Your exact caption. Emojis, line breaks, hashtags — all preserved verbatim. |
| `status` | ✅ | Set `pending`. The workflow moves it to `processing` → `posted` / `failed`. |
| `priority` | optional | Higher number posts first. Blank = 0. |
| `scheduled_for` | optional | ISO date `2026-09-20T00:00:00Z` — won't post before this. |
| `attempts` `locked_at` `run_id` `image_hash` `media_id` `permalink` `posted_at` `error_log` | ❌ | Written by the workflow. Leave blank. |

**Image rules Instagram enforces (the workflow checks these before spending an API call):**
- JPEG only — not PNG, not WebP, not HEIC
- Aspect ratio between 0.8 (4:5 portrait) and 1.91 (landscape)
- Minimum width 320px
- Under 8 MB

---

## Part 3 — Connect Instagram

This is the fiddly part. Go slowly.

1. **developers.facebook.com** → My Apps → Create App → type **Business**.
2. Add the **Instagram Graph API** product.
3. App Settings → Basic. Copy your **App ID** and **App Secret**.
4. Open **Graph API Explorer** (Tools menu). Select your app.
5. Add these permissions: `instagram_basic`, `instagram_content_publish`, `pages_show_list`, `pages_read_engagement`, `business_management`.
6. Generate Access Token → authorise → pick your Page.
7. **Convert it to a long-lived token.** The one you just got lasts ~1 hour. Run:

```
https://graph.facebook.com/v21.0/oauth/access_token
  ?grant_type=fb_exchange_token
  &client_id=YOUR_APP_ID
  &client_secret=YOUR_APP_SECRET
  &fb_exchange_token=YOUR_SHORT_TOKEN
```

The `access_token` in the response is good for **60 days**. That's your `IG_ACCESS_TOKEN`.

8. **Find your Instagram User ID:**

```
https://graph.facebook.com/v21.0/me/accounts?access_token=LONG_TOKEN
   → gives you PAGE_ID

https://graph.facebook.com/v21.0/PAGE_ID?fields=instagram_business_account&access_token=LONG_TOKEN
   → gives you IG_USER_ID  (a ~17-digit number)
```

9. Put all four into your env vars and restart n8n:

```bash
cd ~/n8n && nano docker-compose.yml   # fill IG_USER_ID, IG_ACCESS_TOKEN, META_APP_ID, META_APP_SECRET
docker compose down && docker compose up -d
```

> Secrets live in env vars, never inside the workflow JSON. You can export and share the workflow without leaking anything.

---

## Part 4 — Wire up n8n

1. **Import** → paste `instagram-autopilot-workflow.json`. Repeat for `instagram-token-keeper-workflow.json`.
2. **Credentials → Add → Google Sheets OAuth2.** Follow the in-app steps. Then open each of the three Google Sheets nodes (`Read Queue`, `Lock Row`, `Mark Posted`, `Re-Read Queue`, `Mark Failed Or Retry`) and select the credential from the dropdown.
3. **Telegram bot:** message `@BotFather` → `/newbot` → copy the token. Add it as a Telegram credential in n8n and select it on every Telegram node. Get your chat ID by messaging `@userinfobot`.
4. **Open the `Config` node** and fill in:
   - `sheet_id` — the long string in your Sheet URL between `/d/` and `/edit`
   - `sheet_tab` — `Queue`
   - `telegram_chat_id` — your numeric chat ID
   - `post_times` — `09:00,21:00`
5. Also paste the chat ID into `Notify: Workflow Error` (it runs outside the normal flow, so it can't read Config).
6. **Settings → Error Workflow → select this same workflow.** That arms the crash handler.

---

## Part 5 — Test before you go live

**Test 1 — dry run.** In Config set `dry_run: true`. Click **Execute Workflow** manually. Expect a Telegram message showing the image dimensions and caption preview, with nothing published. If a slot doesn't match, temporarily add the current time to `post_times`.

**Test 2 — real post.** Set `dry_run: false`, execute manually, watch it appear on Instagram. Confirm the sheet row now says `posted` with a permalink.

**Test 3 — bad image.** Add a row pointing at a PNG. Expect: rejected, Telegram tells you why, `status` goes back to `pending` (attempt 1 of 2), and nothing posts.

**Test 4 — empty box.** Mark every row `posted`, execute. Expect the urgent refill ping.

**Test 5 — low queue.** Leave exactly 3 pending. Post one. Expect the ⚠️ refill warning.

Once all five pass, toggle the workflow **Active**. Activate the Token Keeper too.

---

## Living with it

**Change your schedule** — Config node only. Nothing else moves.

```
Twice a day    post_times: 09:00,21:00
Three times    post_times: 08:00,14:00,20:00
Weekdays only  active_days: Mon,Tue,Wed,Thu,Fri
Pause a week   paused: true
Skip Diwali    blackout_dates: 2026-11-08
Look human     jitter_minutes: 12
```

Times snap to 15-minute buckets — `09:00`, `09:15`, `09:30`, `09:45`. `09:07` behaves as `09:00`.

**Adding posts** — new row, unique `id`, `status: pending`. That's it. Add 50 if you like; they queue in order.

**When something fails** — the row goes back to `pending` and retries at the next slot. After 2 attempts it becomes `failed` and stops. Read `error_log` in the sheet, fix the image, set `status` back to `pending`.

**Token rotation** — the Token Keeper Telegrams you a fresh token 14 days before expiry. Paste it into `IG_ACCESS_TOKEN`, `docker compose restart`. Twice a year, two minutes.

---

## Cost

| Component | Cost |
|---|---|
| Instagram Graph API | ₹0 — Meta doesn't charge for publishing |
| Google Sheets + Drive | ₹0 |
| Telegram | ₹0 |
| n8n on Oracle Always Free | ₹0 |
| AI | ₹0 — **there is no AI in this pipeline** |
| **Total** | **₹0 / month** |

There is no model to run out of credits on. Every decision here is deterministic code: the clock gate, the queue picker, the image validator, the duplicate hash. That's why it can't hallucinate and can't get expensive.

---

## Guarantees, and their limits

**What cannot happen:**
- Your caption cannot be altered — it moves from sheet cell to API parameter untouched
- The same image cannot post twice — byte-hash checked against everything already posted
- A row cannot be marked `posted` unless Instagram returned a real media ID
- The workflow cannot retry forever — 10 polls max on the container, 2 attempts max per post
- A crash cannot strand a post — locked rows self-release after 2 hours
- It cannot DM, comment, follow, or scrape. Those endpoints aren't in the workflow at all.

**What can still bite you:**
- Meta changes their API. It happens roughly yearly. The version is pinned at `v21.0`; bump it when they deprecate.
- Instagram caps publishing at **25 posts per rolling 24 hours**. Nowhere near your 2/day, but don't set 30 post times.
- Google Drive sometimes serves an HTML interstitial instead of the file. Validation catches it — it just reads as "not JPEG". Rehost that image if it recurs.
- Automated publishing is allowed via the official API. Nothing here touches unofficial endpoints or scraping, which is what actually gets accounts actioned.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `(#200) Permissions error` | Missing scope | Regenerate token with all five permissions |
| `Invalid OAuth access token` | Expired | Run Token Keeper manually, paste new token, restart |
| `Media ID is not available` | Container still processing | Already handled — it polls. If it persists, image URL is likely unreachable from Meta's servers. |
| `The Aspect Ratio is not supported` | Slipped past validation | Crop to between 4:5 and 1.91:1 |
| Nothing fires at all | Clock gate | Open Clock Gate output — it prints `reason` and `local_time`. Check `paused` and `active_days`. |
| Row stuck on `processing` | Crash mid-run | Self-releases after 2h. To force it, set `status` back to `pending` manually. |
| `Cannot read properties of undefined` in Validate Image | Download failed | The URL isn't publicly reachable. Test it in an incognito window. |
