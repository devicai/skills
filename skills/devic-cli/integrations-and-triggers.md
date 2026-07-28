# Integrations & Triggers

Two command groups for connected apps and the events they emit:

- **`integrations`** — browse the catalogue of apps you can connect (Gmail,
  Drive, HubSpot, …), read each app's event types, and connect an account.
- **`triggers`** — the subscriptions that start an agent or assistant when a
  connected app emits an event.

A trigger listens to an **integration tool server** (its source of events) and
points at one agent or assistant. So the order is: connect an app → get an
integration tool server → create triggers on it.

---

## Integrations

The catalogue is the provider's — every app you *could* connect, not only the
ones you have. It is large, so it is browsed by cursor and searchable.

```bash
# Browse / search the app catalogue
devic integrations list
devic integrations list --search gmail --limit 20
devic integrations list --cursor <nextCursor>   # next page

# One app
devic integrations get gmail
```

### An app's event types

```bash
# The events this app can emit (paginated + searchable)
devic integrations triggers gmail
devic integrations triggers gmail --search "new message"

# One event type in full — its config and payload schemas
devic integrations triggers gmail GMAIL_NEW_GMAIL_MESSAGE
```

The detail view returns two JSON Schemas:

- **`config`** — the trigger's own settings (poll interval, Gmail label, …).
  This is what you pass to `triggers create --from-json`.
- **`payload`** — the shape of the event body. Its leaves are the fields a
  message template can reference, e.g. `{{data.file.name}}`.

### Connecting an account

Connecting is a browser OAuth flow, so it runs in two steps: authorize in a
browser, then build the tool server. **Step 1 alone does not create anything** —
it only returns the authorization URL. The integration is inactive until step 2
builds its tool server, so `--finalize` (or `--wait`) is not optional.

```bash
# 1. Get the authorization URL, open it, authorize the account
devic integrations connect gmail

# 2a. Build the tool server once authorized (required — this is what activates it)
devic integrations connect gmail --finalize --name "Gmail" --tools GMAIL_GET_PROFILE,GMAIL_SEND_EMAIL

# 2b. Or do both in one call — polls until the account is active, then builds it
devic integrations connect gmail --wait --tools GMAIL_GET_PROFILE
```

Use `--wait` when the same session drives the whole flow; use bare `connect`
then `--finalize` when someone else authorizes in the browser (open the URL, let
them authorize, then finalize). Either way, an account authorized in the browser
does **not** appear in `devic integrations connected` until its server is built —
an empty list right after authorizing means "not finalized yet", not a failure.

`--tools` limits which of the app's tools the server exposes; omit it to expose
all of them (a large prompt on every message — narrowing is recommended).

The resulting integration also shows up in `devic tool-servers list` as
`type=integration`, but manage it through `integrations` (below), not
`tool-servers` — that group is for MCP and custom servers. Its provider account
id never appears; it is resolved server-side.

> **Multiple accounts of the same app:** `--finalize` binds the most recently
> authorized account for that app. Choosing among several accounts of one app is
> not available from the CLI (the account ids are not exposed); connect and
> finalize one at a time.

### The integrations you have connected

```bash
devic integrations connected      # id, app, account state, exposed-tool count
```

This is where the **integration id** for the tools commands comes from — you do
not need `tool-servers list`.

### Changing which tools an integration exposes

You can widen or narrow the exposed tools after connecting, without
reconnecting. Empty selection means **all** of the app's tools.

```bash
devic integrations tools list <id>              # what it exposes now
devic integrations tools list <id> --available  # the app's whole catalogue, marking enabled

devic integrations tools enable <id> GMAIL_SEND_EMAIL GMAIL_CREATE_EMAIL_DRAFT
devic integrations tools disable <id> GMAIL_SEND_EMAIL
devic integrations tools enable <id> --all      # expose every tool the app has
```

`enable`/`disable` are incremental (they add to / remove from the current
selection). Two edges follow from "empty means all": `enable` on an integration
that already exposes all is a no-op, and a `disable` that would remove the last
tool is refused — keep at least one, or use `--all` to expose all on purpose.
Only the tool selection changes; the connected account is never touched.

