# Assistants API

The Assistants API allows you to interact with AI assistants that can process messages, maintain conversation history, and provide specialized capabilities.

## Endpoints Overview

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/assistants` | List all assistant specializations |
| GET | `/api/v1/assistants/:identifier` | Get specific assistant |
| POST | `/api/v1/assistants` | Create a new assistant |
| PATCH | `/api/v1/assistants/:identifier` | Update an assistant |
| DELETE | `/api/v1/assistants/:identifier` | Delete an assistant |
| POST | `/api/v1/assistants/:identifier/messages` | Send message to assistant (sync or async) |
| GET | `/api/v1/assistants/:identifier/chats` | List chat histories for assistant |
| GET | `/api/v1/assistants/:identifier/chats/:chatUid` | Get specific chat history |
| GET | `/api/v1/assistants/:identifier/chats/:chatUid/realtime` | Get real-time chat history (for async polling) |
| POST | `/api/v1/assistants/:identifier/chats/:chatUid/stop` | Stop an in-progress async chat |
| POST | `/api/v1/assistants/:identifier/chats/:chatUid/tool-response` | Submit tool responses (Model Interface Protocol) |
| POST | `/api/v1/assistants/chats` | Get all chat histories with filters |
| GET | `/api/v1/assistants/tags` | Get unique tags from chat histories |

---

## Assistant Specialization Structure

Assistant Specializations define how an assistant behaves, what tools it can use, and how it processes messages.

### Key Properties

| Property | Type | Description |
|----------|------|-------------|
| `identifier` | string | Unique identifier used in API paths |
| `name` | string | Display name |
| `description` | string | Description of the assistant's purpose |
| `presets` | string | System prompt / instructions |
| `availableToolsGroupsUids` | string[] | Tool group IDs the assistant can use |
| `enabledTools` | string[] | Explicit subset of enabled tool names |
| `model` | string | Default LLM model |
| `provider` | string | Default LLM provider |
| `memoryDocuments` | object[] | Persistent context documents |
| `inputDelayMs` | number | Message buffer delay in ms for async mode (500–30000). See [Message Buffering](#message-buffering-input-delay) |
| `accessConfiguration` | object | Visibility and external access settings |

### Tool Access

Assistants access tools through `availableToolsGroupsUids`:

1. Each UID references a **Tools Group**
2. Tools Groups can include built-in tools or reference a **Tool Server**
3. When processing messages, the assistant can invoke available tools
4. Use `enabledTools` to restrict access to specific tools

### Access Configuration

Controls visibility and external API access:

```json
{
  "accessConfiguration": {
    "externalAccess": true,
    "visibilityByRole": ["admin", "user"]
  }
}
```

- `externalAccess: true` - Makes the assistant available via the public API
- `visibilityByRole` - Controls which user roles can see/use the assistant in the dashboard

---

## List Assistant Specializations

Retrieves a list of all available assistant specializations (both default and custom).

```
GET /api/v1/assistants
```

### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `external` | boolean | No | Include only assistants configured for external access |

### Response

```json
{
  "success": true,
  "data": [
    {
      "identifier": "default",
      "name": "General Assistant",
      "description": "A general-purpose AI assistant",
      "enabled": true,
      "externalEnabled": true
    },
    {
      "identifier": "code-helper",
      "name": "Code Assistant",
      "description": "Specialized in coding tasks",
      "enabled": true,
      "externalEnabled": false
    }
  ]
}
```

### Example

```bash
curl -X GET "https://api.devic.ai/api/v1/assistants?external=true" \
  -H "Authorization: Bearer devic-your-api-key"
```

---

## Get Assistant Specialization

Retrieves detailed information about a specific assistant specialization.

```
GET /api/v1/assistants/:identifier
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `identifier` | string | The unique identifier of the assistant |

### Response

```json
{
  "success": true,
  "data": {
    "identifier": "default",
    "name": "General Assistant",
    "description": "A general-purpose AI assistant",
    "enabled": true,
    "externalEnabled": true,
    "systemPrompt": "You are a helpful assistant...",
    "model": "gpt-4",
    "provider": "openai"
  }
}
```

### Error Responses

| Status | Description |
|--------|-------------|
| 404 | Assistant specialization not found |

---

## Create Assistant

Creates a new assistant specialization.

