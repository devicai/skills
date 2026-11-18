# Projects API

A **project** groups the agents, assistants, documents, tool servers and
environments of one account under a common scope. Entities join a project
through their own `projectId` field; the project itself only carries a name, a
URL-friendly `identifier` and a visibility. This API creates and edits projects
and reads what happened inside one: its threads, its conversations and its costs.

---

## Endpoints Overview

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/projects` | List projects |
| POST | `/api/v1/projects` | Create a project |
| GET | `/api/v1/projects/:projectId` | Get a project |
| PATCH | `/api/v1/projects/:projectId` | Update a project |
| DELETE | `/api/v1/projects/:projectId` | Archive a project (it is not deleted) |
| GET | `/api/v1/projects/:projectId/stats` | Current-month cost and usage summary |
| GET | `/api/v1/projects/:projectId/threads` | Agent threads of the project's agents |
| GET | `/api/v1/projects/:projectId/conversations` | Conversations of the project's assistants |
| GET | `/api/v1/projects/:projectId/costs/daily` | Daily cost records |
| GET | `/api/v1/projects/:projectId/costs/monthly` | Monthly cost records |

`:projectId` is always the project's `_id`, never its `identifier`.

### Response envelope

Responses are wrapped as `{ "success": true, "data": …, "meta": { "timestamp", "duration" } }`.
The examples below show only `data`.

Errors come back with the HTTP status and the message documented for each case:

```json
{ "statusCode": 404, "message": "Project not found", "error": "Not Found" }
```

An unexpected failure (for example a `:projectId` that is not a valid id) is a
`500` with `{ "error": "INTERNAL_SERVER_ERROR", "message": "…" }`.

There is no request-body validation layer: the fields documented here are the
only ones read, and anything else in the body is ignored.

---

## Project Structure

| Field | Type | Description |
|-------|------|-------------|
| `_id` | string | Project id |
| `clientUID` | string | Owning account |
| `name` | string | Display name |
| `identifier` | string | Unique per account. Lowercase alphanumeric with hyphens |
| `description` | string | Free text. `""` when not set on create |
| `imageUrl` | string | Optional image. Only settable on update |
| `visibility` | string | `private` (default) or `public` |
| `archived` | boolean | `true` once archived |
| `createdByUserUID` | string | Who created it |
| `creationTimestampMs` | number | Creation time, ms |
| `lastEditTimestampMs` | number | Last update, ms |

**Visibility.** A `public` project is reachable by every user of the account
without a grant. A `private` one only by admins and the users it was granted to.
An API key acts as an account admin and reaches every project either way.

---

## List Projects

```
GET /api/v1/projects?archived=&offset=&limit=
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `archived` | string | `false` | `true` returns **only archived** projects; anything else returns only active ones. It never returns both |
| `offset` | number | 0 | Items to skip |
| `limit` | number | 20 | Page size, clamped to 1–100 |

Sorted newest first (`creationTimestampMs` descending). A caller that is not an
admin sees only the projects it can reach (public ones plus its grants).

```json
{
  "projects": [
    {
      "_id": "6874f21c9a3b4e5d2f108abc",
      "clientUID": "client-123",
      "name": "Customer support",
      "identifier": "customer-support",
      "description": "Support assistants and their knowledge",
      "visibility": "private",
      "createdByUserUID": "user-456",
      "creationTimestampMs": 1752500000000,
      "lastEditTimestampMs": 1752500000000,
      "archived": false
    }
  ],
  "total": 1
}
```

`total` counts every project visible to the caller with the same `archived`
value, not only the page.

---

## Create a Project

```
POST /api/v1/projects
```

