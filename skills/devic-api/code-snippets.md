# Code Snippets API

A **code snippet** is a function you write once and give to agents and
assistants as a tool. The model sees it like any other tool: a name, a
description and a JSON Schema of its arguments. When the model calls it, the
code runs in an isolated sandbox and its return value is the tool result.

Snippets belong to the account. An agent or assistant uses one by listing its
id in `codeSnippetIds` — see [Attaching a snippet](#attaching-a-snippet).

---

## Endpoints Overview

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/code-snippets` | List snippets (without their code) |
| POST | `/api/v1/code-snippets` | Create a snippet |
| GET | `/api/v1/code-snippets/{snippetId}` | Get one, with code, parameters and test cases |
| PATCH | `/api/v1/code-snippets/{snippetId}` | Update the fields you send |
| DELETE | `/api/v1/code-snippets/{snippetId}` | Delete it |
| POST | `/api/v1/code-snippets/{snippetId}/test` | Run it in a sandbox |

---

## The snippet contract

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | snake_case (`^[a-z0-9]+(_[a-z0-9]+)*$`), at most 64 characters. It is the name the model calls |
| `toolName` | string | Read-only. The function name runs expose, which is what `enabledTools` and smart tools pins have to name. It equals `name` for every snippet with a snake_case name |
| `description` | string | What the tool does and when to use it. The model reads it to decide when to call the tool |
| `language` | `javascript` \| `typescript` \| `python` | Runtime: Node 24 or Python 3.13 |
| `code` | string | Source code. It must define `main(input)` |
| `parameters` | object | JSON Schema of `input`, with `type: "object"`. Each property is a tool argument |
| `enabled` | boolean | Disabled snippets are never given to a run |
| `tags` | string[] | Free labels |
| `testCases` | `{ name, input }[]` | Saved inputs, at most 20. `test` runs them when you send no inputs |
| `projectId` | string | Files the snippet under a project. It does not restrict who can use it |
| `version` | number | Read-only. A change to the name, description, language, code or parameters saves a new version |
| `lastTest` | object | Read-only. `{ passed, failed, at }` of the last `test` of the current version |

### How the code runs

The code must define a function named `main` that takes one argument, `input`:
the arguments the model sent, as an object that matches `parameters`.

```javascript
function main(input) {
  const { point1, point2 } = input;
  // ...
  return { distance_meters: 3935746 };
}
```

```python
def main(input):
    city = input["city"]
    return {"city": city, "temperature": 21}
```

- `input` is validated against `parameters` **before** the code runs. An input
  that does not match never reaches `main`: the call fails with the schema
  error (`input/name: must have required property 'name'`).
- The return value is the tool result and must be JSON-serializable. `async`
  functions are awaited.
- `console.log` / `print` output is captured as logs. It is not the result.
- Each call runs in a fresh, isolated sandbox with a 10 s timeout by default
  (1–30 s). Nothing persists between calls.

---

## List snippets

```bash
GET /api/v1/code-snippets?language=python&enabled=true&limit=20
```

| Query | Description |
|-------|-------------|
| `search` | Text search over the name and the description |
| `language` | `javascript`, `typescript` or `python` |
| `enabled` | `true`, `false` or `all` (default: `all`) |
| `tag` | Only snippets with this tag |
| `projectId` | Only snippets of this project |
| `limit` / `offset` | Page size (default 20, at most 100) and items to skip |

```json
{
  "success": true,
  "data": {
    "snippets": [
      {
        "id": "69cd04386269dcabdddaa759",
        "name": "calculate_distance",
        "toolName": "calculate_distance",
        "description": "Distance in meters between two coordinates.",
        "language": "javascript",
        "enabled": true,
        "tags": ["geo"],
        "version": 2,
        "createdAt": "2026-04-01T10:00:00.000Z",
        "updatedAt": "2026-04-01T10:05:00.000Z"
      }
    ],
    "total": 1,
    "limit": 20,
    "offset": 0
  }
}
```

## Get a snippet

```bash
GET /api/v1/code-snippets/{snippetId}
```

Returns the listing fields plus `code`, `parameters`, `testCases` and
`lastTest`.

## Create a snippet

```bash
POST /api/v1/code-snippets
```

```json
{
  "name": "calculate_distance",
  "description": "Distance in meters between two coordinates. Use it when the user asks how far apart two places are.",
  "language": "javascript",
  "code": "function main(input) {\n  const toRad = (d) => (d * Math.PI) / 180;\n  const { a, b } = input;\n  const dLat = toRad(b.lat - a.lat);\n  const dLng = toRad(b.lng - a.lng);\n  const h = Math.sin(dLat / 2) ** 2 + Math.cos(toRad(a.lat)) * Math.cos(toRad(b.lat)) * Math.sin(dLng / 2) ** 2;\n  return { distance_meters: 2 * 6371000 * Math.asin(Math.sqrt(h)) };\n}\n",
  "parameters": {
    "type": "object",
    "properties": {
      "a": { "type": "object", "properties": { "lat": { "type": "number" }, "lng": { "type": "number" } }, "required": ["lat", "lng"] },
      "b": { "type": "object", "properties": { "lat": { "type": "number" }, "lng": { "type": "number" } }, "required": ["lat", "lng"] }
    },
    "required": ["a", "b"],
    "additionalProperties": false
  },
  "tags": ["geo"],
  "testCases": [
    { "name": "Madrid to Barcelona", "input": { "a": { "lat": 40.4168, "lng": -3.7038 }, "b": { "lat": 41.3874, "lng": 2.1686 } } }
  ]
}
```

Required: `name`, `description`, `language`, `code`, `parameters`. Returns the
created snippet (`201`), at version 1.

`400` when the name is not snake_case, `parameters` is not a JSON Schema with
`type: "object"`, a field is unknown, or the code is empty.

## Update a snippet

```bash
PATCH /api/v1/code-snippets/{snippetId}
```

Only the fields you send change. `testCases`, when sent, **replaces** the
saved list.

```json
{
  "code": "function main(input) { /* ... */ }",
  "expectedVersion": 2
}
```

Send `expectedVersion` — the `version` you read — to be refused with `409`
instead of overwriting a change someone saved meanwhile. Switching `enabled`,
`tags` or `testCases` does not create a version.

## Delete a snippet

```bash
DELETE /api/v1/code-snippets/{snippetId}
```

```json
{
  "success": true,
  "data": {
    "id": "69cd04386269dcabdddaa759",
    "deleted": true,
    "usedBy": {
      "agents": [{ "_id": "69cced12...", "name": "Nearby Places Finder" }],
      "assistants": []
    }
  }
}
```

Deleting does not edit the agents and assistants that list the snippet: they
keep its id in `codeSnippetIds` and their runs simply stop getting the tool.
`usedBy` tells you which ones to clean up.

## Test a snippet

```bash
POST /api/v1/code-snippets/{snippetId}/test
```

```json
{
  "inputs": [
    { "a": { "lat": 40.4168, "lng": -3.7038 }, "b": { "lat": 41.3874, "lng": 2.1686 } },
    { "a": { "lat": 40.4168 } }
  ],
  "timeout": 10000
}
```

Runs the **saved** snippet once per input, each in its own sandbox and with the
same input validation an agent call gets. Without `inputs`, its saved
`testCases` run; with neither, `400`. At most 20 inputs. A disabled snippet can
be tested.

```json
{
  "success": true,
  "data": {
    "passed": 1,
    "failed": 1,
    "results": [
      {
        "input": { "a": { "lat": 40.4168, "lng": -3.7038 }, "b": { "lat": 41.3874, "lng": 2.1686 } },
        "success": true,
        "output": { "distance_meters": 504629.4 },
        "logs": "",
        "executionTimeMs": 1618
      },
      {
        "input": { "a": { "lat": 40.4168 } },
        "success": false,
        "error": "input/b: must have required property 'b'; input/a/lng: must have required property 'lng'",
        "logs": "",
        "executionTimeMs": 7
      }
    ]
  }
}
```

A failing run is a result, not an error: the response is `200` and the run has
`success: false` with the `error`. Each run takes about 1.5–2 s (a sandbox is
created for it), so test with a handful of inputs rather than hundreds.

---

## Attaching a snippet

A snippet reaches a run through the entity's `codeSnippetIds`:

- **Assistants:** top-level `codeSnippetIds` — see [assistants.md](assistants.md).
- **Agents:** `assistantSpecialization.codeSnippetIds` — see [agents.md](agents.md).

```json
PATCH /api/v1/agents/{agentId}
{
  "assistantSpecialization": {
    "codeSnippetIds": ["69cd04386269dcabdddaa759"]
  }
}
```

Three rules decide whether a run actually gets the tool:

1. **`enabled`.** A disabled snippet is skipped, even if it is listed.
2. **`enabledTools`.** When the entity has an `enabledTools` list (not `null`),
   it is an allowlist over **every** tool, snippets included: add the
   snippet's `toolName` to it, or runs leave the snippet out without any error.
   With `enabledTools: null` every attached snippet is available.
3. **Name collisions.** If another tool of the entity already has the same
   name, that tool wins and the snippet is skipped.

With **smart tools** on, snippets are part of the tool catalogue like any other
tool: discoverable by default, or always loaded if their `toolName` is in
`smartTools.alwaysTools`.
