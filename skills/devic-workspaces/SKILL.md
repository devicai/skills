---
name: devic-workspaces
description: "Devic Workspaces from code and the terminal — shared spaces where people and assistants work in threads. Use when reading or searching a person's threads (across workspaces, only unread, paginated), reading a thread's messages, writing a message as a person or as an integration and waiting for the assistant's reply, answering an assistant's questions or approving its tool calls, downloading the files a thread produced, or configuring a workspace (members, assistants, knowledge and skills, rules, core memory), building automations (app events, schedules, on-demand routines) and connecting the app accounts they use, through the public API (/api/v1/workspaces) or the CLI (devic workspaces / devic ws)."
---

# Devic Workspaces

A **workspace** is a shared space where people of an account and AI assistants
work together. Work happens in **threads**: each thread has messages, the runs
of the assistants that answer them, files, and a read state for every person.

Everything the web and mobile apps do in a workspace can also be done through
the public API under `/api/v1/workspaces` and the CLI (`devic workspaces`, alias
`devic ws`), **with the same permissions**.

This skill is the task guide. The exhaustive references are:

- the API contract, in the `devic-api` skill → `workspaces.md`;
- the full CLI command tree, in the `devic-cli` skill → `workspaces.md`.

---

## 1. Decide who you act as — first

Workspaces belong to people, so every request resolves a **principal**.

| Credential | `X-Devic-User` / `--as` / `DEVIC_USER` | Acts as |
|---|---|---|
| A person's token (`devic auth login`, OAuth app) | — | That person |
| API key | userUID or email of a person **of the key's account** | That person, with **their own** account role and workspace roles |
| API key | not given | The **account integration**, shown as "API" |

What the account integration can and cannot do:

- It sees the account's **shared** workspaces only. It never sees personal workspaces or private (direct) conversations.
- It acts as a workspace **manager**: it reads, writes and configures, but it is never an owner.
- Its messages are authored `api-key:<id>` / "API", and it has its own read state.
- Automations it creates have no person behind them, so they cannot use anyone's connected accounts.

Before acting, check the principal:

```bash
devic ws me                              # {principal: person|integration, userUID, name, role, origin}
devic ws --as ana@acme.com me
```

```http
GET /api/v1/workspaces/me
Authorization: Bearer <api key>
X-Devic-User: ana@acme.com
```

What can go wrong:

- **Naming someone outside the key's account** returns `403 WORKSPACE_ACT_AS_USER_NOT_FOUND`.
- **A person's token naming someone else** returns `403 WORKSPACE_ACT_AS_REQUIRES_API_KEY`.
- **Tenant-session tokens** are rejected: they identify your customers, not members of the account.
- **Acting for a person never adds permissions.** A key with an admin role acting for a `consumer` user can only do what that consumer can do in the app.

Every message records where it came from, as
`origin: {channel: api|cli|sdk, credential, apiKeyId?, onBehalfOf?}`. The apps
show a "via API / CLI / SDK" tag on those messages. The server sets `origin`;
the request body cannot.

---

## 2. Find the work

```bash
devic ws list --search ventas --summary            # workspaces, with unread/thread/document counts
devic ws threads list --unread                      # unread threads across EVERY workspace
devic ws threads list --search "renovaciones"       # matches thread title OR workspace name
devic ws threads list --workspace <wid> --limit 50 --offset 50
```

The same calls through the API:

```http
GET /api/v1/workspaces?search=ventas&withSummary=true
GET /api/v1/workspaces/threads?unread=true&search=renovaciones&limit=20&offset=0
```

- **Search** ignores case and accents, and every word must appear.
- **Unread** means messages from others after your last read, or a thread marked unread by hand.
- **Thread rows** carry `unreadCount`, `unreadMentionCount`, `waitingForMe` (a question or an approval waiting for you) and `activeRunCount`.
- **Pagination** uses `offset` and `limit`, with a limit of at most 100. The response carries `total` and `hasMore`.

## 3. Read a thread

```bash
devic ws threads get <wid> <tid> -o human           # transcript + pending questions
devic ws threads messages <wid> <tid> --before 120  # older page
devic ws threads messages <wid> <tid> --after 164   # only what is new
devic ws threads read <wid> <tid>                   # mark read (reading alone never does)
```

- Messages are paged by `seq`, at most 200 per page.
- The page also returns `invocations`: the assistant runs answering those messages, **plus every run still in progress**.

## 4. Write, and wait for the assistant

```bash
devic ws send <wid> <tid> "Resume la semana" --assistant <aid> --wait --timeout 600
echo "texto largo" | devic ws send <wid> <tid> - --client-id weekly-2026-47
```