Worth narrowing: a whole toolkit is dozens of tools whose schemas are re-sent on
every message (Gmail's ≈ 8.6k prompt tokens).

### Testing a tool before an agent depends on it

```bash
devic integrations tools test <id> GMAIL_GET_PROFILE                     # no arguments
echo '{"max_results":3}' | devic integrations tools test <id> GMAIL_FETCH_EMAILS --from-json -
```

It runs the tool once through the same call path the engine uses, so what you
read back is what the agent would receive: the app is resolved the same way, the
same connected account is used, and a broken connection surfaces the same
message. It answers the two questions worth asking before wiring an agent — *is
this account really working*, and *what shape does this tool return* — without
creating an agent to find out.

> **It is a real call on real data.** There is no sandbox: `GMAIL_SEND_EMAIL`
> sends the email, `GITHUB_CREATE_AN_ISSUE` opens the issue. Composio slugs say
> what they do, so pick the read-only ones (`*_GET_*`, `*_LIST_*`, `*_FETCH_*`,
> `*_SEARCH_*`) to validate a connection, and test a writing tool only when the
> user has asked for it, on disposable data, after telling them what will appear
> in their account.

Reading the outcome:

| What you get | What it means |
|---|---|
| `success: true` with `data` | the tool works for this account; the payload is what lands in the model's context |
| `success: false` with an app error | the connection is fine, the **arguments** are not — the app's own message names the offending field |
| `success: false` + *"the connection itself is the problem"* | the account is expired or revoked. Reconnect it (`devic integrations connect <app>`); nothing about the tool is wrong |
| `400` *"does not expose it"* | the tool exists in the app but this integration does not expose it — `tools enable` it first, or test one it does expose |
| `404` *"No tool …"* | the slug does not exist in the app. Find the real one with `tools list <id> --available` |
| `400` *"is not an app integration"* | that id is an MCP or custom tool server. Its tools are tested with `devic tool-servers tools test` |

---

## Triggers

```bash
# List (filter by tool server or target)
devic triggers list
devic triggers list --tool-server <serverId>
devic triggers list --agent <agentId>
devic triggers list --assistant <assistantId>

# One trigger, with its message template and config
devic triggers get <triggerId>
```

### Creating a trigger

Provide the integration tool server, exactly one target (`--agent` **or**
`--assistant`), and the trigger slug from `integrations triggers <app>`:

```bash
devic triggers create \
  --tool-server <serverId> \
  --agent <agentId> \
  --trigger GMAIL_NEW_GMAIL_MESSAGE \
  --name "New email → triage agent" \
  --message "New email from {{data.from}}: {{data.subject}}"

# The trigger type's own config goes through --from-json:
devic integrations triggers gmail GMAIL_NEW_GMAIL_MESSAGE   # inspect the config schema
echo '{ "labelIds": "INBOX", "interval": 2 }' | devic triggers create \
  --tool-server <serverId> --assistant <identifier> \
  --trigger GMAIL_NEW_GMAIL_MESSAGE --from-json -
```

Flags:

| Flag | Meaning |
|------|---------|
| `--tool-server <id>` | The integration tool server (required) |
| `--agent <id>` / `--assistant <id>` | The target — exactly one |
| `--trigger <slug>` | Trigger type slug (required) |
| `--message <template>` | Renders the event into the message. Empty ⇒ raw event JSON |
| `--chat-uid-template <t>` | Assistants: a stable chat id so related events share one conversation |
| `--state queued\|paused` | Agents: initial thread state. `paused` lands the thread without spending tokens |
| `--rate-limit <n>` | Per-trigger events/minute ceiling |
| `--from-json <file>` | The trigger type's `config` (`-` for stdin) |

A create fails with a clear error if the target agent or assistant does not
exist, or if the tool server is not an app integration.

### Updating, deleting, and the delivery log

```bash
devic triggers update <id> --name "Renamed" --disabled
devic triggers update <id> --message "New template" --rate-limit 10
devic triggers update <id> --enabled

devic triggers delete <id>          # also removes the provider instance if unshared

# Recent deliveries, including the ones that did not run
devic triggers events <id>
devic triggers events <id> --limit 50
```

Event outcomes: `executed`, `duplicate` (retried delivery), `rate_limited`,
`disabled`, `error`.
