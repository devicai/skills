---
name: devic-mcp-apps-github
description: Link Devic MCP App widgets to a folder in a GitHub repository so the source code becomes the runtime source of truth. Covers the opinionated repo layout, the code Devic injects around your JS, the connect/scan/bulk-import UI flow, and the gotcha of "self-contained" widgets that try to ship their own MCP Apps client.
---

# Devic MCP Apps — GitHub-linked widgets

A widget in Devic can be **linked to a folder in a GitHub repository**.
When the widget is rendered, `mcp-api-wrapper` resolves each file from
GitHub at runtime using the user's stored access token, instead of from
the copy persisted in MongoDB. The repo becomes the source of truth:
push to `main` and the next widget render serves the new code, no Devic
re-save required.

This skill assumes you already know the basics — for the runtime
contract (`window.devic.app`, `callServerTool`, tool↔widget binding,
visibility, CSP) see the **`devic-mcp-apps`** skill. This one covers
only the GitHub layer on top.

---

## 1. Opinionated folder layout

Each widget lives under its own folder. Devic's scanner classifies any
folder containing `index.html` as a widget. The full set of recognised
files:

```
<your-repo>/
  widgets/                       (folder name is arbitrary; "widgets/" is convention)
    hello/
      index.html                 REQUIRED — body markup (no <html>/<head>/<body>)
      index.css                  Styles, scoped via a root class on your markup
      index.js                   ES module — your widget code
      csp.yml      OR  csp.json  CSP declarations (defaults to deny-all)
      widget.md                  Human-readable spec — Devic parses it
```

Devic doesn't enforce the parent folder name. `widgets/hello`,
`apps/map-picker`, or `hello` at the repo root are all valid as long as
the leaf folder has at least `index.html`. The scanner walks the whole
tree once and reports every match.

### What Devic reads from `widget.md`

When a widget folder has a `widget.md`, Devic parses two things and
surfaces them in the picker UI:

- **Title** — the first `# H1` heading.
- **Description** — the first prose paragraph (skipping headings,
  lists, blockquotes, code fences and table rows).

This is purely metadata for humans browsing the repo. The widget's
behaviour comes only from `index.html` / `index.css` / `index.js`.

---

## 2. How Devic resolves the files at runtime

When a tool with a GitHub-linked widget is invoked:

1. The MCP host opens an iframe pointing at the widget resource served
   by `mcp-api-wrapper`.
2. The wrapper looks up the link on the widget definition
   (`{ owner, repo, path, ref }`) and calls Security's internal
   GitHub Contents API for each of the five expected files in parallel.
3. The base64-encoded file contents come back from GitHub, are decoded,
   composed into a single HTML document, and returned to the host.
4. Missing files are tolerated: a folder with just `index.html` works;
   the others fall back to empty strings (and the CSP defaults to
   deny-all).

Concretely the wrapper uses:

```
GET /auth/internal/github/repos/{owner}/{repo}/contents
    ?path=<widget-folder>/index.html&ref=<ref>
```

…repeated for each file, gated by a shared internal-service header
between `mcp-api-wrapper` and Security. The user's GitHub access token
is **never** sent to the widget or stored outside Security.

---

## 3. What Devic injects around your `index.js`

This is the part most people get wrong. Even though your repo only
contains five files, the iframe the host loads is **not** a 1:1 mirror
of those files. The wrapper composes an HTML document that looks
roughly like:

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8" />
  <title>{widget-name}</title>

  <!-- (1) Import map that maps the bare "ext-apps" specifier to an     -->
  <!--     inlined data:// URL of @modelcontextprotocol/ext-apps.       -->
  <script type="importmap">
    { "imports": { "ext-apps": "data:text/javascript;base64,..." } }
  </script>

  <style>/* your index.css */</style>