```json
{
  "name": "Customer support",
  "identifier": "customer-support",
  "description": "Support assistants and their knowledge",
  "visibility": "private"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Display name |
| `identifier` | string | yes | 3–50 characters, matching `^[a-z0-9]+(-[a-z0-9]+)*$` (e.g. `my-project-1`). Unique within the account and **cannot be changed later** |
| `description` | string | no | Defaults to `""` |
| `visibility` | string | no | `private` (default) or `public` |

`imageUrl` and `archived` are not read on create; the project always starts
active. Only `identifier` is validated: `name` is not checked for presence and
`visibility` is stored as sent, so send exactly `private` or `public`. Returns
`201` with the created project.

| Status | When |
|--------|------|
| 400 | `Identifier must be between 3 and 50 characters` |
| 400 | `Identifier must be lowercase alphanumeric with hyphens (e.g. "my-project-1")` |
| 400 | `Project with identifier "<identifier>" already exists` |

---

## Get a Project

```
GET /api/v1/projects/:projectId
```

Returns the project. `404 Project not found` when it is not in this account;
`403 You do not have access to this project` for a non-admin caller without a
grant.

---

## Update a Project

```
PATCH /api/v1/projects/:projectId
```

```json
{
  "name": "Customer support (EU)",
  "description": "EU support team",
  "visibility": "public",
  "imageUrl": "https://example.com/logo.png",
  "archived": false
}
```

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | New display name |
| `description` | string | New description |
| `visibility` | string | `private` or `public` |
| `archived` | boolean | `true` archives, `false` restores an archived project |
| `imageUrl` | string | Image URL |

Every field is optional; only the ones present are written, and
`lastEditTimestampMs` is always refreshed. `identifier` cannot be changed.
Returns the updated project, or `404 Project not found`.

---

## Archive a Project

```
DELETE /api/v1/projects/:projectId
```

Sets `archived: true` and returns the updated project. Nothing is deleted: the
project and everything that points at it stay in place, and
`PATCH { "archived": false }` restores it. Archived projects only appear in
`GET /api/v1/projects?archived=true`.

---

## Project Stats

```
GET /api/v1/projects/:projectId/stats
```

The cost and usage of the **current calendar month**.

```json
{
  "currentMonth": "2026-11",
  "totalCost": 12.48,
  "totalSecondaryCost": 0.31,
  "secondaryByOperation": {
    "summarization": { "cost": 0.31, "inputTokens": 52000, "outputTokens": 4100, "count": 12 }
  },
  "totalToolCalls": 342,
  "threadsCount": 57,
  "conversationsCount": 120,
  "totalInputTokens": 2350000,
  "totalOutputTokens": 186000,
  "tenants": ["acme", "_none"]
}
```

| Field | Description |
|-------|-------------|
| `currentMonth` | `YYYY-MM` |
| `totalCost` | Total cost of the month, secondary operations included |
| `totalSecondaryCost` | The part of `totalCost` spent on secondary model operations (`summarization`, `evaluation`, `guardrails`, `planning`, `compaction`, …) |
| `secondaryByOperation` | Per operation: `cost`, `inputTokens`, `outputTokens`, cached/reasoning token counters and `count`. Absent when there is none |
| `totalToolCalls`, `threadsCount`, `conversationsCount` | Activity counters |
| `totalInputTokens`, `totalOutputTokens` | Token totals |
| `tenants` | The tenant ids that generated usage this month. `_none` stands for usage without a `tenantId` |

All counters are `0` when the month has no activity yet.

---

## Project Threads

```
GET /api/v1/projects/:projectId/threads?limit=&offset=
```

The threads of every agent whose `projectId` is this project, newest first.
`limit` defaults to 20 and `offset` to 0; neither is capped.

```json
{
  "total": 57,
  "threads": [
    { "_id": "6874f3009a3b4e5d2f108aaa", "agentId": "6874f21c9a3b4e5d2f108abc", "state": "completed", "creationTimestampMs": 1752500000000 }
  ]
}
```

Each thread is returned **without `threadContent`**. Read the full thread with
`GET /api/v1/agents/threads/:threadId` (see [agents.md](agents.md)).

---

## Project Conversations

```
GET /api/v1/projects/:projectId/conversations?limit=&offset=
```

The conversations of every assistant whose `projectId` is this project, newest
first. Same `limit` / `offset` defaults as threads.

```json
{
  "total": 120,
  "conversations": [
    { "_id": "…", "chatUID": "…", "assistantSpecializationIdentifier": "support-bot", "creationTimestampMs": 1752500000000 }
  ]
}
```

Each conversation is returned **without `chatContent`**. Read the messages
through the assistant's chat endpoints (see [assistants.md](assistants.md)).

---

## Project Costs

```
GET /api/v1/projects/:projectId/costs/daily?startDate=&endDate=&tenantId=
GET /api/v1/projects/:projectId/costs/monthly?startMonth=&endMonth=&tenantId=
```

| Parameter | Endpoint | Format | Description |
|-----------|----------|--------|-------------|
| `startDate` / `endDate` | daily | `YYYY-MM-DD` | Inclusive range. Either can be omitted |
| `startMonth` / `endMonth` | monthly | `YYYY-MM` | Inclusive range. Either can be omitted |
| `tenantId` | both | string | Narrow the figures to one tenant |

Without a range, every record of the project is returned. The response is an
array, most recent first, one record per day (`date`) or per month (`month`):

```json
[
  {
    "projectId": "6874f21c9a3b4e5d2f108abc",
    "clientUID": "client-123",
    "date": "2026-11-16",
    "totalCost": 1.92,
    "totalInputTokens": 410000,
    "totalOutputTokens": 28000,
    "totalInputCachedTokens": 120000,
    "totalInputCacheWriteTokens": 0,
    "totalOutputCachedTokens": 0,
    "totalReasoningOutputTokens": 3000,
    "threadsCount": 9,
    "conversationsCount": 14,
    "totalToolCalls": 61,
    "byTenant": { "acme": { "totalCost": 1.2, "...": "…" }, "_none": { "...": "…" } },
    "byAgent": { "6874f21c9a3b4e5d2f108abc": { "totalToolCalls": 40, "totalCost": 0.9 } },
    "byAssistant": { "6874f21c9a3b4e5d2f108def": { "totalToolCalls": 21, "totalCost": 1.02 } },
    "lastUpdated": 1752500000000
  }
]
```

| Field | Description |
|-------|-------------|
| `totalCost` and `total*Tokens` | Totals of the period |
| `totalSecondary*`, `secondaryByOperation` | Secondary operations, when present |
| `byTenant` | The same counters per `tenantId`. `_none` holds usage without a tenant |
| `byAgent` / `byAssistant` | `{ totalToolCalls, totalCost }` keyed by agent `_id` / assistant `_id` |

**With `tenantId`** the top-level counters are replaced by that tenant's figures
(the `byTenant` entry merged over the record). A day or month in which that
tenant had no activity is still returned, with its counters set to `0`. The
`byTenant`, `byAgent` and `byAssistant` maps are left as they are.

Days are cut in UTC.

---

## Access

| Caller | Reaches |
|--------|---------|
| API key | Every project of the account |
| Admin user token | Every project of the account |
| Other user token | Public projects and the ones it was granted. The list leaves the rest out; get, stats, threads, conversations and costs answer `403 You do not have access to this project` for them |

---

## Example: create a project and put an agent in it

```bash
# 1. Create the project
curl -X POST "https://api.devic.ai/api/v1/projects" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{ "name": "Customer support", "identifier": "customer-support" }'

# 2. Move an agent into it (projectId is a field of the agent)
curl -X PATCH "https://api.devic.ai/api/v1/agents/<agentId>" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{ "projectId": "<projectId>" }'

# 3. Read what it cost this month
curl "https://api.devic.ai/api/v1/projects/<projectId>/stats" \
  -H "Authorization: Bearer devic-your-api-key"
```
