# Integrations & Triggers API

**Integrations** connect third-party apps (Gmail, Drive, HubSpot, Jira, …) to
the workspace. A connected app becomes an **integration tool server**: a tool
server of `type: "integration"` whose tools are the app's own actions, run with
the connected account.

**Triggers** go the other way: they listen to an app's events through an
integration tool server and start an agent or an assistant whenever one
arrives.

The order is always the same:

1. Find the app and see how it connects (`GET /integrations`, `GET /integrations/:app/auth`)
2. Connect an account (`POST /integrations/:app/connect`, plus a browser step for OAuth apps)
3. Build the tool server from it (`POST /integrations/:app/servers`)
4. Give the tool server to an agent or assistant, and/or create triggers on it (`POST /triggers`)

The catalogue and the accounts live at the integration provider. Its
per-account identifiers (connected accounts, auth configs, trigger instances)
are resolved server-side and never sent or required: you work with app slugs,
tool slugs, trigger slugs and Devic ids only.

---

## Endpoints Overview

### App catalogue and connection

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/integrations` | List connectable apps (cursor-paged, searchable) |
| GET | `/api/v1/integrations/:app` | Get one app |
| GET | `/api/v1/integrations/:app/auth` | How the app can be connected, and the fields each way asks for |
| POST | `/api/v1/integrations/:app/connect` | Connect an account (directly, or by returning an authorization URL) |
| GET | `/api/v1/integrations/:app/connection-status` | Whether an active account exists for the app |
| POST | `/api/v1/integrations/:app/servers` | Build an integration tool server from the connected account |
| GET | `/api/v1/integrations/:app/triggers` | The event types the app emits (cursor-paged, searchable) |
| GET | `/api/v1/integrations/:app/triggers/:slug` | One event type, with its `config` and `payload` schemas |

### Connected integrations

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/integrations/connected` | The workspace's integration tool servers |
| GET | `/api/v1/integrations/servers/:id/tools` | Tools an integration exposes (or the app's whole catalogue) |
| PATCH | `/api/v1/integrations/servers/:id/tools` | Change which tools it exposes |
| POST | `/api/v1/integrations/servers/:id/tools/:toolName/test` | Run one tool once against the connected account |

### Triggers

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/triggers` | List triggers |
| POST | `/api/v1/triggers` | Create a trigger |
| GET | `/api/v1/triggers/:id` | Get a trigger |
| PATCH | `/api/v1/triggers/:id` | Update a trigger |
| DELETE | `/api/v1/triggers/:id` | Delete a trigger |
| GET | `/api/v1/triggers/:id/events` | Its recent deliveries |

`:app` is an app slug such as `gmail`. `:id` under `/integrations/servers` is the
integration tool server's `_id` (the `id` from `/integrations/connected`).

**Not reachable with a tenant session token.** These endpoints manage the
workspace's own accounts and triggers. End users connect their own apps
through `/api/v1/tenant-integrations` — see
[tenant-sessions.md](tenant-sessions.md).

### Response envelope

Most responses are wrapped as `{ "success": true, "data": …, "meta": { "timestamp", "duration" } }`.
A handler result that already has a top-level `success` key is returned **as is,
without `data`**: this applies to `POST /integrations/servers/:id/tools/:toolName/test`
and `DELETE /triggers/:id` (their exact shapes are shown below). The other
examples show only `data`.

Errors come back with the HTTP status and the message documented for each case:

```json
{ "statusCode": 404, "message": "TRIGGER_NOT_FOUND", "error": "Not Found" }
```

`INTEGRATION_SETUP_REQUIRED` on connect adds a `code` and the missing fields
(see [Connect an Account](#connect-an-account)). An unexpected failure is a
`500` with `{ "error": "INTERNAL_SERVER_ERROR", "message": "…" }`.

There is no request-body validation layer: only the fields documented here are
read, and their types are not checked. Use the exact names and types shown.

When app integrations are not configured on the deployment, the catalogue
listings come back empty and connecting answers `503 App integrations are not
configured on this deployment.`

---

## List Connectable Apps

```
GET /api/v1/integrations?search=&limit=&cursor=
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `search` | string | - | Words matched against the **app name** |
| `limit` | number | 24 | Page size, clamped to 1–200 |
| `cursor` | string | - | `nextCursor` from the previous page |

```json
{
  "items": [
    {
      "slug": "gmail",
      "name": "Gmail",
      "logo": "https://…/gmail.svg",
      "description": "Gmail is Google's email service…",
      "noAuth": false
    }
  ],
  "totalItems": 1200,
  "nextCursor": "MjQ="
}
```

`nextCursor` is absent on the last page. `noAuth: true` means the app needs no
account at all. A provider failure is answered with an empty page, not an error.

## Get an App

```
GET /api/v1/integrations/:app
```

Returns one catalogue item (`slug`, `name`, `logo`, `description`, `noAuth`).
If the provider cannot be read for that slug, the answer is a placeholder whose
`name` is the slug itself — so a `name` equal to the slug you sent usually
means the slug is wrong. Check it against the list.

---

## How an App Connects

```
GET /api/v1/integrations/:app/auth
```

Call it before connecting: most apps need credentials of your own, and this
says which.

```json
{
  "schemes": [
    {
      "mode": "OAUTH2",
      "composioManaged": false,
      "redirect": true,
      "appFields": [
        { "name": "client_id", "label": "Client ID", "type": "string", "required": true, "secret": false },
        { "name": "client_secret", "label": "Client secret", "type": "string", "required": true, "secret": true }
      ],
      "accountFields": [],
      "guideUrl": "https://…"
    },
    {
      "mode": "API_KEY",
      "composioManaged": false,
      "redirect": false,
      "appFields": [],
      "accountFields": [
        { "name": "generic_api_key", "label": "API key", "type": "string", "required": true, "secret": true }
      ]
    }
  ]
}
```

| Scheme field | Description |
|--------------|-------------|
| `mode` | The provider's name for the scheme: `OAUTH2`, `OAUTH1`, `DCR_OAUTH`, `API_KEY`, `S2S_OAUTH2`, … Send it as `authScheme` |
| `composioManaged` | The integration provider holds an OAuth application for this app, so `appCredentials` are not needed. `false` for most of the catalogue |
| `redirect` | Connecting returns an `authorizationUrl` to open in a browser. True for `OAUTH1`, `OAUTH1A`, `OAUTH2` and `DCR_OAUTH` |
| `appFields` | Credentials of **your own application** registered with the provider (client id/secret…). Sent once per workspace as `appCredentials`. Empty when `composioManaged` |
| `accountFields` | What the account being connected supplies (its API key, the subdomain of its instance…). Sent as `accountFields` |
| `guideUrl` | The provider's setup guide, when known |
| `permissions` | Only on apps with a permission policy: the selectable permission `profiles` and `capabilities`, with `version` and `defaultProfile`. When present, `appFields` is empty |

Each field: `name` (the key to send), `label`, `type`, `required`, `secret`
(holds a credential), and optional `description` and `default`.

`schemes: []` means the app declares no way to connect an account.

---

## Connect an Account

```
POST /api/v1/integrations/:app/connect
```

```json
{
  "authScheme": "API_KEY",
  "accountFields": { "generic_api_key": "pplx-…" }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `authScheme` | string | no | A `mode` from `/auth` (case-insensitive). Omitted, the scheme asking for least setup is used: a provider-managed one first, then one with no required `appFields`, then the rest |
| `appCredentials` | object | no | Values for the scheme's `appFields`, keyed by field `name`. Needed only the first time the workspace connects this app with this scheme; afterwards the registered application is reused and this is ignored |
| `accountFields` | object | no | Values for the scheme's `accountFields`, keyed by field `name` |
| `permissions` | object | no | Only for apps whose scheme lists `permissions`: `{ "version": <number>, "profile": "<profile id>", "capabilities": ["…"] }`. `capabilities` is for a custom selection, when the policy allows one |

Response (`201`):

```json
{ "connected": true }
```

```json
{ "connected": false, "authorizationUrl": "https://…" }
```

| Field | Description |
|-------|-------------|
| `connected` | `true`: the account is active now (API-key style schemes). `false`: it still has to be authorized in a browser |
| `authorizationUrl` | Present when `connected` is `false`. Open it and let the account owner authorize |

### The two flows

**No redirect (`redirect: false`, e.g. `API_KEY`)** — one call. Send the
`accountFields`; the answer is `connected: true` and you can build the tool
server straight away. Always read `connected` rather than assuming it: if the
provider still answers with a URL, it comes back as `authorizationUrl` with
`connected: false`, and a connection the provider marks as failed comes back as
`connected: false` with no URL.

**Redirect (`redirect: true`, OAuth)** — three steps:

1. `POST /connect` returns `authorizationUrl`.
2. The account owner opens it and authorizes at the provider. The provider then
   sends the browser back to Devic, which forwards it to the Devic app.
3. Poll `GET /integrations/:app/connection-status` until `connected` is `true`.

Nothing is created in Devic by connecting: the account exists at the provider,
and the integration only appears in `/integrations/connected` after
`POST /integrations/:app/servers`.

### Missing credentials: `INTEGRATION_SETUP_REQUIRED`

A required field without a value is not guessed. The call answers `400` with the
fields still needed:

```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "code": "INTEGRATION_SETUP_REQUIRED",
  "message": "Connecting jira needs an application registered with the provider: Client ID, Client secret. The integration provider holds no credentials of its own for this app.",
  "toolkit": "jira",
  "authScheme": "OAUTH2",
  "stage": "app",
  "fields": [
    { "name": "client_id", "label": "Client ID", "type": "string", "required": true, "secret": false }
  ],
  "guideUrl": "https://…"
}
```

| `stage` | Meaning | Fix |
|---------|---------|-----|
| `app` | The workspace has no application registered with the provider for this app | Send `appCredentials` with the listed `fields` |
| `account` | The account's own values are missing | Send `accountFields` with the listed `fields` |

On an app with a permission policy, `stage: "account"` with an empty `fields`
list means *choose the permissions*: send `permissions`.

### Other connect errors

| Status | Message / code | Meaning |
|--------|----------------|---------|
| 404 | `"<app>" cannot be connected with <authScheme>.` | That scheme is not offered by the app |
| 404 | `"<app>" does not declare a way to connect an account.` | The app has no schemes |
| 400 | `INTEGRATION_PERMISSIONS_UNSUPPORTED` | `permissions` sent to an app without a permission policy |
| 400 | `INTEGRATION_PERMISSION_OVERRIDE_NOT_ALLOWED` | `appCredentials` or `accountFields` sent to an app with a permission policy |
| 503 | `App integrations are not configured on this deployment.` | |

## Connection Status

```
GET /api/v1/integrations/:app/connection-status
```

```json
{ "connected": true }
```

`true` once the workspace has an `ACTIVE` account for the app.

---

## Build the Tool Server

```
POST /api/v1/integrations/:app/servers
```

```json
{
  "name": "Gmail",
  "tools": ["GMAIL_FETCH_EMAILS", "GMAIL_GET_PROFILE"]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | no | Tool server name. Defaults to the app name |
| `tools` | string[] | no | Tool slugs to expose. Empty or omitted exposes **every** tool of the app |

The server is bound to the app's **most recently updated active account**. The
description and image are taken from the catalogue. Returns `201` with the tool
server in its public shape:

```json
{
  "_id": "65a1b2c3d4e5f6789012399",
  "name": "Gmail",
  "description": "Gmail is Google's email service…",
  "imageUrl": "https://…/gmail.svg",
  "enabled": true,
  "type": "integration",
  "integration": {
    "app": "gmail",
    "apps": ["gmail"],
    "connected": true,
    "enabledToolCount": 2
  }
}
```

`400 No connected account for "<app>". Authorize one first with POST /integrations/<app>/connect.`
when there is no active account yet.

Narrowing `tools` is worth it: every exposed tool's schema is sent to the model
on every message, and a whole app is often dozens of tools.

**Using it.** The integration tool server is a tool server like any other: put
its `_id` in an agent's or assistant's `availableToolsGroupsUids` (see
[built-in-tools.md](built-in-tools.md) and [tool-servers.md](tool-servers.md)).
Each call building a server creates a new one; it does not update an existing
server for the same app.

---

## Connected Integrations

```
GET /api/v1/integrations/connected
```

```json
[
  {
    "id": "65a1b2c3d4e5f6789012399",
    "name": "Gmail",
    "app": "gmail",
    "connected": true,
    "exposedToolCount": 2,
    "enabled": true
  }
]
```

| Field | Description |
|-------|-------------|
| `id` | The integration tool server `_id`. Use it for the tools endpoints and as `toolServerId` for triggers |
| `app` | App slug |
| `connected` | Whether an account is bound. `false` means nothing can run |
| `exposedToolCount` | How many tools it exposes. **Absent means all of the app's tools** |
| `enabled` | The tool server's `enabled` flag |

An array, not paginated. MCP and custom tool servers are not included.

---

## Integration Tools

### List

```
GET /api/v1/integrations/servers/:id/tools?available=&limit=&cursor=
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `available` | string | - | `true` or `1`: browse the app's whole catalogue instead of only the exposed tools |
| `limit` | number | - | Page size when `available`, clamped to 1–200 |
| `cursor` | string | - | `nextCursor` from the previous page, when `available` |

Without `available`, the tools the integration exposes — the set the model is
offered — every one with `enabled: true`, unpaginated:

```json
{
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "GMAIL_FETCH_EMAILS",
        "description": "Fetch a list of email messages…",
        "parameters": { "type": "object", "properties": { "max_results": { "type": "integer" } } }
      },
      "enabled": true
    }
  ],
  "total": 1
}
```

With `available=true`, a page of the app's catalogue, each tool flagged
`enabled` when this integration exposes it, plus `nextCursor` while there are
more pages. `function.name` is the tool slug.

`400 "<id>" is not an app integration…` when `:id` is an MCP or custom tool
server; `404` when it does not exist.

### Change the exposed tools

```
PATCH /api/v1/integrations/servers/:id/tools
```

```json
{ "enable": ["GMAIL_SEND_EMAIL"], "disable": ["GMAIL_FETCH_EMAILS"] }
```

```json
{ "all": true }
```

| Field | Type | Description |
|-------|------|-------------|
| `enable` | string[] | Tool slugs to add to the current selection |
| `disable` | string[] | Tool slugs to remove from the current selection |
| `all` | boolean | Expose every tool of the app (clears the selection) |

Send `all`, or `enable`/`disable` — not both. An empty selection **means all
tools**, which shapes the edges:

- `enable` on an integration that already exposes everything changes nothing.
- `disable` on one that exposes everything starts from the app's full catalogue
  and removes the listed tools.
- A `disable` that would leave nothing selected is refused, because an empty
  selection would mean *all*.

Only the tool selection changes; the connected account is untouched. The
response is the new exposed set, in the same shape as the list without
`available`.

| Status | Message |
|--------|---------|
| 400 | ``Pass either `all`, or enable/disable — not both.`` |
| 400 | ``Nothing to do: pass `enable`, `disable`, or `all`.`` |
| 400 | `That would leave the integration exposing every tool (an empty selection means "all")…` |

### Test a tool

```
POST /api/v1/integrations/servers/:id/tools/:toolName/test
```

```json
{ "parameters": { "max_results": 3 } }
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `parameters` | object | no | The tool's arguments, as its `function.parameters` schema describes. Omit for a tool that takes none |

