# Self-Improvement Loop Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close the Night Shift feedback loop by auto-applying Tier 0–2 job findings and flagging Tier 3–4 findings for review, with a daily 6am digest sent to Telegram and email.

**Architecture:** Jobs return `ApplyAction[]` alongside their existing `JobResult.summary`. A new `applier.ts` module processes those actions — writing files, running git commits, or inserting rows into a new `nightshift.flags` table. A standalone `digest.ts` script runs at 6am via a dedicated systemd timer, queries the last 24h of history plus open flags, and sends formatted messages to Telegram and Gmail.

**Tech Stack:** Node.js / TypeScript (ESM), `pg` (PostgreSQL), `nodemailer` (Gmail SMTP), Telegram Bot API (direct HTTP), Express (dashboard)

**Spec:** `docs/superpowers/specs/2026-06-03-self-improve-loop-design.md`

---

## File Map

| Action | Path | Responsibility |
|--------|------|---------------|
| Modify | `agent/src/types.ts` | Add `ApplyAction` union type; add `actions?` to `JobResult` |
| Modify | `agent/src/db.ts` | Add `nightshift.flags` table to `initSchema()` |
| Create | `agent/src/flags.ts` | `insertFlag`, `getOpenFlags`, `setFlagStatus` |
| Create | `agent/src/applier.ts` | `applyActions(actions, jobCategory, machine)` |
| Modify | `agent/src/runner.ts` | Call `applyActions` after successful job completion |
| Modify | `agent/src/jobs/aios-review.ts` | Return `flag` action instead of plain summary text |
| Modify | `agent/src/jobs/n8n-audit.ts` | Return `flag` action instead of plain summary text |
| Create | `agent/src/jobs/digest.ts` | Standalone script: query DB → Telegram + Gmail |
| Modify | `agent/package.json` | Add `nodemailer` and `@types/nodemailer` |
| Modify | `dashboard/src/index.ts` | Add `/flags` route, POST `/flags/:id/status`, flags count on overview |

---

## Task 1: Add `ApplyAction` type and update `JobResult`

**Files:**
- Modify: `H:\AI\Projects\Night-shift\agent\src\types.ts`

- [ ] **Step 1: Add `ApplyAction` type after the `LlmResponse` interface (line 108)**

Open `agent/src/types.ts`. After line 108 (`}`), add:

```typescript
/** Actions a job can return for the applier to execute */
export type ApplyAction =
  | { type: 'file-write'; path: string; content: string; commitMsg?: string }
  | { type: 'git-commit'; repoPath: string; message: string }
  | { type: 'flag'; title: string; detail?: string; tier: number; project: string }
  | { type: 'noop' };
```

- [ ] **Step 2: Add `actions?` field to `JobResult`**

Find the `JobResult` interface (line 111–115) and update it:

```typescript
/** What a job executor returns */
export interface JobResult {
  success: boolean;
  summary: string;
  details?: string;
  actions?: ApplyAction[];
}
```

- [ ] **Step 3: Type-check**

```bash
cd H:\AI\Projects\Night-shift\agent
npx tsc --noEmit
```

Expected: zero errors (existing handlers return `{ success, summary }` which still satisfies the interface since `actions` is optional).

- [ ] **Step 4: Commit**

```bash
cd H:\AI\Projects\Night-shift
git add agent/src/types.ts
git commit -m "feat: add ApplyAction type and actions field to JobResult"
```

---

## Task 2: Add `nightshift.flags` table to schema

**Files:**
- Modify: `H:\AI\Projects\Night-shift\agent\src\db.ts`

- [ ] **Step 1: Add the flags table DDL to `initSchema()`**

In `agent/src/db.ts`, find the end of the `pool.query(...)` string inside `initSchema()`. After the `idx_history_completed` index line (before the closing backtick), add:

```sql
    CREATE TABLE IF NOT EXISTS nightshift.flags (
      id TEXT PRIMARY KEY,
      job_category TEXT NOT NULL,
      title TEXT NOT NULL,
      detail TEXT,
      project TEXT NOT NULL,
      tier INT NOT NULL,
      status TEXT NOT NULL DEFAULT 'open',
      created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
      resolved_at TIMESTAMPTZ,
      machine TEXT
    );

    CREATE INDEX IF NOT EXISTS idx_flags_status ON nightshift.flags(status);
```

- [ ] **Step 2: Apply schema to Pi**

SSH into Pi and run:

```bash
cd ~/night-shift/agent && node -e "
import('./dist/db.js').then(m => m.initSchema()).then(() => { console.log('done'); process.exit(0); })
"
```

Expected output: `DB schema ready`

Verify the table exists:
```bash
psql -h 192.168.7.222 -U n8n -d n8n_ai -c "\dt nightshift.*"
```

