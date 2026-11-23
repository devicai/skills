# Workspaces API

A **workspace** is a shared space where people and assistants work together in
**threads**. Each thread is a conversation that holds messages, files, runs of
assistants and its own read state for every person.

This API reads and writes the same workspaces as the web and mobile apps. The
same permissions apply: you only see the workspaces you belong to, and every
action needs the same workspace role as in the apps.

**Base path:** `/api/v1/workspaces`

**As MCP tools:** every operation here is also a tool (`workspaces_*`, same
arguments, `body` = the request body below) on two MCP servers. Both sign the
person in over OAuth, with no API key:

- **Devic Workspaces**, `https://workspaces.mcp.devic.ai/mcp`: only these
  tools, scopes `workspaces:read|write`, sign-in on workspaces.devic.ai. Use it
  for people who work in Workspaces.
- **Devic API**, `https://devic.mcp.devic.ai/mcp`: these tools plus the rest of
  the platform, sign-in on app.devic.ai.

---

## Who a request acts as

Workspaces belong to people, so every request resolves a **principal** first.

| Credential | `X-Devic-User` header | Acts as |
|------------|-----------------------|---------|
| A person's token (`devic login`, OAuth app) | — | That person, with their membership and roles |
| API key | `userUID` or email of a person of the key's account | That person, with **their own** account role and workspace roles. The key's admin role never adds permissions |
| API key | not sent | The **account integration**, shown as "API" |

The account integration is not a person:

- It sees the account's **shared** workspaces only. It never sees personal workspaces or private (direct) conversations.
- It acts as a workspace **manager**: it can read, write and configure.
- It is never an owner, so it cannot transfer ownership or change owner-only settings.
- Its messages are authored by `api-key:<id>` with the name "API".
- Its read state is its own.

Errors:

- `403 WORKSPACE_ACT_AS_USER_NOT_FOUND`: the named person is not in the key's account.
- `403 WORKSPACE_ACT_AS_REQUIRES_API_KEY`: a person's token named someone else.
- `403 WORKSPACE_TENANT_SESSION_NOT_SUPPORTED`: tenant sessions cannot be used. They identify your customers, not members of your account.

`GET /api/v1/workspaces/me` returns the resolved principal:

```json
{ "principal": "person", "userUID": "u-123", "name": "Ana", "role": "admin",
  "origin": { "channel": "cli", "credential": "api_key", "apiKeyId": "42", "onBehalfOf": true } }
```

### Message origin

The server records on every message where it was written, in a field you
cannot set or forge:

```json
"origin": { "channel": "api" | "cli" | "sdk", "credential": "api_key" | "oauth" | "session",
            "apiKeyId": "42", "oauthClientId": "…", "onBehalfOf": true }
```

The channel comes from the `devic-api-source` header (`cli`, `sdk`), and is `api`
otherwise. The apps show these messages with a "via API / CLI / SDK" tag.
Messages written in the apps have no `origin`.

---

