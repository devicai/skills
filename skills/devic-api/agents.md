# Agents API

The Agents API allows you to manage autonomous AI agents that execute multi-step tasks. Agents can use tools, handle approvals, and maintain execution state through threads.

## Endpoints Overview

### Agent Management

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/agents` | List all agents |
| GET | `/api/v1/agents/:agentId` | Get agent by ID |
| POST | `/api/v1/agents` | Create a new agent |
| PATCH | `/api/v1/agents/:agentId` | Update an agent |
| DELETE | `/api/v1/agents/:agentId` | Delete an agent |

### Thread Management

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/agents/:agentId/threads` | Create a new thread |
| GET | `/api/v1/agents/:agentId/threads` | List threads for an agent |
| GET | `/api/v1/agents/threads/:threadId` | Get specific thread |
| GET | `/api/v1/agents/threads/:threadId/stream` | Follow a thread's lifecycle (server-sent events) |
| GET | `/api/v1/agents/threads/:threadId/search` | Search messages within a thread |
| PATCH | `/api/v1/agents/threads/:threadId` | Update a thread |
| POST | `/api/v1/agents/threads/:threadId/messages` | Continue a thread with a user message |

### Thread Control

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/agents/threads/:threadId/approval` | Handle approval request |
| POST | `/api/v1/agents/threads/:threadId/complete` | Complete a thread |
| POST | `/api/v1/agents/threads/:threadId/pause` | Pause a thread |
| POST | `/api/v1/agents/threads/:threadId/resume` | Resume a thread |

### Evaluation

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/agents/threads/:threadId/evaluate` | Evaluate a thread |
| GET | `/api/v1/agents/threads/:threadId/evaluation` | Get thread evaluation |
| GET | `/api/v1/agents/threads/:threadId/evaluations` | Get all evaluations |
| GET | `/api/v1/agents/threads/:threadId/evaluations/:evaluationId` | Get specific evaluation |
| PUT | `/api/v1/agents/agents/:agentId/evaluation-config` | Update evaluation config |
| GET | `/api/v1/agents/agents/:agentId/evaluation-config` | Get evaluation config |

### Cost Tracking

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/agents/agents/:agentId/costs/daily` | Get daily costs |
| GET | `/api/v1/agents/agents/:agentId/costs/monthly` | Get monthly costs |
| GET | `/api/v1/agents/agents/:agentId/costs/summary` | Get cost summary |

### Tags

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/agents/threads/tags` | Get all unique thread tags |
| GET | `/api/v1/agents/agents/:agentId/threads/tags` | Get tags for agent threads |