Expected: `nightshift.flags` appears in the list.

- [ ] **Step 3: Commit**

```bash
cd H:\AI\Projects\Night-shift
git add agent/src/db.ts
git commit -m "feat: add nightshift.flags table to schema"
```

---

## Task 3: Create `flags.ts`

**Files:**
- Create: `H:\AI\Projects\Night-shift\agent\src\flags.ts`

- [ ] **Step 1: Write `flags.ts`**

```typescript
import { randomBytes } from "node:crypto";
import { getPool } from "./db.js";

export interface FlagEntry {
  id: string;
  jobCategory: string;
  title: string;
  detail: string | null;
  project: string;
  tier: number;
  status: "open" | "dismissed" | "resolved";
  createdAt: string;
  resolvedAt: string | null;
  machine: string | null;
}

export async function insertFlag(entry: {
  jobCategory: string;
  title: string;
  detail?: string;
  project: string;
  tier: number;
  machine?: string;
}): Promise<void> {
  const id = randomBytes(8).toString("hex");
  await getPool().query(
    `INSERT INTO nightshift.flags (id, job_category, title, detail, project, tier, machine)
     VALUES ($1, $2, $3, $4, $5, $6, $7)`,
    [id, entry.jobCategory, entry.title, entry.detail ?? null, entry.project, entry.tier, entry.machine ?? null],
  );
}

export async function getOpenFlags(): Promise<FlagEntry[]> {
  const { rows } = await getPool().query(
    `SELECT id, job_category AS "jobCategory", title, detail, project, tier, status,
            created_at AS "createdAt", resolved_at AS "resolvedAt", machine
     FROM nightshift.flags WHERE status = 'open' ORDER BY created_at DESC`,
  );
  return rows as FlagEntry[];
}

export async function setFlagStatus(id: string, status: "dismissed" | "resolved"): Promise<void> {
  await getPool().query(
    `UPDATE nightshift.flags SET status = $1, resolved_at = now() WHERE id = $2`,
    [status, id],
  );
}
```

- [ ] **Step 2: Type-check**

```bash
cd H:\AI\Projects\Night-shift\agent
npx tsc --noEmit
```

Expected: zero errors.

- [ ] **Step 3: Commit**

```bash
cd H:\AI\Projects\Night-shift
git add agent/src/flags.ts
git commit -m "feat: add flags.ts for nightshift.flags CRUD"
```

---

## Task 4: Create `applier.ts`

**Files:**
- Create: `H:\AI\Projects\Night-shift\agent\src\applier.ts`

- [ ] **Step 1: Write `applier.ts`**

```typescript
import { writeFileSync, existsSync } from "node:fs";
import { execFileSync } from "node:child_process";
import { dirname, join } from "node:path";
import { insertFlag } from "./flags.js";
import { log } from "./log.js";
import type { ApplyAction } from "./types.js";

function findGitRoot(filePath: string): string {
  let dir = dirname(filePath);
  while (dir !== dirname(dir)) {
    if (existsSync(join(dir, ".git"))) return dir;
    dir = dirname(dir);
  }
  throw new Error(`No git repo found above ${filePath}`);
}

function gitCommit(repoRoot: string, message: string): void {
  try {
    // Check if there is anything staged
    execFileSync("git", ["diff", "--cached", "--quiet"], { cwd: repoRoot });
    // Exit 0 means nothing staged — skip
    log("info", `applier: nothing staged in ${repoRoot}, skipping commit`);
  } catch {
    // Exit 1 means something IS staged — commit
    execFileSync("git", ["commit", "-m", message], { cwd: repoRoot });
    log("ok", `applier: committed "${message}"`);
  }
}

export async function applyActions(
  actions: ApplyAction[],
  jobCategory: string,
  machine: string,
): Promise<void> {
  for (const action of actions) {
    try {
      switch (action.type) {
        case "file-write": {
          writeFileSync(action.path, action.content, "utf-8");
          log("ok", `applier: wrote ${action.path}`);
          if (action.commitMsg) {
            const repoRoot = findGitRoot(action.path);
            execFileSync("git", ["add", action.path], { cwd: repoRoot });
            gitCommit(repoRoot, action.commitMsg);
          }
          break;
        }

        case "git-commit": {
          execFileSync("git", ["add", "-A"], { cwd: action.repoPath });
          gitCommit(action.repoPath, action.message);
          break;
        }

        case "flag": {
          await insertFlag({
            jobCategory,
            title: action.title,
            detail: action.detail,
            project: action.project,
            tier: action.tier,
            machine,
          });
          log("info", `applier: flagged "${action.title}"`);
          break;
        }

        case "noop":
          break;
      }
    } catch (err: any) {
      log("error", `applier: action ${action.type} failed — ${err.message}`);
    }
  }
}
```

