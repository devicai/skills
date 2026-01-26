---
name: devic-api
description: Devic AI Platform API reference for assistants, agents, and tool servers. Use when working with Devic API endpoints, creating integrations, or building applications that interact with the Devic platform.
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

## API Sections

The Devic API is organized into three main sections:

### 1. Assistants API

Manage AI assistants that can process messages and maintain conversation history.

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
| 401 | Unauthorized - Invalid or missing API key |
| 404 | Not Found - Resource does not exist |
| 429 | Too Many Requests - Rate limit exceeded |
| 500 | Internal Server Error |

## Quick Start Examples

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
