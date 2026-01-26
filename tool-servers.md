# Tool Servers API

The Tool Servers API allows you to manage external tool integrations that agents and assistants can use. Tool servers define collections of API endpoints that can be called as tools during AI executions.

## Endpoints Overview

### Tool Server Management

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/tool-servers` | List all tool servers |
| GET | `/api/v1/tool-servers/:toolServerId` | Get tool server by ID |
| POST | `/api/v1/tool-servers` | Create tool server |
| PATCH | `/api/v1/tool-servers/:toolServerId` | Update tool server |
| DELETE | `/api/v1/tool-servers/:toolServerId` | Delete tool server |
| POST | `/api/v1/tool-servers/:toolServerId/clone` | Clone tool server |

### Tool Management

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/tool-servers/:toolServerId/tools` | List tools in server |
| GET | `/api/v1/tool-servers/:toolServerId/tools/:toolName` | Get specific tool |
| POST | `/api/v1/tool-servers/:toolServerId/tools` | Add tool to server |
| PATCH | `/api/v1/tool-servers/:toolServerId/tools/:toolName` | Update tool |
| DELETE | `/api/v1/tool-servers/:toolServerId/tools/:toolName` | Delete tool |
| POST | `/api/v1/tool-servers/:toolServerId/tools/:toolName/test` | Test tool call |

### Definition Management

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/tool-servers/:toolServerId/definition` | Get tool server definition |
| PATCH | `/api/v1/tool-servers/:toolServerId/definition` | Update definition |

---

## List Tool Servers

Retrieves a paginated list of all tool servers.

```
GET /api/v1/tool-servers
```

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `offset` | number | 0 | Number of items to skip |
| `limit` | number | 10 | Maximum items to return (max: 100) |

### Example Request

```bash
curl -X GET "https://api.devic.ai/api/v1/tool-servers?limit=20" \
  -H "Authorization: Bearer devic-your-api-key"
