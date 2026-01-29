# Feedback API

The Feedback API allows you to collect user feedback on AI responses from both assistants (chats) and agents (threads). This enables quality monitoring, model improvement tracking, and user satisfaction analysis.

## Endpoints Overview

### Assistant Chat Feedback

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/assistants/:identifier/chats/:chatUid/feedback` | Submit feedback for a chat message |
| GET | `/api/v1/assistants/:identifier/chats/:chatUid/feedback` | Get all feedback for a chat |

### Agent Thread Feedback

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/agents/threads/:threadId/feedback` | Submit feedback for a thread message |
| GET | `/api/v1/agents/threads/:threadId/feedback` | Get all feedback for a thread |

---

## Feedback Structure

Feedback entries capture user sentiment about specific AI responses.

### Key Properties

| Property | Type | Description |
|----------|------|-------------|
| `_id` | string | Unique feedback identifier |
| `requestId` | string | The message ID the feedback is for |
| `feedback` | boolean | `true` = positive, `false` = negative |
| `feedbackComment` | string | Optional text comment explaining the feedback |
| `feedbackData` | object | Optional structured feedback data (ratings, categories, etc.) |
| `creationTimestamp` | string | When the feedback was created |
| `lastEditTimestamp` | string | When the feedback was last updated |

### Chat-Specific Properties

| Property | Type | Description |
|----------|------|-------------|
| `chatUID` | string | The chat conversation ID |

### Thread-Specific Properties

| Property | Type | Description |
|----------|------|-------------|
| `threadId` | string | The agent thread ID |
| `agentId` | string | The agent ID |

---

## Submit Chat Feedback

Submits feedback for a specific message in an assistant chat conversation. If feedback already exists for the message, it will be updated.

```
POST /api/v1/assistants/:identifier/chats/:chatUid/feedback
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `identifier` | string | The assistant specialization identifier |
| `chatUid` | string | The chat conversation UUID |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `messageId` | string | Yes | The unique message ID (uid) to submit feedback for |
| `feedback` | boolean | No | `true` = positive, `false` = negative |
| `feedbackComment` | string | No | Optional comment explaining the feedback |
| `feedbackData` | object | No | Optional structured feedback data |

### Example Request

```bash
curl -X POST "https://api.devic.ai/api/v1/assistants/default/chats/550e8400-e29b-41d4-a716-446655440000/feedback" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "messageId": "msg-abc123",
    "feedback": true,
    "feedbackComment": "This response was very helpful and accurate!",
    "feedbackData": {
      "rating": 5,
      "categories": ["helpful", "accurate", "well-formatted"]
    }
  }'
```

### Response

```json
{
  "success": true,
  "data": {
    "_id": "64f5a5b8c123456789abcdef",
    "requestId": "msg-abc123",
    "chatUID": "550e8400-e29b-41d4-a716-446655440000",
    "feedback": true,
    "feedbackComment": "This response was very helpful and accurate!",
    "feedbackData": {
      "rating": 5,
      "categories": ["helpful", "accurate", "well-formatted"]
    },
    "creationTimestamp": "2024-01-15T10:30:00.000Z"
  }
}
```

### Error Responses

| Status | Description |
|--------|-------------|
| 404 | Chat history not found or does not belong to the specified assistant |

---

## Get Chat Feedback

Retrieves all feedback entries for a specific chat conversation.

```
GET /api/v1/assistants/:identifier/chats/:chatUid/feedback
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `identifier` | string | The assistant specialization identifier |
| `chatUid` | string | The chat conversation UUID |

### Example Request

```bash
curl -X GET "https://api.devic.ai/api/v1/assistants/default/chats/550e8400-e29b-41d4-a716-446655440000/feedback" \
  -H "Authorization: Bearer devic-your-api-key"
```

### Response

```json
{
  "success": true,
  "data": [
    {
      "_id": "64f5a5b8c123456789abcdef",
      "requestId": "msg-abc123",
      "chatUID": "550e8400-e29b-41d4-a716-446655440000",
      "feedback": true,
      "feedbackComment": "This response was very helpful!",
      "creationTimestamp": "2024-01-15T10:30:00.000Z"
    },
    {
      "_id": "64f5a5b8c123456789abcde0",
      "requestId": "msg-xyz789",
      "chatUID": "550e8400-e29b-41d4-a716-446655440000",
      "feedback": false,
      "feedbackComment": "The response was not accurate",
      "creationTimestamp": "2024-01-15T10:35:00.000Z"
    }
  ]
}
```

### Error Responses

| Status | Description |
|--------|-------------|
| 404 | Chat history not found or does not belong to the specified assistant |

---

## Submit Thread Feedback

Submits feedback for a specific message in an agent thread. If feedback already exists for the message, it will be updated.

```
POST /api/v1/agents/threads/:threadId/feedback
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `threadId` | string | The unique identifier of the thread |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `messageId` | string | Yes | The unique message ID (uid) to submit feedback for |
| `feedback` | boolean | No | `true` = positive, `false` = negative |
| `feedbackComment` | string | No | Optional comment explaining the feedback |
| `feedbackData` | object | No | Optional structured feedback data |

### Example Request

```bash
curl -X POST "https://api.devic.ai/api/v1/agents/threads/thread-456/feedback" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "messageId": "step-response-001",
    "feedback": true,
    "feedbackComment": "The agent handled this step perfectly",
    "feedbackData": {
      "rating": 5,
      "criteriaScores": {
        "accuracy": 5,
        "efficiency": 4,
        "clarity": 5
      }
    }
  }'
