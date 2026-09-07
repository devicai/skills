# Environments & Sandboxes API

An **environment** is the machine an agent works on plus everything it is
allowed to reach: its sandbox, its snapshot, the knowledge and tools it
inherits, and its encrypted variables. One environment serves many agents and
assistants, which is what lets a team of agents share one machine state.

A **sandbox** is a real, isolated Linux machine started on an environment. An
agent connected to the environment can run commands on it; so can you, through
these endpoints.

**Secrets belong here, not in a message.** A thread message is stored in
`threadContent` and readable back over the API indefinitely. An environment
variable is encrypted at rest and injected into the machine as an ordinary
environment variable, so a script reads `$DATABASE_URL` without the value ever
appearing in a conversation.

---

## Endpoints Overview

### Environments

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/environments` | List environments |
| POST | `/api/v1/environments` | Create an environment |
| GET | `/api/v1/environments/{environmentId}` | Get one |
| PATCH | `/api/v1/environments/{environmentId}` | Update it |
| DELETE | `/api/v1/environments/{environmentId}` | Delete it and its per-tenant snapshots |

### Connections

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/environments/{environmentId}/connections` | Agents and assistants connected |
| PUT | `/api/v1/environments/{environmentId}/connections/{entityType}/{entityId}` | Connect one |
| DELETE | `/api/v1/environments/{environmentId}/connections/{entityType}/{entityId}` | Disconnect it |

`entityType` is `agent` or `assistant`. An agent can also be connected on
create or update with the top-level `environmentId` field — see
[agents.md](agents.md).

### Snapshots

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/environments/{environmentId}/snapshot/initialize` | Bake (or re-bake) the base snapshot |
| GET | `/api/v1/environments/{environmentId}/tenant-snapshots` | List per-tenant snapshots |
| POST | `/api/v1/environments/{environmentId}/tenant-snapshots/{tenantId}/initialize` | Derive one tenant's snapshot from the base |
| DELETE | `/api/v1/environments/{environmentId}/tenant-snapshots/{tenantId}` | Drop it |

### Sessions and tooling

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/environments/{environmentId}/sessions` | Sessions run on this environment |
| GET | `/api/v1/environments/{environmentId}/sessions/last` | The most recent one, reconciled against the engine |
| GET | `/api/v1/environments/{environmentId}/sessions/{sessionId}` | One session with its command log |
| GET | `/api/v1/environments/{environmentId}/clis` | CLIs available and installed |

