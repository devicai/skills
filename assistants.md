# Assistants API

The Assistants API allows you to interact with AI assistants that can process messages, maintain conversation history, and provide specialized capabilities.

## Endpoints Overview

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/assistants` | List all assistant specializations |
| GET | `/api/v1/assistants/:identifier` | Get specific assistant |
| POST | `/api/v1/assistants/:identifier/messages` | Send message to assistant |
| GET | `/api/v1/assistants/:identifier/chats` | List chat histories for assistant |
| GET | `/api/v1/assistants/:identifier/chats/:chatUid` | Get specific chat history |
| POST | `/api/v1/assistants/chats` | Get all chat histories with filters |
| GET | `/api/v1/assistants/tags` | Get unique tags from chat histories |

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

Sends a message to the specified assistant and returns the response.

```
POST /api/v1/assistants/:identifier/messages
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `identifier` | string | The unique identifier of the assistant |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `message` | string | Yes | The message to send to the assistant |
| `chatUid` | string | No | Chat conversation ID (for continuing conversations) |
| `provider` | string | No | Override the LLM provider (e.g., "openai", "anthropic") |
| `model` | string | No | Override the model (e.g., "gpt-4", "claude-3-opus") |
| `tags` | string[] | No | Tags to associate with this chat |
| `metadata` | object | No | Custom metadata to store with the chat |

### Example Request

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

### Response

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