- [ ] **Step 2: Type-check**

```bash
cd H:\AI\Projects\Night-shift\agent
npx tsc --noEmit
```

Expected: zero errors.

- [ ] **Step 3: Commit**

```bash
cd H:\AI\Projects\Night-shift
git add agent/src/applier.ts
git commit -m "feat: add applier.ts to execute ApplyAction arrays"
```

---

## Task 5: Wire applier into `runner.ts`

**Files:**
- Modify: `H:\AI\Projects\Night-shift\agent\src\runner.ts`

- [ ] **Step 1: Add import for `applyActions` at top of `runner.ts`**

After the existing imports, add:

```typescript
import { applyActions } from "./applier.js";
```

- [ ] **Step 2: Call `applyActions` after `markCompleted` inside the `if (result.success)` block**

Find this block (lines 67–73):

```typescript
    if (result.success) {
      await markCompleted(job.id, result.summary);
      log("ok", `✓ Done (${(durationMs / 1000).toFixed(1)}s): ${job.title}`);
    } else {
      await markFailed(job.id, result.summary);
      log("warn", `✗ Failed: ${job.title} — ${result.summary}`);
    }
```

Replace it with:

```typescript
    if (result.success) {
      await markCompleted(job.id, result.summary);
      log("ok", `✓ Done (${(durationMs / 1000).toFixed(1)}s): ${job.title}`);
      if (result.actions && result.actions.length > 0) {
        await applyActions(result.actions, job.category, mode);
      }
    } else {
      await markFailed(job.id, result.summary);
      log("warn", `✗ Failed: ${job.title} — ${result.summary}`);
    }
```

Note: `mode` is already declared earlier via `const { mode } = getConfig();` — but it's declared AFTER the try block. Move the `const { mode } = getConfig();` declaration to BEFORE the `try` block (it's currently on line 65 inside the try, after `result` is awaited). Pull it out:

Find:

```typescript
    const result: JobResult = await withTimeout(handler.execute(job, llm), timeoutMs, job.title);
    const durationMs = Date.now() - startTime;
    const { mode } = getConfig();
```

Replace with:

```typescript
    const { mode } = getConfig();
    const result: JobResult = await withTimeout(handler.execute(job, llm), timeoutMs, job.title);
    const durationMs = Date.now() - startTime;
```

- [ ] **Step 3: Type-check**

```bash
cd H:\AI\Projects\Night-shift\agent
npx tsc --noEmit
```

Expected: zero errors.

- [ ] **Step 4: Commit**

```bash
cd H:\AI\Projects\Night-shift
git add agent/src/runner.ts
git commit -m "feat: runner calls applyActions on successful job completion"
```

---

## Task 6: Update `aios-review.ts` to return flag actions

**Files:**
- Modify: `H:\AI\Projects\Night-shift\agent\src\jobs\aios-review.ts`

The handler is Tier 4 — findings must produce a `flag` action, not auto-apply.

- [ ] **Step 1: Add `ApplyAction` import**

At the top of `aios-review.ts`, change:

```typescript
import type { Job, JobHandler, JobResult, LlmClient } from "../types.js";
```

To:

```typescript
import type { Job, JobHandler, JobResult, LlmClient, ApplyAction } from "../types.js";
```

- [ ] **Step 2: Update the no-findings return inside `execute()`**

Find:

```typescript
    if (findings.length === 0 && Object.keys(contextContent).length > 0) {
      return { success: true, summary: "AIOS context looks healthy — all files present and recently updated" };
    }
```

Replace with:

```typescript
    if (findings.length === 0 && Object.keys(contextContent).length > 0) {
      return {
        success: true,
        summary: "AIOS context looks healthy — all files present and recently updated",
        actions: [{ type: "noop" } satisfies ApplyAction],
      };
    }
```

- [ ] **Step 3: Update the findings return at the end of `execute()`**

Find:

```typescript
    try {
      const analysis = await llm.ask(
        `You are reviewing an AI Operating System (AIOS) context directory. ` +
        `Based on the findings and context files below, write 3-5 concise bullet points ` +
        `suggesting what should be updated or acted on. Be specific.\n\n` +
        `Findings:\n${findings.map(f => `- ${f}`).join("\n")}\n\n` +
        `Context sample:\n${contextSample}`,
        { cloud: true },
      );
      return {
        success: true,
        summary: `AIOS context review:\n${findings.map(f => `• ${f}`).join("\n")}\n\nSuggestions:\n${analysis.text}`,
      };
    } catch {
      return {
        success: true,
        summary: `AIOS context review:\n${findings.map(f => `• ${f}`).join("\n")}`,
      };
    }
```

