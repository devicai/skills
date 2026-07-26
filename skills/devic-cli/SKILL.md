---
name: devic-cli
description: "@devicai/cli reference — the Devic AI Platform CLI. Use when executing Devic API operations from the command line, scripting automations, building agent workflows that interact with assistants, agents, tool servers, documents and feedback, or creating, browsing and installing Devic skills (including into local coding agents: claude-code, codex, cursor, opencode, cline)."
---

# @devicai/cli

CLI for the Devic AI Platform API. Agent-first — JSON output by default when piped, human-readable in terminal. Single runtime dependency (commander).

## Installation

```bash
npm install -g @devicai/cli
# or
npx @devicai/cli <command>
```

## Authentication

```bash
# Login with API key (validates with a test request, stores in ~/.config/devic/config.json)
devic auth login --api-key devic-xxx

# Check auth status
devic auth status

# Logout (removes stored credentials)
devic auth logout
```

Environment variables override stored config:

| Variable | Description |
|----------|-------------|
| `DEVIC_API_KEY` | API key |
| `DEVIC_BASE_URL` | API base URL (default: `https://api.devic.ai`) |

## Global Options

| Option | Description |
|--------|-------------|
| `-o, --output <format>` | Output format: `json` or `human`. Auto-detected: JSON when piped, human in TTY |
| `--base-url <url>` | API base URL. Overrides config and env var. Priority: `--base-url` flag > `DEVIC_BASE_URL` env > config file > `https://api.devic.ai` |
| `-V, --version` | Show version |
| `-h, --help` | Show help |

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Error |
| 2 | Authentication required |
| 3 | Poll timeout |

Errors are written to stderr. In JSON mode: `{"error":"...","code":"..."}`. In human mode: markdown-formatted error with bold label and inline code.

## Human Output Format

When using `-o human` (or in a TTY), all output is markdown-formatted:

- **Lists**: Rendered as markdown tables with `| Header | ... |` format
- **Details**: Properties with `**Bold Label:** value`, IDs in `` `inline code` ``
- **Conversations**: Messages labeled `**USER:**` / `**ASSISTANT:**` / `**TOOL:**`
- **Status indicators**: `[OK]` success, `[..]` in-progress, `[!!]` needs attention, `[XX]` failed
- **Actions**: Success messages prefixed with `[OK]`
- **Errors**: Written to stderr with `**Error:** message` format
- **Pagination**: Footer with `**Total:** N | **Offset:** N | ...`
- **Nested objects**: Rendered as fenced JSON code blocks
- **Thread tasks**: Displayed as `[x]`/`[ ]` checklists

This format is designed for both human reading and LLM/agent consumption.

## JSON Input

All create/update commands support `--from-json <file>` (or `-` for stdin) for complex payloads:

```bash
# From file
devic agents create --from-json agent-config.json

# From stdin
echo '{"name":"My Agent"}' | devic agents create --from-json -
```

## Pagination

All list commands support:

| Option | Default | Description |
|--------|---------|-------------|
| `--offset <n>` | 0 | Items to skip |
| `--limit <n>` | 10 | Max items (max 100) |

---

## Commands Reference

### devic assistants

Manage assistants and chat conversations.

#### devic assistants list

List all assistant specializations.

```bash
devic assistants list [--external] [--offset <n>] [--limit <n>]
```

| Option | Description |
|--------|-------------|
| `--external` | Only show externally accessible assistants |

#### devic assistants get

Get details of a specific assistant.

```bash
devic assistants get <identifier>
```

#### devic assistants create

Create a new assistant.

```bash
devic assistants create [--name <name>] [--description <desc>] [--from-json <file>]
```

| Option | Description |
|--------|-------------|
| `--name <name>` | Assistant name |
| `--description <desc>` | Assistant description |
| `--from-json <file>` | Read full assistant config from JSON file (- for stdin) |

The `--from-json` payload supports all assistant specialization fields: `name`, `description`, `presets`, `model`, `provider`, `imgUrl`, `state`, `availableToolsGroupsUids`, `enabledTools`, `accessConfiguration`, `widgetConfiguration`, `memoryDocuments`, `structuredOutput`, `guardrailsConfiguration`, `codeSnippetIds`, `availableSkillIds`, `subagentsIds`, `maxChatMessages`, `maxToolResponseInputTokens`, `contextManagement` (`{ fullContextTurnDepth?: number, alwaysIncludeUserMessages?: boolean }` — send only the most recent N turns in full and summarize older ones).