### Sandboxes

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/environments/{environmentId}/sandbox/start` | Start a session |
| GET | `/api/v1/environments/{environmentId}/sandbox/status` | Whether one is live |
| POST | `/api/v1/environments/{environmentId}/sandbox/exec` | Run a line of shell |
| POST | `/api/v1/environments/{environmentId}/sandbox/stop` | Stop it — this is what saves the snapshot |
| POST | `/api/v1/environments/{environmentId}/sandbox/files/list` | List a directory |
| POST | `/api/v1/environments/{environmentId}/sandbox/files/read` | Read a text file |
| POST | `/api/v1/environments/{environmentId}/sandbox/files/write` | Write a file |

**Not reachable with a tenant session token.** These endpoints hand out a root
shell, the account's secrets and the snapshot every tenant boots from; scoping
an end user to their own data is no answer to that. Use an API key. See
[tenant-sessions.md](tenant-sessions.md).

---

## Environment Structure

| Field | Type | Description |
|-------|------|-------------|
| `_id` | string | Environment id |
| `name` | string | Display name |
| `description` | string | What it is for |
| `projectId` | string | Project it belongs to |
| `envVars` | object | `{ "KEY": "value" }`, encrypted at rest, **returned masked** |
| `sandboxConfig` | object | The machine (see below) |
| `availableToolsGroupsUids` | string[] | Tool groups and MCPs the connected entities inherit |
| `knowledgeDocumentIds` | string[] | Knowledge documents inherited |
| `knowledgeFolderIds` | string[] | Knowledge folders inherited |
| `knowledgeSkills` | array | `{ id, type: "document" \| "folder" }` skills inherited |

### sandboxConfig

| Field | Type | Description |
|-------|------|-------------|
| `runtime` | string | `node24` (default), `node22`, `python3.13` |
| `memoryMib` | number | RAM for the machine. Baked into the snapshot |
| `initScript` | string | Shell script run when a **new** sandbox is created. Skipped on a restore |
| `envVars` | object | Sandbox-level variables, layered under the environment ones |
| `snapshotEnabled` | boolean | Save the filesystem between sessions |
| `replaceSnapshotOnStop` | boolean | **Opt-in.** `true` lets a session's changes replace the snapshot. Unset (not just `false`) means fixed: sessions start from the saved state and cannot write back |
| `perTenantSnapshots` | boolean | Give every tenant its own snapshot, derived from the base |
| `persistAfterSessionClose` | boolean | Keep the machine alive after the conversation ends |
| `autoExtend` | boolean | An operation arriving near the deadline buys another full timeout. Idle sandboxes still expire |
| `autoRestart` | boolean | Restore when the public URL is visited. Opt-out: absent means enabled |
| `publicSlug` | string \| null | Subdomain for `<slug>.sandbox.devic.ai`; `null` releases it |
| `startCommand` | string \| null | Run after each restore to bring the published service back up |
| `provider` | string | `vercel` or `devic-sandbox`. Left unset, the platform default |
| `snapshotId` | string | Managed by the backend. Ignored on write |

`initScript` prepares a machine and runs when one is created. `startCommand`
serves the snapshot and runs on every restore, including one triggered by a
visitor arriving at the public URL. Restoring brings back a filesystem, not a
process someone had started by hand — which is exactly why both exist.

---

## How secrets behave

**Plaintext in, masked out.**

```bash
curl -X POST https://api.devic.ai/api/v1/environments \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Reporting box",
    "envVars": { "DATABASE_URL": "postgres://user:pass@host/db" }
  }'
```

```json
{ "_id": "…", "name": "Reporting box", "envVars": { "DATABASE_URL": "••••••••" } }
```

`envVars` **replaces the whole map** on PATCH. That would make a partial edit
destructive, so the backend reads a returned mask as *keep the stored secret*:

```jsonc
// Adds REGION and keeps DATABASE_URL intact.
{ "envVars": { "DATABASE_URL": "••••••••", "REGION": "eu-west-1" } }
```

The safe pattern is read → modify → write: GET the environment, edit the map
you got back, PATCH it. Omitting a key is how you delete it.

> A sandbox is real. Anything in `envVars` is reachable from inside it by any
> agent connected to the environment. Give an environment the narrowest set of
> secrets its agents actually need, and use separate environments where the
> blast radius should differ.

---

## Running something on the machine

The lifecycle is explicit because none of it is free: starting provisions and
bills a machine, and stopping is what saves the snapshot.

### 1. Start

```bash
curl -X POST https://api.devic.ai/api/v1/environments/{environmentId}/sandbox/start \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{ "timeoutMinutes": 15 }'
```

```json
{
  "sandboxId": "sbx_…",
  "runtime": "node24",
  "expiresAt": 1788780063726,
  "initScript": { "exitCode": 0, "stdout": "…", "stderr": "" }
}
```

`timeoutMinutes` is 1–30 (default 10). With `autoExtend` on the environment, a
busy sandbox renews; an idle one still expires. `initScript` comes back only
when a fresh machine was created — verifying a provisioning script is the whole
reason to start one by hand.

**One session per environment.** A second concurrent start returns `409` with
code `SESSION_ACTIVE`, because two sandboxes would race the shared snapshot on
save. Pass `force: true` to take over, discarding the other's unsaved state.
With `perTenantSnapshots`, the contended resource is the tenant's own snapshot,
so pass `tenantId` and two tenants can be worked on side by side.

A `409 SNAPSHOT_SAVE_IN_PROGRESS` means the snapshot is still being written;
`forceUnsavedSnapshot: true` starts from its last saved version instead.

### 2. Exec

```bash
curl -X POST https://api.devic.ai/api/v1/environments/{environmentId}/sandbox/exec \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "sandboxId": "sbx_…",
    "command": "cd /workspace && ./daily-report.sh | tail -20"
  }'
