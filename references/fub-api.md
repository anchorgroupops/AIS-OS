# Follow Up Boss (FUB) API — Reference

**Docs:** https://api.followupboss.com/
**Auth:** HTTP Basic auth — `Authorization: Basic base64(API_KEY:)`  
**Base URL:** `https://api.followupboss.com/v1/`
**API Key:** `H:\AI\Secrets\.env.master.private` → `FUB_API_KEY`

---

## Webhooks (for n8n triggers)

FUB can send webhooks on:
- Deal status changes (Under Contract, Closed/SOLD)
- New lead assigned
- Task completed
- Note added

**Setup:** FUB → Admin → API → Webhooks → Add webhook → paste n8n webhook URL → select events.

Webhook payload includes: `person` (contact), `deal` (property/price/status), `agent` (assigned agent).

---

## Key Endpoints

### Get a person (contact)
```
GET /v1/people/{id}
```

### Get deals
```
GET /v1/deals
GET /v1/deals/{id}
```

Deal object includes:
- `name` — deal title
- `stage` — current status (Under Contract, Closed, etc.)
- `value` — sale price
- `properties` — array of property objects

### Get property details
```
GET /v1/properties/{id}
```
Returns: `address`, `city`, `state`, `zip`, `bedrooms`, `bathrooms`, `price`

### List agents
```
GET /v1/users
```

---

## n8n Credential Setup

1. n8n → Credentials → New → HTTP Basic Auth
2. User: `<FUB_API_KEY>`
3. Password: (leave blank)
4. Name: "Follow Up Boss API"

---

## Social Media Workflow Integration

The `social-under-contract.json` and `social-sold.json` workflows receive FUB webhook payloads. Expected payload shape (FUB sends on status change):

```json
{
  "event": "dealUpdated",
  "deal": {
    "stage": "Under Contract",
    "value": 425000,
    "property": {
      "address": "123 Ocean Blvd",
      "city": "Palm Coast",
      "state": "FL",
      "bedrooms": 3,
      "bathrooms": 2
    }
  },
  "agent": {
    "name": "Agent Name",
    "email": "agent@example.com"
  },
  "contact": {
    "firstName": "Buyer",
    "lastName": "Name"
  }
}
```

If payload is missing fields, add a FUB API HTTP Request node after the webhook to fetch full deal details.

---

## n8n Variables Needed

| Variable | Value |
|----------|-------|
| `FUB_API_KEY` | From FUB Admin → API |
| `META_PAGE_ID` | Facebook Page ID |
| `META_PAGE_ACCESS_TOKEN` | Long-lived page token from Meta for Developers |
| `N8N_BASE_URL` | `https://n8n.joelycannoli.com` |
