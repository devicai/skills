# Workspaces API

A **workspace** is a shared space where people and assistants work together in
**threads**. Each thread is a conversation that holds messages, files, runs of
assistants and its own read state for every person.

This API reads and writes the same workspaces as the web and mobile apps. The
same permissions apply: you only see the workspaces you belong to, and every
action needs the same workspace role as in the apps.

**Base path:** `/api/v1/workspaces`

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
| GET | `/api/v1/workspaces/threads` | Threads across every workspace (`workspaceId`, `search`, `unread`, `includeArchived`, `offset`, `limit`) |
| GET | `/api/v1/workspaces/{wid}` | Get a workspace |
| PATCH | `/api/v1/workspaces/{wid}` | Update its settings |
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
| GET/POST | `/api/v1/workspaces/{wid}/automations` | List or create automations |
| GET/DELETE | `/api/v1/workspaces/{wid}/automations/{id}` | Read or delete an automation |
| PATCH | `/api/v1/workspaces/{wid}/automations/{id}/settings` | Edit its settings |
| POST | `/api/v1/workspaces/{wid}/automations/{id}/{activate\|pause\|invoke\|test}` | Control it |
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
- `waitingForMe` counts runs waiting for you, for example a question or an approval.

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
- The author and the origin come from the principal. The body cannot set them.

## Answer an assistant's questions

A run with `status: "waiting_input"` carries `requestedInput.questions`.

```http
POST /api/v1/workspaces/{wid}/threads/{tid}/invocations/{runId}/answer
{ "revision": 63, "inputId": "call_…",
  "response": { "answers": [ { "id": "q1", "selected": ["annual"] }, { "id": "q2", "other": "Estado = Activo" } ] } }
```

To skip the questions, send `{ "revision": 63, "inputId": "call_…", "skipped": true }`.

Other actions use the same route with their own name, such as `cancel` and `retry`.

To approve a tool call, send `approve` when the run is `waiting_approval`. The body needs the run's `revision` and `requestedAction.id`:
`{ "revision": 14, "actionId": "call_…", "always": false }`. Set `always: true` to also allow this tool from now on, if the run permits it.

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

An automation is created from a plan, then controlled with
`activate`, `pause`, `invoke` and `test`. Runs are listed with
`GET /{wid}/automation-runs`.

- `manageContent` is required for everything except listing.
- Automations run on behalf of whoever created them. One created by the account integration has no person behind it, so it cannot use anyone's connected accounts.

---

## Response format and errors

Responses use the public API envelope: `{ "success": true, "data": … }`.
Errors return `{ "error": "CODE", "message": … }`. Common codes:

- `WORKSPACE_ACCESS_DENIED`
- `WORKSPACE_READ_ONLY`
- `WORKSPACE_MANAGER_REQUIRED`
- `WORKSPACE_ADMIN_REQUIRED`
- `WORKSPACE_THREAD_NOT_FOUND`
- `PRIVATE_CONVERSATION`
- `INVALID_LIMIT`
- `INVALID_MESSAGE_CURSOR`
- `WORKSPACE_INPUT_CONFLICT`

API keys restricted with `allowedEndpoints` need `/api/v1/workspaces/*`. OAuth
apps need the `workspaces:read` and `workspaces:write` scopes, or a
`platform:*` scope.
