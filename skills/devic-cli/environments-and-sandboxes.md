# Environments & Sandboxes

Two command groups for the machine an agent works on:

- **`environments`** (alias `envs`) — the reusable package: the sandbox
  configuration, its snapshot, the knowledge and tools connected entities
  inherit, and the encrypted variables they can read.
- **`sandbox`** — a real Linux machine started on an environment, to run
  commands and files on.

One environment serves many agents and assistants, which is what lets a team of
agents share one machine state.

**Every command accepts the environment by `_id` or by name.** Names are matched
case-insensitively, and an unknown one lists what exists.

---

## Environments

### CRUD

```bash
devic environments list [--project <project>] [--offset <n>] [--limit <n>]
devic environments get <environment>

devic environments create --name <name> [--description <desc>] [--project <project>]
                          [--runtime node24|node22|python3.13] [--memory <mib>]
                          [--init-script <script> | --init-script-file <file>]
                          [--env KEY=VALUE ...]
                          [--snapshots] [--evolving-snapshot | --fixed-snapshot]
                          [--per-tenant-snapshots] [--auto-extend] [--persist]
                          [--public-slug <slug>] [--start-command <cmd>]
                          [--from-json <file>]

devic environments update <environment> [the same flags, each with a --no- form]
                                        [--unset-env KEY ...]
devic environments delete <environment>
```

| Flag | What it does |
| --- | --- |
| `--runtime` | Base image. `node24` (default), `node22`, `python3.13` |
| `--memory` | RAM in MiB. Baked into the snapshot and inherited on every restore |
| `--init-script` / `--init-script-file` | Shell script run when a **new** machine is created. Skipped on a restore — by then the work is already done |
| `--auto-extend` | An operation arriving near the deadline buys another full timeout. Idle sandboxes still expire |
| `--persist` | Keep the machine alive after the session closes |
| `--public-slug` | Publish the snapshot at `<slug>.sandbox.devic.ai` |
| `--start-command` | Run after each restore to bring the published service back up |

`--init-script` prepares a machine and runs when one is created;
`--start-command` serves the snapshot and runs on every restore. Restoring brings
back a filesystem, not a process someone started by hand — which is why both
exist.

### Variables

Values are encrypted at rest and **always read back masked**:

```bash
devic environments update "Reporting box" --env "REGION=eu-west-1"
devic environments update "Reporting box" --unset-env REGION
```

`--env` **merges over the stored map** rather than replacing it, so the secrets
you do not name survive untouched. This matters: the API replaces the map
wholesale, so a hand-written `PATCH` that sends only the new variable deletes
every other one. The CLI reads first and lets the backend recognise its own mask
as *keep the stored secret*.

This is where a database password or an SSH key belongs — not in a thread
message, which is stored and readable back over the API indefinitely.

Three layers, most specific wins: **connection → environment → sandbox**. Put
shared secrets on the environment; use `connect --env` only for values that
genuinely differ per agent.

### Snapshot modes

`--snapshots` turns saving on. Whether a session may *replace* what is saved is a
separate question, and the answer is **no unless you ask**:

| Mode | Flag | What sessions may do |
| --- | --- | --- |
| **Fixed** (default) | `--fixed-snapshot`, or just `--snapshots` | Start from the saved state; cannot write back |
| **Evolving** | `--evolving-snapshot` | Every session's changes replace the snapshot |

Fixed is the useful default: it is what stops a stray agent run from destroying a
machine you spent time provisioning. Switch either way later:

```bash
devic environments update "Build box" --evolving-snapshot
devic environments update "Build box" --fixed-snapshot
```

### Connections

```bash
devic environments connections <environment>
devic environments connect <environment> agent|assistant <entityId> [--env KEY=VALUE ...]
devic environments disconnect <environment> agent|assistant <entityId>
```

Agents can also be wired at creation: `devic agents create --environment "Build box"`
(and `--environment null` disconnects). Assistants use the connections command.

The connection is **additive** for tools, knowledge and skills — what the
environment brings is added to what the entity had. The sandbox is the
exception: an environment with a sandbox **governs it completely**, and the
entity's own terminal configuration is ignored at runtime.

### Snapshots

```bash
devic environments snapshot init <environment>                   # bake or re-bake the base
devic environments snapshot tenants <environment>                # per-tenant snapshots
devic environments snapshot init-tenant <environment> <tenantId> # discards that tenant's state
devic environments snapshot delete-tenant <environment> <tenantId>
```

`snapshot init` creates a machine, runs the init script **and the configured
CLIs**, and saves the result. It needs snapshots enabled, and **re-baking
discards the state the snapshot had accumulated** — that is the point: it is how
a machine is brought back in line with a changed init script.

With `--per-tenant-snapshots`, each customer's first session derives from the
shared base and keeps its own from then on. Sessions with no tenant keep using
the shared one.

### Sessions and tooling

```bash
devic environments sessions list <environment>
devic environments sessions get <environment> <sessionId>
devic environments clis <environment>
```

---

## Sandboxes