### Feedback

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/agents/threads/:threadId/feedback` | Submit feedback for a thread message (see [feedback.md](feedback.md)) |
| GET | `/api/v1/agents/threads/:threadId/feedback` | Get all feedback for a thread (see [feedback.md](feedback.md)) |

### Response Envelope

Successful responses are wrapped as `{ "success": true, "data": ..., "meta": {...} }`.
A body that already contains a `success` or `error` key is returned **as is**, without
the wrapper. Several endpoints below report failures that way: they answer with the
normal success status (200/201) and a body such as `{ "error": "THREAD_NOT_FOUND" }`.
Always check for an `error` key, not only the HTTP status.

---

## Thread States

The `state` field is always one of these **lowercase** values. Filters and request
bodies use the same values; uppercase names are not recognised.

| State | Description |
|-------|-------------|
| `under_construction` | Thread is being created; not yet picked up |
| `queued` | Waiting to be picked up by the processing cron (default state of a new thread) |
| `processing` | Thread is actively executing |
| `paused` | Thread paused (created paused, or paused via the API) |
| `paused_for_resume` | The agent paused itself with `pause_and_resume` until a scheduled time |
| `paused_for_approval` | Waiting for a decision on the [approval endpoint](#handle-approval) |
| `approval_rejected` | The approval was rejected; the thread ended |
| `waiting_for_response` | Waiting for an external response (e.g. a channel reply) |
| `handed_off` | Waiting for one or more subagent threads to complete |
| `completed` | Thread finished successfully |
| `failed` | Thread encountered an error |
| `terminated` | Thread was terminated |
| `guardrail_trigger` | Thread stopped by a guardrail |
| `limit_exceeded` | Thread blocked by a usage limit before execution (terminal — not resumed by the cron). See [tenants.md](tenants.md) |

---

## Subagent Handoff

Agents can delegate work to other agents (subagents) during execution via the `hand_off_subagent` tool. This creates a parent-child relationship between threads.

### How It Works

1. An agent configured with `subagentsIds` in its `assistantSpecialization` can invoke the `hand_off_subagent` tool during execution
2. A **subthread** is created for the target subagent, linked to the parent thread
3. The parent thread transitions to `handed_off` state
4. The subagent executes independently in its own thread
5. When the subagent calls `finish_execution`, the result is appended to the parent thread as a tool response
6. If all pending subthreads have completed, the parent thread transitions back to `queued` and resumes execution

### Parallel Subagent Handoffs

An agent can hand off to **multiple subagents in parallel**. The parent thread tracks all pending subthreads via `pendingHandOffSubThreadIds` and only resumes once every subthread has completed (or been terminated).

### Subthread Termination

If a user terminates a subthread (via the [Complete endpoint](#complete-thread) with state `terminated`), the parent thread is automatically notified with a tool response containing `"Terminated by the user"`, and the subthread is removed from the pending list. The same notification is sent when a subthread ends in `failed` or `limit_exceeded` without calling `finish_execution`.

---

## Scheduled Execution

**An agent can run itself on a schedule. You do not need an external cron.**

This is worth stating plainly because the field is easy to miss and the
consequence of missing it is a wrong architecture: a scheduler bolted on
outside Devic to poke `POST /agents/{agentId}/threads` every morning, when the
agent could have woken itself up.

Set `periodicExecution` on create or update. A cron in the platform checks every
minute and starts a thread when a schedule is due.

```json
{
  "periodicExecution": {
    "enabled": true,
    "cronExpression": "0 7 * * 1-5"
  }
}
```

### The three ways to say when

Exactly one of the three. Sending two is a 400.

| Field | Shape | Use it for |
|-------|-------|-----------|
| `cronExpression` | 5-field cron string | Anything cron expresses: `"0 7 * * 1-5"` is 07:00 on weekdays |
| `specificHour` | `{ hour, minute, timezone }` | A time of day, optionally narrowed by `executionDays` |
| `intervalSchedule` | `{ days, hour, minute, timezone, startDate }` | Every N days from an anchor date |

`intervalSchedule` is not redundant with cron: cron's day-of-month restarts
every month, so it cannot encode a true 14-day cadence. `startDate` is
`YYYY-MM-DD`, read in `timezone`.

`executionDays` narrows `specificHour` to certain weekdays:

```json
{
  "periodicExecution": {
    "enabled": true,
    "specificHour": { "hour": 9, "minute": 0, "timezone": "Europe/Madrid" },
    "executionDays": { "monday": true, "friday": true }
  }
}
```

`timezone` is an IANA name (`Europe/Madrid`, `America/New_York`). It defaults to
UTC, which is rarely what a business report wants.

### Reading it back

`GET /api/v1/agents/{agentId}` returns `periodicExecution` including
`lastExecutionTimestampMs`, which is how you check a schedule actually fired.

### Turning it off

```json
{ "periodicExecution": { "enabled": false } }
```

A disabled or archived agent does not fire either, whatever its schedule says.

### Validation

Inconsistent schedules are refused before anything is written, with a message
naming the field:

```
400 periodicExecution.cronExpression 'daily' is not a valid 5-field cron expression
```

### From the CLI

```bash
devic agents create --name "Daily report" --cron "0 7 * * 1-5" --timezone Europe/Madrid
devic agents update <agentId> --schedule-at 09:00 --days monday,friday --timezone Europe/Madrid
devic agents update <agentId> --every-days 14 --at 08:30 --start-date 2026-01-06
devic agents update <agentId> --no-schedule
```

---

## Agent Structure

Agents are configured through an embedded `assistantSpecialization` object that determines their behavior and capabilities.

### Key Agent Properties

| Property | Type | Description |
|----------|------|-------------|
| `_id` | string | Unique agent identifier |
| `name` | string | Agent display name |
| `description` | string | Agent description |
| `imgUrl` | string | Image URL |
| `avatarStyle` | object | Generated-avatar style used when there is no `imgUrl`: `{ shape?, hue?, tone? }` — `shape`: `round`\|`organic`\|`boxy`\|`nub`\|`cloud`\|`sun`; `hue`: 0-359; `tone`: `pastel`\|`neutral`\|`mid`\|`deep`\|`bright`\|`ink` |
| `projectId` | string | Project the agent belongs to |
| `assistantSpecialization` | object | Core configuration (see below) |
| `provider` | string | LLM provider of the agent's runs (`open-ai`, `anthropic`, `google-gemini`, `deepseek`, `kimi`, `xai` or a custom provider uid). Runs default to `open-ai` when unset |
| `llm` | string | Model of the agent's runs |
| `effectiveModel` / `effectiveProvider` / `modelSource` | string | Only on `GET /agents/:agentId`: the model and provider a run actually uses when the thread does not override them, and where that model comes from (`agent`, `account_default`, `platform_default`) |
| `disabled` | boolean | Whether agent is disabled (new threads are refused) |
| `archived` | boolean | Whether agent is archived (new threads are refused) |
| `planningConfig` | object | `{ enabled?: boolean }`. `false` disables planning tools and the task board; omitted keeps planning enabled |
| `maxExecutionInputTokens` | number | Token limit per execution |
| `maxExecutionToolCalls` | number | Max tool calls per execution |
| `maxExecutionFrequency` | number | Max executions within `executionFrequencyIntervalMs` |
| `executionFrequencyIntervalMs` | number | Timeframe (ms) for `maxExecutionFrequency` |
| `concurrentExecutionLimit` | number | Max threads of this agent in `processing` at once (5 when unset) |
| `agentNotificationConfig` | object | `{ enabled?, enableEmailNotification?, notifiedEmails?: string[], approvalEmails?: string[] }` |
| `evaluationConfig` | object | `{ enabled?, customCriteria?, disableDefaultEvaluation? }`. `enabled` is `false` on a new agent unless sent |
| `subAgentConfig` | object | Configuration for when this agent acts as a subagent (see below) |
| `periodicExecution` | object | Schedule on which the agent runs itself (see [Scheduled Execution](#scheduled-execution)) |
| `environmentId` | string | Environment the agent is connected to: its sandbox, secrets, knowledge and tools (see [environments.md](environments.md)) |
| `sandboxPreprovision` | object | `{ enabled?: boolean, timeoutMinutes?: number (1-30, default 5), ports?: number[] }`. Starts the terminal sandbox in the background when a thread starts or is re-queued with no live sandbox |
| `creationTimestampMs` / `lastEditTimestampMs` | number | Timestamps (ms) |

### assistantSpecialization Object

The `assistantSpecialization` defines the agent's behavior, tools, and capabilities:

| Property | Type | Description |
|----------|------|-------------|
| `identifier` | string | Unique specialization identifier. Generated on create (a value sent on create is ignored) |
| `name` | string | Specialization name |
| `presets` | string | System prompt / instructions. Must be a plain string |
| `availableToolsGroupsUids` | string[] | Tool group IDs the agent can use |
| `enabledTools` | string[] \| null | Explicit subset of enabled tool names, attached code snippets included (by their `toolName`). `null` enables every tool, `[]` enables none |
| `model` | string | Stored, but agent runs do **not** read it: the run model is the thread's `model` or the agent's top-level `llm`. Set `llm` instead |
| `reasoningEffort` | string \| null | Default reasoning effort of the agent's runs; choose from the model catalog. See [reasoning effort](conversation-options.md#select-the-scope). |
| `provider` | string | Stored, but agent runs use the thread's `provider` or the agent's top-level `provider`. Set the top-level field instead |
| `memoryDocuments` | object[] | Persistent context documents |
| `codeSnippetIds` | string[] | Ids of the [code snippets](code-snippets.md) the agent can call as tools |
| `availableSkillIds` | string[] | Skill IDs the agent can mount |
| `knowledgeSkills` | object[] | Knowledge skills `{ id, type: "document" \| "folder" }`. Ids that do not resolve to a skill of the declared type are rejected with 400 |
| `subagentsIds` | string[] | Other agents this agent can invoke. Each must exist, be non-archived and have `subAgentConfig.enabled: true`, else 400 |
| `smartTools` | object | Smart tools mode (tools loaded on demand via `discover_tools` / `load_tools`) |
| `maxChatMessages` | number | Maximum number of chat messages to include in context |
| `maxToolResponseInputTokens` | number | Maximum input tokens for a tool response |
| `toolResponseOverflow` | object | What happens to a tool response over `maxToolResponseInputTokens` (`{ saveToDocument?, previewCharacters? }`) |
| `memory` | object | Long-term memory configuration, identical to the assistant one |
| `contextManagement` | object | Context depth control: `{ fullContextTurnDepth?: number, alwaysIncludeUserMessages?: boolean }` — send only the most recent N turns in full and summarize older ones. See [Context Depth](#context-depth). Also holds the opt-in `compaction` configuration |

### subAgentConfig Object

Configures how an agent behaves when invoked as a subagent by another agent.

| Property | Type | Description |
|----------|------|-------------|
| `enabled` | boolean | Whether this agent can be invoked as a subagent |
| `inputFormat` | string | Input format: `"text"` or `"json"` |
| `inputStructure` | object | Expected input (when `inputFormat` is `"json"`). The JSON Schema goes **under a `parameters` key** |
| `outputFormat` | string | Output format: `"text"` or `"json"` |
| `outputStructure` | object | Expected output (when `outputFormat` is `"json"`). Same shape: the JSON Schema **under `parameters`** |

`enabled: true` is the prerequisite for any other agent or assistant to list
this one in `subagentsIds` — that field is validated, and a callee without it is
rejected with `INVALID_SUBAGENTS` / `NOT_ENABLED`. Configure the callee first,
then reference it.

The structures are not bare JSON Schemas. The schema is read from `parameters`,
and a structure without that key is rejected with `INVALID_SUBAGENT_CONFIG`:

```json
{
  "subAgentConfig": {
    "enabled": true,
    "inputFormat": "json",
    "inputStructure": {
      "name": "repo_summary_input",
      "description": "What to summarise",
      "strict": true,
      "parameters": {
        "type": "object",
        "properties": { "repo": { "type": "string", "description": "owner/name" } },
        "required": ["repo"],
        "additionalProperties": false
      }
    },
    "outputFormat": "text"
  }
}
```

On `PATCH` the object is **merged key by key** with what is stored, so flipping
`enabled` does not erase the declared structures. To go back to free text set
`"inputFormat": "text"` — omitting a key changes nothing.

### Tool Access

Agents access tools through the `availableToolsGroupsUids` property:

1. Each UID references a **Tools Group**
2. Tools Groups can contain:
   - Built-in platform tools
   - External tools from a **Tool Server**
3. When an agent executes, it can call any tool from its assigned groups
4. Use `enabledTools` to restrict to a specific subset: it is an allowlist of
   tool names across all the assigned groups, where `null` (or absent) enables
   every tool of those groups and `[]` enables none
5. Code snippets are attached apart, by id, in
   `assistantSpecialization.codeSnippetIds`. `enabledTools` filters them too:
   when the agent has a list, add each snippet's `toolName` to it or runs leave
   the snippet out. See [code-snippets.md](code-snippets.md#attaching-a-snippet)

**Example**: An agent with `availableToolsGroupsUids: ["crm-tools", "email-tools"]` can use all tools from both the CRM and Email tool groups during execution.

---

## List Agents

Retrieves a paginated list of all agents.

```
GET /api/v1/agents
```

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `offset` | number | 0 | Number of items to skip |
| `limit` | number | 10 | Maximum items to return (max: 100) |
| `archived` | boolean | - | `true` also includes archived agents. Omitted or `false`: archived agents are excluded |
| `projectId` | string | - | Only agents of this project. For a non-admin caller (personal OAuth token), a project it cannot access returns 403 |

Agents are sorted by name.

### Example Request

```bash
curl -X GET "https://api.devic.ai/api/v1/agents?limit=20" \
  -H "Authorization: Bearer devic-your-api-key"