Replace with:

```typescript
    const findingsSummary = findings.map(f => `• ${f}`).join("\n");

    try {
      const analysis = await llm.ask(
        `You are reviewing an AI Operating System (AIOS) context directory. ` +
        `Based on the findings and context files below, write 3-5 concise bullet points ` +
        `suggesting what should be updated or acted on. Be specific.\n\n` +
        `Findings:\n${findings.map(f => `- ${f}`).join("\n")}\n\n` +
        `Context sample:\n${contextSample}`,
        { cloud: true },
      );
      const detail = `Findings:\n${findingsSummary}\n\nSuggestions:\n${analysis.text}`;
      return {
        success: true,
        summary: `AIOS context review: ${findings.length} finding(s)`,
        actions: [{
          type: "flag",
          title: `AIOS context needs review (${findings.length} finding${findings.length > 1 ? "s" : ""})`,
          detail,
          tier: 4,
          project: job.project,
        } satisfies ApplyAction],
      };
    } catch {
      return {
        success: true,
        summary: `AIOS context review: ${findings.length} finding(s)`,
        actions: [{
          type: "flag",
          title: `AIOS context needs review (${findings.length} finding${findings.length > 1 ? "s" : ""})`,
          detail: `Findings:\n${findingsSummary}`,
          tier: 4,
          project: job.project,
        } satisfies ApplyAction],
      };
    }
```

- [ ] **Step 4: Type-check**

```bash
cd H:\AI\Projects\Night-shift\agent
npx tsc --noEmit
```

Expected: zero errors.

- [ ] **Step 5: Commit**

```bash
cd H:\AI\Projects\Night-shift
git add agent/src/jobs/aios-review.ts
git commit -m "feat: aios-review returns flag action instead of plain summary"
```

---

## Task 7: Update `n8n-audit.ts` to return flag actions

**Files:**
- Modify: `H:\AI\Projects\Night-shift\agent\src\jobs\n8n-audit.ts`

Tier 3 — same pattern as aios-review.

- [ ] **Step 1: Add `ApplyAction` import**

Change:

```typescript
import type { Job, JobHandler, JobResult, LlmClient } from "../types.js";
```

To:

```typescript
import type { Job, JobHandler, JobResult, LlmClient, ApplyAction } from "../types.js";
```

- [ ] **Step 2: Update the no-findings return inside `execute()`**

Find:

```typescript
    if (findings.length === 0) {
      return { success: true, summary: `n8n: ${workflows.length} workflows, ${activeCount} active — no issues found` };
    }
```

Replace with:

```typescript
    if (findings.length === 0) {
      return {
        success: true,
        summary: `n8n: ${workflows.length} workflows, ${activeCount} active — no issues found`,
        actions: [{ type: "noop" } satisfies ApplyAction],
      };
    }
```

- [ ] **Step 3: Update the findings returns at the end of `execute()`**

Find:

```typescript
    // Use LLM to prioritize findings
    try {
      const analysis = await llm.ask(
        `You are auditing n8n automation workflows for a real estate team's operations manager. ` +
        `Based on these findings, write 2-3 actionable recommendations in order of priority. Be concise.\n\n` +
        `Findings:\n${findings.join("\n\n")}`,
        { cloud: false },
      );
      return {
        success: true,
        summary: `${summary}\n\nRecommendations:\n${analysis.text}`,
      };
    } catch {
      return { success: true, summary };
    }
```

Replace with:

```typescript
    // Use LLM to prioritize findings
    const flagAction = (detail: string): ApplyAction => ({
      type: "flag",
      title: `n8n audit: ${findings.length} finding${findings.length > 1 ? "s" : ""}`,
      detail,
      tier: 3,
      project: job.project,
    });

    try {
      const analysis = await llm.ask(
        `You are auditing n8n automation workflows for a real estate team's operations manager. ` +
        `Based on these findings, write 2-3 actionable recommendations in order of priority. Be concise.\n\n` +
        `Findings:\n${findings.join("\n\n")}`,
        { cloud: false },
      );
      const detail = `${summary}\n\nRecommendations:\n${analysis.text}`;
      return {
        success: true,
        summary: `n8n audit: ${findings.length} finding(s)`,
        actions: [flagAction(detail)],
      };
    } catch {
      return {
        success: true,
        summary: `n8n audit: ${findings.length} finding(s)`,
        actions: [flagAction(summary)],
      };
    }
