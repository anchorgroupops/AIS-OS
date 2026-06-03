# Decisions Log

Append-only record of meaningful decisions and why they were made. `/level-up` Phase 2 (Method interview) writes scoped automation specs here. You can also append manually whenever you decide something worth remembering.

**Format per entry:**

```
## YYYY-MM-DD — Short title

**Decision:** what was decided.

**Why:** the reasoning, constraints, and what would change your mind.

**Alternatives considered:** what else was on the table.

**Owner:** who's accountable.
```

Keep it terse. Future-you will thank present-you for capturing the *why*, not just the *what*.

---

## 2026-06-03 — Self-improvement loop: Auto-Apply + Daily Digest (v2.1.0)

**Decision:** Tier 0–2 job findings auto-apply (file writes + git commits); Tier 3–4 findings go to `nightshift.flags` table for human review. A 6am daily digest fires via systemd timer, sends Telegram (HTML) + optional Gmail summary.

**Why:** Joel chose Option C (auto-apply) over the draft queue approach. The tier boundary at 2/3 preserves control over LLM-heavy analysis (n8n-audit, aios-review) while letting maintenance jobs run fully hands-off.

**Alternatives considered:** Dashboard-only (no auto-apply), Draft queue (approve before apply). Chose auto-apply because the loop exists to reduce manual work — requiring approval negates much of the value.

**Owner:** Joel / Claude Code

---
