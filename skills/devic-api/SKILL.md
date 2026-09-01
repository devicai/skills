---
name: devic-api
description: Devic AI Platform API reference for assistants, agents, tool servers, knowledge documents and skills, tenants (multi-tenant cost & usage limits) and tenant sessions (signed tokens for browser credentials). Use when working with Devic API endpoints, creating integrations, or building applications that interact with the Devic platform.
---

# Devic API

Devic is an AI platform that enables developers to build, deploy, and manage AI-powered assistants and autonomous agents. The platform provides a comprehensive REST API for programmatic access to all platform features.

## Base URL

```
https://api.devic.ai
```


## Authentication

All API requests require authentication using a JWT Bearer token. API keys follow the pattern. Generate your API key from the Devic dashboard on https://app.devic.ai/api-keys:

```
devic-{random_string}
```

Example: `devic-wengpqengqp1234abcd`

### Request Header

Include the API key in the Authorization header:

```
Authorization: Bearer devic-your-api-key-here
```

### Credentials that reach a browser

An API key states *which account* is calling, never *which of your customers*.
Anywhere the key can be read — a bundle, a mobile app — the tenant beside it is
a claim anyone can edit. For those, mint a **tenant session** on your server and
send that as the bearer token instead: it carries the tenant in a signed claim
and is imposed on every query. A key can also be restricted to `signed` mode, in
which minting sessions is the only thing it can do. See
[tenant-sessions.md](tenant-sessions.md).

### Example Request

```bash
curl -X GET "https://api.devic.ai/api/v1/assistants" \
  -H "Authorization: Bearer devic-your-api-key-here" \
  -H "Content-Type: application/json"
```

## Response Format

All API responses follow a standardized format:

### Success Response