```

- [ ] **Step 4: Type-check**

```bash
cd H:\AI\Projects\Night-shift\agent
npx tsc --noEmit
```

Expected: zero errors.

- [ ] **Step 5: Commit**

```bash
cd H:\AI\Projects\Night-shift
git add agent/src/jobs/n8n-audit.ts
git commit -m "feat: n8n-audit returns flag action instead of plain summary"
```

---

## Task 8: Install `nodemailer` and create `digest.ts`

**Files:**
- Modify: `H:\AI\Projects\Night-shift\agent\package.json`
- Create: `H:\AI\Projects\Night-shift\agent\src\jobs\digest.ts`

- [ ] **Step 1: Install nodemailer**

```bash
cd H:\AI\Projects\Night-shift\agent
npm install nodemailer
npm install -D @types/nodemailer
```

Expected: `package.json` and `package-lock.json` updated, no errors.

- [ ] **Step 2: Write `digest.ts`**

Create `H:\AI\Projects\Night-shift\agent\src\jobs\digest.ts`:

```typescript
// Standalone script — run directly: node dist/jobs/digest.js
// NOT a job handler (does not implement JobHandler interface).

import pg from "pg";
import nodemailer from "nodemailer";

const { Pool } = pg;

const pool = new Pool({
  host:     process.env.NS_DB_HOST     ?? "192.168.7.222",
  port:     parseInt(process.env.NS_DB_PORT     ?? "5432"),
  database: process.env.NS_DB_DATABASE ?? "n8n_ai",
  user:     process.env.NS_DB_USER     ?? "n8n",
  password: process.env.NS_DB_PASSWORD ?? "",
});

interface HistoryRow {
  title: string;
  project: string;
  machine: string | null;
  result: string;
  category: string;
  outcome: "applied" | "failed";
}

interface FlagRow {
  title: string;
  detail: string | null;
  project: string;
  tier: number;
  job_category: string;
}

async function fetchData(): Promise<{ history: HistoryRow[]; flags: FlagRow[] }> {
  const [{ rows: history }, { rows: flags }] = await Promise.all([
    pool.query<HistoryRow>(`
      SELECT title, project, machine, result, category,
             CASE WHEN result LIKE 'ERROR:%' THEN 'failed' ELSE 'applied' END AS outcome
      FROM nightshift.history
      WHERE completed_at >= now() - interval '24 hours'
      ORDER BY completed_at DESC
    `),
    pool.query<FlagRow>(`
      SELECT title, detail, project, tier, job_category
      FROM nightshift.flags
      WHERE status = 'open'
      ORDER BY created_at DESC
      LIMIT 20
    `),
  ]);
  return { history, flags };
}

function formatTelegram(history: HistoryRow[], flags: FlagRow[]): string {
  const today = new Date().toLocaleDateString("en-US", { month: "short", day: "numeric" });
  const applied = history.filter(h => h.outcome === "applied");
  const failed  = history.filter(h => h.outcome === "failed");
  const lines: string[] = [`🌙 *Night Shift — ${today}*`];

  if (applied.length > 0) {
    lines.push(`\n✅ *Auto\\-applied (${applied.length}):*`);
    for (const h of applied.slice(0, 8)) {
      lines.push(`• ${h.project}: ${h.title}`);
    }
  }

  if (failed.length > 0) {
    lines.push(`\n⚠️ *Failed (${failed.length}):*`);
    for (const h of failed.slice(0, 3)) {
      lines.push(`• ${h.project}: ${h.title}`);
    }
  }

  if (flags.length > 0) {
    lines.push(`\n🚩 *Flagged for review (${flags.length}):*`);
    for (const f of flags.slice(0, 5)) {
      lines.push(`• ${f.job_category}: ${f.title}`);
    }
  }

  if (applied.length === 0 && flags.length === 0 && failed.length === 0) {
    lines.push("\n✓ Nothing ran overnight — all machines quiet\\.");
  }

  lines.push(`\n→ http://192\\.168\\.7\\.222:3002`);
  return lines.join("\n");
}

function formatEmail(history: HistoryRow[], flags: FlagRow[]): string {
  const today = new Date().toLocaleDateString("en-US", { month: "long", day: "numeric", year: "numeric" });
  const applied = history.filter(h => h.outcome === "applied");
  const failed  = history.filter(h => h.outcome === "failed");

  const lines = [
    `Night Shift Digest — ${today}`,
    `${"─".repeat(40)}`,
    "",
    `AUTO-APPLIED: ${applied.length}`,
    ...applied.map(h => `  • ${h.project}: ${h.title}`),
    "",
    `FAILED: ${failed.length}`,
    ...failed.map(h => `  • ${h.project}: ${h.title}`),
    "",
    `FLAGGED FOR REVIEW: ${flags.length}`,
    ...flags.map(f =>
      `  • [T${f.tier}] ${f.job_category}: ${f.title}` +
      (f.detail ? `\n    ${f.detail.slice(0, 300)}` : ""),
    ),
    "",
    `Dashboard: http://192.168.7.222:3002`,
  ];
  return lines.join("\n");
}

