# MCP Apps in Claude (custom UI widgets)

Devic tool servers expose UI widgets that render inside Claude via the
**MCP Apps** extension (spec 2026-01-26) and inside ChatGPT via the OpenAI
Apps SDK. The same widget definition serves both ecosystems: `mcp-api-wrapper`
emits the standard `_meta.ui.*` keys (Claude) alongside legacy `openai/*` keys
(ChatGPT).

## Connecting a Devic tool server to Claude

Claude renders MCP Apps via **Custom Connectors**, available on Pro / Max /
Team plans.

1. Publish your tool server in Devic. Its public MCP endpoint follows the
   pattern `https://<slug>.mcp.devic.ai/mcp`.
2. In Claude: **Settings → Connectors → Add custom connector**.
3. Paste the MCP endpoint URL. Claude completes the OAuth flow exposed by
   `authRoutes` (the same flow used for ChatGPT).
4. Open a new chat. Tools that have a `widgetUid` will trigger their widget
   inline.

No `openai-apps-challenge` token is required for Claude; that endpoint is
ChatGPT-only.

For local development, run the wrapper locally and expose it with:

```bash
cloudflared tunnel --url http://localhost:<port>
```

Use the generated URL as the connector target.

## Widget configuration fields

Configure widgets in the SuntropyAI frontend under **Tools → MCP UIs**.
Each UI has a **Config** panel with a Sandbox section split into Claude /
ChatGPT tabs.

| Field | Audience | Notes |
|---|---|---|
| `domain` | ChatGPT only | **Leave empty for Claude.** The host serves the iframe from its own sandbox (`<hash>.claudemcpcontent.com`) and rejects any other origin under developer mode. Set only if you've registered a custom domain in the ChatGPT Apps directory. |
| `csp.connectDomains` | Both | Hosts your widget JS can `fetch`/`WebSocket` to. Leave empty if your widget only talks to the host via `callServerTool`. |
| `csp.resourceDomains` | Both | Hosts allowed for `<img>` / `<script>` / `<style>`. Leave empty for self-contained widgets. |
| `csp.frameDomains` | Both | Hosts allowed in nested iframes. |
| `permissions` | Claude only | `camera`, `microphone`, `geolocation`, `clipboard`. |
| `visibility` | Both | `["model","app"]` by default. Pick `["app"]` to hide the tool from the model and only allow widget-initiated calls. |

For a self-contained widget that only uses `callServerTool` /
`updateModelContext` (the recommended starting point), **every Sandbox field
can stay empty** — the host's defaults apply.

## Talking to the host from widget JS

`mcp-api-wrapper` inlines the `@modelcontextprotocol/ext-apps` client and
exposes it as `window.devic.app` already connected. Widget JS uses the same
API in both Claude and ChatGPT:

```js
// Send a tool call back to the server
const result = await window.devic.app.callServerTool({
  name: "get_product",
  arguments: { id: "sku-123" },
});

// Update the model's view of what the user is doing
await window.devic.app.updateModelContext({
  content: [{ type: "text", text: "User selected SKU sku-123" }],
});

// Read host context (theme / locale / displayMode)
const ctx = window.devic.app.getHostContext?.();
```

## Troubleshooting

* **`Invalid ui.domain format: expected "{hash}.claudemcpcontent.com"`** —
  you have a `domain` set in the widget and Claude's developer mode is on.
  Clear the **Domain** field in the editor and reload the connector.
* **Widget renders blank in Claude but works in ChatGPT** — usually a CSP
  violation. Open Claude's iframe devtools and check the console; add the
  blocked origin to the corresponding `csp.*` list.
* **`window.devic` is undefined** — the iframe failed to bootstrap the bridge.
  Make sure the wrapper was built with `yarn build` (which runs
  `bundle:ext-apps`); otherwise `extAppsBundle.generated.ts` is the empty
  placeholder.
* **Tool invisible to the model** — `visibility` is `["app"]`. Add `"model"`
  if the assistant should be able to trigger it.
* **`callServerTool` works but the model doesn't react** — by design.
  Widget-initiated tool calls return their result only to the iframe. To
  surface the data to the model, follow up with `devic.app.updateModelContext`.
