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
| PATCH | `/api/v1/agents/threads/:threadId` | Update a thread |

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

---

## Thread States

Threads can be in one of the following states. **The values are lowercase**, and
they are compared literally — sending `COMPLETED` where `completed` is expected
will not match on a filter, and will not be understood as a state when writing.

| State | Description |
|-------|-------------|
| `queued` | Thread created and waiting to be picked up for execution |
| `processing` | Thread is actively executing |
| `completed` | Thread finished successfully |
| `failed` | Thread encountered an error |
| `terminated` | Thread was terminated by the user |
| `paused` | Thread manually paused |
| `paused_for_approval` | Thread paused awaiting user approval |
| `approval_rejected` | Approval was rejected without a retry (terminal) |
| `waiting_for_response` | Waiting on an external reply (channel message or async tool response) |
| `paused_for_resume` | Paused with a scheduled resumption |
| `handed_off` | Thread is waiting for one or more subagent threads to complete |
| `guardrail_trigger` | Execution stopped by a guardrail |
| `under_construction` | Thread is still being created |
| `limit_exceeded` | Thread blocked by a tenant usage limit before execution (terminal — not resumed by the cron). See [tenants.md](tenants.md) |

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

If a user terminates a subthread (via the Complete endpoint with state `terminated`), the parent thread is automatically notified with a tool response containing `"Terminated by the user"`, and the subthread is removed from the pending list.

---

## Agent Structure

Agents are configured through an embedded `assistantSpecialization` object that determines their behavior and capabilities.

### Key Agent Properties

| Property | Type | Description |
|----------|------|-------------|
| `_id` | string | Unique agent identifier |
| `name` | string | Agent display name |
| `description` | string | Agent description |
| `assistantSpecialization` | object | Core configuration (see below) |
| `provider` | string | LLM provider override |
| `llm` | string | Model name override |
| `disabled` | boolean | Whether agent is disabled |
| `maxExecutionInputTokens` | number | Token limit per execution |
| `maxExecutionToolCalls` | number | Max tool calls per execution |
| `evaluationConfig` | object | Evaluation settings |
| `subAgentConfig` | object | Configuration for when this agent acts as a subagent (see below) |

### assistantSpecialization Object

The `assistantSpecialization` defines the agent's behavior, tools, and capabilities:

| Property | Type | Description |
|----------|------|-------------|
| `identifier` | string | Unique specialization identifier |
| `name` | string | Specialization name |
| `presets` | string | System prompt / instructions |
| `availableToolsGroupsUids` | string[] | Tools the agent can use: built-in tool group UIDs and/or the `_id` of custom tool servers, in the same array. See [tool-servers.md](tool-servers.md) |
| `enabledTools` | string[] \| null | Explicit subset of enabled tool names. `null` enables every tool of the assigned groups, `[]` enables none |
| `model` | string | Default model |
| `provider` | string | Default LLM provider |
| `memoryDocuments` | object[] | Persistent context documents |
| `codeSnippetIds` | string[] | Code snippets available to the agent |
| `subagentsIds` | string[] | Other agents this agent can invoke |
| `contextManagement` | object | Context depth control: `{ fullContextTurnDepth?: number, alwaysIncludeUserMessages?: boolean }` — send only the most recent N turns in full and summarize older ones. See [Context Depth](#context-depth) |

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
| `archived` | boolean | - | Filter by archived status |

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
        "provider": "openai",
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
    "provider": "openai",
    "llm": "gpt-4o",
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
| `name` | string | Yes | Agent name |
| `description` | string | No | Agent description |
| `imgUrl` | string | No | Image URL for the agent |
| `assistantSpecialization` | object | No | Configuration (see below) |
| `provider` | string | No | LLM provider override |
| `llm` | string | No | Model name override |
| `maxExecutionInputTokens` | number | No | Max input tokens per execution |
| `maxExecutionToolCalls` | number | No | Max tool calls per execution |
| `maxExecutionFrequency` | number | No | Max executions in timeframe |
| `executionFrequencyIntervalMs` | number | No | Timeframe for frequency limit |
| `concurrentExecutionLimit` | number | No | Max concurrent executions |
| `agentNotificationConfig` | object | No | Notification settings |
| `evaluationConfig` | object | No | Evaluation settings |
| `subAgentConfig` | object | No | Configuration for subagent behavior (see subAgentConfig Object) |

### assistantSpecialization Object

| Field | Type | Description |
|-------|------|-------------|
| `presets` | string | System prompt / instructions |
| `availableToolsGroupsUids` | string[] | Tools the agent can use: built-in tool group UIDs and/or the `_id` of custom tool servers, in the same array. See [tool-servers.md](tool-servers.md) |
| `enabledTools` | string[] \| null | Explicit subset of enabled tool names. `null` enables every tool of the assigned groups, `[]` enables none |
| `model` | string | Default model |
| `provider` | string | Default LLM provider |
| `codeSnippetIds` | string[] | Code snippets available to the agent |
| `subagentsIds` | string[] | Other agents this agent can invoke |
| `contextManagement` | object | Context depth control: `{ fullContextTurnDepth?: number, alwaysIncludeUserMessages?: boolean }`. See [Context Depth](#context-depth) |

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
      "availableToolsGroupsUids": ["support-tools", "crm-tools"],
      "model": "gpt-4o"
    },
    "maxExecutionToolCalls": 30,
    "evaluationConfig": {
      "enabled": true
    }
  }'