async function sendTelegram(text: string): Promise<void> {
  const token  = process.env.NS_TELEGRAM_BOT_TOKEN;
  const chatId = process.env.NS_TELEGRAM_CHAT_ID;
  if (!token || !chatId) { console.log("Telegram not configured — skipping"); return; }

  const res = await fetch(`https://api.telegram.org/bot${token}/sendMessage`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ chat_id: chatId, text, parse_mode: "MarkdownV2" }),
  });
  if (!res.ok) throw new Error(`Telegram API ${res.status}: ${await res.text()}`);
}

async function sendEmail(subject: string, text: string): Promise<void> {
  const gmailUser = process.env.NS_GMAIL_USER;
  const gmailPass = process.env.NS_GMAIL_APP_PASSWORD;
  const emailTo   = process.env.NS_DIGEST_EMAIL_TO;
  if (!gmailUser || !gmailPass || !emailTo) { console.log("Gmail not configured — skipping"); return; }

  const transporter = nodemailer.createTransport({
    service: "gmail",
    auth: { user: gmailUser, pass: gmailPass },
  });
  await transporter.sendMail({ from: gmailUser, to: emailTo, subject, text });
}

async function main(): Promise<void> {
  const { history, flags } = await fetchData();
  const today = new Date().toLocaleDateString("en-US", { month: "short", day: "numeric" });
  const applied = history.filter(h => h.outcome === "applied");

  const telegramText = formatTelegram(history, flags);
  const emailText    = formatEmail(history, flags);
  const emailSubject = `Night Shift — ${today} (${applied.length} applied, ${flags.length} flagged)`;

  const results = await Promise.allSettled([
    sendTelegram(telegramText),
    sendEmail(emailSubject, emailText),
  ]);

  for (const [i, r] of results.entries()) {
    if (r.status === "rejected") {
      console.error(`[${i === 0 ? "Telegram" : "Email"}] failed:`, r.reason?.message ?? r.reason);
    } else {
      console.log(`[${i === 0 ? "Telegram" : "Email"}] sent`);
    }
  }

  await pool.end();
  console.log("Digest complete.");
}

main().catch(e => { console.error(e); process.exit(1); });
```

- [ ] **Step 3: Type-check**

```bash
cd H:\AI\Projects\Night-shift\agent
npx tsc --noEmit
```

Expected: zero errors.

- [ ] **Step 4: Commit**

```bash
cd H:\AI\Projects\Night-shift
git add agent/src/jobs/digest.ts agent/package.json agent/package-lock.json
git commit -m "feat: add digest.ts standalone script for 6am Telegram + email digest"
```

---

## Task 9: Add `/flags` route to dashboard

**Files:**
- Modify: `H:\AI\Projects\Night-shift\dashboard\src\index.ts`

- [ ] **Step 1: Add `express.json()` middleware**

After `const app = express();` (line 15), add:

```typescript
app.use(express.json());
```

- [ ] **Step 2: Add `Flags` to the nav in the `page()` helper**

Find the `<nav>` block inside `page()`:

```html
  <nav>
    <a href="/">Overview</a>
    <a href="/queue">Queue</a>
    <a href="/history">History</a>
    <a href="/insights">Insights</a>
  </nav>
```

Replace with:

```html
  <nav>
    <a href="/">Overview</a>
    <a href="/queue">Queue</a>
    <a href="/history">History</a>
    <a href="/insights">Insights</a>
    <a href="/flags">Flags</a>
  </nav>
```

- [ ] **Step 3: Add open flags count to the Overview stat row**

In the `/` route handler, add a flags query to the `Promise.all`:

Find:

```typescript
  const [machines, stats, recent] = await Promise.all([
    DB.query(`SELECT * FROM nightshift.machines ORDER BY last_seen DESC`),
    DB.query(`
      SELECT
        COUNT(*) FILTER (WHERE status='pending' AND needs_confirmation=false) AS pending,
        ...
      FROM nightshift.jobs
    `),
    DB.query(`...FROM nightshift.history...`),
  ]);