```json
{
  "success": true,
  "data": { ... },
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

### Error Response

```json
{
  "success": false,
  "error": {
    "message": "Error description",
    "code": "ERROR_CODE"
  },
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

## Architecture Overview

Understanding how the core entities relate to each other is essential for building integrations with the Devic platform.

### Entity Relationships

```
┌─────────────────────────────────────────────────────────────────┐
│                         Tool Server                             │
│  (External API integration with tool definitions)               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ referenced by
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Tools Group                               │
│  (Logical grouping of tools - built-in or from tool server)     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ availableToolsGroupsUids
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              Assistant Specialization                           │
│  (Configuration: presets, tools, model, provider, etc.)         │
└─────────────────────────────────────────────────────────────────┘
                    /                   \
                   /                     \
                  ▼                       ▼
┌──────────────────────┐     ┌──────────────────────────────────┐
│     Assistant        │     │            Agent                  │
│  (Chat interface)    │     │  (Autonomous execution threads)   │
└──────────────────────┘     └──────────────────────────────────┘
         │                              │           │
         ▼                              ▼           │ hand_off_subagent
┌──────────────────────┐     ┌──────────────────────┴───────────┐
│   Chat Histories     │     │          Threads                  │
│  (Conversations)     │     │  (Execution sessions with tasks)  │
└──────────────────────┘     │                                   │
                              │  ┌─────────────────────────────┐ │
                              │  │  Subthreads                  │ │
                              │  │  (Child executions from      │ │
                              │  │   subagent handoffs)         │ │
                              │  └─────────────────────────────┘ │
                              └──────────────────────────────────┘
```

### Key Concepts

**Assistant Specialization**: The core configuration object that defines how an AI assistant or agent behaves. It includes:
- `presets` - System prompt instructions. May contain `{{placeholder}}` variables that are filled at request time via `metadata.promptTemplateParams` (see [assistants.md](assistants.md#prompt-template-parameters) and [agents.md](agents.md#prompt-template-parameters))
- `availableToolsGroupsUids` - Array of tool group identifiers that determine available tools
- `enabledTools` - Optional allowlist of tool identifiers across the assigned groups. `null` (or absent) enables every tool of those groups, `[]` enables none. On a partial update, omit it to leave the current selection untouched
- `model` / `provider` - Default LLM configuration
- `memoryDocuments` - Persistent context documents

**Agents**: Autonomous executors that use an embedded `assistantSpecialization` configuration. When you create an agent, you configure its specialization which determines:
- What tools it can use (via `availableToolsGroupsUids`)
- How it behaves (via `presets`)
- What LLM powers it (via `model`/`provider`)

**Tool Servers**: External API integrations that define tools. A tool server:
1. Is created via the Tool Servers API
2. Its `_id` is added to an assistant/agent's `availableToolsGroupsUids`
3. The tools become available for that assistant/agent to use

That array also holds the UIDs of Devic's built-in tool groups, so custom and
built-in tools are mixed in the same list.

### Workflow Example

To give an agent access to a custom CRM API:

1. **Create Tool Server** with your CRM endpoint definitions
2. **Copy its `_id`** from the response
3. **Update the Agent's assistantSpecialization** to include that `_id` in `availableToolsGroupsUids`
4. The agent can now call CRM tools during execution

## API Sections

The Devic API is organized into three main sections:

### 1. Assistants API

Manage AI assistants that can process messages and maintain conversation history.

- Create, update, and delete assistant specializations
- List and retrieve assistant specializations
- Send messages to assistants
- Manage chat histories

**Base path:** `/api/v1/assistants`

For detailed documentation, see [assistants.md](assistants.md).

### 2. Agents API

Manage autonomous agents that can execute multi-step tasks with tool access.

- Create and manage agent execution threads
- Handle approval workflows
- Pause, resume, and complete agent executions
- Evaluate agent performance
- Track agent costs

**Base path:** `/api/v1/agents`

For detailed documentation, see [agents.md](agents.md).

### 3. Tool Servers API

Configure external tool integrations that agents and assistants can use.

- Create and manage tool servers
- Define tool specifications
- Test tool configurations
- Clone tool servers

**Base path:** `/api/v1/tool-servers`

For detailed documentation, see [tool-servers.md](tool-servers.md).

### 4. Files API

Upload files and get shareable download URLs. Used for attaching files to assistant messages.

- Upload files via multipart/form-data
- Get download URLs for uploaded files

**Base path:** `/api/v1/files`

For detailed documentation, see [files.md](files.md).

### 5. Feedback API

Collect user feedback on AI responses from both assistants and agents.

- Submit feedback (positive/negative) on chat messages
- Submit feedback on agent thread messages
- Retrieve feedback history for chats and threads
- Support for structured feedback data (ratings, categories, etc.)

**Base paths:**
- Chat feedback: `/api/v1/assistants/:identifier/chats/:chatUid/feedback`
- Thread feedback: `/api/v1/agents/threads/:threadId/feedback`

For detailed documentation, see [feedback.md](feedback.md).

### 6. Whisper API (Speech-to-Text)

Transcribe audio to text using OpenAI Whisper with Devic's own OpenAI key.

- Transcribe an audio binary (multipart) or a download URL
- Optional language hint, message and chat linkage
- Returns a `transcriptId` to attach to the resulting assistant message

**Base path:** `/api/v1/whisper`

For detailed documentation, see [whisper.md](whisper.md).

### 7. Tenants API

Manage the tenants and subtenants of your account for multi-tenant products, read aggregated cost/usage, and enforce usage limits (token/cost caps) with reusable tiers (plans).

- Auto-register tenants/subtenants by passing `tenantId`/`subtenantId` on messages and threads
- List/update/delete tenants and subtenants
- Read per-tenant cost time series and live chat/thread stats
- Read usage limits, current consumption and durable history (read-only, devic-ui preset)
- Reset counters and change tier from billing/checkout webhooks (full keys only)
- Usage limits block execution with HTTP `429 TENANT_LIMIT_EXCEEDED`

**Base paths:** `/api/v1/tenants` · `/api/v1/tenant-usage` · `/api/v1/tenant-admin`

For detailed documentation, see [tenants.md](tenants.md).

### 8. Documents & Skills API

Manage the knowledge your assistants and agents read at runtime, and the skills built on top of it.

- Create, version and organise markdown documents and folders
- Attach documents or folders to an assistant, an agent or an environment
- Publish a document or folder as a **skill**: its name and description go into the system prompt and the model loads the full instructions on demand
- Scaffold a folder-skill (folder + `SKILL.md`) in one call, browse the skills catalog, download a skill tree

**Base paths:** `/api/v1/documents` · `/api/v1/document-folders`

For detailed documentation, see [documents.md](documents.md).

### 9. Tenant Sessions API

Short-lived tokens that **prove** which of your customers is calling, for
credentials that end up in a browser. Your backend mints one from your own
login; the page then carries the session instead of an API key.

- Mint a session for a tenant/subtenant, with a lifetime you choose (1 h by default, 12 h max)
- A session reaches only what an end user does — chatting, its own conversations, attachments, its own limits, its own connected apps
- The identity in the token is imposed on path, query and body, so another tenant cannot be reached by editing a request
- An API key can be put in `signed` mode, where minting sessions is the **only** thing it can do

**Base path:** `/api/v1/tenant-sessions`

For detailed documentation, see [tenant-sessions.md](tenant-sessions.md).

### 10. Built-in Tools Reference

List of all built-in tool groups with their UIDs, ready to use in `availableToolsGroupsUids`.

For detailed documentation, see [built-in-tools.md](built-in-tools.md).

## Pagination

List endpoints support pagination with the following query parameters:

| Parameter | Type | Default | Max | Description |
|-----------|------|---------|-----|-------------|
| `offset` | number | 0 | - | Number of items to skip |
| `limit` | number | 10 | 100 | Maximum items to return |

Paginated responses include metadata:

```json
{
  "data": [...],
  "total": 50,
  "offset": 0,
  "limit": 10,
  "hasMore": true
}
```

## Rate Limits

API rate limits are applied per API key. Contact support for rate limit details specific to your plan.

## Common HTTP Status Codes

| Code | Description |
|------|-------------|
| 200 | Success |
| 201 | Created |
| 400 | Bad Request - Invalid input data |
| 401 | Unauthorized - Invalid or missing API key; also a route a tenant session may not call, or a `signed` key used without a session (see [tenant-sessions.md](tenant-sessions.md)) |
| 403 | Forbidden - The credential proved one identity and the request asked for another (tenant sessions) |
| 404 | Not Found - Resource does not exist |
| 429 | Too Many Requests - Rate limit, or tenant usage limit (`TENANT_LIMIT_EXCEEDED`, see [tenants.md](tenants.md)) |
| 500 | Internal Server Error |

## Quick Start Examples

### Create an assistant

```bash
curl -X POST "https://api.devic.ai/api/v1/assistants" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My Assistant",
    "description": "A helpful assistant",
    "model": "gpt-4.1-mini",
    "provider": "openai"
  }'
```

### Send a message to an assistant

```bash
curl -X POST "https://api.devic.ai/api/v1/assistants/default/messages" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Hello, how can you help me?",
    "chatUid": "optional-chat-id"
  }'
```

### Create an agent thread

```bash
curl -X POST "https://api.devic.ai/api/v1/agents/{agentId}/threads" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Analyze the sales data and create a report"
  }'
```

### List tool servers

```bash
curl -X GET "https://api.devic.ai/api/v1/tool-servers?limit=10" \
  -H "Authorization: Bearer devic-your-api-key"
```