## Endpoints Overview

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/workspaces/me` | Who this credential acts as |
| GET | `/api/v1/workspaces` | List workspaces (`search`, `includeArchived`, `withSummary`, `offset`, `limit`) |
| POST | `/api/v1/workspaces` | Create a workspace |
| GET | `/api/v1/workspaces/catalog` | Assistants and people available to configure workspaces |
| GET | `/api/v1/workspaces/threads` | Threads across every workspace, private conversations included (`workspaceId`, `kind=direct\|shared`, `search`, `unread`, `includeArchived`, `offset`, `limit`) |
| GET | `/api/v1/workspaces/direct-messages` | Your private conversations: `{ workspaceId, items }` |
| POST | `/api/v1/workspaces/direct-messages` | Open the conversation with a person (`userUID` or `email`) |
| GET | `/api/v1/workspaces/{wid}` | Get a workspace |
| PATCH | `/api/v1/workspaces/{wid}` | Update its settings |
| PATCH | `/api/v1/workspaces/{wid}/consumer-permissions` | Set what Members with the Consumer role can do (owner only) |
| PUT | `/api/v1/workspaces/{wid}/members/{userUID}` | Add a member or change their role |
| DELETE | `/api/v1/workspaces/{wid}/members/{userUID}` | Remove a member |
| GET | `/api/v1/workspaces/{wid}/assistants` | List its assistants |
| POST | `/api/v1/workspaces/{wid}/assistants` | Add an assistant |
| GET/PATCH | `/api/v1/workspaces/{wid}/assistants/{aid}` | Read or edit an assistant |
| DELETE | `/api/v1/workspaces/{wid}/assistants/{aid}?revision=` | Remove an assistant |
| GET/PUT | `/api/v1/workspaces/{wid}/resources` | Knowledge, skills (`knowledgeSkills`) and environment |
| GET | `/api/v1/workspaces/{wid}/knowledge/catalog?search=` | Knowledge you can attach |
| GET | `/api/v1/workspaces/{wid}/skills/catalog?search=` | Skills you can attach |
| GET/PUT | `/api/v1/workspaces/{wid}/rules` | Workspace rules (instructions) |
| GET | `/api/v1/workspaces/{wid}/rules/history?before=` | Earlier versions of the rules |
| GET | `/api/v1/workspaces/{wid}/documents` | Every document of the workspace |
| GET | `/api/v1/workspaces/{wid}/threads` | Threads of one workspace (same query as `/threads`) |
| POST | `/api/v1/workspaces/{wid}/threads` | Create a thread |
| GET | `/api/v1/workspaces/{wid}/threads/{tid}` | Thread, read state and latest messages |
| PATCH | `/api/v1/workspaces/{wid}/threads/{tid}` | Rename (`title`) or archive (`archived`) |
| GET | `/api/v1/workspaces/{wid}/threads/{tid}/messages` | Page messages (`before` / `after`, `limit`) |
| POST | `/api/v1/workspaces/{wid}/threads/{tid}/messages` | Write a message |
| POST | `/api/v1/workspaces/{wid}/threads/{tid}/read` | Mark read (up to `throughSeq`, default all) |
| POST | `/api/v1/workspaces/{wid}/threads/{tid}/unread` | Mark unread |
| POST | `/api/v1/workspaces/{wid}/threads/{tid}/invocations/{runId}/{action}` | Answer a question, approve, cancel, retry a run |
| GET/PUT | `/api/v1/workspaces/{wid}/threads/{tid}/rules` | Thread rules |
| GET | `/api/v1/workspaces/{wid}/threads/{tid}/files` | Files of the thread |
| GET | `/api/v1/workspaces/{wid}/threads/{tid}/files/{fileId}/download` | Download a file (binary) |
| GET/POST | `/api/v1/workspaces/{wid}/assistants/{aid}/memory/core` | Core memory of an assistant |
| PATCH/DELETE | `/api/v1/workspaces/{wid}/assistants/{aid}/memory/core/{entryId}` | Edit or delete a memory entry |
| GET/POST | `/api/v1/workspaces/{wid}/automations` | List automations, or create an empty draft |
| GET | `/api/v1/workspaces/{wid}/automations/{capabilities\|destinations\|connections\|triggers}` | What a draft can use (see *Composing a draft*) |
| PUT | `/api/v1/workspaces/{wid}/automations/{id}/draft` | Save the next draft version |
| GET/DELETE | `/api/v1/workspaces/{wid}/automations/{id}` | Read or delete an automation |
| PATCH | `/api/v1/workspaces/{wid}/automations/{id}/settings` | Edit its settings |
| POST | `/api/v1/workspaces/{wid}/automations/{id}/{activate\|pause\|invoke\|test}` | Control it |
| GET | `/api/v1/workspaces/{wid}/integrations/catalog` | Apps that can be connected |
| GET/POST | `/api/v1/workspaces/{wid}/connections` | Connected accounts; start a connection (authorization link) |
| POST | `/api/v1/workspaces/{wid}/connections/{cid}/reconnect` | A new authorization link |
| POST/DELETE | `/api/v1/workspaces/{wid}/connections/{cid}/grant` | Enable or disable your own account here |
| POST | `/api/v1/workspaces/{wid}/automations/schedule-preview` | Preview a schedule |
| GET | `/api/v1/workspaces/{wid}/automation-runs` | Its runs |
| GET | `/api/v1/workspaces/{wid}/automation-runs/{id}` | One run (`/summary`, `/input`) |
| POST | `/api/v1/workspaces/{wid}/automation-runs/{id}/rerun` | Run it again |

---

## Permissions

| Level | Who has it | Needed for |
|-------|------------|------------|
| read | Any member | Lists, threads, messages, files, documents, rules, memory, automations |
| write | Member, manager or owner (not a viewer), in a workspace that is not archived | Write messages, create threads, mark read, answer runs, edit core memory, thread rules |
| manageContent | Owner or manager of the workspace's own account | Automations, attaching skills |
| manage | Owner or manager who is also an account admin (any owner in a personal workspace) | Settings, members, assistants, knowledge, environment, workspace rules |
| owner | Workspace owner | Ownership and owner-only settings |

Consumer users can do through the API exactly what they can do in the app.
In a shared workspace, what a **member whose account role is `consumer`** (and
any guest from another account) can do is set by the workspace's
[consumer permissions](#consumer-permissions), not by the table above.

### Consumer permissions

A shared workspace keeps a policy, `consumerPermissions`, with one boolean per
action. It applies to **members and viewers whose account role is `consumer`,
and to guests from other Devic accounts**. Owners and managers keep their role's
permissions (table below), viewers stay read-only, and members with a `user` or
`admin` account role are not limited by it. Personal workspaces and direct
messages have no policy.

| Key | Default | Allows | Owner / Manager get it through |
|-----|---------|--------|--------------------------------|
| `write` | `true` | Participate in threads and run assistants. **Every other key needs it** | write |
| `createThreads` | `true` | Create threads | write |
| `uploadFiles` | `true` | Upload, publish and remove thread files | write |
| `editCanvas` | `true` | Edit the thread canvas | write |
| `editThreadRules` | `false` | Edit thread rules | write |
| `editCoreMemory` | `false` | Edit assistants' shared core memory | write |
| `useSharedConnections` | `true` | Use the workspace's shared integrations | write |
| `usePersonalConnections` | `true` | Connect and use their own integrations | write |
| `editWorkspaceRules` | `false` | Edit workspace rules | manage (account admin) |
| `manageAssistants` | `false` | Create and configure workspace assistants | manage (account admin) |
| `manageSkills` | `false` | Manage workspace skills | manageContent |
| `manageAutomations` | `false` | Manage workspace automations | manageContent |
| `manageSharedConnections` | `false` | Manage shared integrations and MCP servers | manageContent |
| `inviteMembers` | `false` | Invite members and viewers | manageContent |

- The management keys (`editWorkspaceRules`, `manageAssistants`, `manageSkills`, `manageAutomations`, `manageSharedConnections`, `inviteMembers`) are never granted to guests from another account, whatever the policy says.
- `useSharedConnections` / `usePersonalConnections` combine with the workspace's `integrationPermissions` and each member's override: both must allow it.
- Account administration, ownership, member roles, shared account catalogs and environment configuration can never be delegated through this policy.

**Read it:** `GET /{wid}` returns `consumerPermissions` (all 14 keys), `policyRevision`, and `capabilities`. For the caller, `capabilities.manageConsumerPermissions` is `true` only for the workspace owner, and a consumer member sees the policy that applies to them in `capabilities.permissions`.

**Change it** (workspace owner only, as a person — an API key without `X-Devic-User` acts as a manager and is refused):

```http
PATCH /api/v1/workspaces/{wid}/consumer-permissions
{
  "revision": 7,
  "consumerPermissions": {
    "write": true, "createThreads": false, "uploadFiles": true, "editCanvas": true,
    "editThreadRules": false, "editCoreMemory": false,
    "useSharedConnections": true, "usePersonalConnections": true,
    "editWorkspaceRules": false, "manageAssistants": true, "manageSkills": false,
    "manageAutomations": false, "manageSharedConnections": false, "inviteMembers": false
  }
}
```

- `revision` is the workspace's **`policyRevision`** (not `revision`). A stale one returns `409 WORKSPACE_REVISION_CONFLICT`: read the workspace again and retry.
- The body is a **complete replacement**: send all 14 keys as booleans. A missing or unknown key returns `400 INVALID_CONSUMER_PERMISSIONS`.
- Returns the updated workspace.
- A refused action returns `403` with `{ "error": "WORKSPACE_CONSUMER_PERMISSION_DENIED", "code": "WORKSPACE_CONSUMER_PERMISSION_DENIED", "permission": "createThreads", "message": "Ask a workspace owner to enable this consumer permission in Participants." }`. Don't retry: only the owner can change it.
- A non-owner changing the policy gets `403` with `"message": "WORKSPACE_OWNER_REQUIRED"`. As with `409 WORKSPACE_REVISION_CONFLICT` and `400 INVALID_CONSUMER_PERMISSIONS`, the code comes in `message`.

---

## List workspaces

```http
GET /api/v1/workspaces?search=ventas&limit=20&offset=0&withSummary=true
```

`search` matches the name and the description. It ignores case and accents,
and every word must appear. `withSummary` adds
`summary: { threadCount, unreadCount, documentCount, lastActivityAt, … }`.

```json
{ "workspaces": [ { "id": "…", "name": "Ventas", "kind": "shared", "visibility": "private",
    "members": [ … ], "assistants": [ … ], "archived": false, "revision": 12 } ],
  "total": 3, "offset": 0, "limit": 20, "hasMore": false }
