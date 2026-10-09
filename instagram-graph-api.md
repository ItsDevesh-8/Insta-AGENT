# Instagram Graph API — publishing reference

Pinned to `v21.0`. Meta deprecates versions roughly annually; bump deliberately, not automatically.

## Contents
- [Account prerequisites](#account-prerequisites)
- [Token chain](#token-chain)
- [Publishing sequence](#publishing-sequence)
- [Media constraints](#media-constraints)
- [Rate limits](#rate-limits)
- [Error codes](#error-codes)
- [Multi-image and video](#multi-image-and-video)

## Account prerequisites

Publishing requires **all** of:
- Instagram **Business** or **Creator** account (personal accounts are rejected outright)
- A Facebook Page linked to it (may be empty/unpublished)
- A Meta Developer app of type **Business**
- Scopes: `instagram_basic`, `instagram_content_publish`, `pages_show_list`, `pages_read_engagement`, `business_management`

## Token chain

Short-lived user token (~1h) → long-lived token (60 days):

```
GET https://graph.facebook.com/v21.0/oauth/access_token
  ?grant_type=fb_exchange_token
  &client_id={APP_ID}
  &client_secret={APP_SECRET}
  &fb_exchange_token={SHORT_TOKEN}
```

Resolving the IG user ID:

```
GET /v21.0/me/accounts?access_token={TOKEN}
  → PAGE_ID
GET /v21.0/{PAGE_ID}?fields=instagram_business_account&access_token={TOKEN}
  → instagram_business_account.id  (the IG_USER_ID, ~17 digits)
```

Checking remaining lifetime:

```
GET /v21.0/debug_token?input_token={TOKEN}&access_token={TOKEN}
  → data.expires_at (unix seconds; 0 means never), data.is_valid, data.scopes
```

Refresh at ≤14 days remaining. Re-exchanging a still-valid long-lived token returns a fresh 60-day one.

## Publishing sequence

Always two calls, sometimes three.

**1. Create container**
```
POST /v21.0/{IG_USER_ID}/media
  ?image_url={PUBLIC_URL}
  &caption={CAPTION}
  &access_token={TOKEN}
→ { "id": "CONTAINER_ID" }
```

`image_url` must be fetchable by Meta's servers. Localhost, signed-expiring, and auth-gated URLs all fail. Meta downloads it server-side; it is not uploaded from the client.

**2. Poll status** (skip only for small images where you accept occasional failure)
```
GET /v21.0/{CONTAINER_ID}?fields=status_code,status&access_token={TOKEN}
→ status_code: IN_PROGRESS | FINISHED | ERROR | EXPIRED | PUBLISHED
```

Poll every ~8s. Cap at 10 iterations. Containers expire after 24 hours.

**3. Publish**
```
POST /v21.0/{IG_USER_ID}/media_publish
  ?creation_id={CONTAINER_ID}
  &access_token={TOKEN}
→ { "id": "MEDIA_ID" }
```

**4. Permalink** (optional but worth storing)
```
GET /v21.0/{MEDIA_ID}?fields=permalink,timestamp&access_token={TOKEN}
```

Treat the returned `MEDIA_ID` as the only proof of success.

## Media constraints

**Images**
| Constraint | Value |
|---|---|
| Format | JPEG only |
| Aspect ratio | 0.8 (4:5) to 1.91 (1.91:1) |
| Min width | 320 px |
| Max width | 1440 px (wider is downscaled) |
| Max size | 8 MB |

PNG, WebP, HEIC and GIF are all rejected at container creation. Validate before spending the call.

**Reels / video**
| Constraint | Value |
|---|---|
| Container param | `media_type=REELS`, `video_url=` |
| Format | MOV or MP4 (H.264, AAC) |
| Duration | 3–900 seconds |
| Aspect ratio | 0.01–10, but 9:16 is what actually displays well |
| Max size | 1 GB |

Video containers take far longer to process — expect 30s+, and raise the poll cap accordingly.

**Captions**
- Max 2,200 characters
- Max 30 hashtags
- Max 20 `@` mentions
- Newlines and emoji survive intact through the query parameter

## Rate limits

- **25 published posts per rolling 24 hours** per IG account. Container creation doesn't count; publishing does.
- Check consumption: `GET /v21.0/{IG_USER_ID}/content_publishing_limit?fields=config,quota_usage`
- Standard Graph API app-level throttling also applies but is far above single-account posting volume.

## Error codes

| Code | Meaning | Action |
|---|---|---|
| 190 | Token invalid or expired | Refresh token chain |
| 200 | Permission missing | Regenerate with full scope list |
| 4 / 17 | Rate limited | Back off, retry next slot |
| 9007 | Aspect ratio unsupported | Crop within 0.8–1.91 |
| 9004 | `image_url` unreachable by Meta | Rehost publicly |
| 2207026 | Video format unsupported | Re-encode H.264/AAC |
| 24 / 25 | Publishing limit reached | Wait out the 24h window |
| 2207032 | Container creation failed | Usually a bad media URL upstream |

Meta returns errors as HTTP 200 with an `error` object in some paths. Check for `body.error`, don't rely on status codes alone.

## Multi-image and video

**Carousel** (2–10 items): create a child container per item with `is_carousel_item=true`, then a parent with `media_type=CAROUSEL&children={ID1},{ID2}`, then publish the parent. One caption on the parent only.

**Stories**: `media_type=STORIES`. No caption field. Available to Business accounts.

**Not available via API**: editing a published caption, deleting a post, tagging products, adding music, close-friends targeting.