</head>
<body>
  <div id="{widget-name}">/* your index.html */</div>

  <!-- (2) Devic bridge — runs BEFORE your JS, no opt-out. -->
  <script type="module">
    import { App } from "ext-apps";
    const __devicApp = new App({ name: "{widget-name}", version: "1.0.0" });

    // Buffer the host's tool-result so widgets that register
    // `ontoolresult` after an async setup step don't miss it.
    let __devicCachedToolResult = null;
    __devicApp.addEventListener("toolresult", (params) => {
      __devicCachedToolResult = params;
    });
    // (Replay shim — if your code later sets `app.ontoolresult = ...`,
    //  the cached result is delivered immediately.)

    await __devicApp.connect();
    window.devic = {
      app: __devicApp,
      get lastToolResult() { return __devicCachedToolResult; },
    };
  </script>

  <!-- (3) Your index.js, also as an ES module. -->
  <script type="module">/* your index.js */</script>
</body>
</html>
```

Three implications you need to internalise:

1. **`window.devic.app` is ready before your JS runs.** It is a
   connected `App` instance from `@modelcontextprotocol/ext-apps`. Just
   use it.
2. **`ontoolresult` is buffered**. If your widget does
   `await bootSomeLibrary()` before assigning `app.ontoolresult = fn`,
   the cached payload is delivered to `fn` the instant you assign it.
   No more "the tool fired before my widget was ready".
3. **Your JS runs as an ES module** (`<script type="module">`). Top-level
   `await` works. `import` from a CDN works only if the CSP allows the
   origin.

### The minimum useful widget

```js
// index.js
const app = window.devic.app;

app.ontoolresult = (result) => {
  document.getElementById("title").textContent =
    result.structuredContent?.name ?? "Item";
};
```

That's it. No `new App(...)`, no `connect()`, no postMessage plumbing.

---

## 4. "But my widget is self-contained — it already imports ext-apps"

**Short answer: don't. Strip those imports and use `window.devic.app`.**

Long answer, because this is a real foot-gun if you port an existing
ChatGPT-Apps or claude.com Custom-Connector widget into Devic:

| What you wrote | What actually happens |
|---|---|
| `import { App } from "@modelcontextprotocol/ext-apps"` | Fails: the importmap only registers the bare specifier `"ext-apps"`, not the scoped one. Module-not-found, your JS doesn't execute. |
| `import { App } from "https://esm.sh/@modelcontextprotocol/ext-apps"` | Blocked by CSP unless you whitelist `esm.sh` in your `csp.yml` — and even then you now have two `App` clients in the same iframe (the bridge already created one). |
| `import { App } from "ext-apps"` | Resolves (the importmap covers it), but you instantiate a **second** `App` and call `connect()` on it. Two clients race for `postMessage` notifications from the host. Observed effects: duplicated `ontoolresult` callbacks on one client and missed events on the other, double `callServerTool` round-trips, or silent hangs. The MCP Apps spec does not define behaviour for two clients per frame. |

Devic's bridge is **not opt-in** — there's no flag that disables it.
The bridge always runs first, and after it `window.devic.app` is the
canonical, already-connected client. If you're porting a widget that
shipped its own MCP Apps client, the migration is mechanical:

```diff
- import { App } from "@modelcontextprotocol/ext-apps";
- const app = new App({ name: "my-widget", version: "1.0.0" });
- await app.connect();
+ const app = window.devic.app;

  app.ontoolresult = render;
  // …rest stays the same.