```

### Response

```json
{
  "success": true,
  "data": {
    "_id": "64f5a5b8c123456789abcdef",
    "requestId": "step-response-001",
    "threadId": "thread-456",
    "agentId": "agent-123",
    "feedback": true,
    "feedbackComment": "The agent handled this step perfectly",
    "feedbackData": {
      "rating": 5,
      "criteriaScores": {
        "accuracy": 5,
        "efficiency": 4,
        "clarity": 5
      }
    },
    "creationTimestamp": "2024-01-15T10:30:00.000Z"
  }
}
```

### Error Responses

| Status | Description |
|--------|-------------|
| 404 | Thread not found |

---

## Get Thread Feedback

Retrieves all feedback entries for a specific agent thread.

```
GET /api/v1/agents/threads/:threadId/feedback
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `threadId` | string | The unique identifier of the thread |

### Example Request

```bash
curl -X GET "https://api.devic.ai/api/v1/agents/threads/thread-456/feedback" \
  -H "Authorization: Bearer devic-your-api-key"
```

### Response

```json
{
  "success": true,
  "data": [
    {
      "_id": "64f5a5b8c123456789abcdef",
      "requestId": "step-response-001",
      "threadId": "thread-456",
      "agentId": "agent-123",
      "feedback": true,
      "feedbackComment": "Great response",
      "creationTimestamp": "2024-01-15T10:30:00.000Z"
    },
    {
      "_id": "64f5a5b8c123456789abcde0",
      "requestId": "step-response-002",
      "threadId": "thread-456",
      "agentId": "agent-123",
      "feedback": false,
      "feedbackComment": "This step took too long",
      "feedbackData": {
        "issue": "performance"
      },
      "creationTimestamp": "2024-01-15T10:32:00.000Z"
    }
  ]
}
```

### Error Responses

| Status | Description |
|--------|-------------|
| 404 | Thread not found |

---

## Use Cases

### Thumbs Up/Down Feedback

Simple binary feedback for quick user sentiment collection:

```bash
# Positive feedback (thumbs up)
curl -X POST "https://api.devic.ai/api/v1/assistants/default/chats/chat-123/feedback" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "messageId": "msg-001",
    "feedback": true
  }'

# Negative feedback (thumbs down)
curl -X POST "https://api.devic.ai/api/v1/assistants/default/chats/chat-123/feedback" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "messageId": "msg-001",
    "feedback": false
  }'
```

### Detailed Feedback with Ratings

Collect structured feedback with custom rating scales:

```bash
curl -X POST "https://api.devic.ai/api/v1/assistants/default/chats/chat-123/feedback" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "messageId": "msg-001",
    "feedback": true,
    "feedbackComment": "Excellent response, but could be more concise",
    "feedbackData": {
      "overallRating": 4,
      "dimensions": {
        "accuracy": 5,
        "helpfulness": 5,
        "clarity": 3,
        "completeness": 4
      },
      "suggestedImprovements": ["Be more concise"]
    }
  }'
```

### Agent Step-by-Step Feedback

Provide feedback on individual steps in an agent execution:

```bash
# Feedback on planning step
curl -X POST "https://api.devic.ai/api/v1/agents/threads/thread-456/feedback" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "messageId": "planning-step-001",
    "feedback": true,
    "feedbackData": {
      "stepType": "planning",
      "wasEfficient": true
    }
  }'

# Feedback on tool usage
curl -X POST "https://api.devic.ai/api/v1/agents/threads/thread-456/feedback" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "messageId": "tool-call-003",
    "feedback": false,
    "feedbackComment": "Used the wrong tool for this task",
    "feedbackData": {
      "stepType": "tool_usage",
      "toolName": "search_database",
      "shouldHaveUsed": "query_api"
    }
  }'
```

---

## Updating Feedback

Submitting feedback for a message that already has feedback will update the existing entry:

```bash
# Initial feedback
curl -X POST "https://api.devic.ai/api/v1/assistants/default/chats/chat-123/feedback" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "messageId": "msg-001",
    "feedback": false,
    "feedbackComment": "Not helpful"
  }'

# Update the feedback (same messageId)
curl -X POST "https://api.devic.ai/api/v1/assistants/default/chats/chat-123/feedback" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "messageId": "msg-001",
    "feedback": true,
    "feedbackComment": "Actually, after re-reading, this was helpful!"
  }'
```

The response will include `lastEditTimestamp` when feedback has been updated:

```json
{
  "success": true,
  "data": {
    "_id": "64f5a5b8c123456789abcdef",
    "requestId": "msg-001",
    "chatUID": "chat-123",
    "feedback": true,
    "feedbackComment": "Actually, after re-reading, this was helpful!",
    "creationTimestamp": "2024-01-15T10:30:00.000Z",
    "lastEditTimestamp": "2024-01-15T11:00:00.000Z"
  }
}
```