```
POST /api/v1/assistants
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | **Yes** | Assistant display name |
| `description` | string | No | Description of the assistant's purpose |
| `presets` | string | No | System prompt / instructions |
| `model` | string | No | Default LLM model |
| `provider` | string | No | Default LLM provider |
| `imgUrl` | string | No | Image URL for the assistant |
| `state` | string | No | State: `active`, `inactive`, or `coming_soon` |
| `availableToolsGroupsUids` | string[] | No | Tool group UIDs the assistant can use |
| `enabledTools` | string[] | No | Explicit subset of enabled tool names |
| `accessConfiguration` | object | No | `{ externalAccess?: boolean, visibilityByRole?: string[] }` |
| `widgetConfiguration` | object | No | `{ enabled?: boolean, sourcesWhiteList?: string[], color?: string, welcomeMessage?: string }` |
| `memoryDocuments` | object[] | No | RAG memory documents `[{ genericDocument?, name?, summary? }]` |
| `structuredOutput` | object | No | JSON schema configuration for structured output |
| `guardrailsConfiguration` | object | No | `{ enabled?: boolean, guardrails?: any }` |
| `codeSnippetIds` | string[] | No | Code snippet IDs available to the assistant |
| `availableSkillIds` | string[] | No | Skill IDs the assistant can use |
| `subagentsIds` | string[] | No | Subagent IDs the assistant can invoke |
| `maxChatMessages` | number | No | Maximum chat messages to include in context |
| `maxToolResponseInputTokens` | number | No | Maximum input tokens for tool responses |
| `inputDelayMs` | number | No | Message buffer delay in ms for async mode (min: 500, max: 30000). See [Message Buffering](#message-buffering-input-delay) |

### Response (201 Created)

Returns the full assistant detail object (`AssistantDetailResponseDto`):

```json
{
  "success": true,
  "data": {
    "_id": "60f7b2a1c3e4d5f6a7b8c9d0",
    "identifier": "a1289b11-fe5a-4c50-8e90-138850651932",
    "name": "Customer Support Assistant",
    "description": "Helps with customer inquiries",
    "presets": "You are a helpful support agent...",
    "model": "gpt-4.1-mini",
    "provider": "openai",
    "availableToolsGroupsUids": [],
    "codeSnippetIds": [],
    "availableSkillIds": [],
    "subagentsIds": [],
    "creationTimestampMs": 1772547299956
  }
}
```

### Example

```bash
curl -X POST "https://api.devic.ai/api/v1/assistants" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Sales Assistant",
    "description": "Helps with sales inquiries",
    "presets": "You are a sales expert...",
    "model": "gpt-4.1-mini",
    "provider": "openai"
  }'
```

---

## Update Assistant

Updates an existing assistant specialization. Supports partial updates — only include fields you want to change.

```
PATCH /api/v1/assistants/:identifier
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `identifier` | string | The unique identifier of the assistant |

### Request Body

All fields from `CreateAssistantDto` are accepted, but all are optional. Only provided fields will be updated.

### Response (200 OK)

Returns the updated assistant detail object (`AssistantDetailResponseDto`), same structure as the Create response.

### Example

```bash
curl -X PATCH "https://api.devic.ai/api/v1/assistants/a1289b11-fe5a-4c50-8e90-138850651932" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Sales Assistant (Updated)",
    "model": "gpt-4.1"
  }'
```

### Error Responses

| Status | Description |
|--------|-------------|
| 404 | Assistant specialization not found |

---

## Delete Assistant

Deletes an assistant specialization and performs cleanup (removes RAG memory documents and internal API key).

```
DELETE /api/v1/assistants/:identifier
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `identifier` | string | The unique identifier of the assistant |

### Response (200 OK)

```json
{
  "success": true,
  "message": "Assistant deleted successfully",
  "deletedId": "60f7b2a1c3e4d5f6a7b8c9d0"
}
```

### Error Responses

| Status | Description |
|--------|-------------|
| 404 | Assistant specialization not found |

---

## Send Message to Assistant

Sends a message to the specified assistant and returns the response. Supports both synchronous (default) and asynchronous processing modes.

```
POST /api/v1/assistants/:identifier/messages
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `identifier` | string | The unique identifier of the assistant |

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `async` | boolean | false | Enable async mode. Returns chatUid immediately and processes in background. |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `message` | string | Yes | The message to send to the assistant |
| `chatUid` | string | No | Chat conversation ID (for continuing conversations) |
| `provider` | string | No | Override the LLM provider (e.g., "openai", "anthropic") |
| `model` | string | No | Override the model (e.g., "gpt-4", "claude-3-opus") |
| `tags` | string[] | No | Tags to associate with this chat |
| `metadata` | object | No | Custom metadata to store with the chat |

### Synchronous Mode (Default)

When `async` is not set or `false`, the request blocks until the assistant finishes processing and returns the full response.

#### Example Request (Sync)

```bash
curl -X POST "https://api.devic.ai/api/v1/assistants/default/messages" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Explain the concept of recursion in programming",
    "chatUid": "chat-123",
    "tags": ["programming", "education"]
  }'
```

#### Response (Sync)

```json
{
  "success": true,
  "data": [
    {
      "role": "user",
      "content": "Explain the concept of recursion in programming"
    },
    {
      "role": "assistant",
      "content": "Recursion is a programming concept where a function calls itself..."
    }
  ]
}
```