Runs the tool once against the connected account, through the same call path an
agent uses. **It is a real call on real data**: a send tool sends, a delete tool
deletes. Prefer read-only tools (`*_GET_*`, `*_LIST_*`, `*_FETCH_*`) to check a
connection.

Answered with `200` and returned **without the `data` envelope**:

```json
{
  "success": true,
  "data": { "emailAddress": "someone@example.com", "messagesTotal": 1234 },
  "executionTimeMs": 412,
  "app": "gmail",
  "tool": "GMAIL_GET_PROFILE"
}
```

```json
{
  "success": false,
  "error": "…the app's own message…",
  "requiresUserAction": true,
  "executionTimeMs": 230,
  "app": "gmail",
  "tool": "GMAIL_FETCH_EMAILS"
}
```

| Field | Description |
|-------|-------------|
| `success` | Whether the app accepted and ran the call |
| `data` | What the tool returned, exactly as the agent would receive it |
| `error` | Why it failed, in the app's words. Usually the arguments |
| `requiresUserAction` | Present and `true` when the failure is the connection itself (expired or revoked account): reconnect it |
| `executionTimeMs` | Round trip to the app |

| Status | When |
|--------|------|
| 400 | `:id` is not an app integration |
| 400 | The tool exists in the app but this integration does not expose it — enable it first |
| 404 | The tool belongs to another app, or the app has no tool with that slug |

