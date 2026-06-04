# Connections

Registry of every system your AIOS can reach. `/audit` checks this file for domain coverage and freshness.

| # | Domain | Tool | Mechanism | Auth | Last checked |
|---|---|---|---|---|---|
| 1 | Revenue / Financials | FUB (Follow Up Boss) | key+ref | `FUB_API_KEY_JOEL` in .env.master.private | 2026-06-04 |
| 2 | Customer interactions | FUB (CRM), Zillow Preferred, Facebook/Meta | key+ref | FUB: above. Facebook: token EXPIRED — regenerate at developers.facebook.com | 2026-06-04 |
| 3 | Calendar | Google Calendar | not yet connected | OAuth creds in .env.master.private | — |
| 4 | Communication | Gmail (AnchorGroupOps), Blink | key+ref / not yet connected | Gmail: OAuth creds in .env.master.private. Blink: NOT YET — see references/blink-api.md | 2026-06-04 |
| 5 | Project / task tracking | GitHub, n8n | key+ref | `GITHUB_PERSONAL_ACCESS_TOKEN`, `N8N_API_KEY_PI` in .env.master.private | 2026-06-04 |
| 6 | Meeting intelligence | Zoom, Google Drive, Notion | not yet connected | Google Drive OAuth in .env.master.private | — |
| 7 | Knowledge / files | Google Drive, Blink, Notion, Google Docs | not yet connected | OAuth creds in .env.master.private. Notion: `NOTION_API_KEY` | 2026-06-04 |

---

## n8n Workflows Ready to Import

Import from `docs/n8n-workflows/` via n8n UI → Workflows → Import from file:

| File | Status | Needs before activating |
|------|--------|------------------------|
| `fub-social-router.json` | **Import + activate** | `META_PAGE_ACCESS_TOKEN` + `META_PAGE_ID` (Facebook token expired — regenerate), `OPENROUTER_API_KEY` n8n var (Enterprise) or hardcode in node, Gmail SMTP credential |
| `blink-morning-briefing.json` | Import only | Blink admin token (`BLINK_API_TOKEN`) + `BLINK_CATEGORY` n8n vars |
| `blink-lead-alert.json` | Import only | Same as above |
| `blink-on-demand-digest.json` | Import only | Same as above + Pi Postgres credential |

> Note: `social-under-contract.json` and `social-sold.json` are superseded by `fub-social-router.json` (combined router).

---

## FUB Webhooks

| ID | Event | URL | Status |
|----|-------|-----|--------|
| 46 | dealsUpdated | /webhook/fub-deal-updated | Active (existing) |
| 51 | dealsUpdated | /webhook/fub-social-router?token=cea22c... | Active — social media router |

> FUB does not support outbound HMAC signing. Webhook 51 is authenticated by the secret token in its URL. Token stored in `.env.master.private` as `FUB_SOCIAL_WEBHOOK_TOKEN`.

---

## Extended Stack

| Tool | Purpose | Credential location | Status |
|---|---|---|---|
| n8n | Automation platform | `N8N_API_KEY_PI` in .env | Connected (read only — write requires UI) |
| Ollama | Local LLM inference | `OLLAMA_API_KEY` in .env | Connected |
| PostgreSQL | Shared queue + social drafts | `POSTGRES_PASSWORD` in .env | Connected |
| Qdrant (cloud) | Vector DB | `QDRANT_API_KEY` in .env | Key available |
| Qdrant (local) | Vector DB (Pi) | `QDRANT_LOCAL_URL` in .env | Key available |
| Pinecone | Vector DB (cloud) | `PINECONE_API_KEY` in .env | Key available |
| Supabase | Hosted Postgres + auth | `SUPABASE_SECRET_KEY` in .env | Key available |
| OpenRouter | LLM API router | `OPENROUTER_API_KEY` in .env | Connected |
| OpenAI | LLM API | via n8n credential | Key in n8n |
| Gemini | LLM API | `GOOGLE_AI_STUDIO_API_KEY` in .env | Key available |
| ElevenLabs | Text-to-speech | `ELEVENLABS_API_KEY` in .env | Key available |
| Telegram | Bot notifications | `TELEGRAM_BOT_TOKEN_MORNING_ANCHOR` in .env | Connected |
| Twilio | SMS | `TWILIO_AUTH_TOKEN` in .env | Key available |
| Jotform | Forms | `JOTFORM_API_KEY` in .env | Key available |
| Airtable | Spreadsheet/DB | `AIRTABLE_API_KEY` in .env | Key available |
| Notion | Notes/docs | `NOTION_API_KEY` in .env | Key available |
| Cloudflare | DNS + tunnel | `CLOUDFLARE_TUNNEL_TOKEN` in .env | Connected |
| Firecrawl | Web scraping | `FIRECRAWL_API_KEY` in .env | Key available |
| GitHub | Code repos | `GITHUB_PERSONAL_ACCESS_TOKEN` in .env | Connected |
| Facebook/Meta | Social media posting | `FACEBOOK_ACCESS_TOKEN` in .env — **EXPIRED** | Need token refresh |
| Blink | Team messaging | `BLINK_API_TOKEN` in .env — **NOT YET OBTAINED** | Need Blink admin setup |
| AnchorGroupAI Gmail | AI-dedicated email | OAuth creds in .env | Keys available (OAuth flow needed) |
| FUB | CRM | `FUB_API_KEY_JOEL` in .env | Connected — webhook 51 active |

When you wire a new tool, save `references/{tool}-api.md` with endpoints, auth flow, and common queries.
