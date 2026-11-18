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

### Tool-Call Cache

| Method | Endpoint | Description |
|--------|----------|-------------|
| DELETE | `/api/v1/tool-servers/:toolServerId/tools/:toolName/cache` | Clear cached results of one tool |
| DELETE | `/api/v1/tool-servers/:toolServerId/cache` | Clear cached results of every tool in the server |

### MCP Gateway (proxy servers only)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/tool-servers/:toolServerId/users` | List per-user access rows |
| PATCH | `/api/v1/tool-servers/:toolServerId/users/:userUID` | Upsert a user's access row |
| DELETE | `/api/v1/tool-servers/:toolServerId/users/:userUID` | Remove a user's access row |
| GET | `/api/v1/tool-servers/:toolServerId/profiles` | List visibility profiles |
| POST | `/api/v1/tool-servers/:toolServerId/profiles` | Create a visibility profile |
| PATCH | `/api/v1/tool-servers/:toolServerId/profiles/:profileId` | Update a visibility profile |
| DELETE | `/api/v1/tool-servers/:toolServerId/profiles/:profileId` | Delete a visibility profile |

### Response envelope

Most responses are wrapped as `{ "success": true, "data": …, "meta": { "timestamp", "duration" } }`.
A handler result that already contains a top-level `success` or `error` key is returned
**as is, without `data`**: this applies to the delete endpoints, the test endpoint and the
cache endpoints (their exact shapes are shown in each section).