---

## Event Types

### List an app's event types

```
GET /api/v1/integrations/:app/triggers?search=&limit=&cursor=
```

`limit` defaults to 50 (clamped to 1–200). Same paging as the app list:

```json
{
  "items": [
    {
      "slug": "GMAIL_NEW_GMAIL_MESSAGE",
      "name": "New Gmail message",
      "description": "Triggers when a new message arrives…",
      "app": "gmail",
      "type": "poll"
    }
  ],
  "totalItems": 3,
  "nextCursor": "…"
}
```

`type`: `poll` events arrive minutes late, `webhook` events arrive at once.

### Get an event type

```
GET /api/v1/integrations/:app/triggers/:slug
```

The full type, with two JSON Schemas:

| Field | Description |
|-------|-------------|
| `config` | The trigger's own settings (label, interval, …). Its values go in `triggerConfig` when creating the trigger |
| `payload` | The shape of the event body. Its fields are what a message template references, under `data` (e.g. `{{data.sender}}`) |
| `instructions` | Provider setup notes, when there are any |

`404 No trigger type "<slug>"` when it cannot be read. The `:app` segment is
not used for the lookup; the slug alone identifies the type.

---

## Trigger Structure

```json
{
  "id": "6874f21c9a3b4e5d2f108fff",
  "name": "New email → triage agent",
  "targetType": "agent",
  "target": {
    "type": "agent",
    "id": "6874f21c9a3b4e5d2f108abc",
    "name": "Triage agent"
  },
  "toolServerId": "65a1b2c3d4e5f6789012399",
  "app": "gmail",
  "triggerSlug": "GMAIL_NEW_GMAIL_MESSAGE",
  "triggerConfig": { "labelIds": "INBOX", "interval": 2 },
  "messageTemplate": "New email from {{data.sender}}: {{data.subject}}",
  "newThreadsState": "queued",
  "enabled": true,
  "rateLimitPerMinute": 10,
  "eventCount": 42,
  "lastEventAt": 1752500000000,
  "creationTimestampMs": 1752400000000
}
```

