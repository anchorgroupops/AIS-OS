# Night Shift v2 — Pi-first Orchestrator Design

**Date:** 2026-05-29  
**Status:** Approved → Implementation

---

## Problem

Night Shift v1 is broken:
- Config points to `H:/Projects 👷‍♂️` — projects are at `H:/AI/Projects/`
- `state.ts` reads Windows Night Light registry — Pi and MacBook can't use this
- Queue is JSON per-machine — no shared state across machines
- Projects list is outdated
- No dashboard

---

## Architecture

Three roles, one TypeScript codebase with an `NS_MODE` env var (`pi` / `pc` / `mac`).

| Machine | Mode | Activation | Ollama | Tier |
|---------|------|-----------|--------|------|
| Pi | `pi` | systemd service, 24/7 | localhost (qwen2.5:3b, hermes3) | 0–2 |
| PC | `pc` | Windows Night Light + Task Scheduler | localhost (qwen2.5-coder:32b) | 0–3 |
| MacBook | `mac` | launchd plist | localhost (70B model) | 0–4 |

Shared state: **Postgres on Pi** (`192.168.7.222:5432`, db `n8n_ai`).

---

## Job Tiers

| Tier | Model | Machines | Job types |
|------|-------|----------|-----------|
| 0 | None | Any (Pi first) | lint, maintenance, git-health, hardening, todo-scan |
| 1 | ≤4B | Pi+ | basic LLM tasks (qwen2.5:3b) |
| 2 | ≤8B | Pi+ | general analysis (hermes3) |
| 3 | 32B | PC+ | code-quality, incomplete, dep-audit |
| 4 | 70B | MacBook | deep analysis, aios-review, sop-draft |

**Downgrade rule:** Tier 3 jobs unclaimed for 24h → Tier 2. Tier 4 jobs unclaimed for 48h → Tier 3.

---

## Database Schema (Postgres)

```sql
-- Schema: nightshift
CREATE TABLE nightshift.jobs (
  id TEXT PRIMARY KEY,
  category TEXT NOT NULL,
  title TEXT NOT NULL,
  description TEXT,
  priority INT NOT NULL,
  tier INT NOT NULL DEFAULT 0,
  project TEXT NOT NULL,
  target TEXT,
  status TEXT NOT NULL DEFAULT 'pending',
  needs_confirmation BOOLEAN NOT NULL DEFAULT false,
  needs_llm BOOLEAN NOT NULL DEFAULT false,
  claimed_by TEXT,         -- machine name
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  completed_at TIMESTAMPTZ,
  result TEXT,
  error TEXT,
  llm_used TEXT
);

CREATE TABLE nightshift.history (
  id TEXT PRIMARY KEY,
  category TEXT NOT NULL,
  title TEXT NOT NULL,
  project TEXT NOT NULL,
  result TEXT NOT NULL,
  completed_at TIMESTAMPTZ NOT NULL,
  duration_ms INT,
  llm_used TEXT,
  machine TEXT
);

CREATE TABLE nightshift.machines (
  name TEXT PRIMARY KEY,       -- 'pi', 'pc', 'mac'
  mode TEXT NOT NULL,
  last_seen TIMESTAMPTZ NOT NULL DEFAULT now(),
  current_job TEXT,
  current_project TEXT,
  ollama_available BOOLEAN DEFAULT false,
  model TEXT
);
```

---

## Components Changed / Added

1. **`queue.ts`** — rewritten to use Postgres instead of JSON files
2. **`state.ts`** — OS-aware: Windows checks Night Light; Pi/Mac always return `true`
3. **`config.ts`** — reads `NS_MODE`, `NS_PROJECTS_ROOT`, `NS_OLLAMA_URL` from env
4. **`runner.ts`** — adds heartbeat writes to `nightshift.machines`
5. **New: `agent/src/jobs/aios-review.ts`** — reviews AIS-OS context files for staleness
6. **New: `agent/src/jobs/n8n-audit.ts`** — calls n8n API to check workflow health
7. **New: `agent/src/dashboard/`** — Express server, port 3001, reads Postgres

---

## New Job Handlers

### `aios-review` (Tier 4)
Reads `H:/AI/Projects/AI OS (Herk)/AIS-OS/context/` and `connections.md`.
Checks for: stale priorities (older than 90 days), disconnected connections, missing SOPs for recurring processes.
Uses LLM to flag staleness and draft update suggestions.

### `n8n-audit` (Tier 3)
Calls n8n API (`n8n.joelycannoli.com/api/v1/workflows`).
Checks for: inactive workflows that were recently edited, workflows with no error handling, SOLD/social media workflows with open TODOs.

---

## Activation Per Machine

**Pi:**
```bash
# /etc/systemd/system/nightshift.service
[Unit]
Description=Night Shift Agent
After=network.target postgresql.service

[Service]
Environment=NS_MODE=pi
Environment=NS_PROJECTS_ROOT=/home/joel/projects
WorkingDirectory=/home/joel/night-shift/agent
ExecStart=/usr/bin/node dist/index.js
Restart=always
RestartSec=30

[Install]
WantedBy=multi-user.target
```

**PC:**
- Night Light toggle (existing Stream Deck + NightShift.psm1) — kept as-is
- Windows Task Scheduler: also runs agent at login and every 4 hours

**MacBook:**
```xml
<!-- ~/Library/LaunchAgents/com.nightshift.agent.plist -->
<key>ProgramArguments</key>
<array>
  <string>node</string>
  <string>/Users/joel/night-shift/agent/dist/index.js</string>
</array>
<key>RunAtLoad</key><true/>
<key>EnvironmentVariables</key>
<dict>
  <key>NS_MODE</key><string>mac</string>
</dict>
```

---

## Dashboard

Express server on Pi, port 3001.
- `/` — machine grid + queue summary
- `/queue` — full job table (filterable by status/project)
- `/history` — 7-day history with LLM usage stats

Accessible at `http://192.168.7.222:3001` (local) and optionally via Cloudflare tunnel.

---

## Project Scope (first deploy)

```json
{
  "projectsRoot": "H:/AI/Projects",
  "projects": [
    "AI OS (Herk)/AIS-OS",
    "Night-shift"
  ]
}
```

Expand to full repo list after first successful run.

---

## What's NOT changing

- Existing job handlers (lint, maintenance, hardening, git-health, dep-audit, sync, todo-scan, incomplete) — kept, minor cross-platform fixes
- Stream Deck plugin — kept as-is
- `NightShift.psm1` — kept as-is
- Night Light as PC trigger — kept as-is