```

## List and search threads

```http
GET /api/v1/workspaces/threads?search=renovaciones&unread=true&limit=50
```

This covers every workspace you can see, or one workspace with `workspaceId`
(or `GET /{wid}/threads`):

- `search` matches the thread title **or** its workspace's name.
- `unread=true` keeps threads with unread messages or marked unread.
- Results are sorted by latest activity.

```json
{ "threads": [ { "id": "…", "workspaceId": "…", "workspaceName": "Ventas",
    "title": "Renovaciones anuales", "kind": "thread", "archived": false,
    "updatedAt": "2026-11-17T08:43:00.000Z",
    "unread": true, "unreadCount": 2, "unreadMentionCount": 1, "manualUnread": false,
    "lastReadSeq": 61, "lastMessageSeq": 64, "waitingForMe": 1, "activeRunCount": 0 } ],
  "total": 1, "offset": 0, "limit": 50, "hasMore": false }
```

- `unreadCount` counts messages from others after your last read.
- `waitingForMe` counts your runs waiting for you, for example a question or an approval.
- `kind` is `direct` (private conversations) or `shared` (everything else).
- Lists return 20 items by default. `limit` is at most 100, and a larger value returns `400 INVALID_LIMIT` instead of being lowered.

## Read a thread

```http
GET /api/v1/workspaces/{wid}/threads/{tid}?limit=50
GET /api/v1/workspaces/{wid}/threads/{tid}/messages?before=120&limit=50   # older
GET /api/v1/workspaces/{wid}/threads/{tid}/messages?after=164             # follow new ones
```

```json
{ "thread": { "id": "…", "title": "…", "readState": { "unreadCount": 0, "lastReadSeq": 164 } },
  "messages": [ { "id": "…", "seq": 163, "role": "user", "authorId": "u-123", "authorName": "Ana",
                  "text": "…", "createdAt": "…", "origin": { "channel": "cli", "credential": "oauth" } } ],
  "invocations": [ { "id": "…", "status": "waiting_input", "assistantId": "…", "replyToMessageId": "…",
                     "revision": 63, "requestedInput": { "id": "call_…", "questions": [ … ] } } ],
  "hasMore": true, "before": 115, "after": 164, "cursor": 164 }