| Field | Description |
|-------|-------------|
| `id` | Trigger id |
| `targetType` | `agent` or `assistant` |
| `target` | The target resolved for display: `type`, `id`, and `name`, `imgUrl`, `avatarStyle` while it exists |
| `target.id` | Agent `_id`, or assistant **identifier** |
| `toolServerId` | The integration tool server it listens to |
| `app` | The app it belongs to |
| `triggerSlug`, `triggerConfig` | The event type and its settings |
| `messageTemplate`, `chatUidTemplate`, `newThreadsState`, `rateLimitPerMinute` | As set on create/update |
| `enabled` | Whether it runs |
| `eventCount`, `lastEventAt` | Executed deliveries and the last one, ms |
| `autoPausedAt`, `autoPauseReason` | Set when the platform switched it off (see [Guardrails](#guardrails)) |
| `creationTimestampMs` | Creation time, ms |

---

## List Triggers

```
GET /api/v1/triggers?toolServerId=&agentId=&assistantId=&offset=&limit=
```

| Parameter | Description |
|-----------|-------------|
| `toolServerId` | Only triggers on this integration tool server |
| `agentId` | Only triggers that start this agent (`_id`) |
| `assistantId` | Only triggers that start this assistant (identifier). Ignored when `agentId` is also sent |
| `offset` | Default 0 |
| `limit` | Default 20, clamped to 1–100 |

```json
{ "items": [ { "id": "…", "triggerSlug": "GMAIL_NEW_GMAIL_MESSAGE", "...": "…" } ], "total": 1 }
```

Newest first. `total` counts every match, not the page.

## Get a Trigger

```
GET /api/v1/triggers/:id
```

`404 TRIGGER_NOT_FOUND` when it is not in this workspace.

---

## Create a Trigger

```
POST /api/v1/triggers
```

```json
{
  "toolServerId": "65a1b2c3d4e5f6789012399",
  "agentId": "6874f21c9a3b4e5d2f108abc",
  "triggerSlug": "GMAIL_NEW_GMAIL_MESSAGE",
  "triggerConfig": { "labelIds": "INBOX", "interval": 2 },
  "name": "New email → triage agent",
  "messageTemplate": "New email from {{data.sender}}: {{data.subject}}",
  "newThreadsState": "queued",
  "rateLimitPerMinute": 10
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `toolServerId` | string | yes | An **integration** tool server (`id` from `/integrations/connected`) |
| `agentId` | string | one of | The agent to start: its `_id` |
| `assistantId` | string | one of | The assistant to start: its **identifier**, not its `_id` |
| `triggerSlug` | string | yes | From `GET /integrations/:app/triggers` |
| `triggerConfig` | object | no | Values for the event type's `config` schema |
| `name` | string | no | Display name |
| `messageTemplate` | string | no | Turns the event into the message the run starts with. Empty ⇒ the raw event JSON |
| `chatUidTemplate` | string | no | Assistants only: the conversation id, so related events share one conversation. Empty ⇒ a new conversation per event |
| `newThreadsState` | string | no | Agents only: `queued` (default, runs at once) or `paused` (the thread is created paused) |
| `rateLimitPerMinute` | number | no | Executions per minute for this trigger. Defaults to the deployment's limit (20) |

Send **exactly one** of `agentId` / `assistantId`. The trigger is registered
with the provider against the account the tool server is bound to, and starts
`enabled: true`. Returns `201` with the trigger.

| Status | Message | Meaning |
|--------|---------|---------|
| 400 | `Provide exactly one of agentId or assistantId — a trigger starts one target.` | Both or neither |
| 404 | `No agent "<id>" — an agent target is its _id.` / `No agent "<id>".` | Unknown agent |
| 404 | `No assistant "<identifier>".` | Unknown assistant identifier |
| 400 | `Agent "<id>" is archived, so it would never run this trigger. Unarchive it first.` | Same for an archived assistant |
| 404 | `TOOL_SERVER_NOT_FOUND` | No such tool server in this workspace |
| 400 | `That tool server is not an app integration, so it has no events to listen to.` | MCP or custom tool server |
| 400 | `That integration is not bound to a connected account. Reconnect it and try again.` | |
| 400 | `This workspace already has <n> triggers, the maximum for this deployment. Delete one before adding another.` | Per-workspace cap (20 by default) |

### Templates

`messageTemplate` and `chatUidTemplate` are rendered against the whole delivery,
so they read the event body under `data` and its metadata under `metadata`:

| Syntax | Meaning |
|--------|---------|
| `{{data.sender}}` | Dot-path lookup. Missing paths render as `""`; objects and arrays as JSON |
| `{{data.attachments.0.name}}` | Numeric segments index arrays |
| `{{data.thread_id \|\| data.id}}` | Fallback: the first alternative that is not empty |
| `{{metadata.trigger_slug}}` | Delivery metadata |
| `{{$date}}`, `{{$week}}`, `{{$month}}`, `{{$year}}` | The current date at four resolutions (UTC) — e.g. one conversation per day |

The field names come from the event type's `payload` schema.

A template that renders to nothing falls back: the message to the raw event
JSON, the conversation id to a fresh one.

### What a delivery does

- **Agent target**: a new thread starts with the rendered message, in the
  `newThreadsState` state. The raw event also travels with the thread as
  `triggerEvent` metadata.
- **Assistant target**: the rendered message is sent to the conversation named
  by `chatUidTemplate`. If that conversation is still answering, the event is
  recorded as `deferred` and delivered once it is free, merged with any other
  event that piled up meanwhile.
- A delivery the provider retries is recognised and dropped; it does not start
  a second run.
- A delivery for a disabled trigger, or for one whose agent or assistant is
  archived or deleted, does not run and is recorded as `disabled`.

---

## Update a Trigger

```
PATCH /api/v1/triggers/:id
```

```json
{ "messageTemplate": "From {{data.sender}}", "rateLimitPerMinute": 5, "enabled": false }
```

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Display name |
| `messageTemplate` | string | Message template |
| `chatUidTemplate` | string | Conversation id template (assistants) |
| `newThreadsState` | string | `queued` or `paused` (agents) |
| `rateLimitPerMinute` | number | Per-trigger ceiling |
| `triggerConfig` | object | New settings. Re-registered with the provider |
| `enabled` | boolean | Switch the trigger on or off. Also switches it at the provider |

Every field is optional; only those present change. The target, the tool server
and `triggerSlug` cannot be changed — delete and create instead. Re-enabling by
hand clears `autoPausedAt` / `autoPauseReason`. Returns the updated trigger.

---

## Delete a Trigger

```
DELETE /api/v1/triggers/:id
```

Returned **without the `data` envelope**:

```json
{ "success": true, "deletedId": "6874f21c9a3b4e5d2f108fff" }
```

Removes the trigger and its delivery log. The provider-side subscription is
removed too, unless another trigger shares it.

---

## Trigger Events

```
GET /api/v1/triggers/:id/events?limit=&offset=
```

Recent deliveries, newest first, **including the ones that did not run**.
`limit` defaults to 20 (max 100), `offset` to 0.

```json
{
  "items": [
    {
      "_id": "…",
      "subscriptionId": "6874f21c9a3b4e5d2f108fff",
      "triggerSlug": "GMAIL_NEW_GMAIL_MESSAGE",
      "receivedAt": "2026-11-16T09:12:00.000Z",
      "outcome": "executed",
      "renderedMessage": "New email from someone@example.com: Invoice",
      "payload": "{\"data\":{…}}",
      "threadId": "6874f3009a3b4e5d2f108aaa"
    }
  ],
  "total": 1
}
```

| Field | Description |
|-------|-------------|
| `outcome` | `executed`, `deferred` (assistant conversation busy, delivered later), `rate_limited`, `disabled` or `error` |
| `renderedMessage` | The message the run received |
| `payload` | The delivery as JSON, truncated to 8000 characters |
| `threadId` / `chatUID` | Where it ran (agent thread / assistant conversation) |
| `error` | Why it failed (`error`) or why it did not run (`disabled` because of the target: `AGENT_NOT_FOUND`, `AGENT_IS_ARCHIVED`, `ASSISTANT_NOT_FOUND`, `ASSISTANT_IS_ARCHIVED`) |

The rows are stored documents, so they also carry internal fields
(`clientUID`, `webhookId`, `triggerInstanceId`, `__v`); ignore them.

Events are kept for 30 days.

---

## Guardrails

| Guardrail | Behaviour |
|-----------|-----------|
| Rate limit | Deliveries beyond `rateLimitPerMinute` (default 20/min) in a minute are recorded as `rate_limited` and dropped, not queued |
| Auto-pause | Over the limit for 3 consecutive minutes (default), the trigger is switched off with `autoPausedAt` / `autoPauseReason` set. Re-enable it with `PATCH { "enabled": true }` |
| Target archived or deleted | The trigger is paused while its agent or assistant is archived or missing, and switched back on automatically when the target returns |
| Trigger cap | 20 triggers per workspace by default |

---

## Example: Gmail → triage agent

```bash
# 1. How does Gmail connect?
curl "https://api.devic.ai/api/v1/integrations/gmail/auth" \
  -H "Authorization: Bearer devic-your-api-key"

# 2. Connect (OAuth: open the returned authorizationUrl and authorize)
curl -X POST "https://api.devic.ai/api/v1/integrations/gmail/connect" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" -d '{}'

# 3. Wait until it is active
curl "https://api.devic.ai/api/v1/integrations/gmail/connection-status" \
  -H "Authorization: Bearer devic-your-api-key"

# 4. Build the tool server with a narrow tool set
curl -X POST "https://api.devic.ai/api/v1/integrations/gmail/servers" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{ "tools": ["GMAIL_FETCH_EMAILS", "GMAIL_GET_PROFILE"] }'

# 5. Check the connection with a read-only tool
curl -X POST "https://api.devic.ai/api/v1/integrations/servers/<serverId>/tools/GMAIL_GET_PROFILE/test" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" -d '{}'

# 6. Start the agent on every new email
curl -X POST "https://api.devic.ai/api/v1/triggers" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "toolServerId": "<serverId>",
    "agentId": "<agentId>",
    "triggerSlug": "GMAIL_NEW_GMAIL_MESSAGE",
    "messageTemplate": "New email from {{data.sender}}: {{data.subject}}"
  }'
```