```bash
devic sandbox start <environment> [--timeout <minutes>] [--tenant <tenantId>]
                                  [--force] [--force-unsaved-snapshot]
devic sandbox status <environment>
devic sandbox exec <environment> <command> [--sandbox <id>] [--cwd <path>] [--sudo]
devic sandbox stop <environment> [--sandbox <id>] [--save | --no-save] [--force]

devic sandbox ls <environment> [path] [--sandbox <id>]
devic sandbox cat <environment> <path> [--sandbox <id>]
devic sandbox write <environment> <path> (--content <text> | --file <local> | --url <src>)
```

After `start`, the other commands find the live session on their own — there is
no sandbox id to carry between calls. `--sandbox` overrides that.

### Starting

`--timeout` is 1–30 minutes (default 10). With `--auto-extend` on the environment
a busy sandbox renews itself; an idle one still expires.

A fresh machine runs the init script and its **output comes back in the
response** — exit code, stdout, stderr. That is the only way to see whether a
provisioning script actually works, and the reason to start one by hand.

**One session per environment.** A second concurrent `start` answers
`SESSION_ACTIVE`. `--force` takes over and **discards the other session's unsaved
state** — it is not a retry. With per-tenant snapshots the contended resource is
the tenant's own snapshot, so `--tenant` lets two be worked on side by side.

`SNAPSHOT_SAVE_IN_PROGRESS` means a save is still being written;
`--force-unsaved-snapshot` starts from the last saved version, which will be
missing whatever that save is committing.

### Running commands

**`exec` takes a line of shell, not argv.** Quote it: pipes, `&&`, `cd` and
redirection belong inside the command.

```bash
devic sandbox exec "Reporting box" 'cd /workspace && ./report.sh | tail -20'
```

The environment's variables are injected, so a script reads `$DATABASE_URL` the
way it would anywhere else. `cwd` is tracked across calls, so a `cd` persists.

A non-zero exit code is reported in the output, not raised as a CLI error — the
command ran, it just failed.

### Stopping, and whether it saves

It depends on the environment, and it matches what the dashboard terminal does:

| Environment | `stop` with nothing said |
| --- | --- |
| Evolving snapshot | Saves (`--no-save` discards) |
| Fixed snapshot | **Does not save** — pass `--save` to bake |
| Snapshots off | Nothing to save either way |

Saying nothing never overwrites a snapshot someone froze on purpose. `--save` is
how you provision one, and it is deliberately something you have to type — a
manual session is the only thing allowed to write to a fixed snapshot.

The save is **asynchronous**: `stop` returns straight away with `saving: true`
and the snapshot id, and the environment's `snapshotId` picks that id up once the
capture finishes. Starting again before then answers `SNAPSHOT_SAVE_IN_PROGRESS`.

### Files

```bash
devic sandbox ls    "Build box" /workspace
devic sandbox cat   "Build box" /workspace/report.log
devic sandbox write "Build box" /workspace/report.py --file ./report.py
```

`cat` is text-only and capped at 1 MB: binary files and anything larger are
refused rather than truncated. `write` takes `--content`, `--file` (read from
your machine) or `--url` (the sandbox fetches the bytes itself).

---

## Provisioning a machine: bake once, then leave it alone

The workflow the fixed default exists for. Install what the agent will need,
save it, and let every later run start from there without being able to break it.

```bash
# 1. An environment with snapshots — fixed, which is the default.
devic environments create --name "Build box" --runtime node24 --snapshots --auto-extend \
  --env "DATABASE_URL=postgres://..."

# 2. A machine to work on. No snapshot yet, so this one is fresh.
devic sandbox start "Build box" --timeout 20

# 3. Provision it, and verify it before freezing a broken machine.
devic sandbox exec  "Build box" 'cd /workspace && npm install express pg'
devic sandbox write "Build box" /workspace/report.py --file ./report.py
devic sandbox exec  "Build box" 'cd /workspace && python report.py --dry-run'

# 4. Bake it. --save is required because the environment is fixed.
devic sandbox stop "Build box" --save

# 5. Confirm the bake landed (the save is async — snapshotId appears when done).
devic environments get "Build box"

# 6. Every later session, yours or an agent's, starts with all of it in place.
devic agents create --name "Reporter" --environment "Build box" --cron "0 7 * * 1-5"
```

Once you know what provisioning takes, write it into the init script so the
machine can be rebuilt without you:

```bash
devic environments update "Build box" --init-script-file ./provision.sh
devic environments snapshot init "Build box"
```

Use the manual session while you are still working out what the script should
say; use `snapshot init` once it is written down.

---

## Cleaning up

Stop the session before deleting the environment — deleting with one live leaves
a sandbox running until its TTL.

```bash
devic sandbox stop "ZZ Test" --no-save
devic environments delete "ZZ Test"
```

`delete` drops the per-tenant snapshots the environment owns first, so none is
left orphaned.

---

## Not reachable with a tenant session token

These commands hand out a root shell, the account's secrets and the snapshot
every tenant boots from. Scoping an end user to their own data is no answer to
that, so the whole surface refuses session tokens and answers `403`. Use an API
key.
