---
name: devic-mcp-apps
description: Build, configure and ship Devic MCP Apps — sandboxed HTML/JS widgets that render inside MCP clients like Claude Custom Connectors and ChatGPT, attached to your Devic tool servers. Covers the runtime contract, the Devic UI editor, and the public API for headless creation.
---

# Devic MCP Apps

An **MCP App** in Devic is a UI widget attached to a tool of a Devic tool
server. When the AI calls that tool, the host (Claude, ChatGPT) renders the
widget in a sandboxed iframe right inside the conversation, with the tool's
output already wired in. The same widget definition works in both ecosystems
— `mcp-api-wrapper` emits the standard MCP Apps `_meta.ui.*` keys (Claude,
spec 2026-01-26) and the legacy `openai/*` keys (ChatGPT Apps SDK)
simultaneously.

This skill covers:

1. [How MCP Apps work](#how-mcp-apps-work) — architecture and message flow.
2. [Developing a widget](#developing-a-widget) — HTML, CSS, JS and the
   `window.devic.app` runtime.
3. [Configuring in the Devic UI](#configuring-in-the-devic-ui) — the MCP UIs
   editor.
4. [Connecting from Claude and ChatGPT](#connecting-from-claude-and-chatgpt).
5. [Creating MCP Apps via the public API](#creating-mcp-apps-via-the-public-api).
6. [Troubleshooting](#troubleshooting).

---

## How MCP Apps work

```
┌──────────────────────────┐    tools/call (MCP)    ┌────────────────────────┐
│   Claude / ChatGPT host  │ ─────────────────────► │  mcp-api-wrapper       │
│   (sandboxed iframe)     │ ◄──────────────────── │   (Devic MCP proxy)    │
│                          │   tool result + _meta  │                        │
└──────────────────────────┘                        └────────────┬───────────┘
        ▲                                                        │
        │ window.devic.app                                       │ HTTP
        │  ↳ callServerTool                                      ▼
        │  ↳ updateModelContext                  ┌────────────────────────────┐
        │  ↳ sendMessage                         │     Your downstream API    │
        ▼                                        │  (the real tool endpoint)  │
┌──────────────────────────┐                     └────────────────────────────┘
│   Widget JS (your code)  │
└──────────────────────────┘
```

### Anatomy of a widget

A widget lives in the tool server's **definition** as one entry of the
`uiComponents[]` array. Each widget has:

| Field | What it is |
|---|---|
| `uid` | Unique id within the tool server (e.g. `widget_product_card`). |
| `name` | Display name. |
| `html` | The widget body (no `<html>`, `<head>` or `<body>` tags). |
| `css` | Optional styles, injected in a single `<style>` block. |
| `js` | An ES module body. Has `window.devic.app` already connected. |
| `description` | Short text shown to the host. |
| `invokingStates.invokingText` | Status line while the tool is running. |
| `invokingStates.invokedText` | Status line once the result is rendered. |
| `widgetPrefersBorder` | Render with a bordered card. |
| `domain` | iframe origin override (ChatGPT only — leave empty for Claude). |
| `csp` | Standard MCP Apps CSP (Claude). |
| `legacyOpenaiCsp` | Optional snake_case CSP for ChatGPT. Derived from `csp` when absent. |
| `permissions` | Sandbox permissions, Claude-only: `camera`, `microphone`, `geolocation`, `clipboard`. |
| `visibility` | Who can invoke the owning tool: `["model"]`, `["app"]`, or both (default). |
| `host` | Editor hint: `claude` or `openai`. Runtime emits metadata for both regardless. |

To attach a widget to a tool, set the tool's `widgetUid` to the widget's
`uid`. One widget can be reused by many tools.

### Render pipeline

When a tool with a widget is called:

1. The host opens an iframe pointing at the widget resource served by the
   `mcp-api-wrapper`.
2. The wrapper inlines `@modelcontextprotocol/ext-apps` (the official MCP
   Apps client) and a small bridge so the widget can talk to the host
   without writing any boilerplate.
3. Your `js` runs as an ES module. `window.devic.app` is already connected
   and the tool result is available via the bridge.
4. The widget can call other tools, push context updates, or send chat
   messages on the user's behalf.

The widget never has direct access to the user's MCP credentials. Calls
back through the host (`callServerTool`) go through the host's normal
authorization — your widget JS cannot smuggle requests outside the
declared CSP.

---

## Developing a widget

The simplest working widget is three blocks of code (HTML, CSS, JS) plus a
tool that points at it via `widgetUid`. You don't need to set any of the
Sandbox fields for a self-contained widget — the host's defaults apply.

### Minimal widget

```html
<!-- html -->
<div id="card" class="card">
  <h2 id="title">Loading…</h2>
  <p id="body"></p>
  <button id="refresh">Refresh</button>
</div>
```

```css
/* css */
.card { font-family: system-ui; padding: 16px; border-radius: 8px; background: #f8fafc; }
.card h2 { margin: 0 0 6px; font-size: 16px; }
.card button { margin-top: 8px; padding: 6px 10px; cursor: pointer; }
```

```js
// js (ES module)
const app = window.devic.app;

// The host gives us the tool result that triggered this widget render
app.ontoolresult = (result) => {
  document.getElementById("title").textContent = result.structuredContent?.name ?? "Item";
  document.getElementById("body").textContent  = result.structuredContent?.description ?? "";
};

// User clicks → call another tool on the same MCP server
document.getElementById("refresh").addEventListener("click", async () => {
  const fresh = await app.callServerTool({
    name: "get_product",
    arguments: { id: result.structuredContent?.id },
  });
  // Optionally surface this update to the model:
  await app.updateModelContext({
    content: [{ type: "text", text: "User refreshed the product card." }],
  });
});
```

### The `window.devic.app` runtime

The bridge exposes a small surface. All methods are async and return
promises.

| Method | Purpose |
|---|---|
| `app.callServerTool({ name, arguments })` | Invoke another tool of the same MCP server. Returns the raw MCP `tools/call` result. The model does **not** see this unless followed by `updateModelContext`. |
| `app.updateModelContext({ content })` | Inject content into the model's next turn. Use after `callServerTool` if you want the model to react. |
| `app.sendMessage({ role, content })` | Send a message in the chat on the user's behalf. `role` is `"user"` or `"assistant"`. `content` is an array of MCP content blocks. |
| `app.getHostContext?.()` | Read host context: theme, locale, displayMode. |
| `app.ontoolresult = fn` | Callback for the tool result that triggered the widget. |
| `app.onhostcontextchanged = fn` | Callback when host context (theme/locale) changes. |

### Cross-host autodetection

The `App` from `@modelcontextprotocol/ext-apps` autodetects the host: in
Claude it speaks JSON-RPC over `postMessage`; in ChatGPT it delegates to
`window.openai`. Your widget code does **not** need to branch on host.

### CSP / Domain / Permissions / Visibility

Most widgets need none of these. Set them only when you hit a wall:

* `csp.connectDomains` — extra hosts your widget JS can `fetch`/`WebSocket` to.
* `csp.resourceDomains` — extra hosts for `<img>`, `<script>`, `<style>`.
* `csp.frameDomains` — extra hosts allowed in nested iframes.
* `csp.baseUriDomains` — extra hosts allowed in `base-uri`.
* `permissions` (Claude only) — request `camera`, `microphone`,
  `geolocation`, `clipboard`.
* `visibility` — `["model","app"]` by default. Set `["app"]` to hide the
  tool from the model (only callable from the iframe).
* `domain` — leave **empty** for Claude. Set only if you have a registered
  ChatGPT Apps custom domain.

---

## Configuring in the Devic UI

In Devic, open the tool server, then go to the **MCP UIs** tab (gated by
the `VITE_ENABLE_MCP_APPS` flag — enabled on staging and prod).

### List view

The first screen lists all widgets registered on the tool server.

* **Create** → choose Claude or ChatGPT as the primary host (this only
  drives editor defaults; runtime emits metadata for both regardless).
* **Empty state** also shows the two host buttons so you can start from
  scratch.
* Each row shows the host icon (white container behind the OpenAI black
  icon so it reads in dark mode).

### Edit view

The widget editor has three tabs:

* **Editor** — Monaco with three sub-panels for HTML, CSS, JS. The JS
  panel is preloaded with `@modelcontextprotocol/ext-apps` ambient TS
  declarations so autocompletion works for `window.devic.app.*` and tool
  names of the current server are typed as a `MCPToolName` union.
* **Sandbox** — Domain (ChatGPT-only), CSP fields (connect / resource /
  frame / base-uri), Permissions, Visibility. Tooltips explain each one.
* **API** — read-only view of the runtime declarations that the editor
  injects, useful as a quick reference.

A side panel shows **Snippets** for the most common widget tasks
(read tool result, call a tool, push context, send a chat message).

### Binding a tool to a widget

A widget on its own doesn't do anything — the AI needs to call a tool that
declares the widget via `widgetUid`. In the **Tools** tab of the tool
server, pick a tool, scroll to the **UI** section, and select the widget
from the dropdown.

### OAuth proxy (separate from MCP Apps)

When the underlying API needs an API key per user (the common case),
enable **OAuth proxy** in the Authentication tab. Optionally click
"Customize screen" to brand the `/authorize` page (logo, favicon, primary
color, header, message). Claude and ChatGPT will both open that page
during the connector setup.

---

## Connecting from Claude and ChatGPT

### Claude (Custom Connector)

Claude renders MCP Apps via **Custom Connectors**, available on Pro / Max /
Team plans.

1. Publish your tool server in Devic. Its public MCP endpoint follows the
   pattern `https://<slug>.mcp.devic.ai/mcp`.
2. In Claude: **Settings → Connectors → Add custom connector**.
3. Paste the MCP endpoint URL. Claude completes the OAuth flow served by
   the wrapper (`/.well-known/oauth-protected-resource`, `/authorize`,
   `/token`).
4. Start a new chat. Tools with a `widgetUid` will trigger the widget
   inline when invoked.

No `openai-apps-challenge` token is required for Claude — that endpoint is
ChatGPT-only.

For local development, run the wrapper locally and expose it with:

```bash
cloudflared tunnel --url http://localhost:<port>
```

Use the generated URL as the connector target.

### ChatGPT (Apps SDK)

ChatGPT uses the same MCP endpoint plus the `openai-apps-challenge` token
returned by `GET /.well-known/openai-apps-challenge/<server_id>`. Generate
that token from the Devic UI (Authentication → ChatGPT Apps) and paste it
into the ChatGPT custom connector form.

---

## Creating MCP Apps via the public API

Widgets live inside a tool server's **definition**. The public API exposes
two endpoints to manage them:

```
GET   /api/v1/tool-servers/:toolServerId/definition
PATCH /api/v1/tool-servers/:toolServerId/definition
```

`PATCH` accepts the full `uiComponents[]` (and `toolDefinitions[]`) and
overwrites whatever was there.

### Authentication

All Devic API endpoints accept a JWT API key from the dashboard:

```
Authorization: Bearer devic-xxxxxxxxxxxxxxxxxxxxxxxx
```

### End-to-end: create a tool server with a widget

1. **Create the tool server**:

```bash
curl -X POST "https://api.devic.ai/api/v1/tool-servers" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Product API",
    "description": "Product catalog",
    "url": "https://api.example.com",
    "identifier": "product-api",
    "enabled": true,
    "mcpType": true,
    "toolDefinitions": [
      {
        "type": "function",
        "function": {
          "name": "get_product",
          "description": "Get a product card by SKU",
          "parameters": {
            "type": "object",
            "properties": {
              "id": { "type": "string", "description": "Product SKU" }
            },
            "required": ["id"]
          }
        },
        "endpoint": "/products/${id}",
        "method": "GET",
        "pathParametersKeys": ["id"]
      }
    ]
  }'
```

Save the returned `_id` (tool server id) and `toolServerDefinitionId`.

2. **Add a widget to the definition** with `PATCH /:toolServerId/definition`:

```bash
curl -X PATCH "https://api.devic.ai/api/v1/tool-servers/<toolServerId>/definition" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "uiComponents": [
      {
        "uid": "widget_product_card",
        "name": "product-card",
        "description": "Renders a product card with an SKU refresh button",
        "html": "<div id=\"card\"><h2 id=\"title\">Loading…</h2><p id=\"body\"></p><button id=\"refresh\">Refresh</button></div>",
        "css": ".card { font-family: system-ui; padding: 16px; }",
        "js": "const app = window.devic.app; app.ontoolresult = (r) => { document.getElementById(\"title\").textContent = r.structuredContent?.name ?? \"Item\"; };",
        "invokingStates": {
          "invokingText": "Loading product card…",
          "invokedText": "Product card ready"
        },
        "widgetPrefersBorder": true,
        "visibility": ["model", "app"]
      }
    ]
  }'
```

3. **Bind a tool to the widget** by updating the tool definition's
   `widgetUid`:

```bash
curl -X PATCH "https://api.devic.ai/api/v1/tool-servers/<toolServerId>/tools/get_product" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "widgetUid": "widget_product_card"
  }'
```

That's it. The next call to `get_product` from Claude or ChatGPT will
render the widget with the tool result.

### `uiComponents[]` schema (reference)

```json
{
  "uid": "widget_xxx",
  "name": "human-readable-name",
  "html": "<div>…</div>",
  "css": ".card { … }",
  "js": "const app = window.devic.app; …",
  "description": "What this widget shows",
  "widgetPrefersBorder": true,
  "invokingStates": {
    "invokingText": "Loading…",
    "invokedText": "Ready"
  },
  "domain": "https://my-server.mcp.devic.ai",
  "csp": {
    "connectDomains": ["https://api.example.com"],
    "resourceDomains": ["https://cdn.example.com"],
    "frameDomains": [],
    "baseUriDomains": []
  },
  "legacyOpenaiCsp": {
    "connect_domains": ["https://api.example.com"]
  },
  "permissions": ["clipboard"],
  "visibility": ["model", "app"],
  "host": "claude"
}
```

* `uid` is the join key with `toolDefinition.widgetUid`.
* `js` runs as an **ES module**. You can `import` static URLs only when
  the host's CSP allows it (or when the bundle is fully inline).
* `domain` should usually be empty. Claude rejects values that don't match
  its sandbox host (`<hash>.claudemcpcontent.com`) in developer mode.
* `legacyOpenaiCsp` is optional; when absent the wrapper derives it from
  `csp` at registration time.

### Headless full replace

To rewrite the entire UI side of a tool server in one call:

```bash
curl -X PATCH "https://api.devic.ai/api/v1/tool-servers/<toolServerId>/definition" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "uiComponents": [ /* full list, this replaces the array */ ],
    "toolDefinitions": [ /* full list, this replaces the array */ ]
  }'
```

Tools that don't reference a `widgetUid` work unchanged — adding widgets
is additive from the client's perspective.

---

## Troubleshooting

* **`Invalid ui.domain format: expected "{hash}.claudemcpcontent.com"`** —
  there is a `domain` set on the widget and Claude developer mode is on.
  Clear the **Domain** field and reload the connector.
* **Widget renders blank in Claude but works in ChatGPT** — almost always
  a CSP violation. Open Claude's iframe devtools, read the console, add
  the blocked origin to the corresponding `csp.*` list.
* **`window.devic` is undefined** — the iframe failed to bootstrap the
  bridge. Make sure the wrapper was built with `yarn build` (which runs
  `bundle:ext-apps`); otherwise `extAppsBundle.generated.ts` is the empty
  placeholder.
* **Tool invisible to the model** — `visibility` is `["app"]`. Add
  `"model"` if the assistant should be able to trigger it.
* **`callServerTool` works but the model doesn't react** — by design.
  Widget-initiated tool calls return their result only to the iframe. To
  surface the data to the model, follow up with
  `app.updateModelContext({ content: [...] })`.
* **Connector icon shows Devic instead of my logo** — the wrapper serves
  `/favicon.ico` redirecting to `proxyFaviconUrl → proxyLogoUrl →
  imageUrl`. Set the tool server image, or upload a logo/favicon in the
  OAuth proxy "Customize screen" modal.
* **Claude doesn't ask for auth** — the wrapper only triggers the OAuth
  flow when one of `oAuthProxyEnabledConfig.enabled`, `oauth2`, `jwt`, or
  `customHeader` is set on the tool server's authentication config. Make
  sure your tool server has the OAuth proxy enabled if you want Claude to
  prompt the user for an API key.

For deeper Claude-specific guidance (Custom Connector setup, plan
requirements), see [`devic-api/mcp_apps_claude.md`](../devic-api/mcp_apps_claude.md).