```

### Response

```json
{
  "success": true,
  "data": {
    "agents": [
      {
        "_id": "65a1b2c3d4e5f6789012345",
        "name": "Sales Assistant Agent",
        "description": "Helps with sales inquiries",
        "disabled": false,
        "archived": false,
        "provider": "open-ai",
        "llm": "gpt-4o",
        "creationTimestampMs": 1705315800000,
        "assistantSpecialization": {
          "presets": "You are a sales assistant...",
          "availableToolsGroupsUids": ["crm-tools"]
        }
      }
    ],
    "total": 5,
    "offset": 0,
    "limit": 20,
    "hasMore": false
  }
}
```

---

## Get Agent by ID

Retrieves detailed information about a specific agent.

```
GET /api/v1/agents/:agentId
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `agentId` | string | The unique identifier of the agent |

### Response

```json
{
  "success": true,
  "data": {
    "_id": "65a1b2c3d4e5f6789012345",
    "name": "Sales Assistant Agent",
    "description": "Helps with sales inquiries",
    "disabled": false,
    "archived": false,
    "provider": "open-ai",
    "llm": "gpt-4o",
    "effectiveModel": "gpt-4o",
    "effectiveProvider": "open-ai",
    "modelSource": "agent",
    "assistantSpecialization": {
      "identifier": "sales-assistant",
      "presets": "You are a sales assistant...",
      "availableToolsGroupsUids": ["crm-tools", "email-tools"],
      "enabledTools": ["search_contacts", "send_email"]
    },
    "maxExecutionInputTokens": 100000,
    "maxExecutionToolCalls": 50,
    "evaluationConfig": {
      "enabled": true
    }
  }
}
```

### Error Responses

| Status | Description |
|--------|-------------|
| 404 | Agent not found |

---

## Create Agent

Creates a new agent with the provided configuration.