```

### Response

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
| `assistantSpecialization` | object | Configuration update |
| `disabled` | boolean | Disable/enable the agent |
| `archived` | boolean | Archive/unarchive the agent |
| `provider` | string | LLM provider override |
| `llm` | string | Model name override |
| `maxExecutionInputTokens` | number | Max input tokens |
| `maxExecutionToolCalls` | number | Max tool calls |
| `evaluationConfig` | object | Evaluation settings |
| `subAgentConfig` | object | Configuration for subagent behavior (see subAgentConfig Object) |

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

```json
{
  "success": true,
  "data": {
    "success": true,
    "message": "Agent deleted successfully",
    "deletedId": "65a1b2c3d4e5f6789012345"
  }
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
| `message` | string | Yes | Initial message/task for the agent |
| `tags` | string[] | No | Tags to associate with the thread |
| `metadata` | object | No | Custom metadata for the thread. `metadata.promptTemplateParams` fills `{{placeholder}}` variables in the agent's system prompt — see [Prompt Template Parameters](#prompt-template-parameters). |
| `tenantId` | string | No | Tenant the thread belongs to (multi-tenant environments). Auto-registers the tenant on first use and attributes cost/usage to it. See [tenants.md](tenants.md). |
| `subtenantId` | string | No | End user/entity inside the tenant. When omitted it is derived from `metadata.subtenantMetadata.id` or the legacy `metadata.userId`. Used for per-subtenant cost attribution and usage limits. |

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
    "tags": ["sales", "quarterly-report"],
    "tenantId": "acme-corp",
    "subtenantId": "user_67890"
  }'
```

### Response

```json
{
  "success": true,
  "data": {
    "threadId": "thread-456",
    "agentId": "agent-123",
    "state": "processing",
    "message": "Analyze the Q4 sales report and create a summary",
    "createdAt": "2024-01-15T10:00:00.000Z"
  }
}
```

### Error Responses

| Status | Description |
|--------|-------------|
| 404 | Agent not found |
| 429 | Tenant usage limit exceeded — `error: "TENANT_LIMIT_EXCEEDED"` (see below) |

When the thread's `tenantId`/`subtenantId` has a configured usage limit that is already consumed, thread creation is blocked **before** the LLM call and returns HTTP `429` with a structured body (`details.blockingRule`/`current`/`limit`/`resetsAt`) plus a `Retry-After` header — identical in shape to the assistant 429 documented in [assistants.md](assistants.md). The blocked thread is also persisted with state `limit_exceeded` (terminal; the resumption cron does not retry it) so it is visible in the thread list. See [tenants.md](tenants.md) for the usage-limit model.

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
| `state` | string | - | Filter by thread state |
| `startDate` | string | - | Start date filter (ISO string) |
| `endDate` | string | - | End date filter (ISO string) |
| `dateOrder` | string | - | Sort by date (`asc` or `desc`) |
| `durationOrder` | string | - | Sort by duration (`asc` or `desc`) |
| `tokensOrder` | string | - | Sort by tokens (`asc` or `desc`) |
| `tags` | string | - | Comma-separated tags (matches ANY tag) |
| `tenantId` | string | - | Filter by tenant ID |
| `subtenantId` | string | - | Filter by subtenant ID (end user/entity inside a tenant) |

### Example Request

```bash
curl -X GET "https://api.devic.ai/api/v1/agents/agent-123/threads?state=completed&limit=20&tags=urgent,review" \
  -H "Authorization: Bearer devic-your-api-key"
