# Workspaces

`devic workspaces` (alias `ws`) drives Workspaces from the command line: shared
spaces where people and assistants work together in threads. It wraps the public
API under `/api/v1/workspaces` (contract in the devic-api skill, `workspaces.md`).
For task-oriented guidance (who to act as, finding unread work, answering and
approving), see the `devic-workspaces` skill.

Ids are positional and always in the order **workspace → thread → invocation /
file**: `<wid> <tid> ...`.

---

## Identity: who you act as

The API acts as the token's person. With an **API key**, choose a person of the
account with `--as <userUID|email>` (header `X-Devic-User`) or the `DEVIC_USER`
environment variable; that person's permissions apply. Without either, an API key
acts as the account's "API" integration and only sees shared workspaces.

```bash
devic ws me                                   # principal: person | integration
devic ws --as ana@acme.com me
DEVIC_USER=ana@acme.com devic ws threads list --unread
```

`--as` can go anywhere on the line and wins over `DEVIC_USER`. The header is only
sent to the Workspaces API.

---

## Command tree

```
devic ws me
devic ws list [--search] [--archived] [--summary] [--offset] [--limit]
devic ws get <wid> | catalog
devic ws create  [--data <json|file|->] [--name] [--description] [--visibility]
devic ws update <wid> [--data ...] [--name] [--description] [--visibility] [--archive|--unarchive] [--revision]

devic ws threads list [--workspace <wid>] [--kind direct|shared] [--search] [--unread] [--archived] [--offset] [--limit]
devic ws threads get <wid> <tid> [--limit] [--before <seq>]
devic ws threads messages <wid> <tid> [--before <seq> | --after <seq>] [--limit]
devic ws threads create <wid> [--title] [--direct --user <uid>]   # --direct: the account-wide conversation

devic ws dm list                                 # private conversations: {workspaceId, items}
devic ws dm open <userUID|email>                 # the conversation with that person (same one if it exists)
devic ws dm send <userUID|email> <text|->        # opens it if needed, then posts
devic ws threads rename <wid> <tid> <title>
devic ws threads archive|unarchive <wid> <tid>
devic ws threads read <wid> <tid> [--through <seq>]
devic ws threads unread <wid> <tid>

devic ws send <wid> <tid> <text|-> [--assistant <aid>] [--mention kind:id[:label]|everyone]... [--everybody]
                                   [--reply-to <mid>] [--client-id <id>]
                                   [--wait [--timeout <s>] [--poll-interval <s>]]
devic ws answer <wid> <tid> <invocationId> (--answers <json|file|-> | --skip)
devic ws approve <wid> <tid> <invocationId> [--always]  # waiting_approval: revision and actionId fetched
devic ws run-action <wid> <tid> <invocationId> <action> [--data <json|file|->]

devic ws files list <wid> <tid>
devic ws files download <wid> <tid> <fileId> [--out <file|dir/|->] [--force]
devic ws documents <wid> [--search] [--extension] [--type] [--creator] [--thread] [--offset] [--limit]

devic ws members list <wid>
devic ws members set <wid> <userUID> --role owner|manager|member|viewer [--revision]
devic ws members remove <wid> <userUID>

devic ws assistants list|get|add|update|remove ...      # add/update take --data
devic ws resources get <wid> | set <wid> --data <json> [--revision]
devic ws knowledge catalog <wid> [--search]
devic ws skills catalog <wid> [--search] | list <wid>
devic ws skills attach|detach <wid> <skillRef...>

devic ws memory list <wid> <aid>
devic ws memory add <wid> <aid> --section persona|instructions|decisions|profile --content <text|-> [--pinned]
devic ws memory update <wid> <aid> <entryId> [--section] [--content|--file] [--pinned|--unpinned]
devic ws memory remove <wid> <aid> <entryId>

devic ws rules get <wid> [--thread <tid>]
devic ws rules set <wid> [--thread <tid>] (--content <text|-> | --file <path>) [--expected-revision <n>]
devic ws rules history <wid> [--thread <tid>] [--before <cursor>]

devic ws automations list|get|update|delete <wid> ...
devic ws automations create <wid> [--name] [--source-thread <tid>]       # empty draft
devic ws automations capabilities <wid>                                   # sources + included model
devic ws automations destinations <wid>
devic ws automations destination-assistants <wid> <target>
devic ws automations find-threads <wid> <target> [--query] [--mode keywords|literal|regex|list] [--search-in titles|messages|all] [--match all|any] [--limit] [--before]
devic ws automations read-thread <wid> <target> <tid> [--query] [--limit] [--before-seq] [--after-seq]
devic ws automations connections <wid>
devic ws automations connection-tools <wid> <connectionId>
devic ws automations triggers <wid> --integration <app> [--query] [--cursor]
devic ws automations trigger <wid> <slug>
devic ws automations schedule-preview <wid> --data <json>
devic ws automations draft <wid> <automationId> --data <plan|file|-> [--draft-version <n>]   # never activates
devic ws automations activate <wid> <automationId> (--included | --provider <p> --model <m>) [--draft-version <n>] [--max-input-tokens <n>]
devic ws automations pause|invoke|test <wid> <automationId> [--data]

devic ws integrations search <wid> <query> [--semantic]
devic ws connections list <wid>
devic ws connections connect <wid> <integration> --scope workspace|user [--name] [--auth-scheme]   # prints the authorization link
devic ws connections reconnect <wid> <connectionId>
devic ws connections enable|disable <wid> <connectionId>
devic ws automations runs <wid> [--automation <id>] [--status] [--query k=v]...
devic ws automations run <wid> <runId> [--summary] | rerun <wid> <runId>
```