```

### Response

```json
{
  "success": true,
  "data": {
    "toolServers": [
      {
        "_id": "65a1b2c3d4e5f6789012345",
        "name": "CRM API",
        "description": "Tools for CRM operations",
        "url": "https://api.crm.example.com",
        "identifier": "crm-api",
        "enabled": true
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

## Get Tool Server by ID

Retrieves a specific tool server including its tool definition.

```
GET /api/v1/tool-servers/:toolServerId
```

### Response

```json
{
  "success": true,
  "data": {
    "_id": "65a1b2c3d4e5f6789012345",
    "name": "CRM API",
    "description": "Tools for CRM operations",
    "url": "https://api.crm.example.com",
    "identifier": "crm-api",
    "enabled": true,
    "toolServerDefinitionId": "65a1b2c3d4e5f6789012346",
    "toolServerDefinition": {
      "toolDefinitions": [...]
    }
  }
}
```

---

## Create Tool Server

Creates a new tool server with optional inline tool definitions.

```
POST /api/v1/tool-servers
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Tool server name |
| `description` | string | No | Tool server description |
| `url` | string | No | Base URL for the API |
| `identifier` | string | No | URL-friendly identifier |
| `enabled` | boolean | No | Whether the server is enabled (default: true) |
| `mcpType` | boolean | No | Whether this is an MCP server |
| `toolDefinitions` | array | No | Array of tool definitions |
| `authenticationConfig` | object | No | Authentication configuration |
| `imageUrl` | string | No | Icon/image URL |

### Example Request

```bash
curl -X POST "https://api.devic.ai/api/v1/tool-servers" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Weather API",
    "description": "Get weather information",
    "url": "https://api.weather.example.com",
    "identifier": "weather-api",
    "enabled": true,
    "toolDefinitions": [
      {
        "type": "function",
        "function": {
          "name": "get_weather",
          "description": "Get current weather for a location",
          "parameters": {
            "type": "object",
            "properties": {
              "city": {
                "type": "string",
                "description": "City name"
              }
            },
            "required": ["city"]
          }
        },
        "endpoint": "/weather/${city}",
        "method": "GET",
        "pathParametersKeys": ["city"]
      }
    ]
  }'
```

### Response

```json
{
  "success": true,
  "data": {
    "_id": "65a1b2c3d4e5f6789012345",
    "name": "Weather API",
    "toolServerDefinition": {
      "_id": "65a1b2c3d4e5f6789012346",
      "toolDefinitions": [...]
    }
  }
}
```

---

## Update Tool Server

Updates a tool server. If `toolDefinitions` is provided, updates the linked definition.

```
PATCH /api/v1/tool-servers/:toolServerId
```

### Request Body

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Update name |
| `description` | string | Update description |
| `url` | string | Update base URL |
| `enabled` | boolean | Enable/disable |
| `toolDefinitions` | array | Update all tool definitions |
| `authenticationConfig` | object | Update auth config |

---

## Delete Tool Server

Deletes a tool server. Does not delete the associated definition.

```
DELETE /api/v1/tool-servers/:toolServerId
```

### Response

```json
{
  "success": true,
  "data": {
    "success": true,
    "message": "Tool server deleted successfully",
    "deletedId": "65a1b2c3d4e5f6789012345"
  }
}
```

---

## Clone Tool Server

Creates a copy of a tool server and its definition.

```
POST /api/v1/tool-servers/:toolServerId/clone
```

### Response

The cloned server will have "(Copy)" appended to its name.

---

## List Tools in Server

Retrieves all tool definitions from a tool server.

```
GET /api/v1/tool-servers/:toolServerId/tools
```

### Response

```json
{
  "success": true,
  "data": {
    "tools": [
      {
        "type": "function",
        "function": {
          "name": "get_weather",
          "description": "Get current weather for a location",
          "parameters": {...}
        },
        "endpoint": "/weather/${city}",
        "method": "GET"
      }
    ],
    "total": 3
  }
}
```

---

## Get Tool by Name

Retrieves a specific tool definition by its function name.

```
GET /api/v1/tool-servers/:toolServerId/tools/:toolName
```

---

## Add Tool to Server

Adds a new tool definition to the tool server.

```
POST /api/v1/tool-servers/:toolServerId/tools
```

### Request Body

```json
{
  "tool": {
    "type": "function",
    "function": {
      "name": "create_contact",
      "description": "Create a new contact in the CRM",
      "parameters": {
        "type": "object",
        "properties": {
          "name": {
            "type": "string",
            "description": "Contact name"
          },
          "email": {
            "type": "string",
            "description": "Contact email"
          }
        },
        "required": ["name", "email"]
      }
    },
    "endpoint": "/contacts",
    "method": "POST",
    "bodyMode": "advanced",
    "bodyJsonTemplate": "{\"name\": \"${name}\", \"email\": \"${email}\"}"
  }
}
```

---

## Update Tool by Name

Updates a specific tool definition.

```
PATCH /api/v1/tool-servers/:toolServerId/tools/:toolName
```

### Request Body

| Field | Type | Description |
|-------|------|-------------|
| `function` | object | Update function definition (name, description, parameters) |
| `endpoint` | string | New API endpoint |
| `method` | string | HTTP method (GET, POST, PUT, DELETE, PATCH) |
| `queryParametersKeys` | string[] | Parameters to include as query params |
| `pathParametersKeys` | string[] | Parameters to include in URL path |
| `bodyPropertyKey` | string | Parameter to use as request body (bodyMode: simple) |
| `bodyMode` | string | Body mode: `simple` or `advanced` |
| `bodyJsonTemplate` | string | JSON body template with `${paramName}` placeholders |
| `isFormDataBody` | boolean | Send body as multipart/form-data |
| `customHeaders` | array | Custom headers `[{headerName, value}]` |
| `responsePostProcessingEnabled` | boolean | Enable JS expression post-processing |
| `responsePostProcessingTemplate` | string | JS expression to transform response |

### Example: Update with Response Post-Processing

```bash
curl -X PATCH "https://api.devic.ai/api/v1/tool-servers/server-123/tools/get_users" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "responsePostProcessingEnabled": true,
    "responsePostProcessingTemplate": "response.body.data.map(u => ({id: u.id, name: u.name}))"
  }'
```

---

## Delete Tool by Name

Removes a tool definition from the server.

```
DELETE /api/v1/tool-servers/:toolServerId/tools/:toolName
```

---

## Test Tool Call

Executes a tool call with provided parameters for testing.

```
POST /api/v1/tool-servers/:toolServerId/tools/:toolName/test
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `parameters` | object | Yes | Parameters to pass to the tool |

### Example Request

```bash
curl -X POST "https://api.devic.ai/api/v1/tool-servers/server-123/tools/get_weather/test" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "parameters": {
      "city": "London"
    }
  }'
```

### Response

```json
{
  "success": true,
  "data": {
    "success": true,
    "data": {
      "temperature": 15,
      "condition": "cloudy",
      "humidity": 75
    },
    "executionTimeMs": 245,
    "requestUrl": "https://api.weather.example.com/weather/London",
    "requestMethod": "GET"
  }
}
```

### Error Response

```json
{
  "success": true,
  "data": {
    "success": false,
    "error": "Connection timeout",
    "executionTimeMs": 5000,
    "requestUrl": "https://api.weather.example.com/weather/London",
    "requestMethod": "GET"
  }
}
```

---

## Tool Definition Schema

Complete schema for a tool definition:

```json
{
  "type": "function",
  "function": {
    "name": "tool_name",
    "description": "What this tool does",
    "parameters": {
      "type": "object",
      "properties": {
        "param1": {
          "type": "string",
          "description": "Parameter description"
        }
      },
      "required": ["param1"]
    }
  },
  "endpoint": "/api/endpoint/${pathParam}",
  "method": "POST",
  "pathParametersKeys": ["pathParam"],
  "queryParametersKeys": ["queryParam"],
  "bodyPropertyKey": "bodyParam",
  "bodyMode": "simple",
  "bodyJsonTemplate": "{\"key\": \"${value}\"}",
  "isFormDataBody": false,
  "customHeaders": [
    {"headerName": "X-Custom-Header", "value": "custom-value"}
  ],
  "responsePostProcessingEnabled": true,
  "responsePostProcessingTemplate": "response.body.data"
}
```

### Parameter Handling

- **Path Parameters**: Use `${paramName}` in the endpoint URL and list in `pathParametersKeys`
- **Query Parameters**: List parameter names in `queryParametersKeys`
- **Body (Simple Mode)**: Specify the parameter name in `bodyPropertyKey`
- **Body (Advanced Mode)**: Use `bodyJsonTemplate` with `${paramName}` placeholders

### Response Post-Processing

When `responsePostProcessingEnabled` is true, the `responsePostProcessingTemplate` is evaluated as a JavaScript expression with access to:

- `response.body` - The response body (parsed JSON)
- `response.status` - HTTP status code

Example templates:
- `response.body.data` - Extract data field
- `response.body.items.map(i => i.name)` - Map to names only
- `response.status === 200 ? response.body : {error: 'Failed'}` - Conditional

---

## Authentication Configuration

Tool servers support various authentication methods:

```json
{
  "authenticationConfig": {
    "type": "bearer",
    "token": "your-api-token"
  }
}
```

```json
{
  "authenticationConfig": {
    "type": "apiKey",
    "headerName": "X-API-Key",
    "apiKey": "your-api-key"
  }
}
```

```json
{
  "authenticationConfig": {
    "type": "oauth2",
    "clientId": "client-id",
    "clientSecret": "client-secret",
    "tokenUrl": "https://auth.example.com/token"
  }
}
```
