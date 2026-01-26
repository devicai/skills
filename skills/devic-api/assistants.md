# Assistants API

The Assistants API allows you to interact with AI assistants that can process messages, maintain conversation history, and provide specialized capabilities.

## Endpoints Overview

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/assistants` | List all assistant specializations |
| GET | `/api/v1/assistants/:identifier` | Get specific assistant |
| POST | `/api/v1/assistants/:identifier/messages` | Send message to assistant (sync or async) |
| GET | `/api/v1/assistants/:identifier/chats` | List chat histories for assistant |
| GET | `/api/v1/assistants/:identifier/chats/:chatUid` | Get specific chat history |
| GET | `/api/v1/assistants/:identifier/chats/:chatUid/realtime` | Get real-time chat history (for async polling) |
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