#### devic assistants update

Update an existing assistant (partial updates supported).

```bash
devic assistants update <identifier> [--name <name>] [--description <desc>] [--enabled-tools <a,b,c>] [--all-tools] [--from-json <file>]
```

Same options as `create`, plus the tool selection flags below. Only provided fields will be updated.

| Option | Description |
|--------|-------------|
| `--enabled-tools <a,b,c>` | Replace the enabled tool names. Use `""` to enable none |
| `--all-tools` | Enable every tool of the assigned tool groups |

`--enabled-tools` and `--all-tools` are mutually exclusive. Pass neither and the
current tool selection is left untouched — read it with `devic assistants get`,
which reports it as `all`, `none`, or the explicit list.

Requires CLI 0.14.0 or newer. Earlier versions could only change the selection
through `--from-json`.

#### devic assistants delete

Delete an assistant.

```bash
devic assistants delete <identifier>
```

#### devic assistants chat

Send a message to an assistant. Uses async mode with polling by default.

```bash
devic assistants chat <identifier> -m "message" [options]
```

| Option | Description |
|--------|-------------|
| `-m, --message <text>` | **Required.** Message to send |
| `--chat-uid <uid>` | Continue an existing conversation |
| `--provider <provider>` | LLM provider override (openai, anthropic, azure, google) |
| `--model <model>` | Model override |
| `--tags <tags>` | Comma-separated tags |
| `--wait` | Async mode + poll for result (default: true) |
| `--no-wait` | Synchronous mode — blocks until response |
| `--from-json <file>` | Read full ProcessMessageDto from file (- for stdin) |

When `--wait` is active, status updates are emitted during polling:

- **JSON mode** (`-o json`): NDJSON lines on stdout:
  ```jsonl
  {"type":"chat_status","chatUid":"...","status":"processing","timestamp":1234567890}
  {"type":"chat_status","chatUid":"...","status":"completed","timestamp":1234567891}
  ```
- **Human mode** (`-o human`): Readable status lines on stderr:
  ```
  [..] Chat `550e8400...` — **processing**
  [OK] Chat `550e8400...` — **completed**
  ```

Chat polling: 1s initial interval, 1.5x backoff, 10s max, 5min timeout.

Status values: `processing`, `completed`, `error`, `waiting_for_tool_response`, `handed_off`.

Status indicators: `[OK]` completed/active, `[..]` processing/queued, `[!!]` paused/waiting, `[XX]` failed/error.

#### devic assistants stop

Stop an in-progress async chat.

```bash
devic assistants stop <identifier> <chatUid>
```

#### devic assistants chats list

List chat histories for an assistant.

```bash
devic assistants chats list <identifier> [--omit-content] [--offset <n>] [--limit <n>]
```

#### devic assistants chats get

Get a specific chat history.

```bash
devic assistants chats get <identifier> <chatUid>
```

#### devic assistants chats search

Search chat histories across all assistants with filters.

```bash
devic assistants chats search [options]
```

| Option | Description |
|--------|-------------|
| `--assistant <identifier>` | Filter by assistant |
| `--tags <tags>` | Comma-separated tags |
| `--start-date <date>` | Start date (ISO string) |
| `--end-date <date>` | End date (ISO string) |
| `--omit-content` | Exclude chat content |
| `--from-json <file>` | Read filters from file |

---

### devic agents

Manage agents, execution threads, and costs.

#### devic agents list

```bash
devic agents list [--archived] [--offset <n>] [--limit <n>]
```

#### devic agents get

```bash
devic agents get <agentId>
```

#### devic agents create

```bash
devic agents create [--name <name>] [--description <desc>] [--from-json <file>]
```

The `--from-json` payload supports all agent fields: `name`, `description`, `assistantSpecialization` (with `presets`, `availableToolsGroupsUids`, `enabledTools`, `model`, `provider`, `subagentsIds`, `contextManagement`), `provider`, `llm`, `maxExecutionInputTokens`, `maxExecutionToolCalls`, `evaluationConfig`, `subAgentConfig`.

#### devic agents update

```bash
devic agents update <agentId> [--name <name>] [--description <desc>] [--from-json <file>]
```

#### devic agents delete

```bash
devic agents delete <agentId>
```

---

### devic agents threads

Manage agent execution threads.

#### devic agents threads create

Create and optionally poll a new thread.

```bash
devic agents threads create <agentId> -m "task" [options]
```