```

If you depend on listeners attached **before** `connect()` to avoid
missing the first `toolresult`, you don't need that workaround in
Devic — the bridge already buffers and replays.

---

## 5. CSP — `csp.yml`

Each widget folder may include a `csp.yml` (or `csp.json`). The four
recognised keys map 1:1 to the MCP Apps spec (Claude) and are also
translated to the legacy `openai/widgetCSP` for ChatGPT:

```yaml
connectDomains: []   # fetch(), XHR, WebSocket targets
resourceDomains: []  # img-src / media-src / font-src
frameDomains: []     # iframes inside the widget
baseUriDomains: []   # <base href="…"> targets
```

Defaults: everything **deny**. Inline CSS/JS already work because the
wrapper composes them inline (`unsafe-inline` granted by the host's
iframe sandbox). External CDNs do not — declare them here or bundle.

If you forget the `csp.yml` entirely, the defaults apply. If you have
one but it's empty (e.g. just comments), the same defaults apply.

---

## 6. Linking from the Devic UI

There are three entry points, all in the **MCP UIs** editor of a tool
server:

### a) From inside a specific widget
The widget editor has a **"Link to GitHub"** button. The modal opens,
you pick a repo, Devic scans it and shows every detected widget folder
as a card with file-presence tags. Selecting one and confirming:
- Stores `{ owner, repo, path, ref }` on the widget.
- **Renames** the widget to the folder leaf (e.g. `new_widget` becomes
  `hello`).
- If the repo had more widgets not yet linked to this tool server,
  pops a follow-up dialog asking which extras to import as new
  components.

### b) From the widget list — "Connect repository"
A button in the list-view header (or the empty state) lets you start
from the repo side. Devic synthesises a blank placeholder widget, runs
you through the same picker, and drops the placeholder if you cancel.

### c) Ghost cards
Once any widget in the tool server is bound to a repo, that repo
appears as a **header card** above the list, with counts and Open /
Disconnect actions. Below the header, every widget the scan found in
the repo that **isn't bound yet** appears as a translucent "ghost
card" with an inline **Link** button — and a **Link all** primary
button to import them in one click.

---

## 7. Workflows

### Adding a new widget to an existing linked repo
1. Push the new folder to the repo (`widgets/<new>/{index.html,…}`).
2. Open the Devic widget editor for the tool server.
3. The header card refreshes the scan on mount — a new ghost card
   appears for the new folder.
4. Click **Link** on the ghost (or **Link all** if you added many).

### Iterating on a linked widget
You don't need to do anything in Devic. Push to the repo, refresh the
chat in Claude/ChatGPT — the next tool call serves the new bundle.
There's no caching layer between the wrapper and GitHub other than
GitHub's own.

### Pinning to a tag or commit
Set `ref` on the link to a tag (`v1.2.0`) or commit SHA. The wrapper
forwards `ref` to GitHub's Contents API verbatim. Helpful when you
want a deterministic version in production while iterating on `main`.

### Disconnecting
- **Per-widget**: `Edit link` → `Unlink` on the banner above the editor.
  The widget reverts to whatever was last in its local `html/css/js`
  fields.
- **Whole repo**: the **Disconnect** button on the repo header card
  clears `githubLink` from every widget bound to that repo. The
  widgets themselves remain, just unlinked.

---

## 8. Security model (short version)

- The user's GitHub OAuth access token lives **encrypted at rest** in
  the Security service (`client.githubAccessToken`).
- The wrapper never sees the token — it calls
  `/auth/internal/github/repos/.../contents` on Security with a shared
  internal-service-key header, and `x-devic-client-uid` identifies the
  tenant. Security decrypts, calls GitHub, returns the file content.
- Path traversal and shell-metachar attempts are rejected at every
  layer (frontend DTO, Security regex, mcp-api-wrapper pre-validation).
- The fetched widget code runs in the host's sandboxed iframe under
  the CSP you declared. It does **not** have access to the user's
  Devic session or to GitHub credentials.

---

## 9. Troubleshooting

| Symptom | Likely cause |
|---|---|
| Modal lists no widgets after picking a repo | Repo has no folder containing `index.html`. Check the scan output — `GET /auth/github/repos/{owner}/{repo}/scan-widgets`. |
| Widget renders blank in preview | If linked, ensure the editor finished loading from GitHub — the preview reads from `linkedContents`, not the local fields. |
| `Error: Failed to resolve module specifier "@modelcontextprotocol/ext-apps"` | You're trying to import the lib directly in your `index.js`. Use `window.devic.app` instead — see §4. |
| Duplicate `ontoolresult` callbacks | Same root cause — you instantiated a second `App` alongside the bridge. Drop the `new App(...)` and `connect()` calls. |
| `redirect_uri_mismatch` when connecting GitHub | The OAuth App registered in GitHub doesn't list your environment's `/login` (or a parent) as an authorisation callback URL. |
| Linked widget still serves old code after a push | Force a fresh render in the host (Claude/ChatGPT often caches the iframe by `_meta` URL). Edit and re-save the tool server to bump the resource version, or change the `ref`. |
| Tools tab in editor disabled / "GitHub is not connected" | The user signed in via email/password and hasn't run the standalone Connect GitHub flow yet. Open the widget editor → **Link to GitHub** → **Connect GitHub** button in the warning Alert. |