```

Replace with:

```typescript
  const [machines, stats, recent, openFlags] = await Promise.all([
    DB.query(`SELECT * FROM nightshift.machines ORDER BY last_seen DESC`),
    DB.query(`
      SELECT
        COUNT(*) FILTER (WHERE status='pending' AND needs_confirmation=false) AS pending,
        COUNT(*) FILTER (WHERE status='running')                              AS running,
        COUNT(*) FILTER (WHERE status='completed')                            AS completed,
        COUNT(*) FILTER (WHERE status='failed')                               AS failed,
        COUNT(*) FILTER (WHERE status='pending' AND needs_confirmation=true)  AS needs_confirmation
      FROM nightshift.jobs
    `),
    DB.query(`
      SELECT title, project, machine, completed_at, llm_used,
             CASE WHEN result LIKE 'ERROR:%' THEN 'failed' ELSE 'completed' END AS outcome
      FROM nightshift.history
      ORDER BY completed_at DESC LIMIT 10
    `),
    DB.query(`SELECT COUNT(*) AS cnt FROM nightshift.flags WHERE status = 'open'`),
  ]);
```

Then in the stat-row HTML, add a flags card after the existing stats. Find:

```typescript
      ${parseInt(s.needs_confirmation) > 0 ? `<div class="stat"><div class="val">${s.needs_confirmation}</div><div class="lbl">Need Approval</div></div>` : ""}
```

Add after it:

```typescript
      ${parseInt(openFlags.rows[0].cnt) > 0 ? `<div class="stat"><a href="/flags" style="text-decoration:none"><div class="val" style="color:#f87171">${openFlags.rows[0].cnt}</div><div class="lbl">Open Flags</div></a></div>` : ""}
```

- [ ] **Step 4: Add GET `/flags` route**

Add before the `app.listen(...)` call at the bottom:

```typescript
app.get("/flags", async (_req, res) => {
  const result = await DB.query(`
    SELECT id, job_category, title, detail, project, tier, created_at
    FROM nightshift.flags
    WHERE status = 'open'
    ORDER BY created_at DESC
  `);

  const rows = result.rows.length === 0
    ? `<tr><td colspan="6" class="empty">No open flags. All clear.</td></tr>`
    : result.rows.map(r => `
        <tr>
          <td>${r.title}</td>
          <td>${r.project}</td>
          <td>${r.job_category}</td>
          <td><span class="tier">T${r.tier}</span></td>
          <td>${new Date(r.created_at).toLocaleString()}</td>
          <td>
            <button onclick="flagAction('${r.id}','resolved')"
              style="background:#1a3a1a;color:#4ade80;border:none;padding:3px 10px;border-radius:4px;cursor:pointer;font-size:12px;margin-right:4px">
              Resolve
            </button>
            <button onclick="flagAction('${r.id}','dismissed')"
              style="background:#2a1a1a;color:#f87171;border:none;padding:3px 10px;border-radius:4px;cursor:pointer;font-size:12px">
              Dismiss
            </button>
          </td>
        </tr>
        ${r.detail ? `<tr><td colspan="6" style="padding:0 10px 10px;color:#666;font-size:12px;white-space:pre-wrap">${r.detail.slice(0, 500)}</td></tr>` : ""}
      `).join("");

  res.send(page("Flags", `
    <script>
      async function flagAction(id, status) {
        await fetch('/flags/' + id + '/status', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ status }),
        });
        location.reload();
      }
    </script>
    <h2>Open Flags (${result.rows.length})</h2>
    <table>
      <tr><th>Title</th><th>Project</th><th>Category</th><th>Tier</th><th>Created</th><th>Actions</th></tr>
      ${rows}
    </table>
  `));
});
```

- [ ] **Step 5: Add POST `/flags/:id/status` route**

Add immediately after the GET `/flags` route:

```typescript
app.post("/flags/:id/status", async (req, res) => {
  const { id } = req.params;
  const { status } = req.body as { status?: string };
  if (status !== "resolved" && status !== "dismissed") {
    res.status(400).json({ error: "status must be resolved or dismissed" });
    return;
  }
  await DB.query(
    `UPDATE nightshift.flags SET status = $1, resolved_at = now() WHERE id = $2`,
    [status, id],
  );
  res.json({ ok: true });
});
```

- [ ] **Step 6: Type-check**

```bash
cd H:\AI\Projects\Night-shift\dashboard
npx tsc --noEmit
```

Expected: zero errors.

- [ ] **Step 7: Commit**

```bash
cd H:\AI\Projects\Night-shift
git add dashboard/src/index.ts
git commit -m "feat: add /flags route with resolve/dismiss to dashboard"
```

---

## Task 10: Add env vars for digest to Pi's systemd service

**Files:**
- Modify on Pi: `/etc/systemd/system/nightshift.service` (add env vars)
- Create on Pi: `/etc/systemd/system/nightshift-digest.service`
- Create on Pi: `/etc/systemd/system/nightshift-digest.timer`

These steps run on the Pi (SSH in).

- [ ] **Step 1: Add digest env vars to the main nightshift.service**

On Pi, edit `/etc/systemd/system/nightshift.service` to add the five digest env vars (needed so the agent process can also use them if needed in future):

```ini
Environment=NS_TELEGRAM_BOT_TOKEN=<from .env.master.private>
Environment=NS_TELEGRAM_CHAT_ID=<from .env.master.private>
Environment=NS_GMAIL_USER=anchorgroupai@gmail.com
Environment=NS_GMAIL_APP_PASSWORD=<from .env.master.private>
Environment=NS_DIGEST_EMAIL_TO=anchorgroupops@gmail.com
```

Get the actual values from `H:\AI\Secrets\.env.master.private` on the PC and copy them.

- [ ] **Step 2: Create `/etc/systemd/system/nightshift-digest.service`**

```ini
[Unit]
Description=Night Shift Daily Digest
After=network.target postgresql.service

