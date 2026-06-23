# Tenants API

The Tenants API lets you manage the **tenants** and **subtenants** of your Devic account, read their aggregated cost and usage, and enforce **usage limits** (token / cost caps) that block execution when exceeded. It is the building block for multi-tenant SaaS products built on top of Devic: each of your own customers maps to a tenant, and each end user inside that customer maps to a subtenant.

## Concepts

- **Tenant**: a customer/organization of your account. Identified by a free-form `tenantId` you choose (e.g. `acme-corp`). A tenant is **auto-registered** the first time you send a message/thread carrying its `tenantId` — you don't have to create it explicitly.
- **Subtenant**: an end user/entity inside a tenant (e.g. `user_67890`). Auto-registered the same way. When `subtenantId` is omitted it is derived from `metadata.subtenantMetadata.id` or the legacy `metadata.userId`.
- **Cost aggregation**: every LLM call attributed to a tenant/subtenant rolls up into daily/monthly cost series, broken down per assistant and per agent. Aggregation is **forward-only** (it starts capturing the moment the dimension first appears; there is no historical backfill).
- **Usage limit (rule)**: a cap of `tokens` or `cost` over a time window (e.g. *100K tokens per month*, *$5 per 3 hours*). When a rule is consumed, the next message/thread is **blocked before the LLM call** with HTTP `429`.
- **Tier (plan)**: a reusable bundle of limit rules (e.g. `free`/`pro`/`enterprise`). A tenant/subtenant references a tier by `tierId` **by live reference, never by copy** — editing a tier applies instantly to every tenant assigned to it.

### How tenants are populated

Pass `tenantId` (and optionally `subtenantId`) when sending a message or creating a thread:

```bash
# Assistant chat
curl -X POST "https://api.devic.ai/api/v1/assistants/default/messages" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{ "message": "Hi", "tenantId": "acme-corp", "subtenantId": "user_67890" }'

# Agent thread
curl -X POST "https://api.devic.ai/api/v1/agents/agent-123/threads" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{ "message": "Run the report", "tenantId": "acme-corp", "subtenantId": "user_67890" }'
```

The tenant is registered (auto `touch`) **before** the LLM call, so it appears even if the model call later fails. See [assistants.md](assistants.md) and [agents.md](agents.md) for the full message/thread bodies and the `429` shape.

You can enrich the auto-registered records by sending display metadata alongside the ids:

```jsonc
{
  "message": "Hi",
  "tenantId": "acme-corp",
  "subtenantId": "user_67890",
  "metadata": {
    "tenantMetadata":    { "name": "Acme Corp", "email": "billing@acme-corp.com", "imageUrl": "https://acme-corp.com/logo.png" },
    "subtenantMetadata": { "id": "user_67890", "name": "Jane Doe", "email": "jane@acme-corp.com", "imageUrl": "https://…/jane.png" }
  }
}
```

`tenantMetadata` enriches the tenant (name/email fill-only; an explicit `imageUrl` is pinned manual). `subtenantMetadata` enriches the subtenant (`name`→`displayName`, `email`, `imageUrl`) and is refreshed on every request unless the field was pinned manually via PATCH. All fields are optional.

## Endpoint groups & API-key scope

The Tenants API is split into three URL subtrees so you can grant fine-grained access per API key (configured in the per-key endpoint-scope picker / api-gateway allowlist):

| Subtree | Endpoint group | Intended for |
|---------|----------------|--------------|
| `/api/v1/tenants/*` | **Tenants** | Tenant/subtenant management + cost (read/write). Full keys. |
| `/api/v1/tenant-usage/*` | **Tenant usage** | Read-only limits/consumption/history. Part of the **devic-ui** key preset. |
| `/api/v1/tenant-admin/*` | **Tenant admin** | Reset counters & change tier (e.g. checkout/upgrade webhooks). **Full keys only** — deliberately excluded from the devic-ui preset, so restricted keys are `403`'d at the gateway. |

All endpoints authenticate with the standard `Authorization: Bearer devic-...` header and are scoped to the calling account (`clientUID`); you only ever see your own tenants.

---

# 1. Tenant Management — `/api/v1/tenants`

## List Tenants

```
GET /api/v1/tenants
```

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `search` | string | — | Case-insensitive match on `tenantId`, `displayName`, `email` or `domain` |
| `limit` | number | 50 | Max items (capped at 200) |
| `skip` | number | 0 | Items to skip (pagination) |