| Option | Description |
|--------|-------------|
| `-m, --message <text>` | **Required.** Initial message/task |
| `--tags <tags>` | Comma-separated tags |
| `--wait` | Poll until terminal state |
| `--from-json <file>` | Read thread config from file |

When `--wait` is active, status updates are emitted during polling:

- **JSON mode** (`-o json`): NDJSON lines on stdout:
  ```jsonl
  {"type":"thread_status","threadId":"...","state":"processing","tasks":[...],"timestamp":1234567890}
  {"type":"thread_status","threadId":"...","state":"completed","tasks":[...],"timestamp":1234567891}
  ```
- **Human mode** (`-o human`): Readable status lines on stderr:
  ```
  [..] Thread `thread-456` — **processing** (tasks: 1/3)
  [OK] Thread `thread-456` — **completed** (tasks: 3/3)
  ```

Thread polling: 2s initial interval, 1.5x backoff, 15s max, 10min timeout.

Terminal states: `completed`, `failed`, `terminated`.

Actionable states (returned to caller): `paused_for_approval`.

#### devic agents threads list

```bash
devic agents threads list <agentId> [options]
```

| Option | Description |
|--------|-------------|
| `--state <state>` | Filter by state |
| `--start-date <date>` | Start date filter |
| `--end-date <date>` | End date filter |
| `--date-order <order>` | Sort: `asc` or `desc` |
| `--tags <tags>` | Comma-separated tags |
| `--omit-content` | Exclude thread content from response (returns metadata and state only). Significantly reduces payload size for large thread lists |

#### devic agents threads get

```bash
devic agents threads get <threadId> [--with-tasks] [--grep <pattern>]
```

| Option | Description |
|--------|-------------|
| `--with-tasks` | Include task details |
| `--grep <pattern>` | Filter thread content to only show messages matching the pattern (case-insensitive). Useful for finding specific data within large threads without scanning the full content manually |

#### devic agents threads approve

```bash
devic agents threads approve <threadId> [-m "message"]
```

#### devic agents threads reject

```bash
devic agents threads reject <threadId> [-m "message"]
```

#### devic agents threads pause

```bash
devic agents threads pause <threadId>
```

#### devic agents threads resume

```bash
devic agents threads resume <threadId>
```

#### devic agents threads complete

Manually set a thread's final state.

```bash
devic agents threads complete <threadId> --state <COMPLETED|FAILED|CANCELLED|TERMINATED>
```

#### devic agents threads evaluate

Trigger AI evaluation of a completed thread.

```bash
devic agents threads evaluate <threadId>
```

---

### devic agents costs

Track agent execution costs.

#### devic agents costs daily

```bash
devic agents costs daily <agentId> [--start-date YYYY-MM-DD] [--end-date YYYY-MM-DD]
```

#### devic agents costs monthly

```bash
devic agents costs monthly <agentId> [--start-month YYYY-MM] [--end-month YYYY-MM]
```

#### devic agents costs summary

Get today's and current month's cost summary.

```bash
devic agents costs summary <agentId>
```

---

### devic tool-servers

Manage tool servers, their definitions, and individual tools.

Three kinds of tool server, told apart by the `type` column: `http` (tool
definitions you wrote, called against a base URL), `mcp` (a remote MCP server),
and `integration` (an app connected in Devic — Gmail, Drive, HubSpot). An
integration has no URL and no stored definition; its tools live in the connected
app, and the `target` column names that app, with `(not connected)` when no
account is linked.

Credentials come back masked (`••••••••`). Sending a masked value back in an
update keeps the stored secret, so `get` → edit → `update` is safe; send a real
value to change it.

#### devic tool-servers list

```bash
devic tool-servers list [--offset <n>] [--limit <n>] [--project <project>]
```

`--project` accepts an `_id`, an identifier, or a name.

#### devic tool-servers get

```bash
devic tool-servers get <toolServerId>
```

#### devic tool-servers create

```bash
devic tool-servers create [--name <name>] [--url <url>] [--description <desc>] [--from-json <file>]
```

The `--from-json` payload supports: `name`, `description`, `url`, `identifier`, `enabled`, `mcpType`, `toolDefinitions`, `authenticationConfig`, `imageUrl`.

`url`, each tool's `endpoint` and the advanced body template accept `{{...}}`
template references resolved at call time, so one tool server can target a
different upstream per environment instead of being cloned:

- `{{metadata.<field>}}` — thread or chat metadata (bare `{{field}}` is the same thing)
- `{{env.<VAR>}}` — env var of the Environment connected to the agent or assistant
- `{{fields.<apiName>}}` — published-MCP connection field, resolved by the wrapper only

Unknown references resolve to an empty string. Do not confuse these with the
`${arg}` / `{arg}` placeholders in `endpoint`, which are tool arguments the model
supplies.

```bash
devic tool-servers create --name "Billing API" --url 'https://{{env.BILLING_HOST}}'
```

#### devic tool-servers update

```bash
devic tool-servers update <toolServerId> [--name <name>] [--url <url>] [--description <desc>] [--enabled <bool>] [--from-json <file>]
```

#### devic tool-servers delete

```bash
devic tool-servers delete <toolServerId>
```

#### devic tool-servers clone

```bash
devic tool-servers clone <toolServerId>
```

#### devic tool-servers definition

Get the full tool server definition.

```bash
devic tool-servers definition <toolServerId>
```

#### devic tool-servers update-definition

```bash
devic tool-servers update-definition <toolServerId> --from-json <file>
```

---

### devic tool-servers tools

Manage individual tools within a tool server.

#### devic tool-servers tools list

```bash
devic tool-servers tools list <toolServerId> [--available] [--limit <n>] [--cursor <cursor>]
```

Lists the tools the server exposes. On an integration, `--available` browses
everything the connected app offers instead, with an `enabled` column marking
which of them this server uses. That catalogue is paged — follow the `--cursor`
the output suggests rather than counting against the total, which includes tools
the app has deprecated and are not listed.

#### devic tool-servers tools get

```bash
devic tool-servers tools get <toolServerId> <toolName>
```

#### devic tool-servers tools add

```bash
devic tool-servers tools add <toolServerId> --from-json <file>
```

JSON structure:

```json
{
  "type": "function",
  "function": {
    "name": "tool_name",
    "description": "What it does",
    "parameters": {
      "type": "object",
      "properties": { "param": { "type": "string" } },
      "required": ["param"]
    }
  },
  "endpoint": "/api/path/${param}",
  "method": "GET",
  "pathParametersKeys": ["param"]
}
```

#### devic tool-servers tools update

```bash
devic tool-servers tools update <toolServerId> <toolName> --from-json <file>
```

#### devic tool-servers tools delete

```bash
devic tool-servers tools delete <toolServerId> <toolName>
```

#### devic tool-servers tools test

Test a tool call with parameters.

```bash
devic tool-servers tools test <toolServerId> <toolName> --from-json <file>
```

The JSON file should contain the parameters object: `{"city": "London"}`.

---

### devic integrations / devic triggers

Browse the catalogue of connectable apps (Gmail, Drive, HubSpot, …) and the
events each emits, connect an account, and manage the triggers that start an
agent or assistant from those events.

```bash
devic integrations list --search gmail
devic integrations triggers gmail                 # an app's event types
devic integrations connect gmail --wait           # connect + build the tool server
devic triggers create --tool-server <id> --agent <id> --trigger GMAIL_NEW_GMAIL_MESSAGE
devic triggers events <id>
```

For detailed documentation, see [integrations-and-triggers.md](integrations-and-triggers.md).

---

### devic documents

Manage knowledge documents (markdown). List, read, create, update and manage versions.

```bash
devic documents list [--project <p>] [--folder <id>] [--file-type md]
devic documents get <documentId>
devic documents create --name <name> [content source] [--folder <id>] [--tags <t...>] [--as-skill]
devic documents update <documentId> [fields...] [--tags <t...>] [--as-skill|--no-skill]
devic documents versions list <documentId>
devic documents versions revert <documentId> <version>
devic documents attach <documentId> --target-type agent|assistant|environment --target-id <id>
devic documents usage <documentId>
devic documents folders create --name <name> [--as-skill] [--tags <t...>]
```

`--folder` on `create`, `--tags` and `--as-skill`/`--no-skill` require **CLI ≥ 0.18.0**.
`--as-skill` publishes the document in the skills catalog; its name and
description then come from the YAML frontmatter at the top of the markdown.

#### Providing document content (create / update)

The markdown body is **never read from a shell redirect by itself**. You must
pass it through one of these content sources:

| Source | Example |
|--------|---------|
| Inline | `devic documents update <id> --content "# Title\n..."` |
| From a file | `devic documents update <id> --from-file SKILL.md` |
| From stdin (pipe) | `cat SKILL.md \| devic documents update <id> --from-stdin` |