[Service]
Type=oneshot
User=joel
WorkingDirectory=/home/joel/night-shift/agent
EnvironmentFile=/home/joel/night-shift/agent/.env
ExecStart=/usr/bin/node dist/jobs/digest.js
StandardOutput=journal
StandardError=journal
```

Create `/home/joel/night-shift/agent/.env` with all required env vars:

```
NS_DB_HOST=192.168.7.222
NS_DB_PORT=5432
NS_DB_DATABASE=n8n_ai
NS_DB_USER=n8n
NS_DB_PASSWORD=<from .env.master.private>
NS_TELEGRAM_BOT_TOKEN=<from .env.master.private>
NS_TELEGRAM_CHAT_ID=<from .env.master.private>
NS_GMAIL_USER=anchorgroupai@gmail.com
NS_GMAIL_APP_PASSWORD=<from .env.master.private>
NS_DIGEST_EMAIL_TO=anchorgroupops@gmail.com
```

- [ ] **Step 3: Create `/etc/systemd/system/nightshift-digest.timer`**

```ini
[Unit]
Description=Night Shift Daily Digest Timer
Requires=nightshift-digest.service

[Timer]
OnCalendar=*-*-* 06:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

- [ ] **Step 4: Enable and test**

```bash
sudo systemctl daemon-reload
sudo systemctl enable nightshift-digest.timer
sudo systemctl start nightshift-digest.timer
systemctl list-timers --all | grep nightshift
```

Expected: `nightshift-digest.timer` shows with next trigger ~6am.

**Test the digest manually:**

```bash
cd ~/night-shift/agent
sudo systemctl start nightshift-digest.service
journalctl -u nightshift-digest.service -n 20
```

Expected output includes:
```
[Telegram] sent
[Email] sent
Digest complete.
```

---

## Task 11: Build and deploy to Pi

**Files:**
- `H:\AI\Projects\Night-shift\agent\dist\` (built output)

- [ ] **Step 1: Build on PC**

```bash
cd H:\AI\Projects\Night-shift\agent
npx tsc
```

Expected: `dist/` folder updated, no errors.

- [ ] **Step 2: Sync to Pi**

```bash
rsync -av --exclude=node_modules H:\AI\Projects\Night-shift\agent\dist\ joel@192.168.7.222:~/night-shift/agent/dist/
```

Or if using Syncthing (already set up), wait for sync to complete.

- [ ] **Step 3: Restart nightshift.service on Pi**

```bash
ssh joel@192.168.7.222 "sudo systemctl restart nightshift.service && sudo systemctl status nightshift.service"
```

Expected: `Active: active (running)`

- [ ] **Step 4: Verify flags table is being written**

After a job runs on Pi, check:

```bash
ssh joel@192.168.7.222 "psql -h localhost -U n8n -d n8n_ai -c 'SELECT * FROM nightshift.flags LIMIT 5;'"
```

Expected: rows appear after Tier 3–4 jobs complete.

- [ ] **Step 5: Verify dashboard `/flags` route**

Open `http://192.168.7.222:3002/flags` in a browser.

Expected: Flags table loads. Resolve/Dismiss buttons visible. Clicking Resolve updates the row (disappears on refresh).

- [ ] **Step 6: Final commit**

```bash
cd H:\AI\Projects\Night-shift
git add -A
git commit -m "build: compiled output for Pi deploy"
```

---

## Verification Checklist

After all tasks are complete:

- [ ] `npx tsc --noEmit` in `agent/` — zero errors
- [ ] `npx tsc --noEmit` in `dashboard/` — zero errors
- [ ] Night Shift agent restarts cleanly on Pi (`systemctl status nightshift`)
- [ ] A Tier 3 or 4 job run produces a row in `nightshift.flags`
- [ ] Dashboard `/flags` shows that row with working Resolve/Dismiss
- [ ] Manual digest run sends Telegram message + email to AnchorGroupOps
- [ ] Timer shows in `systemctl list-timers` with 6am trigger
- [ ] Overview page shows "Open Flags" count (red) when flags exist
