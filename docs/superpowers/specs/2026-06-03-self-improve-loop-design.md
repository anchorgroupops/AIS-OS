# Self-Improvement Loop — Auto-Apply + Daily Digest

**Date:** 2026-06-03
**Status:** Approved → Implementation
**Builds on:** `2026-05-29-night-shift-v2-design.md`

---

## Problem

Night Shift v2 is deployed and running background jobs, but findings go nowhere actionable:
- `aios-review.ts` and `n8n-audit.ts` produce output that is only logged, not applied or surfaced
- No mechanism to apply safe changes automatically
- No daily summary of what ran overnight

---

## Design: Auto-Apply + Daily Digest

Night Shift jobs produce structured output. Low-risk findings are applied immediately and logged. High-risk findings are flagged for review. A digest fires every morning summarizing both.

---

## Tier Policy

| Tier | LLM | Auto-apply? | Examples |
|------|-----|-------------|---------|
| 0 | None | Yes | lint, git-health, maintenance, todo-scan |
| 1 | ≤4B | Yes | minor context date updates, file formatting |
| 2 | ≤8B | Yes | basic analysis rewrites, staleness patches |
| 3 | 32B | Flag only | n8n workflow recommendations, dep-audit findings |
| 4 | 70B | Flag only | deep AIOS review, SOP drafts |

**"Flag"** = stored in `nightshift.flags`, shown in dashboard, included in digest.

---

## New Components

### 1. `agent/src/applier.ts`

Receives structured output from job handlers and applies changes.

Job handlers return a typed `ApplyAction`:

```typescript
type ApplyAction =
  | { type: 'file-write'; path: string; content: string; commitMsg?: string }
  | { type: 'git-commit'; repoPath: string; message: string }
  | { type: 'flag'; title: string; detail: string; tier: number }
  | { type: 'noop' }
```

- `file-write`: writes content to path. If `commitMsg` is provided, applier runs `git add <path>` then `git commit -m commitMsg` in the file's repo root.
- `git-commit`: stages all changes in `repoPath` (`git add -A`) and commits with `message`. Used when a job modifies multiple files.
- `flag`: inserts into `nightshift.flags`
- `noop`: job ran, nothing to apply (logged to history only)

Git commits are local only — no auto-push to remote.

### 2. `nightshift.flags` table (Postgres)

```sql
CREATE TABLE nightshift.flags (
  id TEXT PRIMARY KEY,
  job_category TEXT NOT NULL,
  title TEXT NOT NULL,
  detail TEXT,
  project TEXT NOT NULL,
  tier INT NOT NULL,
  status TEXT NOT NULL DEFAULT 'open',  -- open | dismissed | resolved
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  resolved_at TIMESTAMPTZ,
  machine TEXT
);
```

### 3. `agent/src/jobs/digest.ts`

Runs daily at 6am Pi time (systemd timer).

Fetches from Postgres:
- Last 24h of `nightshift.history` (auto-applied changes)
- All `nightshift.flags` with `status = 'open'`

Formats two messages:
- **Telegram**: Short summary (< 30 lines)
- **Email**: Full detail to AnchorGroupOps@gmail.com via Gmail SMTP

Digest format:

```
🌙 Night Shift — Jun 3

Auto-applied (3):
• AIS-OS: priorities.md date updated
• Night-shift: 2 stale branches removed
• FUB-sync: lint 8 issues fixed

Flagged for review (1):
• n8n-audit: SOLD workflow has no error handling

→ 192.168.7.222:3002
```

### 4. Dashboard update

Add a **Flags** panel to the existing Express dashboard at `192.168.7.222:3002`:
- `/flags` route — table of open flags, sortable by tier/project/age
- Inline dismiss/resolve buttons (POST to `/flags/:id/status`)
- Machine grid and queue summary unchanged

---

## Activation

Digest job runs as a **separate** systemd timer on Pi (not through the main job queue):

```ini
# /etc/systemd/system/nightshift-digest.timer
[Timer]
OnCalendar=*-*-* 06:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

The companion `.service` unit runs `node dist/jobs/digest.js` directly. No changes needed to PC/Mac activation or the main nightshift.service.

---

## Job Handler Output (Required Change)

All existing job handlers (`lint.ts`, `maintenance.ts`, `aios-review.ts`, etc.) must be updated to return `ApplyAction[]` instead of `void`. Handlers that previously only logged output now return structured actions.

Example update to `aios-review.ts`:

```typescript
// Before
async function run(project: Project): Promise<void> {
  const result = await llm.prompt('...');
  log(result);
}

// After
async function run(project: Project): Promise<ApplyAction[]> {
  const result = await llm.prompt('...');
  if (result.staleness === 'minor') {
    return [{ type: 'file-write', path: '...', content: result.updated, commitMsg: 'aios-review: update priorities date' }];
  }
  return [{ type: 'flag', title: 'Stale context detected', detail: result.summary, tier: 4 }];
}
```

---

## Credentials Needed

| Service | Credential | Source |
|---------|-----------|--------|
| Gmail SMTP | App password | `H:\AI\Secrets\.env.master.private` |
| Telegram Bot | Bot token + chat ID | `H:\AI\Secrets\.env.master.private` |

Both already exist in the master env file — just need to be wired into Night Shift's env vars.

---

## What's NOT Changing

- Existing Night Shift job handlers (logic unchanged, only output type changes)
- Postgres schema for `jobs`, `history`, `machines` — no modifications
- Tier routing, job scheduling, systemd service — unchanged
- Dashboard machine grid and queue views — only additive (flags panel added)
- PC/Mac activation (Night Light, launchd) — unchanged

---

## Success Criteria

1. After a Night Shift run, auto-applied changes appear in `nightshift.history` with `machine` and `llm_used` fields populated
2. Tier 3+ findings appear in `nightshift.flags` with `status = 'open'`
3. At 6am daily, a Telegram message and email arrive summarizing the night's activity
4. Dashboard shows open flags at `/flags` with working dismiss/resolve controls
5. Zero unintended file changes — applier only writes when job explicitly returns `file-write` action