> ⚠️ **Do NOT use** `devic documents update <id> < file.md` **on its own.** A bare
> shell redirect is only picked up when the process detects a piped stdin; the
> reliable forms are `--from-file <path>` or a real pipe with `--from-stdin`. To
> avoid silent mistakes, `update` now **refuses an empty payload** (it errors out
> instead of returning a misleading success), and the response includes
> `versionCreated: true|false` so you can confirm a new version was actually
> written. Always check that flag after updating.

`update` only creates a new version when the content actually changes; updating
with identical content (or only metadata like `--name`/`--folder`) reports
`versionCreated: false`.

---

### devic skills

Browse and install Devic **skills** (documents or folders flagged as skills, in the
SKILL.md format) into your local coding agents. Mirrors the `skills.sh` model:
each skill is written as a folder named after the skill into the agent's skills
directory, and an install registry (lockfile) tracks what is installed so it can
be refreshed with `update`.

#### devic skills list (alias: ls)

List the skills catalog.

```bash
devic skills list [--tag <tag...>] [--search <text>] [--project <projectId>] [--limit <n>] [--page <n>]
```

| Option | Description |
|--------|-------------|
| `--tag <tag...>` | Filter by tag (repeatable, e.g. `--tag cli --tag qa`) |
| `--search <text>` | Free-text search over name/description |
| `--project <projectId>` | Filter by project id |
| `--limit <n>` | Page size (max 200, default 100) — CLI ≥ 0.18.0 |
| `--page <n>` | 1-based page number — CLI ≥ 0.18.0 |

Human output shows a table with id, name, type (`document`/`folder`), tags, and
usage stats: linked **agents**, **assistants**, and **reads** (how often the skill
was consulted by agents via the knowledge tools).

#### devic skills create

Create a **folder-skill**: the folder plus its `SKILL.md` manifest, with the
`name`/`description` frontmatter already written. Requires **CLI ≥ 0.18.0**.

```bash
devic skills create <name> [-d <description>] [--tags <t...>] [--project <projectId>] [--parent <folderId>] [--from-file <path>]
```

| Option | Description |
|--------|-------------|
| `-d, --description <text>` | One-line description. It is what the model reads before deciding to load the skill, so phrase it as a trigger (*"How to … when …"*) |
| `--tags <tags...>` | Category tags |
| `--project <projectId>` | Scope the skill to a project |
| `--parent <folderId>` | Create it inside an existing folder |
| `--from-file <path>` | Replace the generated stub manifest with your own markdown |

The output prints the folder id — that is what you attach as
`knowledgeSkills: [{ id, type: "folder" }]` — and the manifest document id.

```bash
devic skills create "Incident triage" -d "How to triage a production incident." --tags ops
devic skills create "Release drill" --from-file ./SKILL.md
```

#### devic skills get

Show one catalog entry (by id or name), including its tags and usage counts.
Requires **CLI ≥ 0.18.0**.

```bash
devic skills get <id|name>
```

#### devic skills tree

Show the files a skill contains — what an install would download — **without**
recording an install. Requires **CLI ≥ 0.18.0**.

```bash
devic skills tree <id|name> [--out <dir>]
```

`--out <dir>` also writes the files to disk, preserving their relative paths.
Useful to inspect or diff a skill before installing it.

#### devic skills tags

List the distinct tags across skills (for `--tag` filtering).

```bash
devic skills tags [--project <projectId>]
```

#### devic skills install (alias: add)

Download a skill's whole tree into your coding agents. The skill is resolved by
**id or name**. Records the install locally (lockfile) and in Devic (install
counter + per-user install date).

```bash
devic skills install <id|name> [-a <agents...>] [-g]
```

| Option | Description |
|--------|-------------|
| `-a, --agent <agents...>` | Target agents: `claude-code`, `codex`, `cursor`, `opencode`, `cline`, or `*` for all. Auto-detected from installed agents when omitted (falls back to `claude-code`) |
| `-g, --global` | Install to the user-level (global) agent directories instead of the project ones |

Install locations (a folder named after the skill is created inside):

| Agent | Project path | Global path |
|-------|--------------|-------------|
| claude-code | `.claude/skills/` | `~/.claude/skills/` |
| codex | `.agents/skills/` | `~/.codex/skills/` |
| cursor | `.agents/skills/` | `~/.cursor/skills/` |
| opencode | `.agents/skills/` | `~/.config/opencode/skills/` |
| cline | `.agents/skills/` | `~/.agents/skills/` |