```

```json
{ "exitCode": 0, "stdout": "…", "stderr": "", "durationMs": 396, "cwd": "/workspace" }
```

**`command` is a line of shell, not an argv[0].** It runs through `bash -c`, so
pipes, redirection, `&&` and `cd` all work and belong inside it. The optional
`args` array is appended space-separated **as text** — nothing is quoted or
escaped for you, so an argument containing spaces must arrive already quoted, or
be written into `command` instead.

A non-zero `exitCode` is a result, not an HTTP error: the call is a 200 and the
failure is in the body.

`cwd` is tracked across calls — send back the one you got and `cd` persists.

### 3. Stop

```bash
curl -X POST https://api.devic.ai/api/v1/environments/{environmentId}/sandbox/stop \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{ "sandboxId": "sbx_…" }'
```

With snapshots enabled this is what persists the work, and the save runs in the
background: `saving: true` means the machine stops once it finishes.
`saveChanges: false` throws the session away — the right choice when snapshots
are off and you were only verifying something.

Forgetting to stop is survivable: the session expires on its own. What is lost
is the snapshot, not the machine.

---

## Files

All three take `sandboxId` and go through POST, because a path is a body field
rather than something to escape into a URL.

```jsonc
// files/list  → { path, entries: [{ name, path, type, size }] }
{ "sandboxId": "sbx_…", "path": "/workspace" }

// files/read  → { path, content, size }   text only, 1 MB max
{ "sandboxId": "sbx_…", "path": "/workspace/report.log" }

// files/write → { success, path, size }
{ "sandboxId": "sbx_…", "destinationPath": "/workspace/report.sh", "content": "#!/bin/bash\n…" }
```

`files/write` also takes `sourceUrl` instead of `content`, and the sandbox
fetches the bytes itself.

`files/read` refuses binary files and anything over 1 MB rather than truncating
it — read a large file with `exec` and a pager, or write it somewhere it can be
served from.

---

## Snapshots

By default a sandbox is thrown away when the session ends. With
`snapshotEnabled` the filesystem is saved instead, so the next session starts
where the last one left off — dependencies installed, working directory intact.

The snapshot belongs to the **environment**, not to any single agent. That is
what makes a team of agents work on the same material.

```bash
# Bake (or re-bake) the base snapshot: creates a machine, runs the init script
# and the configured CLIs, saves the result.
curl -X POST https://api.devic.ai/api/v1/environments/{environmentId}/snapshot/initialize \
  -H "Authorization: Bearer devic-your-api-key"
```

Re-baking discards the state the snapshot had accumulated. That is the point:
it is how a machine is brought back in line with a changed init script.

### Fixed by default, and who may write

`replaceSnapshotOnStop` is checked for `true`. **Unset means fixed** — agent
runs boot from the snapshot and cannot write back to it. This is usually what
you want: it is what stops a stray run from destroying a machine you spent time
provisioning.

Manual sandbox sessions are the exception, and deliberately: `POST /sandbox/stop`
with `saveChanges: true` writes the snapshot whatever the mode says. That is the
only way to provision a fixed environment, and it is why the CLI makes it
explicit (`devic sandbox stop --save`) rather than doing it silently.

### Provisioning a machine: bake once, then leave it alone

The workflow the fixed default exists for — install dependencies by hand, save
them, and let every later run start from there without being able to break it:

```bash
# 1. An environment with snapshots, fixed (the default).
devic environments create --name "Build box" --runtime node24 --snapshots

# 2. A machine to work on. No snapshot yet, so this one is fresh and runs the
#    init script.
devic sandbox start "Build box" --timeout 20