```

`limit` defaults to 50 and is at most 200. Messages are ordered by `seq`. Pass
`before` to get older pages and `after` to follow new messages; you cannot send
both. `invocations` holds the assistant runs that answer the messages on the
page.

## Write a message

```http
POST /api/v1/workspaces/{wid}/threads/{tid}/messages
X-Devic-User: ana@acme.io

{ "text": "@Analista resume la semana", "directedAssistantId": "asst-1",
  "mentions": [ { "kind": "assistant", "id": "asst-1", "label": "Analista" } ],
  "clientMessageId": "weekly-2026-47" }
```

The response is `{ message, invocations }`. `invocations` lists the assistant
runs the message started; follow them with `messages?after=`.

- `clientMessageId` makes retries safe. If you send the same id with the same body again, you get the same message back. The same id with a different body returns `409 CLIENT_MESSAGE_ID_REUSED`. If you leave it out, one is generated.
- To ask an assistant, set `directedAssistantId`, mention it, or reply (`replyToMessageId`) to one of its messages.
- **`@everybody`** notifies everyone who can read the thread, like a mention, and never asks an assistant: add `{ "kind": "everyone", "id": "everyone", "label": "everybody" }` to `mentions` **and** `@everybody` (or `@everyone`) to the text. Guests from another account get `403 EVERYONE_MENTION_NOT_ALLOWED`; one author can use it once every 10 minutes per thread, otherwise `429 EVERYONE_MENTION_COOLDOWN` and the message is **not** posted.
- The author and the origin come from the principal. The body cannot set them.

## Direct messages

Private conversations are **account-wide**: any two people of the account can have
one, whatever workspaces they share. They all live in one hidden space of the
account, never listed as a workspace.

```http
GET /api/v1/workspaces/direct-messages
X-Devic-User: ana@acme.io
→ { "workspaceId": "dm-space-id", "items": [ { "id": "t1", "kind": "direct", "directUserUIDs": ["u-ana", "u-bea"], ... } ] }