### Asynchronous Mode

When `async=true`, the request returns immediately with a `chatUid`. Use this for long-running requests or when you want to avoid blocking. Poll the realtime endpoint to check for results.

#### Example Request (Async)

```bash
curl -X POST "https://api.devic.ai/api/v1/assistants/default/messages?async=true" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Analyze this complex dataset and provide insights"
  }'
```

#### Response (Async)

```json
{
  "success": true,
  "data": {
    "chatUid": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

After receiving the `chatUid`, poll the realtime endpoint to get the processing status and results:

```bash
curl -X GET "https://api.devic.ai/api/v1/assistants/default/chats/550e8400-e29b-41d4-a716-446655440000/realtime" \
  -H "Authorization: Bearer devic-your-api-key"
```

### Error Responses

| Status | Description |
|--------|-------------|
| 400 | Invalid input data, provider not configured, or model not supported |
| 404 | Assistant specialization not found |
| 409 | Chat is currently processing (only when `inputDelayMs` is configured and chat status is `processing`) |

---

## List Chat Histories for Assistant

Retrieves a paginated list of chat histories for the specified assistant.

```
GET /api/v1/assistants/:identifier/chats
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `identifier` | string | The unique identifier of the assistant |

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `offset` | number | 0 | Number of items to skip |
| `limit` | number | 10 | Maximum items to return (max: 100) |
| `omitContent` | boolean | false | Exclude chat content from response (metadata only) |

### Response

```json
{
  "success": true,
  "data": {
    "chats": [
      {
        "chatUid": "chat-123",
        "assistantSpecializationIdentifier": "default",
        "createdAt": "2024-01-15T10:00:00.000Z",
        "updatedAt": "2024-01-15T10:30:00.000Z",
        "tags": ["support"],
        "messages": [...]
      }
    ],
    "total": 25,
    "offset": 0,
    "limit": 10,
    "hasMore": true
  }
}
```

---

## Get Chat History

Retrieves the chat history for a specific conversation.

```
GET /api/v1/assistants/:identifier/chats/:chatUid
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `identifier` | string | The unique identifier of the assistant |
| `chatUid` | string | The unique identifier of the chat conversation |

### Response

```json
{
  "success": true,
  "data": {
    "chatUid": "chat-123",
    "assistantSpecializationIdentifier": "default",
    "createdAt": "2024-01-15T10:00:00.000Z",
    "updatedAt": "2024-01-15T10:30:00.000Z",
    "tags": ["support"],
    "metadata": {},
    "messages": [
      {
        "role": "user",
        "content": "Hello"
      },
      {
        "role": "assistant",
        "content": "Hi! How can I help you today?"
      }
    ]
  }
}
```

### Error Responses

| Status | Description |
|--------|-------------|
| 404 | Chat history not found or does not belong to the specified assistant |

---

## Get Real-Time Chat History

Retrieves the real-time chat history from Redis cache. Use this endpoint to poll for results when using async mode.

```
GET /api/v1/assistants/:identifier/chats/:chatUid/realtime
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `identifier` | string | The unique identifier of the assistant |
| `chatUid` | string | The unique identifier of the chat conversation |

### Response

```json
{
  "success": true,
  "data": {
    "chatUID": "550e8400-e29b-41d4-a716-446655440000",
    "clientUID": "client-123",
    "chatHistory": [
      {
        "role": "user",
        "content": { "message": "Analyze this data" }
      },
      {
        "role": "assistant",
        "content": { "message": "Based on the analysis..." }
      }
    ],
    "status": "completed",
    "lastUpdatedAt": 1705312200000
  }
}
```

### Status Values

| Status | Description |
|--------|-------------|
| `buffering` | Messages are being collected before processing (see [Message Buffering](#message-buffering-input-delay)) |
| `processing` | Message is currently being processed by the assistant |
| `completed` | Processing finished successfully |
| `error` | An error occurred during processing |

### Polling Pattern for Async Mode

When using async mode, implement a polling pattern:

```javascript
async function waitForResult(identifier, chatUid) {
  const maxAttempts = 60; // 60 seconds timeout
  const pollInterval = 1000; // 1 second

  for (let i = 0; i < maxAttempts; i++) {
    const response = await fetch(
      `https://api.devic.ai/api/v1/assistants/${identifier}/chats/${chatUid}/realtime`,
      { headers: { "Authorization": "Bearer devic-your-api-key" } }
    );
    const result = await response.json();

    if (result.data.status === 'completed') {
      return result.data.chatHistory;
    }

    if (result.data.status === 'error') {
      throw new Error('Processing failed');
    }

    // 'buffering' and 'processing' — keep polling

    await new Promise(resolve => setTimeout(resolve, pollInterval));
  }

  throw new Error('Timeout waiting for response');
}
```

### Error Responses

| Status | Description |
|--------|-------------|
| 404 | Real-time chat history not found |

---

## Stop an In-Progress Async Chat

Requests a graceful stop of an in-progress async chat completion. The current LLM call or tool execution will finish, and the chat will be marked as completed with the history accumulated so far. Use the realtime endpoint to poll until the status changes to `completed`.

```
POST /api/v1/assistants/:identifier/chats/:chatUid/stop
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `identifier` | string | The unique identifier of the assistant |
| `chatUid` | string | The unique identifier of the chat conversation |