A folder-skill is written with its full tree (`SKILL.md` + referenced files, at
their relative paths). A document-skill is written as a single `SKILL.md`.

#### devic skills update

Refresh installed skills to their latest version. Compares the installed version
(from the lockfile) against Devic and only rewrites what changed.

```bash
devic skills update [id|name] [-g]
```

Omit the argument to update every installed skill; pass an id/name to update just
one. `-g` targets the global install registry.

#### devic skills installed

List what is installed locally (from the lockfile), with the installed version and
date.

```bash
devic skills installed [-g]
```

The lockfile lives at `.devic/skills.json` (project) or `~/.devic/skills.json`
(global).

---

### devic feedback

Submit and view feedback on chat messages and thread messages.

#### devic feedback submit-chat

```bash
devic feedback submit-chat <identifier> <chatUid> --message-id <id> [options]
```

| Option | Description |
|--------|-------------|
| `--message-id <id>` | **Required.** Message UID to give feedback on |
| `--positive` | Positive feedback (thumbs up) |
| `--negative` | Negative feedback (thumbs down) |
| `--comment <text>` | Feedback comment |
| `--from-json <file>` | Read full feedback payload from file |

#### devic feedback list-chat

```bash
devic feedback list-chat <identifier> <chatUid>
```

#### devic feedback submit-thread

```bash
devic feedback submit-thread <threadId> --message-id <id> [options]
```

Same options as `submit-chat`.

#### devic feedback list-thread

```bash
devic feedback list-thread <threadId>
```

---

## Usage Examples

### Send a message and get the result

```bash
# Simple chat
devic assistants chat default -m "What is the capital of France?"

# Continue a conversation
CHAT_UID=$(devic assistants chat default -m "Hello" -o json | jq -r '.chatUID')
devic assistants chat default -m "Tell me more" --chat-uid "$CHAT_UID"
```

### Create an agent and run a thread

```bash
# Create agent from JSON
cat <<'EOF' | devic agents create --from-json -
{
  "name": "Data Analyst",
  "description": "Analyzes data and creates reports",
  "assistantSpecialization": {
    "presets": "You are a data analyst. Analyze data and provide insights.",
    "model": "gpt-4o"
  }
}
EOF

# Run a thread and wait for completion
devic agents threads create <agentId> -m "Analyze Q4 sales data" --wait
```

### Handle thread approvals in a script

```bash
# Create thread
RESULT=$(devic agents threads create <agentId> -m "Delete old records" --wait -o json)
STATE=$(echo "$RESULT" | jq -r '.state')

if [ "$STATE" = "paused_for_approval" ]; then
  THREAD_ID=$(echo "$RESULT" | jq -r '._id')
  devic agents threads approve "$THREAD_ID" -m "Approved"
fi
```

### Set up a tool server with tools

```bash
# Create tool server
cat <<'EOF' | devic tool-servers create --from-json -
{
  "name": "Weather API",
  "url": "https://api.weather.example.com",
  "toolDefinitions": [{
    "type": "function",
    "function": {
      "name": "get_weather",
      "description": "Get weather for a city",
      "parameters": {
        "type": "object",
        "properties": { "city": { "type": "string" } },
        "required": ["city"]
      }
    },
    "endpoint": "/weather/${city}",
    "method": "GET",
    "pathParametersKeys": ["city"]
  }]
}
EOF

# Test the tool
echo '{"city":"London"}' | devic tool-servers tools test <serverId> get_weather --from-json -
```

### Search for specific data in agent threads

```bash
# List threads without content (fast, metadata only)
devic agents threads list <agentId> --omit-content --limit 50

# Find a specific email within a thread's content
devic agents threads get <threadId> --grep "user@example.com"

# Combine: list threads, then search each for a keyword
devic agents threads list <agentId> --omit-content -o json | \
  jq -r '.[].threadId' | \
  while read tid; do
    RESULT=$(devic agents threads get "$tid" --grep "target@email.com" -o json)
    COUNT=$(echo "$RESULT" | jq '.threadContent | length')
    if [ "$COUNT" -gt "0" ]; then echo "Found in thread: $tid"; fi
  done
```

### Pipe JSON between commands

```bash
# Get all completed threads and their evaluations
devic agents threads list <agentId> --state COMPLETED -o json | \
  jq -r '.[].threadId' | \
  while read tid; do devic agents threads evaluate "$tid"; done
```