```
POST /api/v1/agents
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Agent name (400 `Agent name is required` when missing) |
| `description` | string | No | Agent description |
| `imgUrl` | string | No | Image URL for the agent |
| `avatarStyle` | object | No | Generated-avatar style (see [Key Agent Properties](#key-agent-properties)) |
| `projectId` | string | No | Project the agent belongs to |
| `assistantSpecialization` | object | No | Configuration (see [assistantSpecialization Object](#assistantspecialization-object)). If it carries an `_id`, every other key is dropped and the agent references that existing assistant specialization |
| `provider` | string | No | LLM provider of the agent's runs (defaults to `open-ai`) |
| `llm` | string | No | Model of the agent's runs |
| `planningConfig` | object | No | `{ enabled?: boolean }`; `false` disables planning and skips the task board. A non-boolean `enabled` is a 400 |
| `maxExecutionInputTokens` | number | No | Max input tokens per execution |
| `maxExecutionToolCalls` | number | No | Max tool calls per execution |
| `maxExecutionFrequency` | number | No | Max executions in timeframe |
| `executionFrequencyIntervalMs` | number | No | Timeframe (ms) for frequency limit |
| `concurrentExecutionLimit` | number | No | Max concurrent executions (5 when unset) |
| `agentNotificationConfig` | object | No | `{ enabled?, enableEmailNotification?, notifiedEmails?: string[], approvalEmails?: string[] }` |
| `evaluationConfig` | object | No | `{ enabled?, customCriteria?, disableDefaultEvaluation? }`; `enabled` defaults to `false` |
| `subAgentConfig` | object | No | Configuration for subagent behavior (see subAgentConfig Object) |
| `periodicExecution` | object | No | Run the agent on a schedule (see [Scheduled Execution](#scheduled-execution)) |
| `environmentId` | string | No | Environment to connect the agent to; `null` disconnects (see [environments.md](environments.md)). An unknown id is a 400 **after** the agent was saved |
| `sandboxPreprovision` | object | No | `{ enabled?, timeoutMinutes?, ports? }` (see [Key Agent Properties](#key-agent-properties)) |

`assistantSpecialization` takes the fields listed in [assistantSpecialization Object](#assistantspecialization-object). Set the run model with the top-level `llm`/`provider`, not `assistantSpecialization.model`.

Validation (400, nothing is written): `presets` not a string, inconsistent `periodicExecution`, invalid `subAgentConfig`, invalid `subagentsIds`, invalid `knowledgeSkills`, invalid `planningConfig`.

### Example Request

```bash
curl -X POST "https://api.devic.ai/api/v1/agents" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Customer Support Agent",
    "description": "Handles customer support tickets",
    "assistantSpecialization": {
      "presets": "You are a customer support agent. Be helpful and professional.",
      "availableToolsGroupsUids": ["support-tools", "crm-tools"]
    },
    "provider": "open-ai",
    "llm": "gpt-4o",
    "maxExecutionToolCalls": 30,
    "evaluationConfig": {
      "enabled": true
    }
  }'
```

### Response (201 Created)

```json
{
  "success": true,
  "data": {
    "_id": "65a1b2c3d4e5f6789012345",
    "name": "Customer Support Agent",
    "description": "Handles customer support tickets",
    "creationTimestampMs": 1705315800000,
    "assistantSpecialization": {
      "presets": "You are a customer support agent...",
      "availableToolsGroupsUids": ["support-tools", "crm-tools"]
    }
  }
}
```

---

## Update Agent

Updates an existing agent. Supports partial updates.

```
PATCH /api/v1/agents/:agentId
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `agentId` | string | The unique identifier of the agent |

### Request Body