### Response

```json
{
  "success": true,
  "data": {
    "chatUid": "550e8400-e29b-41d4-a716-446655440000",
    "message": "Stop requested. The chat will finish its current step and complete."
  }
}
```

### Behavior

- The stop is **graceful**: the loop finishes whatever step it is currently executing (LLM call or tool execution) and then exits.
- After stopping, the realtime status is set to `completed` with the chat history accumulated up to that point.
- Continue polling the realtime endpoint — the chat will transition from `processing` to `completed`.

### Error Responses

| Status | Description |
|--------|-------------|
| 400 | Chat is not currently processing |

### Example

```bash
curl -X POST "https://api.devic.ai/api/v1/assistants/default/chats/550e8400-e29b-41d4-a716-446655440000/stop" \
  -H "Authorization: Bearer devic-your-api-key"
```

---

## Get All Chat Histories with Filters

Retrieves a paginated list of chat histories across all assistants with optional filters.

```
POST /api/v1/assistants/chats
```

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `offset` | number | 0 | Number of items to skip |
| `limit` | number | 10 | Maximum items to return (max: 100) |
| `omitContent` | boolean | false | Exclude chat content from response |

### Request Body (Filters)

Send an empty JSON object `{}` if no filters are needed.

| Field | Type | Description |
|-------|------|-------------|
| `assistantIdentifier` | string | Filter by assistant identifier |
| `tags` | string[] | Filter by tags |
| `startDate` | string | Filter by start date (ISO string) |
| `endDate` | string | Filter by end date (ISO string) |

### Example Request

```bash
curl -X POST "https://api.devic.ai/api/v1/assistants/chats?limit=20" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "tags": ["support", "urgent"],
    "startDate": "2024-01-01T00:00:00.000Z"
  }'
```

---

## Get Unique Tags

Retrieves a list of all unique tags used across chat histories.

```
GET /api/v1/assistants/tags
```

### Response

```json
{
  "success": true,
  "data": ["support", "sales", "technical", "urgent", "feedback"]
}
```

---

## Provider and Model Options

When sending messages, you can specify custom providers and models:

### Supported Providers

- `openai` - OpenAI API
- `anthropic` - Anthropic API
- `azure` - Azure OpenAI
- `google` - Google AI (Gemini)

### Example with Custom Model

```bash
curl -X POST "https://api.devic.ai/api/v1/assistants/default/messages" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Write a haiku about programming",
    "provider": "anthropic",
    "model": "claude-3-opus-20240229"
  }'
```

**Note:** The provider must be configured in your account settings before use. The model must belong to the specified provider.

---

## Message Buffering (Input Delay)

When integrating assistants with external chat platforms (WhatsApp, Telegram, etc.), users often send multiple short messages in quick succession. Without buffering, each message triggers an independent LLM call, producing fragmented responses and unnecessary costs.

The `inputDelayMs` field on an assistant configures a **debounce window**. When set, consecutive messages sent in async mode are accumulated in a buffer and merged into a single processing call after the delay expires.

### How It Works

1. A message arrives via `POST /:identifier/messages?async=true`
2. If the assistant has `inputDelayMs` configured, the message is buffered instead of processed immediately
3. The realtime status is set to `buffering`
4. Each new message within the delay window resets the timer
5. Once the timer expires (no new messages for `inputDelayMs` milliseconds), all buffered messages are merged and processed as one
6. Status transitions: `buffering` → `processing` → `completed`

### Merging Behavior

- **Text**: Messages are joined with newline (`\n`) in arrival order
- **Files**: All files from all buffered messages are flattened into a single array

### Configuration

Set `inputDelayMs` when creating or updating an assistant:

```bash
curl -X PATCH "https://api.devic.ai/api/v1/assistants/my-whatsapp-bot" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{ "inputDelayMs": 5000 }'
```

To disable buffering, set it to `undefined` or `0`, or omit the field.

### Constraints

- **Async mode only**: Buffering is ignored in synchronous mode
- **Range**: 500 – 30000 ms
- **Conflict during processing**: Sending a message while status is `processing` returns `409 Conflict` — the client must wait for completion before sending new messages
- **Multi-instance safe**: Uses Redis distributed locks so only one instance processes the buffer