# 3. Provision it: dependencies, libraries, files, whatever the agent will need.
devic sandbox exec "Build box" 'cd /workspace && npm install'
devic sandbox write "Build box" /workspace/report.py --file ./report.py

# 4. Bake it. `--save` is required because the environment is fixed.
devic sandbox stop "Build box" --save

# 5. From here every session — yours or an agent's — starts with all of it
#    already in place, and none of them can overwrite it.
```

The save runs in the background: `stop` returns `saving: true` and the snapshot
id, and the environment's `sandboxConfig.snapshotId` picks up that id once the
capture finishes. Starting a session again before it does answers `409
SNAPSHOT_SAVE_IN_PROGRESS`.

To let sessions evolve the snapshot instead, `devic environments update
"Build box" --evolving-snapshot`, and `--fixed-snapshot` to freeze it again.

`POST /snapshot/initialize` does the same baking unattended, from the init
script and the configured CLIs, with nobody at a terminal. Use it when the
provisioning is already written down; use the manual session when you are
still working out what it should say.

### Per tenant

With `perTenantSnapshots`, each customer's first session starts from the shared
base and from then on keeps its own. Sessions with no tenant keep using the
shared one. Re-deriving a tenant (`/tenant-snapshots/{tenantId}/initialize`)
brings that one customer back in line with a re-baked base, discarding their
accumulated state; deleting it sends their next session back to the base.

---

## A worked example: a nightly report against a private database

The shape this API exists for.

```bash
# 1. An environment holding the credentials and the client tooling.
curl -X POST https://api.devic.ai/api/v1/environments \
  -H "Authorization: Bearer $DEVIC_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "name": "Reporting box",
    "envVars": {
      "DATABASE_URL": "postgres://reporter:…@db.internal/analytics",
      "SSH_PRIVATE_KEY": "-----BEGIN OPENSSH PRIVATE KEY-----\n…"
    },
    "sandboxConfig": {
      "runtime": "python3.13",
      "initScript": "pip install psycopg2-binary && mkdir -p /workspace",
      "snapshotEnabled": true,
      "autoExtend": true
    }
  }'

# 2. Bake the snapshot once, so no run pays for the install.
curl -X POST https://api.devic.ai/api/v1/environments/$ENV_ID/snapshot/initialize \
  -H "Authorization: Bearer $DEVIC_API_KEY"

# 3. An agent that wakes itself at 07:00 on weekdays, connected to it.
curl -X POST https://api.devic.ai/api/v1/agents \
  -H "Authorization: Bearer $DEVIC_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "name": "Nightly analytics report",
    "environmentId": "'"$ENV_ID"'",
    "periodicExecution": {
      "enabled": true,
      "cronExpression": "0 7 * * 1-5"
    },
    "assistantSpecialization": {
      "presets": "Every morning, open an SSH tunnel with $SSH_PRIVATE_KEY, query $DATABASE_URL and summarise yesterday."
    }
  }'
```

No external scheduler, and no credential in a message: the agent wakes itself,
and the keys live encrypted on the environment where only the machine sees them.

---

## Error Responses

| Status | Meaning |
|--------|---------|
| 400 | Malformed body; or an `environmentId` on an agent that names no environment of this account — the agent was still saved, only the connection was not |
| 403 | A tenant session token reached this surface. Use an API key |
| 404 | No such environment, session or snapshot in this account |
| 409 | `SESSION_ACTIVE` (a session is already live) or `SNAPSHOT_SAVE_IN_PROGRESS` (the snapshot is still being written) |

---

## From the CLI

Every command takes the environment by id **or by name**.

```bash
devic environments create --name "Reporting box" --runtime python3.13 \
  --env "DATABASE_URL=postgres://…" --init-script-file ./provision.sh --snapshots
devic environments snapshot init "Reporting box"
devic environments connect "Reporting box" agent <agentId>

devic sandbox start "Reporting box" --timeout 15
devic sandbox exec "Reporting box" 'cd /workspace && ./report.sh | tail -20'
devic sandbox stop "Reporting box"
```

See the [devic-cli skill](../devic-cli/SKILL.md).