- **Address an assistant** with `--assistant` (`directedAssistantId`), by mentioning it, or by replying to one of its messages. A message that addresses no assistant is posted for the people in the thread.
- **`--everybody`** notifies everyone who can read the thread (a mention, never an assistant). Use it sparingly: once every 10 minutes per author and thread (`EVERYONE_MENTION_COOLDOWN`), never for guests.
- **Write to one person privately** with `devic ws dm send <userUID|email> "<text>"`: conversations are account-wide (`devic ws dm list`; their `workspaceId` is the account's hidden space), and need a person, not the integration.
- **Retries:** `--client-id` (`clientMessageId`) makes them safe. The same id with the same body returns the same message; with a different body it returns `409`.
- **`--wait`** polls until every run the message started has finished or is waiting for you. It returns `outcome`: `completed`, `failed`, `waiting` or `no_invocations`.
  - In JSON mode it first prints NDJSON status lines, then the final document.
  - A timeout exits with code `3`.

## 5. When the assistant waits for you

`outcome: "waiting"`, or a run in `waiting_input` / `waiting_approval`:

```bash
# It asked questions (requestedInput.questions[].options[].value)
devic ws answer <wid> <tid> <runId> --answers '[{"id":"periodicidad","selected":["anual"]}]'
devic ws answer <wid> <tid> <runId> --skip

# It wants to run a tool that needs approval (e.g. the terminal)
devic ws approve <wid> <tid> <runId>            # --always to allow that tool from now on
```

Both commands read the thread to fill in what the server requires:

- answering needs `{revision, inputId}`;
- approving needs `{revision, actionId: requestedAction.id}`.

Through the raw API you send those yourself:

```http
POST /api/v1/workspaces/{wid}/threads/{tid}/invocations/{runId}/answer
{ "revision": 63, "inputId": "call_…", "response": { "answers": [ { "id": "q1", "selected": ["anual"] } ] } }

POST /api/v1/workspaces/{wid}/threads/{tid}/invocations/{runId}/approve
{ "revision": 14, "actionId": "call_…" }
```

A stale `revision` returns `409`. Read the thread again and retry. Other
actions (`cancel`, `retry`, `publish`, `resolve`, …) go through `devic ws run-action`;
the states and what each one accepts are in `devic-api` → `workspaces.md`, *Runs*.

- **Only the person who started a run can answer, approve or cancel it**
  (`403 WORKSPACE_INVOCATION_OWNER_REQUIRED`). Acting for someone else's run means
  acting **as** that person, with `--as`.
- **There is no reject.** To refuse a tool call, cancel the run.
- **Cancelling while a tool runs** leaves the run `outcome_unknown`: the effect
  may have happened. Check it, then `run-action … resolve --data '{"revision":…,"acknowledged":true}'`.
- **A question to an assistant that mentions no person is private** until the
  run completes; a reply built on your own connected accounts stays a draft
  (`awaiting_publication`) until you `publish` it.

## 6. Files and documents

```bash
devic ws files list <wid> <tid>
devic ws files download <wid> <tid> <fileId> --out ./out/     # never overwrites without --force
devic ws documents <wid> --search contrato --extension pdf     # every document of the workspace
```

The download is the original bytes, with its name in `Content-Disposition`.

## 7. Configure a workspace

```bash
devic ws update <wid> --name "Ventas" --description "…"
devic ws members set <wid> <userUID> --role member            # owner|manager|member|viewer
devic ws assistants add <wid> --data '{"sourceAssistantId":"…"}'
devic ws skills catalog <wid> && devic ws skills attach <wid> <skillId> folder:<folderId>
devic ws rules set <wid> --file rules.md                       # standing instructions
devic ws memory add <wid> <aid> --section decisions --content "Precios en EUR" --pinned
devic ws automations list <wid>
```

- **Skills** are linked as `{type: document|folder, id}` in `resources.knowledgeSkills`. Create or edit a skill's content with the Skills API or `devic skills`, then attach it here.
- **Core memory** holds what an assistant always reads in this workspace. Sections are `persona`, `instructions`, `decisions` and `profile`, and `content` is at most 4000 characters.
- **Writes that carry a `revision`** (rules, resources, members, assistants) return `409` if someone changed the object in between. The CLI fills in the current revision.

## 7b. Build an automation

An automation reacts to an app event, runs on a schedule or runs on demand, and
posts in a destination workspace. You save a **draft**; **a person activates
it**. The full plan contract is in `devic-api` → `workspaces.md`, *Automations*.

```bash
devic ws automations capabilities <wid>                       # cron? app events? included model?
devic ws automations destinations <wid>                       # where it may post
devic ws automations connections <wid>                        # accounts it may use
devic ws integrations search <wid> "meeting recordings" --semantic        # app missing?
devic ws connections connect <wid> fathom --scope workspace   # → authorization link for the person
devic ws automations triggers <wid> --integration fathom --query recording
devic ws automations trigger <wid> <slug>                      # config schema + setup instructions
devic ws automations connection-tools <wid> <connectionId>    # exact tool names to grant
devic ws automations find-threads <wid> <target> --query acme --search-in all
devic ws automations schedule-preview <wid> --data '{"expression":"0 9 * * 1-5","timezone":"Europe/Madrid"}'
devic ws automations create <wid> --name "Meeting summaries"
devic ws automations draft <wid> <id> --data plan.json        # validated; never activates
devic ws automations activate <wid> <id> --included           # only when the person says so
```

- **Discover, never guess.** Trigger slugs, connection ids and tool names come from the listings above; the validation rejects anything else before writing.
- **A missing app is a link, not a dead end.** `connections connect` returns the provider's consent link. The person opens it; nothing is connected until they finish. Shared workspaces need `--scope workspace` (an account for everyone).
- **Write instructions a run can follow alone**: how to identify the customer or record, how to find the existing thread (stable source ids in message text first), when to create a new one, what to post, and what to do when the match is ambiguous.
- **Schedules** need an explicit IANA time zone. Ask for it, and show the preview dates.
- **Grant the minimum**: only the tools each run needs, per account.
- **`--draft-version`** guards against overwriting someone else's edit (`409 AUTOMATION_DRAFT_CHANGED`).
- **Activation is the approval.** Show the plan, then activate with the model the person chooses: `--included`, or `--provider` and `--model` with the account's own key.

## 8. Permissions (same as the app)

| Level | Who | For |
|---|---|---|
| read | any member | lists, threads, files, documents, rules, memory |
| write | member, manager, owner (not viewer), workspace not archived | messages, threads, read state, answers, approvals, memory, thread rules |
| manageContent | owner or manager of the workspace's account | automations, attaching skills |
| manage | owner or manager who is also an account admin | settings, members, assistants, knowledge, workspace rules |
| owner | workspace owner | ownership |

Consumer users can do the same through the API as in the app. In a shared
workspace, members with the `consumer` account role (and guests from other
accounts) are governed by the workspace's **consumer permissions**: 14 booleans
such as `createThreads`, `uploadFiles`, `editThreadRules`, `manageAssistants` or
`inviteMembers`. By default they can participate, create threads, upload files,
edit the canvas and use integrations, and nothing else. A refused action returns
`403 WORKSPACE_CONSUMER_PERMISSION_DENIED` with the `permission` key that is off:
tell the person to ask a workspace owner, don't retry.

- Read the policy in `devic ws get <wid>` → `consumerPermissions`, `policyRevision`; your own effective policy is in `capabilities.permissions`.
- Only the **workspace owner** can change it, with `PATCH /api/v1/workspaces/{wid}/consumer-permissions` (`{revision: policyRevision, consumerPermissions: {all 14 keys}}`). There is no `devic ws` command for it yet. Full key list in `devic-api` → `workspaces.md`, *Consumer permissions*.

## 9. Credentials and access

- **API keys with `allowedEndpoints`** need the "Workspaces" group (`/api/v1/workspaces/*`).
- **OAuth apps** need `workspaces:read` or `workspaces:write`, or a `platform:*` scope.
- **Responses** use the public envelope `{success, data}`. Errors come as `{error: CODE, message}`, for example `WORKSPACE_ACCESS_DENIED`, `WORKSPACE_READ_ONLY`, `WORKSPACE_MANAGER_REQUIRED`, `WORKSPACE_CONSUMER_PERMISSION_DENIED`, `PRIVATE_CONVERSATION`, `WORKSPACE_INPUT_CONFLICT` or `WORKSPACE_APPROVAL_CONFLICT`.

---

## Recipe: an inbox assistant for a person

```bash
export DEVIC_USER=ana@acme.com
devic ws threads list --unread -o json |
  jq -r '.threads[] | select(.waitingForMe > 0) | "\(.workspaceId) \(.id) \(.title)"' |
  while read wid tid title; do
    devic ws threads get "$wid" "$tid" -o human   # see what is being asked
  done
```

## Recipe: post an automatic notice as the integration

```bash
unset DEVIC_USER    # no person: authored "API"
devic ws send <wid> <tid> "Despliegue 2026-11-17 completado" --client-id deploy-2026-11-17
```
