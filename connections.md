# Connections

Registry of every system your AIOS can reach. `/audit` checks this file for domain coverage and freshness.

| # | Domain | Tool | Mechanism | Auth | Last checked |
|---|---|---|---|---|---|
| 1 | Revenue / Financials | FUB (Follow Up Boss) | not yet connected | — | — |
| 2 | Customer interactions | FUB (CRM), Zillow Preferred, Facebook/Meta | not yet connected | — | — |
| 3 | Calendar | Google Calendar | not yet connected | — | — |
| 4 | Communication | Gmail (AnchorGroupOps), Blink | not yet connected | — | — |
| 5 | Project / task tracking | GitHub, n8n | not yet connected | — | — |
| 6 | Meeting intelligence | Zoom, Google Drive, Notion | not yet connected | — | — |
| 7 | Knowledge / files | Google Drive, Blink, Notion, Google Docs | not yet connected | — | — |

**Mechanism options:** `mcp` (MCP server), `script` (Python/Bash hitting an API, in `scripts/`), `export` (CSV/JSON dump pipeline), `key+ref` (`.env` key + `references/{tool}-api.md` guide), `not yet connected`.

---

## Extended Stack

Tools identified in the environment. Wire these as needed — credentials in `H:\AI\Secrets\.env.master.private`.

| Tool | Purpose | Notes |
|---|---|---|
| n8n | Automation platform | Hosted at n8n.joelycannoli.com (Pi). Also PC and Mac instances. MCP server available. |
| Ollama | Local LLM inference | 192.168.7.222:11434. Models: qwen3, deepseek-r1, phi4, llama3. |
| PostgreSQL | Database | 192.168.7.222:5432. Used by n8n. |
| Qdrant | Vector DB (local) | 192.168.7.222:6333 |
| Qdrant Cloud | Vector DB (cloud) | GCP us-east4 instance |
| Pinecone | Vector DB (cloud) | GravityClaw index |
| Supabase | Hosted Postgres + auth | Cloud instance |
| OpenRouter | LLM API router | GravityClaw account |
| OpenAI | LLM API | Via n8n credential |
| Gemini | LLM API | Google account |
| ElevenLabs | Text-to-speech | — |
| Telegram | Bot notifications | MorningAnchorReportBot + DORI bot |
| Twilio | SMS | — |
| Jotform | Forms | — |
| Airtable | Spreadsheet/DB | — |
| Notion | Notes/docs | Scattered use |
| Cloudflare | DNS + tunnel | Tunnel to Pi services |
| Firecrawl | Web scraping | — |
| Postman | API testing | — |
| AnchorGroupAI Gmail | AI-dedicated email | Use freely for automation |
| GitHub | Code repos | Personal access token available |

---

When you wire a new tool, save `references/{tool}-api.md` with endpoints, auth flow, and common queries.
