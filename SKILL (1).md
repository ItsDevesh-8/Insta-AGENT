---
name: n8n-instagram-autopilot
description: Build queue-driven social media publishing pipelines in n8n — a spreadsheet "box" of scheduled posts, a config-driven clock, deterministic media validation, and low-queue notifications. Use this skill whenever the user wants to auto-post to Instagram, Facebook, or any social platform from n8n; mentions scheduling or queueing posts, a content calendar that publishes itself, "upload my posts automatically", an "Instagram agent" or "posting bot"; asks about the Instagram Graph API, media containers, or IG access tokens; or wants a content queue that alerts them when it runs low. Also use it for any n8n workflow that must drain a queue on a schedule without double-processing rows, and whenever the user insists an AI must not rewrite their content.
---

# n8n Instagram Autopilot

Builds a **BOX ➜ AGENT ➜ PLATFORM** pipeline: the user drops content into a spreadsheet, a scheduled workflow drains it one item per slot, and the user gets pinged before the queue empties.

The defining constraint: **the user's caption is sacred.** It moves from spreadsheet cell to API parameter untouched. Do not introduce an LLM into the caption path unless the user explicitly asks for rewriting.

## When someone asks for an "AI agent" for social posting

They usually mean *automation*, not *inference*. Ask directly: should the AI write or change captions, or just publish what you wrote? Most people want the second. Say so plainly — an LLM in the caption path is the single largest source of hallucination risk, cost, and unpredictability in this pipeline, and removing it makes the whole thing free and deterministic.

Keep a vision check only if the user wants semantic image verification (right image, not blank, matches caption subject). Everything else — format, dimensions, aspect ratio, file size, duplicates — is plain code and should never be delegated to a model.

## Non-negotiable architecture

Build these seven stages. Skipping any one of them creates a failure mode that shows up in week three, not day one.

1. **Schedule trigger every 15 minutes** — not at the posting interval. Frequent ticks plus a logic gate is what makes the schedule editable without rewiring.
2. **Config node** — one Set node holding every tunable: `paused`, `dry_run`, `timezone`, `post_times`, `active_days`, `blackout_dates`, `jitter_minutes`, `low_queue_alert`, `max_attempts`, sheet ID, chat ID. The user must never edit anything else.
3. **Clock gate (Code)** — resolves "is this tick a posting slot?" in the user's timezone. Match slots to 15-minute buckets so exactly one tick per slot can fire. Always output a human-readable `reason` and `local_time`; this is the first thing anyone debugs.
4. **Claim-and-lock** — read queue, pick one row, immediately write `status: processing` with a run ID and `locked_at` timestamp *before* any API call. Auto-recover rows stuck in `processing` beyond 2 hours.
5. **Deterministic validation** — download bytes, parse the header for real format and dimensions, check platform constraints, hash for duplicates. Free, instant, no network beyond the fetch.
6. **Two-step publish with polling** — create container, poll until `FINISHED` with a hard poll cap, then publish. Mark `posted` only on a returned media ID.
7. **Watchdog + error trigger** — recount the queue after each post and alert at the threshold; an error trigger that names the failed node and releases the lock.

## Rules that prevent the common disasters

- **Never mark a row `posted` on any path except a real returned media ID.** Throw rather than guess.
- **Always cap loops.** Container polling ≤10 iterations. Per-post attempts ≤2, then dead-letter to `failed`.
- **Fail toward `pending`, not `failed`.** First failure returns the row to the queue for one retry. Only the last attempt marks it dead.
- **Secrets in env vars, never in nodes.** Reference `{{ $env.IG_ACCESS_TOKEN }}`. Makes the workflow JSON safely shareable.
- **Config values must be reachable downstream** via `$('Config').first().json.x`, since spreadsheet nodes replace the item payload.
- **Ship a `dry_run` flag.** Testing a publishing pipeline by publishing is how people burn real content.
- **Build the token refresh workflow at the same time.** Long-lived Instagram tokens expire in 60 days. A pipeline that dies silently in month two is worse than no pipeline.

## Platform facts worth not re-deriving

Detailed API sequences, error codes, and per-platform media constraints live in `references/instagram-graph-api.md`. Read it before writing any HTTP node against Meta.

The short version for Instagram:
- Business or Creator account linked to a Facebook Page. Personal accounts cannot publish via API.
- Image posts accept **JPEG only**. Aspect ratio 0.8–1.91. Min width 320px. Under 8 MB.
- 25 published posts per rolling 24 hours.
- `image_url` must be publicly fetchable **by Meta's servers**, not just by the user's browser.
- Publishing via the official Graph API is permitted; scraping and unofficial endpoints are what get accounts actioned. Never build the latter.

## What to refuse

Do not build auto-DM, auto-comment, auto-follow/unfollow, engagement pods, scraping of other accounts, or anything using unofficial/reverse-engineered Instagram endpoints. These violate platform terms and get the user's account banned. Offer the compliant publishing pipeline instead and say why.

## Delivery checklist

- [ ] Main workflow JSON, importable, with sticky notes explaining each stage
- [ ] Token refresh companion workflow
- [ ] Spreadsheet template with exact column names
- [ ] Setup guide: hosting → platform credentials → n8n wiring → first post
- [ ] Test plan covering: dry run, real post, bad image, empty queue, low queue
- [ ] Honest cost breakdown, including what "free" actually costs in attention

Validate the generated JSON before delivering: every connection endpoint must resolve to a real node name, node IDs must be unique, and no non-trigger node should be orphaned.

## Adapting to other platforms

Stages 1–4 and 7 are platform-agnostic — reuse them wholesale for LinkedIn, X, YouTube, or a blog CMS. Only stage 5 (media constraints) and stage 6 (publish sequence) change. When the user asks for multi-platform, keep one queue with a `platforms` column and branch at stage 6 rather than duplicating the whole workflow.