### Response

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "_id": "665f...",
        "tenantId": "acme-corp",
        "displayName": "Acme Corp",
        "email": "billing@acme-corp.com",
        "imageUrl": "https://www.google.com/s2/favicons?domain=acme-corp.com&sz=128",
        "domain": "acme-corp.com",
        "source": "auto",
        "subtenantsCount": 12,
        "firstSeenAt": "2026-06-01T09:00:00.000Z",
        "lastSeenAt": "2026-06-21T11:30:00.000Z",
        "metadata": {}
      }
    ],
    "total": 37
  }
}
```

`imageUrl` is auto-derived from the tenant's corporate domain (Google favicons). The domain is inferred either from ≥2 subtenants sharing a non-generic email domain, or — as a fallback — from the tenant's own `email` domain. Generic domains (gmail, outlook, …) are ignored.

## Get Tenant

```
GET /api/v1/tenants/:tenantId
```

Returns a single tenant object (same shape as a list item).

## Update Tenant

```
PATCH /api/v1/tenants/:tenantId
```

### Request Body

| Field | Type | Description |
|-------|------|-------------|
| `displayName` | string | Human-friendly name |
| `email` | string | Contact email |
| `imageUrl` | string | Logo URL — **setting it freezes** the auto favicon (`imageSource = manual`) |
| `domain` | string | Corporate domain — **setting it freezes** auto domain detection (`domainSource = manual`) |

Manual values always win over auto-detection.

## Delete Tenant

```
DELETE /api/v1/tenants/:tenantId
```

Deletes the tenant **and all its subtenants**. Returns `{ "success": true }`.

## Tenant Stats (live counts)

```
GET /api/v1/tenants/:tenantId/stats
```

Live chat/thread counts per assistant and per agent (computed from the indexed `tenantId`, so they reflect full history), plus a cost summary.

```json
{
  "success": true,
  "data": {
    "byAssistant": [ { "assistantIdentifier": "support-bot", "chats": 240 } ],
    "byAgent": [ { "agentId": "665f...", "threads": 58 } ],
    "totals": { "chats": 240, "threads": 58 }
  }
}
```

## Tenant Costs (time series)

```
GET /api/v1/tenants/:tenantId/costs
```

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `granularity` | `daily` \| `monthly` | `daily` | Bucketing of the series |
| `from` | string | — | Start date (ISO `date`, or `YYYY-MM` for monthly) |
| `to` | string | — | End date |
| `subtenantId` | string | — | Scope the series to a single subtenant |

Returns an array of cost documents (`date`/`month`, totals, and `byAgent`/`byAssistant` breakdowns). Only the **primary** LLM cost is broken down per tenant in this version.

## List Subtenants

```
GET /api/v1/tenants/:tenantId/subtenants
```

Query: `search`, `limit`, `skip`. Returns `{ items, total }` of subtenants (`subtenantId`, `displayName`, `email`, `emailDomain`, `imageUrl`, `source`, `firstSeenAt`, `lastSeenAt`).

`imageUrl` is the subtenant's avatar/logo. Unlike the tenant logo it is **never** auto-derived from a domain favicon — it is only populated from explicit `metadata.subtenantMetadata.imageUrl` (or `logoUrl`, or the flat `metadata.userImageUrl`) seen on a request, or set manually via the PATCH below. When absent, render initials of `displayName` (falling back to `subtenantId`).

## Get Subtenant

```
GET /api/v1/tenants/:tenantId/subtenants/:subtenantId
```

## Update Subtenant

```
PATCH /api/v1/tenants/:tenantId/subtenants/:subtenantId
```

### Request Body

| Field | Type | Description |
|-------|------|-------------|
| `displayName` | string | Human-friendly subtenant name |
| `imageUrl` | string | Avatar / logo URL |

Editing either field **pins it as manual** (`displayNameSource` / `imageSource = manual`), so subsequent auto-registration from `subtenantMetadata` no longer overwrites it — same "manual wins" semantics as the tenant. Returns the updated subtenant object.

---

# 2. Tenant Usage (read-only) — `/api/v1/tenant-usage`

Read-only view of limits, current consumption and durable history. Available to **devic-ui keys** (in the preset).

## List Tiers (plans)

```
GET /api/v1/tenant-usage/tiers
```

Returns the account's tiers. Each tier:

```json
{
  "tierId": "pro",
  "name": "Pro",
  "description": "Standard paid plan",
  "rules": [
    { "scope": "tenant", "metric": "tokens", "windowUnit": "month", "windowEvery": 1, "limit": 1000000, "enabled": true },
    { "scope": "subtenant", "metric": "cost", "windowUnit": "day", "windowEvery": 1, "limit": 5, "enabled": true }
  ],
  "excludedAssistants": ["onboarding-bot"],
  "excludedAgents": ["665f..."],
  "isDefault": false
}
```

**Rule scopes:** `scope: "tenant"` = aggregate cap across the whole tenant; `scope: "subtenant"` without an id = the same cap applied to **each** subtenant; `scope: "subtenant"` with a `subtenantId` = override for that specific subtenant. The window is `windowEvery × windowUnit` (`hour`/`day`/`week`/`month`). **Exempt** assistants/agents (by slug or `_id`) neither count toward the counters nor get blocked, though their cost is still aggregated.

## Tenant Usage

```
GET /api/v1/tenant-usage/:tenantId
```

```json
{
  "success": true,
  "data": {
    "tenantId": "acme-corp",
    "tierId": "pro",
    "usage": [
      {
        "scope": "tenant",
        "subtenantId": null,
        "metric": "tokens",
        "windowUnit": "month",
        "windowEvery": 1,
        "limit": 1000000,
        "current": 734512,
        "percent": 73.45,
        "resetsAt": 1719792000000,
        "origin": "tier",
        "tierId": "pro"
      }
    ]
  }
}
```

`usage` is one entry per applicable rule with live `current`/`percent` from the realtime counters and the `resetsAt` epoch (ms) when the window rolls over. `origin` is `tier` (from the assigned/default tier) or `adhoc` (a per-tenant override).

## Subtenant Usage

```
GET /api/v1/tenant-usage/:tenantId/subtenants/:subtenantId
```

Same shape, scoped to the subtenant (includes per-subtenant rules).

## Usage History (durable per-window snapshots)

```
GET /api/v1/tenant-usage/:tenantId/history
GET /api/v1/tenant-usage/:tenantId/subtenants/:subtenantId/history
```

Live counters are ephemeral (they expire when the window closes); a background cron snapshots each **closed** window so you keep a durable record. Use this to chart historical consumption and to see when limits were hit.

### Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `subtenantId` | string | Scope to a subtenant (tenant-level route only) |
| `scope` | `tenant` \| `subtenant` | Filter by scope |
| `metric` | `tokens` \| `cost` | Filter by metric |
| `windowUnit` | `hour` \| `day` \| `week` \| `month` | Filter by window unit |
| `from` / `to` | number | Time range as epoch ms |
| `limit` / `skip` | number | Pagination |

Each row carries `windowStart`/`windowEnd`, `consumption` (counted toward the limit), `exemptConsumption` (incurred by exempt entities — included in cost aggregation but **not** counted toward the limit, so a `total` can legitimately exceed the limit without blocking), `limit`, `percent`, `metric`, `scope` and `tierId`.

---

# 3. Tenant Admin — `/api/v1/tenant-admin`

Privileged actions. **Full keys only** (excluded from the devic-ui preset → restricted keys get `403` at the gateway). Typical caller: your backend's billing/checkout webhook.

## Reset Usage Counters

```
POST /api/v1/tenant-admin/:tenantId/usage-limits/reset
POST /api/v1/tenant-admin/:tenantId/subtenants/:subtenantId/usage-limits/reset
```

The tenant-level route also accepts `?subtenantId=` as an alternative to the nested path. Clears the realtime counters so consumption restarts from zero (e.g. after a manual top-up).

## Change Tier (plan)

```
POST /api/v1/tenant-admin/:tenantId/tier
POST /api/v1/tenant-admin/:tenantId/subtenants/:subtenantId/tier
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `tierId` | string | Yes | The tier (plan) to assign |
| `reason` | string | No | Free-text reason, stored in the tier-change audit log |
| `resetUsage` | boolean | No | Also reset the usage counters on change (e.g. fresh quota on upgrade) |

Assignment is **by live reference**: the tenant just stores the new `tierId`, the limit set is resolved from the tier at request time, and the change is recorded with `source: "webhook"`. Example — upgrade a customer to Pro from a Stripe checkout webhook:

```bash
curl -X POST "https://api.devic.ai/api/v1/tenant-admin/acme-corp/tier" \
  -H "Authorization: Bearer devic-your-full-api-key" \
  -H "Content-Type: application/json" \
  -d '{ "tierId": "pro", "reason": "checkout.session.completed", "resetUsage": true }'
```

---

## Handling limit blocks (429)

When a tenant/subtenant has consumed a limit, message and thread endpoints return HTTP `429` with `error: "TENANT_LIMIT_EXCEEDED"`, a `details` object (`blockingRule`, `current`, `limit`, `resetsAt`) and a `Retry-After` header (seconds until the window resets). The **most restrictive** applicable rule wins. See the `429` sections in [assistants.md](assistants.md) and [agents.md](agents.md) for the exact response body and the async/realtime variants.