POST /api/v1/workspaces/direct-messages
{ "email": "bea@acme.io" }            // or { "userUID": "u-bea" }
→ the thread (the same one if it already exists)
```

- `workspaceId` is `null` until the account has its first conversation.
- Read, write, mark read and download files with the usual thread endpoints, using that `workspaceId`: `/api/v1/workspaces/{workspaceId}/threads/{tid}/...`.
- Only the two participants can open a conversation. The account integration (an API key without `X-Devic-User`) has no private conversations: `403 DIRECT_MESSAGES_REQUIRE_USER`.
- `POST /{wid}/threads { "kind": "direct", "userUID" }` still works and returns the same account-wide conversation. Old `wid`/`tid` pairs of a conversation keep resolving.
- Errors: `RECIPIENT_NOT_IN_ACCOUNT`, `INVALID_RECIPIENT` (yourself). A conversation being moved returns `503 DIRECT_MESSAGE_MIGRATING` for a few seconds: retry.

## Answer an assistant's questions

A run with `status: "waiting_input"` carries `requestedInput.questions`.

```http
POST /api/v1/workspaces/{wid}/threads/{tid}/invocations/{runId}/answer
{ "revision": 63, "inputId": "call_…",
  "response": { "answers": [ { "id": "q1", "selected": ["annual"] }, { "id": "q2", "other": "Estado = Activo" } ] } }