---

## Reading threads

- `threads list` without `--workspace` covers **every workspace** you see.
  `--search` matches the thread title or the workspace name (all words,
  accent/case-insensitive). Rows carry `unreadCount`, `unreadMentionCount`,
  `waitingForMe` and `activeRunCount`.
- `threads get` returns the thread, its `readState`, the latest messages and the
  open `invocations`. Human output is a transcript: author, `#seq`, and a
  `via CLI` / `via API` / `via SDK` tag on messages posted through the public
  API; invocations in `waiting_*` are listed with their questions and the exact
  `answer` command to run.
- `threads messages` pages by `seq`: `--before` for older, `--after` for newer
  (limit ≤ 200, default 50).
- Reading does not mark anything read — call `threads read` (optionally
  `--through <seq>`).

## Sending and waiting

`send` posts a message (text argument, `-` or piped stdin). The response holds
the `message` and the `invocations` it started. With `--wait` the CLI polls
`messages?after=<seq of your message>` until every started invocation is
terminal (`completed`, `failed`, `cancelled`, `outcome_unknown`) or waiting
(`waiting_*`), then prints `{message, outcome, invocations, messages}` where
`outcome` is `completed`, `failed`, `waiting` or `no_invocations`. Poll timeout
(`--timeout`, default 300 s) exits **3**. In JSON mode, status changes stream as
NDJSON lines `{"type":"workspace_invocation",...}` before the final document.

`--client-id` sets `clientMessageId`, so retrying the same send does not post
twice.

**Direct messages** are account-wide (not inside a workspace): `dm list` gives the
hidden space's `workspaceId`, and every thread command works on them with it. They
need a person: an API key without `--as` gets `DIRECT_MESSAGES_REQUIRE_USER`.

**`--everybody`** mentions everyone who can read the thread (adds `@everybody` to the
text if missing). Not for guests (`EVERYONE_MENTION_NOT_ALLOWED`); once every 10
minutes per author and thread (`EVERYONE_MENTION_COOLDOWN`, the message is not posted).

## Answering questions

When an assistant asks something (`status: waiting_input`, `requestedInput`),
`answer` reads the thread to fill `revision` and `inputId` for you:

```bash
devic ws answer <wid> <tid> <rid> --answers '[{"id":"q1","selected":["eur"]},{"id":"q2","other":"Q4 only"}]'
devic ws answer <wid> <tid> <rid> --skip
```

Pass `--revision` and `--input-id` to skip the lookup.

## Approving tool calls

When a run is `waiting_approval` (for example before using the terminal),
`approve` fills `revision` and `actionId` (`requestedAction.id`) the same way:

```bash
devic ws approve <wid> <tid> <rid>            # --always: allow that tool from now on
```

Pass `--revision` and `--action-id` to skip the lookup. Any other invocation
action (`cancel`, `retry`, ...) goes through
`run-action <wid> <tid> <rid> <action> [--data]`.

## Files

`files download` saves under the server's filename (from `Content-Disposition`)
in the current directory, or under `--out` (a file path, an existing directory or
one ending in `/`, or `-` for stdout). It **never overwrites** an existing file
without `--force` (error code `FILE_EXISTS`).

## Concurrency (revisions)

- `rules set` sends the current revision as `expectedRevision` unless you give
  `--expected-revision`.
- `resources set` fills `revision` from the current resources when the body has
  none.
- `skills attach|detach` take `<id>` / `document:<id>` (skill document) or
  `folder:<id>` (skill folder), read `resources.knowledgeSkills`, add or remove
  the `{type, id}` links and write back `{knowledgeSkills, revision}`.
- `assistants remove` uses `--revision`, else the assistant's `revision`, else
  the workspace's.

A `409` means someone changed it in between: read again and retry.

---

## Examples

```bash
# What needs me, across every workspace
devic ws --as ana@acme.com threads list --unread -o human

# Ask an assistant in a thread and wait for its reply
devic ws send <wid> <tid> "Summarise this week's tickets" --assistant <aid> --wait --timeout 600

# It asked a question: see it, then answer it
devic ws threads get <wid> <tid> -o human
devic ws answer <wid> <tid> <rid> --answers '[{"id":"q1","selected":["eur"]}]'

# Pull every PDF a thread produced
devic ws files list <wid> <tid> | jq -r '.items[] | select(.mimeType=="application/pdf") | .id' |
  xargs -I{} devic ws files download <wid> <tid> {} --out ./pdfs/

# Standing rules and memory
devic ws rules set <wid> --file rules.md
devic ws memory add <wid> <aid> --section decisions --content "Prices are quoted in EUR" --pinned
```
