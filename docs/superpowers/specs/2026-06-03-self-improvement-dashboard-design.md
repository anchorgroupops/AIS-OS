# Self-Improvement Insights Dashboard — Design

**Date:** 2026-06-03
**Status:** Approved → Implementation
**Spec location:** `docs/superpowers/specs/2026-06-03-self-improvement-dashboard-design.md`

---

## Problem

Night Shift v2 is processing jobs across 19 projects, but the only visibility is a raw queue monitor (`:3002/history`). Joel has no way to answer: "Is AI actually doing more of my work week over week? Which areas are getting automated? Is the Default Shift happening?"

---

## Goal

A `/insights` page on the existing Night Shift dashboard that answers:
1. How much automated work happened this week vs. last?
2. Which categories/projects are getting the most AI attention?
3. Rough time-saved estimate (AI leverage in practice)

---

## Architecture

**No new infrastructure.** Extend `dashboard/src/index.ts` with a new Express route `/insights`. All data comes from `nightshift.history` in Postgres on the Pi (already populated, already accessible at `192.168.7.222:5432`).

---

## Data Model

All queries against `nightshift.history`:

| Metric | Query |
|--------|-------|
| This week's jobs | `WHERE completed_at >= date_trunc('week', now())` |
| Last week's jobs | `WHERE completed_at >= date_trunc('week', now()) - interval '1 week' AND < date_trunc('week', now())` |
| By category | `GROUP BY category` |
| By project | `GROUP BY project ORDER BY COUNT DESC LIMIT 10` |
| Time saved | Heuristic: T0=2min, T1/2=5min, T3=15min, T4=30min per completed job |

---

## UI Sections

1. **Stats row** — This week: N jobs, ~X hours saved | Last week: N jobs | Trend arrow
2. **By Category** — Horizontal bar breakdown (hardening / sync / lint / incomplete / n8n-audit / aios-review)
3. **Top Projects** — Table: project name, job count this week, time saved estimate
4. **30-day trend** — One row per week for past 4 weeks: jobs completed, categories, time saved

---

## Time-Saved Heuristic

| Tier | Per-job estimate | Rationale |
|------|-----------------|-----------|
| T0 | 2 min | Automated scan that would take 2 min to run manually |
| T1–T2 | 5 min | Small LLM task (summarize, lint, flag) |
| T3 | 15 min | Code analysis or stub fill — meaningful dev work |
| T4 | 30 min | Deep analysis on 70B model |

These are conservative estimates. The point is trend, not precision.

---

## Implementation

- Single new route `GET /insights` in `dashboard/src/index.ts`
- 3 parallel Postgres queries (this week, last week, by-category/project/tier breakdown)
- Same HTML helper pattern as existing routes (dark theme, stat cards, tables)
- Added to the nav bar alongside Overview / Queue / History

---

## Scope

**In:** New `/insights` route, time-saved heuristic, week-over-week trend, category/project breakdown.

**Out:** decisions/log.md integration (future), streak tracking, custom time ranges, notifications.

---

## Success Criteria

Joel opens `http://192.168.7.222:3002/insights` and can answer within 10 seconds: "Did AI do more this week than last week, and where?"