```

To skip the questions, send `{ "revision": 63, "inputId": "call_…", "skipped": true }`.

To approve a tool call, send `approve` when the run is `waiting_approval`. The body needs the run's `revision` and `requestedAction.id`:
`{ "revision": 14, "actionId": "call_…", "always": false }`. Set `always: true` to also allow this tool from now on, if the run permits it.
There is no "reject" action: to refuse a tool call, `cancel` the run.

## Runs: states and actions

Every action goes through `POST /{wid}/threads/{tid}/invocations/{runId}/{action}`.
**Only the person who started a run can act on it**: anyone else gets
`403 WORKSPACE_INVOCATION_OWNER_REQUIRED`, even an owner or a manager of the
workspace. A run started by the account integration can only be driven by the
integration. `waitingForMe` counts only your own runs.

| Status | Meaning | Actions |
|---|---|---|
| `queued`, `running` | Working | `cancel`, `pause` |
| `paused` | Stopped by `pause` | `resume`, `cancel` |
| `waiting_input` | Asked questions (`requestedInput`) | `answer` |
| `waiting_approval` | Wants to run a tool (`requestedAction`) | `approve`, `cancel` |
| `waiting_connection` | Needs a connected account | `continue` with `{ "connectionIds": [ … ] }` |
| `awaiting_publication` | A private draft only you can see | `publish`, `revise` (`{ revision, text }`), `discard` (`{ revision }`) |
| `outcome_unknown` | Cancelled while a tool was already running: the effect may have happened | `resolve` with `{ "revision": …, "acknowledged": true }`, after checking |
| `failed` | Failed | `retry`, only if `retryable` and no tool had started (`WORKSPACE_UNSAFE_RETRY` otherwise) |
| `completed`, `cancelled` | Finished | — |

- Actions that carry `revision` fail with `409` when the run changed in between. Read the thread again and retry.
- `cancel` works in any state that is not finished, and returns a finished run unchanged.
- An unknown action returns `400 INVALID_INVOCATION_ACTION`.

**Private requests.** A message that asks an assistant and mentions no person
is visible only to you while the run works. When the run completes, the request
and the reply appear in the thread together. If the reply used data only you can
reach, such as your own connected accounts, it stays a private draft
(`awaiting_publication`) until you `publish` it.

## Read state

`POST /{wid}/threads/{tid}/read` with `{ "throughSeq": 164 }`, or `{}` to mark
everything read. `POST /{wid}/threads/{tid}/unread` marks the thread unread.
Both return `{ threadId, readState }`. Each person, and the account
integration, has their own read state.

## Documents and files

- `GET /{wid}/documents` lists every document of the workspace.
  - Filters: `search`, `extension`, `type`, `creator`, `threadId`.
  - Paging: `offset` and `limit` (at most 100).
  - Response: `{ items, total, offset, limit, filters }`.
- `GET /{wid}/threads/{tid}/files` lists the files of one thread as `{ items }`.
- `GET /{wid}/threads/{tid}/files/{fileId}/download` streams the original as `application/octet-stream`. The name is in `Content-Disposition: attachment; filename*=UTF-8''…`.

## Configuration

Settings use the same bodies as the app.

- **Settings:** `PATCH /{wid}` with `{ name, description, logoUrl, visibility, assistantIds, intervention, archived, integrationPermissions, experimentalFeatures, revision }`.
- **Members:** `PUT /{wid}/members/{userUID}` with `{ role: "owner" | "manager" | "member" | "viewer", revision? }`. `DELETE` removes the member.
- **Consumer permissions:** `PATCH /{wid}/consumer-permissions` with `{ revision: policyRevision, consumerPermissions: {…all 14 keys} }`. Owner only. See [Consumer permissions](#consumer-permissions).
- **Assistants:** `POST /{wid}/assistants` and `PATCH /{wid}/assistants/{aid}` take the assistant's configuration (name, model, instructions, tools, `knowledgeSkills`, …). `DELETE` takes `?revision=` for a safe removal.
- **Resources:** `PUT /{wid}/resources` with `{ knowledgeSkills?, knowledge?, environment?, revision? }`. Find candidates with `GET /{wid}/skills/catalog` and `GET /{wid}/knowledge/catalog`. Create or edit a skill's content with the Documents API (`POST /api/v1/documents/skills/scaffold`, `PATCH /api/v1/documents/{id}`; see `documents.md`), then attach it here.
- **Rules:** `PUT /{wid}/rules` (or `/{wid}/threads/{tid}/rules`) with `{ content, expectedRevision }`. A stale `expectedRevision` returns `409`.

## Core memory

These are the facts an assistant always reads in this workspace.

```http
GET    /api/v1/workspaces/{wid}/assistants/{aid}/memory/core
POST   /api/v1/workspaces/{wid}/assistants/{aid}/memory/core          { "section": "decisions", "content": "…", "pinned": true }
PATCH  /api/v1/workspaces/{wid}/assistants/{aid}/memory/core/{entryId} { "content": "…" }
DELETE /api/v1/workspaces/{wid}/assistants/{aid}/memory/core/{entryId}
```

- `section` is one of `persona`, `instructions`, `decisions` or `profile`.
- `content` is at most 4000 characters.
- Reading needs `read`; editing needs `write`.

## Automations

An automation is a routine of a workspace: it reacts to an app event, runs on a
schedule or runs on demand, and posts its results in a destination workspace.
It is always built in two steps:

1. Someone saves a **draft**: a validated plan, with a `draftVersion`.
2. **A person activates** that version and picks the model. A draft never runs.

The app has a designer assistant that writes drafts in a conversation. Through
the API you compose the draft yourself with the same reads the designer uses,
and with the same validation.

> **Step by step:** [workspace-automations.md](workspace-automations.md) is the
> recipe for an agent building an automation for someone (MCP tool names and
> endpoints, decisions to ask, validation errors, when to stop).

Everything needs `manageContent` on the automation's workspace (its owner or a
manager), except listing automations and runs, which needs `read`. Automations
run on behalf of whoever created them. One created by the account integration
has no person behind it, so it cannot use anyone's connected accounts.

### Composing a draft

| Method | Endpoint | Returns |
|--------|----------|---------|
| GET | `/{wid}/automations/capabilities` | `{ sources: ["composio","cron","manual"], includedModel: { modelAccess, provider, model } \| null, designAvailable, triggersAvailable, schedulesAvailable }` |
| GET | `/{wid}/automations/destinations` | `{ items }`: workspaces usable as `targetWorkspaceId` |
| GET | `/{wid}/automations/destinations/{target}/assistants` | `{ workspaceId, items }`: assistants usable as `targetAssistantId` |
| GET | `/{wid}/automations/destinations/{target}/threads` | Threads of one destination: `query`, `mode` (`keywords` \| `literal` \| `regex` \| `list`), `searchIn` (`titles` \| `messages` \| `all`), `match` (`all` \| `any`), `limit` (1–50), `before` (`nextCursor`) |
| GET | `/{wid}/automations/destinations/{target}/threads/{tid}` | Latest public messages of a thread: `query` (literal filter), `limit`, `beforeSeq`, `afterSeq` |
| GET | `/{wid}/automations/connections` | `{ items }`: connected accounts this automation may use |
| GET | `/{wid}/automations/connections/{cid}/tools` | `{ tools }`: exact tool names and schemas of one account |
| GET | `/{wid}/automations/triggers?integration=gmail&query=` | App events (trigger types) of one app; `cursor` for the next page |
| GET | `/{wid}/automations/triggers/{slug}` | Configuration schema, payload schema and provider setup instructions |
| POST | `/{wid}/automations/schedule-preview` | `{ expression, timezone }` → the next five run dates |
| POST | `/{wid}/automations` | `{ name?, sourceThreadId? }` → `{ automation, thread }`, an empty draft |
| PUT | `/{wid}/automations/{id}/draft` | The plan, plus optional `version` → `{ automation, requiresActivation: true }` |

Rules that the validation enforces:

- **Destinations.** A shared workspace can only post to workspaces whose members include all of its own members (`AUTOMATION_DESTINATION_AUDIENCE_MISMATCH`). A personal workspace can post to any workspace its owner can write to.
- **Accounts.** A shared workspace uses accounts connected **for everyone** in it. A personal workspace uses its owner's own accounts. Anything else returns `AUTOMATION_CONNECTION_NOT_SHARED` or `AUTOMATION_CONNECTION_NOT_GRANTED`. Two accounts of the same app are not interchangeable: use each connection's `id`.
- **Tools.** Every name in `externalTools` must come from `connections/{cid}/tools` for that account (`AUTOMATION_TOOL_UNAVAILABLE`).
- **Triggers.** The trigger must belong to the app of `connectionId` (`AUTOMATION_TRIGGER_CONNECTION_MISMATCH`), and `triggerConfig` must carry its required fields (`AUTOMATION_MISSING_CONFIG_FIELD:<field>`). Read the trigger's instructions: some providers also need a webhook set up in their own console, which OAuth alone does not do.

### The plan

The `PUT .../draft` body is the plan.

```json
{
  "version": 0,
  "name": "Meeting summaries",
  "instructions": "Find the customer by the attendee's email domain. Search the destination for a thread with that domain in its messages; if none, create one titled with the company name. Post the summary and the transcript.",
  "targetWorkspaceId": "w-sales",
  "connectionId": "c-fathom",
  "triggerSlug": "FATHOM_NEW_RECORDING",
  "triggerConfig": {},
  "connections": [
    { "connectionId": "c-fathom", "externalTools": ["FATHOM_GET_TRANSCRIPT"], "alwaysLoadedTools": ["FATHOM_GET_TRANSCRIPT"] },
    { "connectionId": "c-zoho", "externalTools": ["ZOHO_SEARCH_CONTACTS"] }
  ]
}
```

| Field | |
|---|---|
| `name`, `instructions`, `targetWorkspaceId` | Required. `instructions` (at most 16000 characters) is everything a run will know: say how to resolve the entity, how to find or create the thread, what to post, and what to do when the match is ambiguous |
| `source` | Omit it for an app event. `{ "type": "cron", "expression": "0 9 * * 1-5", "timezone": "Europe/Madrid" }` for a schedule; preview it first, and ask for the time zone rather than guessing it. `{ "type": "manual" }` for a routine people start from a thread |
| `connectionId`, `triggerSlug`, `triggerConfig` | App events only: the account the trigger listens on, and its configuration |
| `connections` | `[{ connectionId, externalTools, alwaysLoadedTools? }]`: the tools each account grants to the runs. It may be empty. Include the trigger account only if its tools are needed. `alwaysLoadedTools` is a subset loaded from the first call; the rest are discovered on demand |
| `targetAssistantId` | Optional assistant of the destination that each run asks to answer |
| `nativeTools` | `{ "alwaysLoadedTools": ["search_web", "web_extract"] }` to load web search from the first call. Runs can always discover them. Add `"ocr": true` for native document OCR and optionally `"sandbox": "workspace"` (shared files and credentials) or `"sandbox": "clean"` (isolated per execution). Both grants default off and require activation |
| `version` | Optional: the current `draftVersion`. If someone saved another version in between, the save returns `409 AUTOMATION_DRAFT_CHANGED` and writes nothing |

Each save that passes validation increments `draftVersion`. A plan that fails
validation writes nothing.

### Activating and running

| Method | Endpoint | |
|--------|----------|---|
| POST | `/{wid}/automations/{id}/activate` | `{ version, modelAccess: "included" \| "byok", provider, model, maxInputTokens? }`. Use the included model from `capabilities`, or a provider and model with the account's own key. `version` must be the current `draftVersion` |
| POST | `/{wid}/automations/{id}/pause` | Stops it; queued occurrences are cancelled |
| POST | `/{wid}/automations/{id}/test` | `{ operationId }`: runs the approved version of a scheduled automation now. A real run |
| POST | `/{wid}/automations/{id}/invoke` | `{ clientRequestId: uuid, threadId, input? }`: starts an on-demand automation from a thread |
| PATCH | `/{wid}/automations/{id}/settings` | `{ maxInputTokens, revision }`: the input limit per event or tool result, without a new approval |
| DELETE | `/{wid}/automations/{id}` | Deletes it and stops its trigger and schedule |
| GET | `/{wid}/automation-runs` | Runs, newest first: `automationId`, `limit` (≤ 50), `before` |
| GET | `/{wid}/automation-runs/{id}` | One run (`/summary`, `/input`); `POST .../rerun` runs it again |

Activation is the approval. An agent working for someone should save the draft,
show them the plan, and activate only when they ask.

## Integrations

The app accounts (Gmail, HubSpot, Notion…) a workspace can use, for automations
and for its assistants.

| Method | Endpoint | |
|--------|----------|---|
| GET | `/{wid}/integrations/catalog?search=` | Apps by name, or by task with `semantic=true`, and whether each is connected |
| GET | `/{wid}/connections` | Accounts shared in this workspace, and your own with `granted` (enabled here) |
| POST | `/{wid}/connections` | `{ integration, scope: "workspace" \| "user", name?, authScheme? }` → the connection, with `redirectUrl` |
| POST | `/{wid}/connections/{cid}/reconnect` | A new `redirectUrl` for a pending or expired connection |
| POST / DELETE | `/{wid}/connections/{cid}/grant` | Enable or disable one of your own accounts in this workspace |

**Connecting is done by a person.** `POST /connections` creates a pending
connection and returns the provider's authorization link in `redirectUrl`.
Give that link to the person. The account is connected only once they open it
and accept the consent screen; then it appears in `GET /connections` with
`status: "active"`. The API never completes an authorization on anyone's behalf.

- `scope: "workspace"` connects an account **for everyone** in the workspace. It needs `manageContent`, and it is what shared automations require.
- `scope: "user"` connects the caller's own account and enables it in this workspace. It needs a person: the account integration gets `403 PERSONAL_CONNECTION_REQUIRES_USER`.
- Only app integrations can be connected this way. Remote MCP servers come from the account's tool servers in the app.

---

## Response format and errors

Responses use the public API envelope: `{ "success": true, "data": … }`.
Errors return `{ "error": "CODE", "message": … }`. Common codes:

- `WORKSPACE_ACCESS_DENIED`
- `WORKSPACE_READ_ONLY`
- `WORKSPACE_MANAGER_REQUIRED`
- `WORKSPACE_ADMIN_REQUIRED`
- `WORKSPACE_OWNER_REQUIRED`
- `WORKSPACE_CONSUMER_PERMISSION_DENIED` (with `permission`: the policy key that is off)
- `INVALID_CONSUMER_PERMISSIONS`, `WORKSPACE_REVISION_CONFLICT` (409)
- `WORKSPACE_THREAD_NOT_FOUND`
- `PRIVATE_CONVERSATION`
- `DIRECT_MESSAGES_REQUIRE_USER`
- `RECIPIENT_NOT_IN_ACCOUNT`
- `EVERYONE_MENTION_NOT_ALLOWED`, `EVERYONE_MENTION_COOLDOWN` (429)
- `DIRECT_MESSAGE_MIGRATING` (503, retry)
- `INVALID_LIMIT`
- `INVALID_MESSAGE_CURSOR`
- `WORKSPACE_INPUT_CONFLICT`, `WORKSPACE_APPROVAL_CONFLICT` (stale `revision`, or the run is no longer waiting)
- `WORKSPACE_INVOCATION_OWNER_REQUIRED` (the run belongs to someone else)
- `WORKSPACE_UNSAFE_RETRY`

API keys restricted with `allowedEndpoints` need `/api/v1/workspaces/*`. OAuth
apps need the `workspaces:read` and `workspaces:write` scopes, or a
`platform:*` scope.
