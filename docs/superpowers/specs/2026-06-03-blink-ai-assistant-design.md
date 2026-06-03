# Blink AI Assistant — Design

**Date:** 2026-06-03
**Status:** Approved → Implementation
**Spec location:** `docs/superpowers/specs/2026-06-03-blink-ai-assistant-design.md`

---

## Problem

The team communicates via Blink but all posts are manual. There's no AI in the loop for:
- Daily briefings (what's active, what needs attention)
- Lead notifications (new Zillow leads arrive but team hears about them late)
- AI-drafted summaries to reduce writing overhead

---

## Blink API Reality

Blink's Feed Events API (developer.joinblink.com) supports **broadcasting** rich posts to users/groups via Bearer token. It is **not** a two-way chat API — there's no documented webhook for incoming messages. Interactive Q&A chatbot is not feasible.

**What IS feasible:** Automated feed posts triggered by schedules or external events.

---

## Architecture

Three n8n workflows, all posting to Blink via HTTP Request → Feed Events API.

| Workflow | Trigger | Content | LLM |
|----------|---------|---------|-----|
| Morning Briefing | Cron 7:30 AM daily | Day summary: date, 1-2 priorities, team reminder | claude-haiku (OpenRouter) |
| Lead Alert | Webhook (FUB or Zapier) | New lead name, source, quick action suggestion | None (template) |
| On-Demand Digest | Manual webhook | AI-drafted summary of open tasks/priorities | claude-haiku |

---

## Blink Setup Required (manual, one-time)

1. Log into Blink admin → "Manage Apps" → Create new app → "Night Shift Bot"
2. Generate API token → save to `H:\AI\Secrets\.env.master.private` as `BLINK_API_TOKEN`
3. Get the target group ID (or user ID) to post to → save as `BLINK_GROUP_ID`
4. Create a "post category" in Blink admin → save slug as `BLINK_CATEGORY` (e.g. `nightshift`)

---

## Workflow 1: Morning Briefing

**n8n workflow: `Blink - Morning Briefing`**

```
Schedule (7:30 AM M-F)
  → Set node (build prompt: today's date, priorities from context)
  → OpenRouter/Claude (draft 3-bullet briefing in Joel's voice register)
  → HTTP Request → POST https://api.joinblink.com/v1/feed-events
    Body: { category, title: "Morning Brief", body: <AI output> }
```

Content template:
```
Good morning team — {date}

• {priority_1}
• {priority_2}
• {reminder}

— AnchorGroupOps AI
```

---

## Workflow 2: Lead Alert

**n8n workflow: `Blink - Lead Alert`**

```
Webhook (called by FUB webhook or Zapier when new lead assigned)
  → Parse lead data (name, source, phone)
  → HTTP Request → POST Blink Feed Event
    Body: template (no LLM needed — fast, reliable)
```

Template:
```
New lead assigned: {name}
Source: {source} | Phone: {phone}
Action: Call within 5 min for Zillow Preferred scoring
```

---

## Workflow 3: On-Demand Digest

**n8n workflow: `Blink - On-Demand Digest`**

```
Webhook (manual trigger from n8n UI or URL call)
  → Postgres query (nightshift.history last 7 days)
  → OpenRouter/Claude (summarize: what got done, what's pending)
  → HTTP Request → POST Blink Feed Event
```

---

## Credentials Needed

| Key | Where | Note |
|-----|-------|------|
| `BLINK_API_TOKEN` | .env.master.private | From Blink admin → Manage Apps |
| `BLINK_GROUP_ID` | .env.master.private | Target group for team posts |
| `BLINK_CATEGORY` | .env.master.private | Post category slug created in admin |
| `NS_OPENROUTER_API_KEY` | Already in .nightshift.env | For Claude haiku calls |

---

## Scope

**In:** 3 n8n workflows (morning brief, lead alert, on-demand digest), Blink API reference saved to `references/blink-api.md`.

**Out:** Two-way chat (Blink API doesn't support it), FUB integration wiring (separate task), mobile push logic.

---

## Success Criteria

Team sees an AI-drafted morning briefing in Blink by 7:30 AM on weekdays. Lead alert fires within 30 seconds of a new lead webhook. On-demand digest reachable by hitting a webhook URL.

---

## Blockers

Blink app creation and token generation must be done manually in the Blink admin panel before workflows can be activated. n8n workflows can be imported and configured now; they'll be inactive until credentials are added.