Errors come back with the HTTP status and, for most of these endpoints, the framework
format `{ "statusCode": 404, "message": "Tool server not found", "error": "Not Found" }`,
where `error` is the status name. An unexpected failure (for example a malformed id)
returns 500 with `{ "error": "INTERNAL_SERVER_ERROR", "message": "…" }`. See the error
shapes in [SKILL.md](SKILL.md#error-responses).

There is no request-body validation layer: unknown fields are ignored where the endpoint
copies fields one by one (create/update server, update tool) and stored verbatim where it
does not (add tool, update definition). Use the exact field names documented here.

---

## How Tool Servers Connect to Assistants and Agents

Tool Servers provide external API capabilities to assistants and agents through a layered architecture:

```
Tool Server (API Definition)
       │
       │ assigned to
       ▼
  Tools Group (Logical Grouping)
       │
       │ referenced by availableToolsGroupsUids
       ▼
Assistant Specialization
       │
       ├──► Assistant (processes messages with tools)
       │
       └──► Agent (executes threads with tools)
```

### Integration Steps

1. **Create Tool Server**: Define your external API tools via the Tool Servers API
2. **Assign to Tools Group**: Link the tool server to a Tools Group (via Devic dashboard)
3. **Configure Assistant/Agent**: Add the Tools Group UID to `availableToolsGroupsUids`
4. **Tools Available**: The assistant/agent can now invoke tools during execution

### Tool Server Properties

| Property | Type | Description |
|----------|------|-------------|
| `_id` | string | Unique identifier |
| `name` | string | Display name |
| `description` | string | Purpose description |
| `url` | string | Base URL for API calls. Accepts template references — see [Template references](#template-references) |
| `identifier` | string | URL-friendly identifier |
| `enabled` | boolean | Whether tools are active |
| `type` | string | Read-only: `http`, `mcp` or `integration` — see [Server types](#server-types) |
| `toolServerDefinitionId` | string | Link to tool definitions |
| `authenticationConfig` | object | API authentication settings — see [Authentication Configuration](#authentication-configuration) |
| `mcpType` | boolean | Whether it's a remote MCP server |
| `proxy` | boolean | MCP Gateway: re-exposes the remote MCP at `url` publicly through the MCP wrapper |
| `remoteConnection` | object | Remote MCP connection config (`customHeaders`, `oauthSessionId`, `authMode`, `userManagement`, `defaultAccessMode`, `upstreamOAuth`…) |
| `mcpServerProperties` | object | Publication as an MCP: `{ mode: "public"\|"private", slug, status: "active"\|"inactive" }` |
| `permissionPolicy` | object | Host-enforced tool approval policy — see below |
| `imageUrl` | string | Icon URL |
| `projectId` | string | Project the server belongs to (read-only through this API) |

`permissionPolicy` shape (all optional; a tool with no rule, category or default runs
without approval):

```json
{
  "categories": [{ "id": "write-data", "name": "Write data", "description": "...", "approvalMode": "always" }],
  "tools": [{ "toolName": "delete_invoice", "categoryIds": ["write-data"], "approvalMode": "inherit" }],
  "defaultApprovalMode": "never"
}
```

`categories[].approvalMode` is `always` | `never`; `tools[].approvalMode` is
`inherit` | `always` | `never`.

### Tool Execution Flow

When an assistant or agent needs to call a tool:

1. AI model decides to use a tool based on the user request
2. Platform looks up the tool in the agent's available tool groups
3. Finds the tool server associated with that tool
4. Resolves any `{{...}}` template reference in the base URL and the endpoint (the body template receives `metadata` and `env` as objects instead)
5. Constructs the HTTP request using tool definition (endpoint, method, parameters)
6. Applies authentication from `authenticationConfig`
7. Executes the request and returns response to the AI model

---

## Template references

The base URL and the tool `endpoint` accept `{{namespace.name}}` references
resolved at call time. This is what lets ONE tool server — and therefore one
published MCP — serve several environments, instead of cloning the server per
environment.

The advanced body template (`bodyJsonTemplate`) does not substitute `{{...}}`
text; it reads the same values from the `metadata` and `env` objects (see the
example below). `authenticationConfig` has its own, narrower template support —
see [Authentication Configuration](#authentication-configuration).

| Reference | Resolved from | Available on |
|-----------|---------------|--------------|
| `{{metadata.<field>}}` | Thread or chat `metadata` sent by whoever started the run. Dot notation walks nested objects (`{{metadata.tenant.region}}`) | Internal path (a Devic agent or assistant calls the tool) |
| `{{<field>}}` | Shorthand for `{{metadata.<field>}}` — the form the authentication config has always used | Internal path |
| `{{env.<VAR>}}` | Env var of the Environment connected to the agent or assistant | Internal path |
| `{{fields.<apiName>}}` | Value the end user typed on the OAuth-proxy connection screen | Published MCP only — resolved by the mcp-api-wrapper |

Rules:

- **An unknown reference resolves to an empty string.** `https://{{env.API_HOST}}/v1`
  with no `API_HOST` produces `https:///v1`, not a URL with literal braces. Empty
  path segments are collapsed, so `/{{metadata.region}}/study/42` becomes `/study/42`.
- **`{{...}}` is resolved before `${arg}` / `{arg}`.** Those two are tool *arguments*
  supplied by the model; `{{...}}` references never come from the model. A bare
  `{{name}}` in an endpoint is therefore metadata, **not** the tool argument `name`:
  use `${name}` or `{name}` for arguments.
- **`{{fields.*}}` is not resolved on the internal path** — it collapses to empty
  there. Conversely the wrapper does not resolve `{{metadata.*}}` or `{{env.*}}`, so
  a tool server published as a public MCP should not rely on them in its URL.
- Cached tool results are keyed by the resolved target, so two environments never
  share a cached response.

Example — one tool server, one upstream per environment:

```json
{
  "name": "Billing API",
  "url": "https://{{env.BILLING_HOST}}",
  "toolDefinitions": [
    {
      "type": "function",
      "function": { "name": "get_invoice", "parameters": { "type": "object", "properties": { "invoiceId": { "type": "string" } } } },
      "endpoint": "/{{metadata.region}}/invoices/${invoiceId}",
      "method": "POST",
      "bodyMode": "advanced",
      "bodyJsonTemplate": "{ invoiceId: params.invoiceId, stage: env.STAGE, tenant: metadata.tenantSlug }"
    }
  ]
}
```

In the advanced body template, `metadata` and `env` are exposed as objects next
to `params` (the tool arguments).

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
| `projectId` | string | - | Only tool servers belonging to this project |

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

### Server types

`type` says where a server's tools come from, which is not otherwise derivable:
an integration has no `url` and no `identifier`.

| `type` | Tools come from | Notes |
|--------|-----------------|-------|
| `http` | Tool definitions you supply, called against `url` | |
| `mcp` | A remote MCP server | |
| `integration` | An app connected in Devic (Gmail, Drive, HubSpot...) | see below |

For an `integration`, an `integration` object describes the connected app:

```json
{
  "_id": "65a1b2c3d4e5f6789012399",
  "name": "Gmail",
  "enabled": true,
  "type": "integration",
  "integration": {
    "app": "gmail",
    "apps": ["gmail"],
    "connected": true,
    "enabledToolCount": 3
  }
}
```

`connected: false` means no account is linked and nothing can run.
`enabledToolCount` is absent when the server exposes every tool the app has.

Note that `total` is the size of the whole collection, not of the page.

### Credentials

`authenticationConfig` comes back with its secrets replaced by a mask
(`••••••••`): `jwt.token`, `basic.password`, `oauth2.refreshToken`,
`oauth2.clientSecret` and every `customHeader` value. Everything else — the
method, the username, the client id, the endpoints — is returned as stored. A
secret whose stored value is a `{{...}}` template or a vault reference is
returned verbatim, not masked. `remoteConnection.upstreamOAuth.clientSecret` is
masked too.

Sending a masked value back in a create or update **keeps the stored secret**,
so the ordinary read-modify-write round trip is safe. Send a real value to
change a secret, or an empty string to clear it. Custom headers are matched to
their stored value by `headerName`, so they can be reordered. Secrets are
encrypted at rest.

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
    "type": "http",
    "toolServerDefinitionId": "65a1b2c3d4e5f6789012346",
    "toolServerDefinition": {
      "_id": "65a1b2c3d4e5f6789012346",
      "name": "CRM API",
      "toolDefinitions": [...],
      "uiComponents": {}
    }
  }
}
```

`toolServerDefinition` is `null` when the server has no linked definition.

---

## Create Tool Server

Creates a new tool server with optional inline tool definitions.

```
POST /api/v1/tool-servers
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Tool server name (400 `Tool server name is required` if missing) |
| `description` | string | No | Tool server description |
| `url` | string | No | Base URL for the API (the upstream MCP URL for `mcpType` / `proxy`). Required when `proxy` is true |
| `identifier` | string | No | URL-friendly identifier |
| `enabled` | boolean | No | Currently ignored: a new server is always created enabled. Disable it afterwards with `PATCH` |
| `mcpType` | boolean | No | Whether this is a remote MCP server |
| `proxy` | boolean | No | Create an MCP Gateway that re-exposes the remote MCP at `url` |
| `remoteConnection` | object | No | Remote MCP connection config (`customHeaders`, `oauthSessionId`, `authMode: "shared"\|"per-user"`, `userManagement`, `defaultAccessMode: "deny"\|"allow"`, `upstreamOAuth: { clientId, clientSecret, scopes }`) |
| `mcpServerProperties` | object | No | `{ mode: "public"\|"private", slug, status: "active"\|"inactive" }`. A slug already in use returns 409 |
| `toolDefinitions` | array | No | Array of tool definitions ([schema](#tool-definition-schema)). Creates a new definition and links it |
| `toolServerDefinitionId` | string | No | Link an existing definition instead. Ignored when `toolDefinitions` is non-empty |
| `authenticationConfig` | object | No | [Authentication configuration](#authentication-configuration) |
| `imageUrl` | string | No | Icon/image URL |
| `permissionPolicy` | object | No | [Approval policy](#tool-server-properties) |

Any other field (for example `projectId`) is ignored.

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

`toolServerDefinition` is only present when `toolDefinitions` was sent.

---

## Update Tool Server

Updates a tool server. If `toolDefinitions` is provided, it **replaces the whole
tool list** of the linked definition (a definition is created and linked if the
server has none). To change a single tool, use the tool endpoints below.

```
PATCH /api/v1/tool-servers/:toolServerId
```

### Request Body

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Update name |
| `description` | string | Update description |
| `url` | string | Update base URL. On a server with `remoteConnection`, changing it drops the stored OAuth session and probe data (only `customHeaders` and a new `oauthSessionId` are kept) |
| `enabled` | boolean | Enable/disable |
| `toolDefinitions` | array | Replace all tool definitions |
| `authenticationConfig` | object | Replace the auth config (send the whole object; masked secrets are kept) |
| `imageUrl` | string | Update icon URL |
| `proxy` | boolean | Toggle MCP Gateway mode |
| `remoteConnection` | object | Replace the remote MCP connection config |
| `mcpServerProperties` | object | Replace `{ mode, slug, status }`. A new slug already in use returns 409 |
| `permissionPolicy` | object | Replace the approval policy |

Only these fields are applied. `identifier`, `mcpType` and `toolServerDefinitionId`
cannot be changed through this endpoint and are silently ignored. Object fields are
replaced as a whole, not merged.

---

## Delete Tool Server

Deletes a tool server. Does not delete the associated definition.

```
DELETE /api/v1/tool-servers/:toolServerId
```

### Response

Returned without the `data` wrapper:

```json
{
  "success": true,
  "message": "Tool server deleted successfully",
  "deletedId": "65a1b2c3d4e5f6789012345"
}
```

---

## Clone Tool Server

Creates a copy of a tool server and its definition. No request body.

```
POST /api/v1/tool-servers/:toolServerId/clone
```

### Response

`201` with the new tool server (same shape as [Get Tool Server by ID](#get-tool-server-by-id),
without `toolServerDefinition`). The copy is named `<name> (Copy)`, is enabled, keeps
every other field — including the stored credentials — and points at a new definition
also named `<definition name> (Copy)`.

---

## List Tools in Server

Retrieves the tools a server exposes.

```
GET /api/v1/tool-servers/:toolServerId/tools
```

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `available` | boolean | false | Integrations only: list every tool the connected app offers instead of only the exposed ones |
| `limit` | number | - | Page size when browsing a catalogue (`available=true`), clamped to 1–200 |
| `cursor` | string | - | Page token from a previous response's `nextCursor` |

### App integrations

An integration has no stored tool definitions: its tools are resolved from the
connected app. By default this endpoint returns the tools **the server
exposes** — the same set the model is offered.

Pass `available=true` to browse the app's whole catalogue instead, to find
something worth turning on. There each tool carries `enabled`, saying whether
this server exposes it, and the response is paged:

```bash
curl -X GET "https://api.devic.ai/api/v1/tool-servers/<id>/tools?available=true&limit=50" \
  -H "Authorization: Bearer devic-your-api-key"
```

```json
{
  "success": true,
  "data": {
    "tools": [
      {
        "type": "function",
        "function": { "name": "GMAIL_SEND_EMAIL", "description": "...", "parameters": {} },
        "enabled": false
      }
    ],
    "total": 63,
    "nextCursor": "Mi01"
  }
}
```

Page with `nextCursor`, not with `total`: when browsing a catalogue, `total` is
the app's own count and includes tools the provider has deprecated, which are
not offered here.

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

For a non-integration server the query parameters are ignored and the stored
definitions are returned in full. A server without a definition returns
`{ "tools": [], "total": 0 }`.

---

## Get Tool by Name

Retrieves a specific tool definition by its function name.

```
GET /api/v1/tool-servers/:toolServerId/tools/:toolName
```

For an app integration the tool is looked up in the app's whole catalogue and
carries `enabled`. Returns 404 when the tool, or the server's definition, does
not exist.

---

## Add Tool to Server

Adds a new tool definition to the tool server. If the server has no definition,
one is created and linked.

```
POST /api/v1/tool-servers/:toolServerId/tools
```

The body is `{ "tool": <tool definition> }` — see [Tool Definition Schema](#tool-definition-schema).
`tool.function.name` is required (400 otherwise) and must be unique in the server
(400 `Tool with name '…' already exists`). The tool is stored exactly as sent, so
every schema field is accepted here, and the response (`201`) echoes it.

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
    "bodyJsonTemplate": "{ name: params.name, email: params.email }"
  }
}
```

---

## Update Tool by Name

Updates a specific tool definition. The body is merged into the stored tool, field
by field; the response is the updated tool.

```
PATCH /api/v1/tool-servers/:toolServerId/tools/:toolName
```

### Request Body

| Field | Type | Description |
|-------|------|-------------|
| `function` | object | Shallow-merged into the stored `function`: send `name`, `description` and/or `parameters` (a sent `parameters` replaces the stored one entirely). Changing `name` renames the tool |
| `endpoint` | string | New API endpoint |
| `method` | string | HTTP method (GET, POST, PUT, DELETE, PATCH) |
| `queryParametersKeys` | string[] | Parameters to include as query params |
| `pathParametersKeys` | string[] | Parameters to include in URL path |
| `bodyPropertyKey` | string | Parameter to use as request body (bodyMode: simple) |
| `bodyMode` | string | Body mode: `simple` or `advanced` |
| `bodyJsonTemplate` | string | JavaScript expression returning request body object. Access parameters via `params.<paramName>` |
| `isFormDataBody` | boolean | Send body as multipart/form-data |
| `customHeaders` | array | Custom headers `[{headerName, value}]` |
| `responsePostProcessingEnabled` | boolean | Enable JS expression post-processing |
| `responsePostProcessingTemplate` | string | JS expression to transform response |
| `binaryResponse` | boolean | Fetch the response as raw bytes (see [Response Post-Processing](#response-post-processing)) |
| `annotations` | object | MCP tool annotations (replaced as a whole) |
| `permissionCategoryIds` | string[] | Categories of the server's `permissionPolicy` this tool belongs to |
| `approvalMode` | string | `inherit`, `always` or `never` |

**Only the fields above are applied.** Anything else — notably `cacheEnabled`,
`cacheTtlSeconds`, `widgetUid`, `asyncTool` and `hidden` — is silently dropped by
this endpoint. To change those, rewrite the tool list with
[Update Definition](#update-tool-server-definition) or with `toolDefinitions` on
[Update Tool Server](#update-tool-server).

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

### Response

Returned without the `data` wrapper:

```json
{
  "success": true,
  "message": "Tool 'get_users' deleted successfully",
  "deletedId": "get_users"
}
```

---

## Test Tool Call

Executes a tool call with provided parameters for testing.

```
POST /api/v1/tool-servers/:toolServerId/tools/:toolName/test
```

> **App integrations use their own endpoint.** This one runs a stored tool
> definition, and an integration has none — its tools live at the provider — so
> it answers `404 … does not have a tool definition`. Test those with
> `POST /api/v1/integrations/servers/:id/tools/:toolSlug/test`
> (`devic integrations tools test <id> <SLUG>`), which runs the tool against the
> connected account. Remote MCP and proxy servers are not testable from either.

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `parameters` | object | Yes | Parameters to pass to the tool |
| `metadata` | object | No | Values for `{{metadata.*}}` references (and the bare `{{field}}` shorthand) |
| `envVars` | object | No | Values for `{{env.*}}` references. At runtime these come from the connected Environment; there is none on the test path, so supply them here |

### Example Request

```bash
curl -X POST "https://api.devic.ai/api/v1/tool-servers/server-123/tools/get_weather/test" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "parameters": {
      "city": "London"
    },
    "metadata": { "region": "eu-west" },
    "envVars": { "API_HOST": "staging.weather.example.com" }
  }'
```

### Response

HTTP 200, returned without the `data` wrapper (`data` here is the tool result):

```json
{
  "success": true,
  "data": {
    "temperature": 15,
    "condition": "cloudy",
    "humidity": 75
  },
  "executionTimeMs": 245,
  "requestUrl": "https://api.weather.example.com/weather/London",
  "requestMethod": "GET",
  "rawResponse": { "body": { "temperature": 15, "condition": "cloudy", "humidity": 75 }, "status": 200 },
  "postProcessing": { "enabled": false, "applied": false }
}
```

- `data` is already post-processed when `postProcessing.applied` is true; do not
  run the template again on it.
- `rawResponse` is the upstream response before post-processing (`isBinary`,
  `contentType` and `truncated` appear for binary bodies, which are base64 and
  clipped).
- `cacheInfo` (`{ cached, cachedAt, ageMs, ttlSeconds, remainingMs }`) replaces
  `rawResponse`/`postProcessing` when the result was served from the tool cache.

### Error Response

A failed upstream call or a failing post-processing template is still HTTP 200,
with `success: false`. When the upstream answered with an HTTP error, `error` is
its response body (JSON-stringified if not a string) and `rawResponse` is
included; when the template failed, `postProcessing.error` carries the reason:

```json
{
  "success": false,
  "error": "Connection timeout",
  "executionTimeMs": 5000,
  "requestUrl": "https://api.weather.example.com/weather/London",
  "requestMethod": "GET"
}
```

Real HTTP errors: 404 when the server, its definition or the tool does not exist;
400 when the tool has no `endpoint` or no `method`.

---

## Tool Definition Schema

Complete schema for a tool definition (every field except `function.name` is optional):

```json
{
  "type": "function",
  "function": {
    "name": "tool_name",
    "description": "What this tool does",
    "strict": false,
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
  "bodyJsonTemplate": "{ key: params.value }",
  "isFormDataBody": false,
  "customHeaders": [
    {"headerName": "X-Custom-Header", "value": "custom-value"}
  ],
  "responsePostProcessingEnabled": true,
  "responsePostProcessingTemplate": "response.body.data",
  "binaryResponse": false,
  "cacheEnabled": false,
  "cacheTtlSeconds": 300,
  "asyncTool": false,
  "widgetUid": "widget_123",
  "annotations": {
    "title": "Tool title",
    "readOnlyHint": true,
    "destructiveHint": false,
    "idempotentHint": true,
    "openWorldHint": false
  },
  "permissionCategoryIds": ["write-data"],
  "approvalMode": "inherit",
  "hidden": false
}
```

| Field | Description |
|-------|-------------|
| `method` | `GET`, `POST`, `PUT`, `DELETE` or `PATCH`. `GET` and `DELETE` send no body |
| `customHeaders` | Per-tool headers. Values may contain `${arg}` / `{arg}` tool-argument placeholders |
| `binaryResponse` | Fetch the response as raw bytes; see [Response Post-Processing](#response-post-processing) |
| `cacheEnabled` / `cacheTtlSeconds` | Cache results per client + arguments + resolved URL. Both are required for caching to apply (`cacheTtlSeconds` > 0, max 86400). Clear with the [cache endpoints](#clear-tool-call-cache) |
| `asyncTool` | The request body is the full argument object plus an injected `__async_context` (`chatUid`/`threadId`, `toolCallId`, `callbackUrl`), and the conversation waits until the external system posts the real result to that callback. Not cached |
| `widgetUid` | Links the tool to a UI component (`uiComponents` of the definition) |
| `annotations` | MCP tool annotations — informational hints only, never authorization |
| `permissionCategoryIds` / `approvalMode` | Approval settings, combined with the server's `permissionPolicy`. `approvalMode`: `inherit`, `always`, `never` |
| `hidden` | MCP Gateway (`proxy`) servers only: hides the upstream tool of that name. On a proxy the definition is an override sidecar (`hidden`, `annotations`) matched by `function.name`; the REST fields are unused |

### Parameter Handling

The URL is `<server url>/<endpoint>`, built in this order:

- **`{{...}}` references** in the base URL and the endpoint are resolved first (metadata / env — see [Template references](#template-references)).
- **Endpoint placeholders**: `${paramName}` or `{paramName}` is replaced by that tool argument (left untouched if the argument is missing).
- **Path Parameters**: every key in `pathParametersKeys` that was *not* already used by an endpoint placeholder is appended as an extra `/value` segment, in order. Listing a key that already has a placeholder is harmless.
- **Query Parameters**: keys in `queryParametersKeys` (not used by a placeholder) become `?key=value`, URL-encoded; an array value repeats the key; null/undefined values are skipped.
- **Body (Simple Mode, also the default when `bodyMode` is absent)**: the body is the value of the single argument named by `bodyPropertyKey` (typically an object-typed argument). Without `bodyPropertyKey` no body is sent — the arguments are **not** sent as a body automatically.
- **Body (Advanced Mode)**: `bodyMode: "advanced"` plus `bodyJsonTemplate`, a JavaScript **expression** (not JSON with `{{...}}` placeholders) evaluated in a sandbox with `params` (tool arguments), `metadata` and `env` in scope, e.g. `{ name: params.name, email: params.email, tenant: metadata.tenantSlug }`. 5 s timeout; `require`, `import`, `process.`, `eval(`, `Function(`, timers and `global.` are rejected.
- **`isFormDataBody`**: sends the body as `multipart/form-data`, one field per top-level key (objects are JSON-stringified).

### Response Post-Processing

When `responsePostProcessingEnabled` is true, the `responsePostProcessingTemplate` is evaluated as an async JavaScript function body with access to:

- `response.body` - The response body (parsed JSON; a base64 string when `binaryResponse` is true)
- `response.status` - HTTP status code
- `response.contentType`, `response.isBinary` - set when `binaryResponse` is true
- `devic` - helper namespace, e.g. `await devic.saveFile(base64, { fileName?, mimeType? })` returns `{ downloadUrl, fileName }`

A single expression is returned as is; code with several statements or declarations must use an explicit `return`. It runs only on a 2xx response: an upstream HTTP error is returned to the model as `{ error: <response body> }` without going through the template. With `binaryResponse` and no post-processing, the bytes are not forwarded to the model (only a short description).

Example templates:
- `response.body.data` - Extract data field
- `response.body.items.map(i => i.name)` - Map to names only
- `const f = await devic.saveFile(response.body, { fileName: 'report.pdf' }); return f.downloadUrl;` - Persist a binary payload

---

## Authentication Configuration

`authenticationConfig` selects one method with `authenticationMethod` and carries
the matching sub-object. Values are applied as request headers on every tool call.

| `authenticationMethod` | Sub-object | Sent as |
|------------------------|------------|---------|
| `jwt` | `jwt: { token }` | `Authorization: Bearer <token>` |
| `basic` | `basic: { username, password }` | `Authorization: Basic base64(username:password)` |
| `customHeader` | `customHeader: [{ headerName, value }]` (a single object is also accepted) | One header per entry |
| `oauth2` | `oauth2: { … }` (below) | `Authorization: Bearer <access token>` |

```json
{ "authenticationConfig": { "authenticationMethod": "jwt", "jwt": { "token": "your-api-token" } } }
```

```json
{
  "authenticationConfig": {
    "authenticationMethod": "customHeader",
    "customHeader": [{ "headerName": "X-API-Key", "value": "your-api-key" }]
  }
}
```

```json
{
  "authenticationConfig": {
    "authenticationMethod": "oauth2",
    "oauth2": {
      "grant_type": "client_credentials",
      "clientId": "client-id",
      "clientSecret": "client-secret",
      "refreshTokenEndpoint": "https://auth.example.com/oauth/token",
      "scope": "read write",
      "requestFormat": "form"
    }
  }
}
```

`oauth2` fields:

| Field | Description |
|-------|-------------|
| `refreshTokenEndpoint` | Token URL that is POSTed to obtain an access token (there is no `tokenUrl` field) |
| `grant_type` | `refresh_token` or `client_credentials` (defaults to `refresh_token` when a refresh token is set) |
| `refreshToken` | Refresh token (secret) |
| `clientId` / `clientSecret` | Client credentials (`clientSecret` is a secret) |
| `scope` | Optional scope |
| `requestFormat` | `json` (default) or `form` (`application/x-www-form-urlencoded`) |
| `customBody` | Replaces the whole token request body |
| `useBasicAuth` | Send client credentials as a Basic `Authorization` header instead of in the body |
| `provider`, `authorizationEndpoint`, `tokenEndpoint`, `oauthFlowCompleted`, `connectedUserEmail`, `connectedUserName`, `connectedUserUsername` | Set by the dashboard's interactive OAuth flow; not used to refresh the token |

Access tokens are cached until one minute before `expires_in` (when the token
response includes it). If the configuration is incomplete or the token request
fails, the tool call is still sent, without authentication headers.

`oAuthProxyEnabledConfig` configures the connection screen of a tool server
published as a public MCP: `enabled`, `proxyLogoUrl`, `proxyHeaderText`,
`proxyPrimaryColor`, `proxyMessage`, `proxyFaviconUrl`, `hideApiKeyField` and
`customFields: [{ apiName, label, type, required?, placeholder?, helpText?, options? }]`
(`type`: `text`, `secret`, `number`, `select`, `email`, `url`, `date`). Values the
end user types there are referenced as `{{fields.<apiName>}}`.

Templates in credential values: a bare `{{field}}` (dot paths allowed) resolves
against the run's metadata and `{{tenantId}}` against the tenant id. Unlike the URL
templates, `{{metadata.*}}` and `{{env.*}}` are **not** resolved here, and an
unresolved reference is left as literal text. Credentials may also be vault
references. See [Credentials](#credentials) for how secrets are masked and kept.

---

## Get Tool Server Definition

```
GET /api/v1/tool-servers/:toolServerId/definition
```

Returns the linked definition document: `_id`, `name`, `description`,
`toolDefinitions`, `uiComponents`, `creationUserUID`, `creationTimestampMs`,
`lastEditUserUID`, `lastEditTimestampMs`. 404 when the server has no definition.

---

## Update Tool Server Definition

```
PATCH /api/v1/tool-servers/:toolServerId/definition
```

The body is written onto the definition as sent (no field whitelist). Use only:

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Definition name |
| `description` | string | Definition description |
| `toolDefinitions` | array | Replaces the whole tool list ([schema](#tool-definition-schema)); every field is kept |
| `uiComponents` | object | UI widgets referenced by `widgetUid` |

Returns the updated definition. 404 when the server has no definition (create one
by adding a tool or sending `toolDefinitions` to the server first).

---

## Clear Tool-Call Cache

```
DELETE /api/v1/tool-servers/:toolServerId/tools/:toolName/cache
DELETE /api/v1/tool-servers/:toolServerId/cache
```

Removes the cached results (see `cacheEnabled`) of one tool, or of every tool in
the server, for the calling client. Returned without the `data` wrapper:

```json
{
  "success": true,
  "removed": 12,
  "scope": "mcp-tool-cache:65a1b2c3d4e5f6789012345:get_products:<clientUID>:*"
}
```

---

## MCP Gateway: Users and Profiles

Only for tool servers with `proxy: true` (400 otherwise). They control which Devic
users may use a user-managed gateway (`remoteConnection.userManagement: true`) and
which tools each one sees. Without an access row a user gets
`remoteConnection.defaultAccessMode` (`deny` by default). Visibility only ever hides
more: the gateway's `hidden` tools ∪ the assigned profile's `hiddenToolNames` ∪ the
user's `userHiddenToolNames`.

### Users

```
GET    /api/v1/tool-servers/:toolServerId/users
PATCH  /api/v1/tool-servers/:toolServerId/users/:userUID
DELETE /api/v1/tool-servers/:toolServerId/users/:userUID
```

`GET` returns the access rows (`toolServerId`, `userUID`, `allowed`, `profileId`,
`userHiddenToolNames`, …).

`PATCH` upserts the row for `userUID` and returns it. Body (all optional):

| Field | Type | Description |
|-------|------|-------------|
| `allowed` | boolean | Access gate. Defaults to `true` when a new row is created without it |
| `profileId` | string \| null | Assigned visibility profile; `null` unassigns |
| `userHiddenToolNames` | string[] | Per-user hidden tool names |

`DELETE` removes the row (the user falls back to the default access mode) and
returns `{ "removed": true }`.

### Profiles

```
GET    /api/v1/tool-servers/:toolServerId/profiles
POST   /api/v1/tool-servers/:toolServerId/profiles
PATCH  /api/v1/tool-servers/:toolServerId/profiles/:profileId
DELETE /api/v1/tool-servers/:toolServerId/profiles/:profileId
```

Body for `POST` / `PATCH`:

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Required on `POST`; unique per gateway (409 on a duplicate) |
| `description` | string | Optional |
| `hiddenToolNames` | string[] | Tool names hidden for users assigned to the profile |

`GET` lists the profiles sorted by name; `POST` / `PATCH` return the profile.
`DELETE` returns `{ "removed": true, "detachedUsers": <n> }`; users that had it
assigned keep only their own overrides.
