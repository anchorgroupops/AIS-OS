# Social Media Process Revamp — Design

**Date:** 2026-06-03
**Status:** Approved → Implementation
**SOP source:** https://docs.google.com/document/d/1wBdBfo0IVl1AGgtjgaDSW5IGUBuAyHaKC9bRIQl7zXo/edit
**Spec location:** `docs/superpowers/specs/2026-06-03-social-media-revamp-design.md`

---

## Problem

Currently, Under Contract and SOLD posts require someone to:
1. Write the post content from scratch (or from a template)
2. Pick the tone
3. Verify the photo matches the property
4. Post to Facebook/Instagram manually

Agents forget, write inconsistently, or skip the process. Joel ends up doing it himself.

---

## Goal

Agents check one box — "looks good" — and walk away. AI handles writing, tone selection, and posting.

---

## Architecture

One n8n workflow per milestone type (Under Contract, SOLD). Both follow the same pattern:

```
FUB Webhook (status change)
  → Extract property details (address, city, price, agent, beds/baths)
  → AI generates 3 tone variations
  → Email agent with 3 options + one-click approve buttons
  → Agent clicks approve → n8n receives approval
  → Post to Facebook Page via Meta Graph API
  → Log to nightshift history (or Slack/Blink confirmation)
```

---

## Post Templates (from SOP)

### Under Contract
5 tone options per SOP — workflow generates 3 (coastal, family, luxury):
- **Coastal:** Lifestyle-focused, beach/Florida references
- **Family:** Milestone/life chapter tone  
- **Luxury:** Premium experience, quality emphasis

Word count: 50–100 words. Fair Housing compliant (no school/crime refs). Ends with team tag.

### SOLD
Celebrates transaction + agent service quality. 50–120 words. Includes agent name.

---

## Approval Flow

1. Agent receives email: "Under Contract post ready — pick a tone"
   - 3 post previews (text only, they already have the photo)
   - 3 buttons: "Approve Coastal" / "Approve Family" / "Approve Luxury"
   - Each button hits a unique n8n webhook URL
2. On click: n8n posts to Facebook Page immediately
3. Joel gets a confirmation notification via Blink or email

**Agent effort: 10 seconds, one click.**

---

## Data Flow from FUB

FUB sends a webhook on status change. Required fields:
- `contact.first_name`, `contact.last_name` (buyer)
- `property.address`, `property.city`, `property.state`
- `property.price` (sale price)
- `property.bedrooms`, `property.bathrooms`
- `agent.name` (listing/selling agent)
- `tags` (to identify milestone: "Under Contract" or "Closed")

If FUB webhook doesn't include all fields, a secondary FUB API call fetches the deal record.

---

## Facebook/Meta Integration

**API:** Meta Graph API v19+
**Required:** Facebook Page Access Token (long-lived, from Meta for Developers)
**Endpoint:** `POST /{page-id}/feed`
**Instagram:** Separate endpoint via Business Account linked to the Page

Credentials: `META_PAGE_ACCESS_TOKEN`, `META_PAGE_ID` → save to `.env.master.private` and as n8n variables.

---

## Workflow Files

| File | Trigger | Posts to |
|------|---------|---------|
| `social-under-contract.json` | FUB webhook: Under Contract | Facebook + Instagram |
| `social-sold.json` | FUB webhook: SOLD/Closed | Facebook + Instagram |

---

## Scope

**In:** Under Contract + SOLD posts, 3-tone AI generation, email-based one-click approval, Facebook Page posting.

**Out:** Open House / Price Improvement / New Listing (Phase 2), Instagram carousel generation, image resizing/overlay, Blink approval flow (use email for now — simpler).

---

## Credentials Required

| Key | Source |
|-----|--------|
| `FUB_API_KEY` | FUB Settings → API |
| `META_PAGE_ACCESS_TOKEN` | Meta for Developers → Page Token |
| `META_PAGE_ID` | Facebook Page → About → Page ID |
| `OPENROUTER_API_KEY` | Already configured |
| n8n SMTP credential | For approval emails |

---

## Success Criteria

An agent gets a FUB status update to "Under Contract" at 10 AM. By 10:01 AM, they have an email in their inbox with 3 post options. They click one. The post appears on the Anchor Group Facebook page within 30 seconds. Joel gets a Blink confirmation. Agent wrote zero words.