```

### Response

```json
{
  "success": true,
  "data": {
    "threads": [
      {
        "threadId": "thread-456",
        "agentId": "agent-123",
        "state": "completed",
        "message": "Task description",
        "createdAt": "2024-01-15T10:00:00.000Z",
        "completedAt": "2024-01-15T10:15:00.000Z",
        "duration": 900000,
        "totalTokens": 5432
      }
    ],
    "total": 45,
    "offset": 0,
    "limit": 20,
    "hasMore": true
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
| `withTasks` | boolean | false | Include tasks in the response |

### Response

```json
{
  "success": true,
  "data": {
    "threadId": "thread-456",
    "agentId": "agent-123",
    "state": "processing",
    "message": "Analyze the Q4 sales report",
    "createdAt": "2024-01-15T10:00:00.000Z",
    "tags": ["sales"],
    "steps": [
      {
        "stepNumber": 1,
        "action": "Reading sales data",
        "status": "completed"
      },
      {
        "stepNumber": 2,
        "action": "Generating analysis",
        "status": "in_progress"
      }
    ],
    "tasks": [...],
    "isSubthread": false,
    "pendingHandOffSubThreadIds": [],
    "handOffSubThreadIds": []
  }
}
```

### Subthread Properties

When a thread is a subthread (created via subagent handoff), the following additional properties are present:

| Property | Type | Description |
|----------|------|-------------|
| `isSubthread` | boolean | Whether this thread is a subthread created by a parent handoff |
| `parentThreadId` | string | ID of the parent thread (if this is a subthread) |
| `parentAgentId` | string | ID of the parent agent (if this is a subthread) |
| `subThreadToolCallId` | string | Tool call ID in the parent thread that triggered this subthread |

For parent threads that have handed off to subagents:

| Property | Type | Description |
|----------|------|-------------|
| `pendingHandOffSubThreadIds` | string[] | IDs of subthreads still executing |
| `handOffSubThreadIds` | string[] | All subthread IDs ever created during handoffs |
```

---

## Update Thread

Updates a thread with the provided data.

```
PATCH /api/v1/agents/threads/:threadId
```

### Request Body

| Field | Type | Description |
|-------|------|-------------|
| `tags` | string[] | Update thread tags |
| `metadata` | object | Update thread metadata |

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

## Handle Approval

Processes an approval request for a thread that requires user confirmation.

```
POST /api/v1/agents/threads/:threadId/approval
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `approved` | boolean | Yes | Whether to approve or reject |
| `message` | string | No | Optional message or feedback |

### Example Request

```bash
curl -X POST "https://api.devic.ai/api/v1/agents/threads/thread-456/approval" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "approved": true,
    "message": "Proceed with the data export"
  }'
```

---

## Complete Thread

Marks a thread as completed with the specified final state.

```
POST /api/v1/agents/threads/:threadId/complete
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `state` | string | Yes | Final state — one of `completed`, `failed`, `terminated` |

---

## Pause Thread

Pauses the execution of a running thread.

```
POST /api/v1/agents/threads/:threadId/pause
```

### Response

```json
{
  "success": true,
  "data": {
    "threadId": "thread-456",
    "state": "paused",
    "pausedAt": "2024-01-15T10:10:00.000Z"
  }
}
```

---

## Resume Thread

Resumes the execution of a paused thread.

```
POST /api/v1/agents/threads/:threadId/resume
```

---

## Evaluate Thread

Triggers evaluation of a completed thread based on multiple criteria.

```
POST /api/v1/agents/threads/:threadId/evaluate
```

### Response

```json
{
  "success": true,
  "data": {
    "evaluation": {
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

Retrieves the evaluation results for a thread.

```
GET /api/v1/agents/threads/:threadId/evaluation
```

---

## Get All Thread Evaluations

Retrieves all evaluation results for a thread.

```
GET /api/v1/agents/threads/:threadId/evaluations
```

---

## Get Agent Daily Costs

Retrieves daily cost breakdown for an agent.

```
GET /api/v1/agents/agents/:agentId/costs/daily
```

### Query Parameters

| Parameter | Type | Format | Description |
|-----------|------|--------|-------------|
| `startDate` | string | YYYY-MM-DD | Start date |
| `endDate` | string | YYYY-MM-DD | End date |

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
      "date": "2024-01-15",
      "totalCost": 12.50,
      "inputTokens": 50000,
      "outputTokens": 25000,
      "threadCount": 15
    }
  ]
}
```

---

## Get Agent Monthly Costs

Retrieves monthly cost breakdown for an agent.

```
GET /api/v1/agents/agents/:agentId/costs/monthly
```

### Query Parameters

| Parameter | Type | Format | Description |
|-----------|------|--------|-------------|
| `startMonth` | string | YYYY-MM | Start month |
| `endMonth` | string | YYYY-MM | End month |

---

## Get Cost Summary

Retrieves cost summary for today and current month.

```
GET /api/v1/agents/agents/:agentId/costs/summary
```

### Response

```json
{
  "success": true,
  "data": {
    "today": {
      "date": "2024-01-15",
      "totalCost": 5.25,
      "inputTokens": 21000,
      "outputTokens": 10500,
      "threadCount": 8
    },
    "currentMonth": {
      "month": "2024-01",
      "totalCost": 125.75,
      "inputTokens": 503000,
      "outputTokens": 251500,
      "threadCount": 180
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

```json
{
  "autoEvaluate": true,
  "evaluationModel": "gpt-4",
  "customCriteria": [
    {
      "name": "accuracy",
      "description": "Evaluate the accuracy of data analysis",
      "weight": 2
    }
  ]
}
```

---

## Get Evaluation Config

Retrieves evaluation configuration for an agent.

```
GET /api/v1/agents/agents/:agentId/evaluation-config
```

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

```json
{
  "success": true,
  "data": ["urgent", "review", "sales", "quarterly", "approved"]
}
```

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
- Configuration example:

```bash
curl -X PATCH "https://api.devic.ai/api/v1/agents/{agentId}" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{ "assistantSpecialization": { "contextManagement": { "fullContextTurnDepth": 4 } } }'
```