All fields are optional. Only provided fields will be updated.

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Agent name |
| `description` | string | Agent description |
| `imgUrl` | string | Image URL |
| `avatarStyle` | object | Generated-avatar style |
| `projectId` | string \| null | Project the agent belongs to |
| `assistantSpecialization` | object | Configuration update (merged, see below) |
| `disabled` | boolean | Disable/enable the agent |
| `archived` | boolean | Archive/unarchive the agent |
| `provider` | string | LLM provider of the agent's runs |
| `llm` | string | Model of the agent's runs |
| `planningConfig` | object | `{ enabled?: boolean }`. Re-enabling planning provisions the task board |
| `maxExecutionInputTokens` | number | Max input tokens |
| `maxExecutionToolCalls` | number | Max tool calls |
| `maxExecutionFrequency` | number | Max executions in timeframe |
| `executionFrequencyIntervalMs` | number | Timeframe (ms) for frequency limit |
| `concurrentExecutionLimit` | number | Max concurrent executions |
| `agentNotificationConfig` | object | Notification settings (replaces the stored object) |
| `evaluationConfig` | object | Evaluation settings (replaces the stored object) |
| `subAgentConfig` | object | Configuration for subagent behavior (see subAgentConfig Object); merged key by key |
| `periodicExecution` | object | Schedule (replaces the stored object; see [Scheduled Execution](#scheduled-execution)) |
| `environmentId` | string \| null | Environment to connect; `null` or `""` disconnects |
| `sandboxPreprovision` | object | `{ enabled?, timeoutMinutes?, ports? }` |

**`assistantSpecialization` is merged one level deep**: the keys you send replace the
stored ones, the others are kept. One exception: if the agent has an `enabledTools`
list and the payload contains `assistantSpecialization` **without** `enabledTools`,
`enabledTools` is reset to `null` (every tool enabled). Resend `enabledTools` on every
update that touches `assistantSpecialization`.

The response is the updated agent. 404 if the agent does not exist; the same 400
validations as Create apply.

### Example Request

```bash
curl -X PATCH "https://api.devic.ai/api/v1/agents/65a1b2c3d4e5f6789012345" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "description": "Updated description",
    "assistantSpecialization": {
      "presets": "Updated system prompt...",
      "enabledTools": ["search_contacts"]
    },
    "maxExecutionToolCalls": 50
  }'
```

---

## Delete Agent

Deletes an agent. Note: This does not delete associated threads.

```
DELETE /api/v1/agents/:agentId
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `agentId` | string | The unique identifier of the agent |

### Response

The body carries its own `success` key, so it is **not** wrapped in `data`:

```json
{
  "success": true,
  "message": "Agent deleted successfully",
  "deletedId": "65a1b2c3d4e5f6789012345"
}
```

### Error Responses

| Status | Description |
|--------|-------------|
| 404 | Agent not found |

---

## Create Thread

Creates a new execution thread for the specified agent.

```
POST /api/v1/agents/:agentId/threads
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `agentId` | string | The ID of the agent |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `message` | string \| object | No | Initial message/task for the agent. An object is sent to the agent as its JSON string. When omitted, every body property that is **not** one of the control fields in this table is forwarded to the agent as the message (so a raw webhook payload can be posted as is); with nothing left, the message is `{}` |
| `state` | string | No | Initial state: `queued` (default, runs on the next processing cycle) or `paused` (created paused until [resumed](#resume-thread)). Any other value is a 400 `VALIDATION_ERROR` |
| `metadata` | object | No | Custom metadata for the thread. `metadata.promptTemplateParams` fills `{{placeholder}}` variables in the agent's system prompt — see [Prompt Template Parameters](#prompt-template-parameters). `metadata.tenantToken` is a token for tool authentication |
| `tenantId` | string | No | Tenant the thread belongs to (multi-tenant environments). Auto-registers the tenant on first use and attributes cost/usage to it. See [tenants.md](tenants.md). |
| `subtenantId` | string | No | End user/entity inside the tenant, stored as `metadata.subtenantId`. When omitted it is derived from `metadata.subtenantMetadata.id` or the legacy `metadata.userId`. Used for per-subtenant cost attribution and usage limits. |
| `enabledTools` | string[] | No | Tools available to this thread, overriding the agent's `assistantSpecialization.enabledTools`. `[]` means no tools |
| `provider` | string | No | LLM provider for this thread (`open-ai`, `deepseek`, `anthropic`, `google-gemini`, `kimi`, `xai`). Defaults to the agent's `provider`, then `open-ai` |
| `model` | string | No | Model for this thread. Defaults to the agent's `llm`. A retired model is refused with 410 `MODEL_DEPRECATED` |
| `tags` | string[] | No | Reserved: not forwarded as message, but **not stored** on creation. Set tags with [Update Thread](#update-thread) |

### Prompt Template Parameters

The agent's system prompt (the `presets` of its `assistantSpecialization`) can contain `{{variableName}}` placeholders. When the thread is created with `metadata.promptTemplateParams`, the system prompt is processed before execution, replacing each `{{variableName}}` placeholder with the matching value:

| Metadata Field | Type | Description |
|----------------|------|-------------|
| `promptTemplateParams` | object (`Record<string, any>`) | Key-value map of template variables. Each key replaces the `{{key}}` placeholder in the system prompt. |

For example, with an agent whose presets contain:

```
Process data for {{customerName}} in region {{region}}.
```

Create the thread with the values in metadata:

```bash
curl -X POST "https://api.devic.ai/api/v1/agents/agent-123/threads" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Run the monthly data processing task",
    "metadata": {
      "promptTemplateParams": {
        "customerName": "Acme Corp",
        "region": "North America"
      }
    }
  }'
```

The agent executes with the system prompt already resolved: "Process data for Acme Corp in region North America."

**Notes:**
- The misspelled legacy key `propmptTemplateParams` is also accepted for backward compatibility (and takes precedence when both are present). Use the correct spelling `promptTemplateParams` in new integrations.
- This keeps a single agent reusable across customers/contexts: per-thread context travels in the thread metadata instead of requiring one agent per customer.
- The same mechanism is available for assistant messages — see [assistants.md](assistants.md#prompt-template-parameters).

### Example Request

```bash
curl -X POST "https://api.devic.ai/api/v1/agents/agent-123/threads" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Analyze the Q4 sales report and create a summary",
    "tenantId": "acme-corp",
    "subtenantId": "user_67890"
  }'
```

### Response (201 Created)

`message` here is the literal `"Agent notified"`; `thread` is the thread as returned by
[Get Thread](#get-thread-by-id).

```json
{
  "success": true,
  "data": {
    "message": "Agent notified",
    "threadId": "65b2c3d4e5f6789012345678",
    "thread": {
      "_id": "65b2c3d4e5f6789012345678",
      "agentId": "65a1b2c3d4e5f6789012345",
      "state": "queued",
      "threadContent": [...],
      "creationTimestampMs": 1705315800000,
      "metadata": { "subtenantId": "user_67890" },
      "tenantId": "acme-corp"
    }
  }
}
```

### Error Responses

| Status | Body `error` | Description |
|--------|--------------|-------------|
| 400 | `VALIDATION_ERROR` | A control field has the wrong type, or `state` is not `queued`/`paused` |
| 404 | `NOT_FOUND` | Agent not found |
| 410 | `MODEL_DEPRECATED` | The thread's model (`model` or the agent's `llm`) is retired |
| 429 | `TENANT_LIMIT_EXCEEDED` | Tenant usage limit exceeded (see below) |
| 402 | `BILLING_BLOCKED` | The account's billing blocks new runs |
| 201 | `AGENT_IS_DISABLED`, `AGENT_IS_ARCHIVED`, `ASSISTANT_SPECIALIZATION_NOT_FOUND`, `INTERNAL_SERVER_ERROR` | Returned unwrapped as `{ "error": ... }` with the success status; no thread runs |

When the thread's `tenantId`/`subtenantId` has a configured usage limit that is already consumed, thread creation is blocked **before** the LLM call and returns HTTP `429` with a structured body (`details.blockingRule`/`current`/`limit`/`resetsAt`, plus `retryAfter` in seconds when the reset is in the future; it is a body field, not a header) — identical in shape to the assistant 429 documented in [assistants.md](assistants.md). A marker thread is also persisted with state `limit_exceeded` (terminal; the resumption cron does not retry it) so it is visible in the thread list. See [tenants.md](tenants.md) for the usage-limit model.

---

## List Threads for Agent

Retrieves a paginated list of threads for the specified agent.

```
GET /api/v1/agents/:agentId/threads
```

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `offset` | number | 0 | Number of items to skip |
| `limit` | number | 10 | Maximum items to return (max: 100) |
| `state` | string | - | Exact match on ONE lowercase [thread state](#thread-states) (e.g. `completed`). Comma-separated lists and uppercase names match nothing |
| `startDate` | string | - | Created at or after this date (ISO string) |
| `endDate` | string | - | Created at or before this date (ISO string) |
| `dateOrder` | string | - | Sort by creation date (`asc` or `desc`) |
| `durationOrder` | string | - | Sort by duration (`asc` or `desc`); only threads with start and finish timestamps are returned, sorted within the page |
| `tokensOrder` | string | - | Sort by tokens (`asc` or `desc`) |
| `tags` | string | - | Comma-separated tags (matches ANY tag) |
| `tenantId` | string | - | Filter by tenant ID |
| `subtenantId` | string | - | Filter by subtenant ID (`metadata.subtenantId`) |

Only one ordering applies, in this priority: `dateOrder`, `durationOrder`, `tokensOrder`. Default: newest first.

### Example Request

```bash
curl -X GET "https://api.devic.ai/api/v1/agents/agent-123/threads?state=completed&limit=20&tags=urgent,review" \
  -H "Authorization: Bearer devic-your-api-key"
```

### Response

Each item has the shape returned by [Get Thread](#get-thread-by-id). There is no `hasMore`; compare `offset + limit` with `total`.

```json
{
  "success": true,
  "data": {
    "threads": [
      {
        "_id": "65b2c3d4e5f6789012345678",
        "agentId": "65a1b2c3d4e5f6789012345",
        "state": "completed",
        "threadContent": [...],
        "creationTimestampMs": 1705312800000,
        "finishTimestampMs": 1705313700000,
        "tokenUsage": { "inputTokens": 4000, "outputTokens": 1432 },
        "finishReason": "Task completed"
      }
    ],
    "total": 45,
    "offset": 0,
    "limit": 20
  }
}
```

---

## Get Thread by ID

Retrieves detailed information about a specific thread.

```
GET /api/v1/agents/threads/:threadId
```

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `withTasks` | boolean | false | Include the agent's planning tasks for this thread in `tasks`. Send `withTasks=true`; omit it otherwise (the raw query string is used, so even `withTasks=false` turns it on) |

### Response

```json
{
  "success": true,
  "data": {
    "_id": "65b2c3d4e5f6789012345678",
    "agentId": "65a1b2c3d4e5f6789012345",
    "clientUID": "client-uid",
    "state": "processing",
    "threadContent": [
      { "role": "developer", "content": { "message": "..." }, "uid": "...", "timestamp": 1705315800000 },
      { "role": "user", "content": { "message": "Analyze the Q4 sales report" }, "uid": "..." }
    ],
    "creationTimestampMs": 1705315800000,
    "inputTokens": 4000,
    "outputTokens": 1432,
    "tokenUsage": { "...": "..." },
    "threadStatesChanges": [
      { "state": "processing", "timestamp": 1705315801000, "userUID": "system" }
    ],
    "metadata": {},
    "tenantId": "acme-corp",
    "tasks": [...]
  }
}
```

### Thread Properties

| Property | Type | Description |
|----------|------|-------------|
| `_id` | string | Thread ID |
| `agentId` | string | Agent ID |
| `state` | string | Lowercase [thread state](#thread-states) |
| `threadContent` | object[] | Messages (`role`: `developer`, `user`, `assistant`, `tool`; `content.message`; `uid`; `timestamp`) |
| `creationTimestampMs` / `finishTimestampMs` | number | Timestamps (ms) |
| `finishReason` | string | Why the thread finished |
| `inputTokens` / `outputTokens` | number | Legacy token counters |
| `tokenUsage` | object | Detailed token usage and cost |
| `threadStatesChanges` | object[] | State history `{ state, timestamp, userUID }` |
| `metadata` | object | Thread metadata |
| `tenantId` | string | Tenant ID |
| `tasks` | object[] | Planning tasks (only with `withTasks=true`) |
| `pendingAsyncToolCalls` | object[] | Pending async tool callbacks, while the thread waits for one |
| `pausedReason` / `pausedUntil` / `pausedTimestampMS` | string / number / number | Why and until when (ms) a paused thread is paused |
| `coreMemories` / `recalledMemories` | object[] | Memory audit trails |

The public representation does **not** include `tags`, `userUID`, `evaluations` or the
subthread fields (`isSubthread`, `parentThreadId`, `parentAgentId`, `subThreadToolCallId`,
`pendingHandOffSubThreadIds`, `handOffSubThreadIds`), even though the thread stores them.
Use [Get Thread Tags](#get-unique-thread-tags) and the
[evaluation endpoints](#get-all-thread-evaluations) to read tags and evaluations.

### Error Responses

| Status | Description |
|--------|-------------|
| 404 | Thread not found |

---

## Update Thread

Updates a thread with the provided data.

```
PATCH /api/v1/agents/threads/:threadId
```

### Request Body

There is no allowlist: every top-level key of the body is written to the thread as is.
Use it for:

| Field | Type | Description |
|-------|------|-------------|
| `tags` | string[] | Replaces the thread tags (the only way to set tags; thread creation does not store them) |
| `metadata` | object | **Replaces** the whole stored `metadata` object (not merged): resend the keys you want to keep, e.g. `subtenantId` |

Prefer the dedicated endpoints to change `state` or `threadContent`
([Complete](#complete-thread), [Pause](#pause-thread), [Add Message](#add-thread-message)):
this endpoint writes them without checking the current state.

The response is the updated thread. Any failure, including an unknown thread, is a 400
`Error updating thread: ...`.

### Example Request

```bash
curl -X PATCH "https://api.devic.ai/api/v1/agents/threads/thread-456" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "tags": ["sales", "reviewed", "approved"]
  }'
```

---

## Add Thread Message

Appends a user message to an existing thread to continue it.

```
POST /api/v1/agents/threads/:threadId/messages
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `message` | string \| object | Yes | The user message |

The optional `devic-api-source` header is recorded as the message source (default `public-api`).

What happens depends on the current state:

| Current state | Effect |
|---------------|--------|
| `processing`, `queued`, `paused`, `paused_for_resume` | The message is parked and delivered on the thread's next turn (after a pause, once resumed). State unchanged |
| `completed`, `failed`, `terminated`, `guardrail_trigger`, `limit_exceeded`, `approval_rejected` | The message is appended and the thread is re-queued (`queued`) |
| `waiting_for_response` | The pending wait is closed, the message is appended and the thread is re-queued |
| `paused_for_approval` | 409 `THREAD_WAITING_FOR_APPROVAL` — use [Handle Approval](#handle-approval) |
| `handed_off` | 409 `THREAD_HANDED_OFF` — message the active subthread instead |
| `under_construction` | 409 `THREAD_UNDER_CONSTRUCTION` — retry shortly |

### Response (200)

The updated thread (see [Get Thread](#get-thread-by-id)). 404 if the thread does not exist.

```bash
curl -X POST "https://api.devic.ai/api/v1/agents/threads/thread-456/messages" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{ "message": "Also include a summary of the previous steps" }'
```

---

## Handle Approval

Processes an approval request for a thread that requires user confirmation.

```
POST /api/v1/agents/threads/:threadId/approval
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action` | string | Yes | `"approved"` or `"rejected"`. Any other value, including a missing field, is treated as a rejection |
| `message` | string | No | Your answer. The agent receives it as `User responded: <message>` (literally `User responded: undefined` when omitted). On a rejection it is also stored as `approvalRejectedMessage` |
| `retry` | boolean | No | Only with `"rejected"`: instead of ending the thread, the agent continues and reads your message, so it can try another way |

What happens next:

| Body | Thread state |
|------|--------------|
| `action: "approved"` | `queued`. The agent resumes |
| `action: "rejected"`, `retry: true` | `queued`. The agent resumes with your message as feedback |
| `action: "rejected"` | `approval_rejected`. The thread ends |

The thread must be in `paused_for_approval`. Otherwise, or when the thread does not
exist, the endpoint answers 400 `Error handling approval: ...` and nothing changes.
On success the response is the updated thread.

> There is no `approved: true` field. A body such as `{ "approved": true }` has no `action` and **rejects** the request.

### Example Request

```bash
curl -X POST "https://api.devic.ai/api/v1/agents/threads/thread-456/approval" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "action": "approved",
    "message": "Proceed with the data export"
  }'

# Reject, but let the agent try another way
curl -X POST "https://api.devic.ai/api/v1/agents/threads/thread-456/approval" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "action": "rejected",
    "retry": true,
    "message": "Do not export customer emails; export only the totals"
  }'
```

---

## Complete Thread

Manually finishes a thread with the given final state.

```
POST /api/v1/agents/threads/:threadId/complete
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `state` | string | Yes | Final state, lowercase: `completed`, `failed` or `terminated`. The value is stored verbatim and is not validated, so any other string (e.g. `COMPLETED` or `cancelled`) leaves the thread in a state no client recognises |

The thread must currently be `processing`, `queued` or `waiting_for_response`. The
finish reason is set to `Manually finished thread`. Ending a subthread as `terminated` or
`failed` notifies its parent (see [Subthread Termination](#subthread-termination)).

### Response (200)

The updated thread. Failures are returned unwrapped **with status 200**:

| Body | Cause |
|------|-------|
| `{ "error": "THREAD_NOT_FOUND" }` | Unknown thread |
| `{ "error": "THREAD_CANNOT_BE_MANUALLY_COMPLETED" }` | The thread is not `processing`, `queued` or `waiting_for_response` |

```bash
curl -X POST "https://api.devic.ai/api/v1/agents/threads/thread-456/complete" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{ "state": "terminated" }'
```

---

## Pause Thread

Sets a thread to `paused`. No request body.

```
POST /api/v1/agents/threads/:threadId/pause
```

Only a thread in `queued`, `paused` or `paused_for_resume` can be paused; a thread that
is already `processing` cannot. Pausing a `paused_for_resume` thread also answers the
agent's pending `pause_and_resume` call.

### Response (200)

The updated thread (`"state": "paused"`). Failures are returned unwrapped with status 200:
`{ "error": "THREAD_NOT_FOUND" }` or `{ "error": "THREAD_CANNOT_BE_PAUSED_OR_RESUMED" }`.

---

## Resume Thread

Resumes a paused thread. No request body.

```
POST /api/v1/agents/threads/:threadId/resume
```

Accepted from `paused`, `queued` or `paused_for_resume`; the thread is set to
`processing`. Resuming a `paused_for_resume` thread before its scheduled time answers the
agent's `pause_and_resume` call with "The user resumed the thread manually before
schedule".

### Response (200)

The updated thread. Failures are returned unwrapped with status 200:
`{ "error": "THREAD_NOT_FOUND" }` or `{ "error": "THREAD_CANNOT_BE_PAUSED_OR_RESUMED" }`.

---

## Evaluate Thread

Triggers evaluation of a finished thread based on multiple criteria. The thread must be
`completed` or `failed`. The new evaluation is appended to the thread's evaluations.

```
POST /api/v1/agents/threads/:threadId/evaluate
```

When the agent has `evaluationConfig.enabled: false` (the default for a new agent; see
[Update Evaluation Config](#update-evaluation-config)), no evaluation runs and the
response is `{ "evaluation": null }`.

Failures are returned unwrapped with status 200: `{ "error": "THREAD_NOT_FOUND" }`,
`THREAD_NOT_COMPLETED`, `AGENT_NOT_FOUND` or `EVALUATION_FAILED`.

### Response

```json
{
  "success": true,
  "data": {
    "evaluation": {
      "evaluationId": "eval-uuid",
      "overallScore": 8.5,
      "evaluationTimestampMs": 1705315800000,
      "evaluationModel": "gpt-4",
      "generalFeedback": "The agent completed the task effectively...",
      "criteria": {
        "instructionFollowing": {
          "score": 9,
          "reasoning": "Followed all instructions accurately",
          "observations": ["Correctly identified key metrics"]
        },
        "taskPlanning": {
          "score": 8,
          "reasoning": "Good task breakdown",
          "observations": ["Could have parallelized some steps"]
        },
        "taskExecution": {
          "score": 9,
          "reasoning": "Executed efficiently",
          "observations": []
        },
        "toolUsageEffectiveness": {
          "score": 8,
          "reasoning": "Good tool selection",
          "observations": ["Used appropriate tools for analysis"]
        },
        "executionFinalization": {
          "score": 8,
          "reasoning": "Clean completion",
          "observations": []
        }
      }
    }
  }
}
```

---

## Get Thread Evaluation

Retrieves the most recent evaluation of a thread, as `{ "evaluation": {...} }`.

```
GET /api/v1/agents/threads/:threadId/evaluation
```

Failures are returned unwrapped with status 200: `{ "error": "THREAD_NOT_FOUND" }`,
`EVALUATION_NOT_FOUND` (the thread has none) or `FAILED_TO_GET_EVALUATION`.

---

## Get All Thread Evaluations

Retrieves all evaluation results for a thread, as `{ "evaluations": [...] }`.

```
GET /api/v1/agents/threads/:threadId/evaluations
```

An unknown thread answers unwrapped with status 200:
`{ "evaluations": [], "error": "THREAD_NOT_FOUND" }`.

---

## Get Thread Evaluation by ID

Retrieves one evaluation of a thread by its `evaluationId`, as `{ "evaluation": {...} }`.

```
GET /api/v1/agents/threads/:threadId/evaluations/:evaluationId
```

Failures are returned unwrapped with status 200: `{ "error": "THREAD_NOT_FOUND" }` or
`EVALUATION_NOT_FOUND`.

---

## Get Agent Daily Costs

Retrieves daily cost breakdown for an agent.

```
GET /api/v1/agents/agents/:agentId/costs/daily
```

### Query Parameters

| Parameter | Type | Format | Description |
|-----------|------|--------|-------------|
| `startDate` | string | YYYY-MM-DD | Start date (default: 30 days ago) |
| `endDate` | string | YYYY-MM-DD | End date (default: today) |

Days without activity are absent. Results are sorted newest first.

### Example Request

```bash
curl -X GET "https://api.devic.ai/api/v1/agents/agents/agent-123/costs/daily?startDate=2024-01-01&endDate=2024-01-31" \
  -H "Authorization: Bearer devic-your-api-key"
```

### Response

```json
{
  "success": true,
  "data": [
    {
      "agentId": "65a1b2c3d4e5f6789012345",
      "date": "2024-01-15",
      "totalCost": 12.50,
      "totalInputTokens": 50000,
      "totalOutputTokens": 25000,
      "totalInputCachedTokens": 12000,
      "totalInputCacheWriteTokens": 0,
      "totalOutputCachedTokens": 0,
      "totalReasoningOutputTokens": 3000,
      "totalSecondaryCost": 0.12,
      "threadsCount": 15,
      "lastUpdated": 1705334400000
    }
  ]
}
```

`totalSecondary*` fields hold the cost and tokens of secondary operations (summaries,
compaction, ...), with a per-operation breakdown in `secondaryByOperation`.

---

## Get Agent Monthly Costs

Retrieves monthly cost breakdown for an agent.

```
GET /api/v1/agents/agents/:agentId/costs/monthly
```

### Query Parameters

| Parameter | Type | Format | Description |
|-----------|------|--------|-------------|
| `startMonth` | string | YYYY-MM | Start month (default: current month) |
| `endMonth` | string | YYYY-MM | End month (default: current month) |

Each item has the same fields as a daily item, with `month` instead of `date`.

---

## Get Cost Summary

Retrieves cost summary for today and current month (UTC).

```
GET /api/v1/agents/agents/:agentId/costs/summary
```

### Response

Each side has the daily/monthly item shape, or is `null` when there was no activity.

```json
{
  "success": true,
  "data": {
    "today": {
      "date": "2024-01-15",
      "totalCost": 5.25,
      "totalInputTokens": 21000,
      "totalOutputTokens": 10500,
      "threadsCount": 8
    },
    "currentMonth": {
      "month": "2024-01",
      "totalCost": 125.75,
      "totalInputTokens": 503000,
      "totalOutputTokens": 251500,
      "threadsCount": 180
    }
  }
}
```

---

## Update Evaluation Config

Updates evaluation settings and custom criteria for an agent.

```
PUT /api/v1/agents/agents/:agentId/evaluation-config
```

### Request Body

Only these two fields are read; any other key is ignored. Each one sent replaces the
stored value.

| Field | Type | Description |
|-------|------|-------------|
| `enabled` | boolean | Enables automatic evaluation (and the [Evaluate](#evaluate-thread) endpoint) |
| `customCriteria` | object[] | Replaces the custom criteria: `{ id, name, description, weight?, enabled?, isBinary? }` |

```json
{
  "enabled": true,
  "customCriteria": [
    {
      "id": "accuracy",
      "name": "accuracy",
      "description": "Evaluate the accuracy of data analysis",
      "weight": 0.5
    }
  ]
}
```

### Response

`{ "agent": {...} }` with the updated agent. Failures are returned unwrapped with status
200: `{ "error": "AGENT_NOT_FOUND" }` or `FAILED_TO_UPDATE_EVALUATION_CONFIG`.

---

## Get Evaluation Config

Retrieves evaluation configuration for an agent, as `{ "evaluationConfig": {...} }`.

```
GET /api/v1/agents/agents/:agentId/evaluation-config
```

An agent without one returns `{ "enabled": true, "customCriteria": [] }`. Failures are
returned unwrapped with status 200: `{ "error": "AGENT_NOT_FOUND" }`.

---

## Get Unique Thread Tags

Retrieves all unique tags used across agent threads.

```
GET /api/v1/agents/threads/tags
```

For agent-specific tags:

```
GET /api/v1/agents/agents/:agentId/threads/tags
```

### Response

Tags are returned sorted alphabetically.

```json
{
  "success": true,
  "data": ["approved", "quarterly", "review", "sales", "urgent"]
}
```

---

## Stream Thread (SSE)

Follows a thread's lifecycle as server-sent events instead of polling.

```
GET /api/v1/agents/threads/:threadId/stream
```

- The first `event: snapshot` frame is sent immediately; another one whenever the thread
  changes state. Each `data` is the thread as returned by [Get Thread](#get-thread-by-id)
  (not wrapped in `success`/`data`).
- `: keep-alive` comment lines keep idle connections open.
- The stream closes when the thread reaches `completed`, `failed`, `terminated`,
  `approval_rejected`, `guardrail_trigger` or `limit_exceeded`, or after 60 seconds:
  reconnect to keep following it. An `event: reconnect` frame is sent if the server hits
  an error.
- 404 if the thread does not exist.

```bash
curl -N "https://api.devic.ai/api/v1/agents/threads/thread-456/stream" \
  -H "Authorization: Bearer devic-your-api-key"
```

---

## Search Thread Messages

Substring search across the messages of one thread.

```
GET /api/v1/agents/threads/:threadId/search
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `q` | string | - | Search term (substring). Required |
| `tenantId` | string | - | Optional tenant filter |
| `limit` | number | 20 | Max hits (1-100) |
| `offset` | number | 0 | Hits to skip |

### Response

```json
{
  "success": true,
  "data": {
    "data": [
      {
        "messageUid": "msg-uid",
        "source": "thread",
        "conversationId": "65b2c3d4e5f6789012345678",
        "role": "assistant",
        "partType": "text",
        "snippet": "...matching text...",
        "occurredAt": 1705315800000,
        "agentId": "65a1b2c3d4e5f6789012345"
      }
    ],
    "pagination": { "total": 1, "limit": 20, "offset": 0, "hasMore": false }
  }
}
```

404 if the thread does not exist.

---

## Thread Feedback

`POST /api/v1/agents/threads/:threadId/feedback` (body `{ messageId, feedback?: boolean, feedbackComment?, feedbackData? }`, where `messageId` is the message `uid`) and `GET /api/v1/agents/threads/:threadId/feedback` are documented in [feedback.md](feedback.md).

---

## Context Depth

On long-running agent threads the input context grows every turn, increasing latency and cost. The `contextManagement` object (inside `assistantSpecialization`) lets the agent send only the most recent turns verbatim and replace older ones with a short, AI-generated summary.

```json
{
  "assistantSpecialization": {
    "contextManagement": {
      "fullContextTurnDepth": 4,
      "alwaysIncludeUserMessages": false
    }
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `fullContextTurnDepth` | number | Number of most-recent **turns** sent to the model in full. A turn is a user message and its whole assistant/tool response chain. Older turns are replaced by their summary plus a placeholder that tells the model it is a recap of a previous action. `0` or omitted disables compression. |
| `alwaysIncludeUserMessages` | boolean | When `true`, user messages are always sent in full even if their turn is outside the depth window; only assistant replies and tool interactions get summarized. |

### Notes

- `system`/`developer` (instruction) messages are never compacted. Because the boundary always falls on a user message, tool-call / tool-response pairs are never split.
- Summarizing older turns changes the prompt prefix, which can reduce the reusable prompt cache on upcoming turns.
- A `PATCH` that sends `assistantSpecialization` without `enabledTools` resets the agent's `enabledTools` to `null` (see [Update Agent](#update-agent)); include it if the agent has a tool allowlist.
- Configuration example:

```bash
curl -X PATCH "https://api.devic.ai/api/v1/agents/{agentId}" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{ "assistantSpecialization": { "contextManagement": { "fullContextTurnDepth": 4 } } }'
```
