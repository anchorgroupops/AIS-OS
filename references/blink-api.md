# Blink API — Reference

**Portal:** https://developer.joinblink.com/
**Base URL:** `https://api.joinblink.com/v2/`
**Auth:** `Authorization: Bearer <token>` — token from Blink admin → Manage Apps

---

## One-Time Setup

1. Log into Blink admin panel
2. Go to **Manage Apps** → Create new app → name it "Night Shift Bot"
3. Generate API token → save as `BLINK_API_TOKEN` in `H:\AI\Secrets\.env.master.private`
4. Create a post category → name it `nightshift` → save slug as `BLINK_CATEGORY`
5. Get your team group ID → save as `BLINK_GROUP_ID`
6. Add `BLINK_API_TOKEN`, `BLINK_CATEGORY`, `BLINK_GROUP_ID` as n8n variables (Settings → Variables)

---

## Feed Events API

### POST a feed event
```
POST https://api.joinblink.com/v2/feed/post
Authorization: Bearer <token>
Content-Type: application/json

{
  "category": "nightshift",
  "title": "Post Title",
  "body": "Post content (plain text or CardKit)"
}
```

### Response
```json
{ "id": "...", "status": "ok" }
```

---

## Notes

- API is **broadcast-only** — no webhook for incoming messages, no two-way chat
- Posts appear in the team's Blink feed under the category you create
- CardKit supports rich formatting (images, buttons, links) — see developer.joinblink.com/docs/cardkit
- Rate limits: not documented publicly; stay under 1 req/sec for safety

---

## n8n Workflows (import from `docs/n8n-workflows/`)

| File | Trigger | What it does |
|------|---------|-------------|
| `blink-morning-briefing.json` | Cron 7:30 AM M-F | AI-drafted daily brief → Blink feed |
| `blink-lead-alert.json` | Webhook POST | New lead details → Blink alert |
| `blink-on-demand-digest.json` | Webhook POST | Week's Night Shift activity → AI summary → Blink |

**To import:** n8n UI → Workflows → Import from file → select JSON → configure credentials → activate.

**Credentials needed in n8n:**
- `BLINK_API_TOKEN` — as n8n variable
- `BLINK_CATEGORY` — as n8n variable (e.g. `nightshift`)
- `OPENROUTER_API_KEY` — as n8n variable (already in `.nightshift.env`)
- Postgres credential pointing to `192.168.7.222:5432/n8n_ai` (for digest workflow)
