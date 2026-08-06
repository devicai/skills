---
name: devic-ui
description: Devic UI is a react component library to integrate AI UI components like chats and agents executions handler directly in your code base connected to devicai API. Covers tenant sessions (signed credentials instead of an API key in the bundle) and connected apps.
---

# Devic UI Integration Guide

This guide explains how to integrate the `@devicai/ui` library into your React application to add AI assistant chat capabilities.

## Prerequisites

- Node.js 20+
- React 17+ application
- Devic API key (obtain from Devic dashboard) — or, for a page real users reach,
  a backend endpoint that mints tenant sessions (see
  [tenant-sessions.md](tenant-sessions.md))

## Installation

```bash
npm install @devicai/ui
# or
yarn add @devicai/ui
# or
pnpm add @devicai/ui
```

## Basic Integration

### Step 1: Import Styles

Add the CSS import to your application entry point:

```tsx
// App.tsx or index.tsx
import '@devicai/ui/styles.css';
```

### Step 2: Wrap Your App with DevicProvider

```tsx
import { DevicProvider } from '@devicai/ui';

function App() {
  return (
    <DevicProvider
      apiKey="your-devic-api-key"
      baseUrl="https://api.devic.ai"  // Optional, defaults to this
    >
      <YourApp />
    </DevicProvider>
  );
}
```

> For anything user-facing, replace `apiKey` with `getTenantSession` — a key in
> a bundle lets any visitor claim to be any of your customers. See
> [Proving the tenant](#proving-the-tenant-tenant-sessions).

### Step 3: Add ChatDrawer Component

```tsx
import { ChatDrawer } from '@devicai/ui';

function YourApp() {
  return (
    <div>
      {/* Your app content */}

      <ChatDrawer
        assistantId="your-assistant-identifier"
        options={{
          position: 'right',
          welcomeMessage: 'Hello! How can I help you today?',
          suggestedMessages: [
            'Help me get started',
            'What can you do?',
            {
              content: <><span>🚀</span> Launch a workflow</>,
              message: 'I want to launch a workflow',
            },
          ],
        }}
      />
    </div>
  );
}
```

## Multi-Tenant Integration

For SaaS applications with multiple tenants:

```tsx
<DevicProvider
  apiKey="your-api-key"
  tenantId="acme-corp"                                  // the customer/organization
  tenantMetadata={{ name: 'Acme Corp', email: 'billing@acme.com', imageUrl: 'https://acme.com/logo.png' }}
  subtenantId="user-456"                                // the end user inside the tenant
  subtenantMetadata={{ id: 'user-456', name: 'Jane', email: 'jane@acme.com', imageUrl: 'https://…/jane.png' }}
>
  <ChatDrawer
    assistantId="support-assistant"
    tenantId="acme-corp"            // overrides the provider value
    subtenantId="user-456"          // overrides the provider value
    showUsageBar="onDemand"         // show a "Usage" toggle above the input
  />
</DevicProvider>
```

The `tenantId` / `subtenantId` you pass are forwarded to the Devic API on every message and **auto-register** the tenant and subtenant on the platform: the conversation is attributed to them and cost/usage roll up under that tenant (and per subtenant) in the Devic dashboard and Tenants API. `subtenantId` is sent explicitly; when omitted it falls back to `subtenantMetadata.id` (or the legacy `tenantMetadata.userId`). Both `tenantId`/`subtenantId` and their `*Metadata` can be set globally on the provider and overridden per `ChatDrawer`. See the `devic-api` skill's tenants reference for the management/usage/limits endpoints.

`tenantMetadata` and `subtenantMetadata` are typed (exported **`TenantMetadata`** and **`SubtenantMetadata`**, since `0.29.0`): they give first-class fields — `name`/`displayName`, `email`, `imageUrl` (and `id` for subtenants) — used to enrich the tenant/subtenant display record (e.g. the avatar shown in the Devic Tenants UI), while still allowing any extra integrator-defined key (`[key: string]: any`). Sending these is optional; the avatar falls back to the subtenant's initials when no `imageUrl` is provided.

### Conversation list scoping

The conversation selector in `ChatDrawer` filters the listed conversations by the resolved scope. When `subtenantId` is set (via provider or `ChatDrawer` prop), each end user sees **only their own** conversations rather than every conversation of the tenant; with only `tenantId` set, the list is scoped to the tenant. Resolution follows the usual precedence (explicit `ChatDrawer` prop overrides the `DevicProvider` value).

Using the client directly, `listConversations` accepts both scopes:

```tsx
const { histories, total } = await client.listConversations('assistant-id', {
  tenantId: 'acme-corp',
  subtenantId: 'user-456', // optional — omit to list at the tenant level
  offset: 0,
  limit: 20,
});
```

Backed by the `subtenantId` query param on `GET /api/v1/assistants/:id/chats` (matched against the canonical `metadata.subtenantId`). Sending it against an older backend is harmless — unknown query params are ignored, so the list simply isn't subtenant-filtered.

## Proving the tenant (tenant sessions)

Everything above declares the tenant. In a browser that is a claim, not a fact:
the `apiKey` is in your bundle, so anyone who reads it can put another
customer's `tenantId` beside it and get their conversations back.
`allowedDomains` narrows *where* a key works, never *who* is behind it.

Since **0.41.0** the page can carry a short-lived token instead, minted by your
backend from your own login. `apiKey` becomes optional — and should be absent:

```tsx
<DevicProvider
  getTenantSession={async () => {
    const r = await fetch('/api/devic-session', { credentials: 'include' });
    return r.json();          // { token, expiresAt } — or the bare token string
  }}
  onSessionExpired={() => location.assign('/login')}
>
  <ChatDrawer assistantId="support-bot" />
</DevicProvider>
```

The widget renews it on its own before expiry, and one session is shared by
every component below the provider. The tenant inside the token is then imposed
on path, query and body server-side, so `tenantId` props become redundant:
passing the matching value is harmless, passing a different one answers `403`.

Mint it on your server with a **server-side** key — the issuer refuses any
request carrying an `Origin` header, or made with a key that has
`allowedDomains`:

```ts
app.post('/api/devic-session', requireLogin, async (req, res) => {
  const r = await fetch('https://api.devic.ai/api/v1/tenant-sessions', {
    method: 'POST',
    headers: { Authorization: `Bearer ${process.env.DEVIC_API_KEY}`,
               'Content-Type': 'application/json' },
    body: JSON.stringify({                 // from YOUR session, never the body
      tenantId: req.user.organisationId,
      subtenantId: req.user.id,
    }),
  });
  res.json(await r.json());
});
```

Put that key in **`signed`** mode in the console and minting becomes the only
thing it can do — every other `/api/v1` call with the key alone answers `401`.

For the full guide — `createSharedSession` (≥ 0.41.1), sessions from a cookie
with no renewal endpoint, what a session may reach, and the migration
checklist — see [tenant-sessions.md](tenant-sessions.md).

## Connected apps

Where the **end user** connects their **own** third-party accounts to the
assistant they are talking to (≥ 0.38.0). The drawer grows a stack of app logos
in its header when the assistant offers apps to its tenants, and renders nothing
— and requests nothing — when it does not:

```tsx
<ChatDrawer
  assistantId="support-bot"
  tenantId="acme-corp"
  subtenantId="user-456"
  options={{
    showIntegrationsButton: true,     // default; false keeps it out entirely
    integrationsLabel: 'Connected apps',
    maxIntegrationLogos: 6,
    showIntegrationsHint: true,       // strip above the composer, dismissible
  }}
/>
```

Since **0.42.0** it knows without asking: `GET /api/v1/assistants/:id` returns
`tenantIntegrations: { enabled, count }` on the call the drawer already makes
for its header, so an assistant offering nothing costs no extra request, and
`count` holds the header's place while the listing arrives. An **absent** field
(older backend) means *cannot tell*, not *no* — the listing is requested as
before.

`IntegrationsModal`, `IntegrationsLauncher`, `IntegrationsHint`,
`IntegrationLogo` and `useIntegrations` are exported for your own UI, as are
`useAssistantInfo` / `forgetAssistant` (the once-per-assistant cache behind the
header). See [connected-apps.md](connected-apps.md).

## Conversation Tags

Tags label a conversation. They are a **top-level** property of the message
(the public API's `tags: string[]`), **distinct from `metadata` and from
`tenantId`/`subtenantId`** — do **not** put them in `tenantMetadata` (that
travels inside `metadata`, a different field, and won't become real tags).

You can set them at three levels; they are **merged and deduped**
(provider ∪ component ∪ per-message), following the same precedence model as
`tenantId`:

```tsx
// 1) Globally on the provider — applied to every conversation
<DevicProvider apiKey="devic-xxx" tags={['web-app']}>
  {/* 2) Per ChatDrawer (also AICommandBar / AIGenerationButton) */}
  <ChatDrawer assistantId="support" tags={['support']} />
</DevicProvider>
```

```tsx
// 3) Per message — via useDevicChat or a customPromptBox
const { sendMessage } = useDevicChat({ assistantId: 'support', tags: ['support'] });
sendMessage('My card was declined', { tags: ['urgent', 'billing'] });
// sent tags = ['web-app', 'support', 'urgent', 'billing'] (deduped)

// customPromptBox receives the same per-message channel:
//   sendMessage(text, files, { tags: ['urgent'] })
```

`tags` is accepted as a prop by `ChatDrawer`, `AICommandBar` and
`AIGenerationButton`, and as an option by `useDevicChat` / `useAICommandBar` /
`useAIGenerationButton`. All three components also inherit the provider's
global `tags`.

### Listing existing tags

To power autocompletion or filters, read the unique tags used across the
account's chat histories:

```tsx
const tags = await client.getChatTags(); // string[] — GET /api/v1/assistants/tags
```

Conversations returned by `listConversations` / chat history endpoints carry
their own `tags` array. See the `devic-api` skill (assistants reference) for the
server-side `tags` filter on the chats listing.

### Usage bar

When the tenant/subtenant has usage limits configured, `ChatDrawer` can render a usage bar above the input, fed by the read-only `GET /api/v1/tenant-usage` endpoint (renders nothing when there are no limits):

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `showUsageBar` | `boolean \| 'onDemand'` | `false` | `true` = always visible; `'onDemand'` = a small "Usage" toggle reveals it; `false` = hidden |
| `usageBarMetric` | `'tokens' \| 'cost'` | — | Restrict the bar to a single metric (default: all applicable rules) |
| `usageBarDisplay` | `UsageBarDisplay` | `{ showPercent: true, showAllRules: true }` | `{ showValues?, showPercent?, showAllRules? }` — absolute values, %, and whether to show every applicable rule or only the most restrictive |
| `customUsageBar` | `(data: UsageBarData) => ReactNode` | — | Render your own bar in the same slot; receives `{ rules, tierId, loading }`. Providing it is enough to show the bar |

The `UsageBar` and `LimitBanner` components are also exported standalone.

### Handling usage-limit blocks (429)

When a tenant/subtenant hits a configured limit, the message is blocked **before** the LLM call. `ChatDrawer` shows a built-in **limit banner** and disables the input while the limit is active (both sync `429 TENANT_LIMIT_EXCEEDED` and async `status: "limit_exceeded"` are handled transparently):

| Prop | Type | Description |
|------|------|-------------|
| `hideLimitBanner` | `boolean` | Suppress the built-in banner (e.g. to render your own) |
| `limitBannerRenderer` | `(limit: TenantLimitExceeded) => ReactNode` | Custom banner. The input stays disabled regardless while the limit is active |

`useDevicChat()` exposes the live block as `limitExceeded: TenantLimitExceeded | null` (`{ message, blockingRule, current, limit, resetsAt }`), so you can build your own UI. The client also offers read-only `getTenantUsage(tenantId, subtenantId?)` and `getTenantUsageHistory(tenantId, options?)`. Exported types: `TenantLimitExceeded`, `TenantUsage`, `TenantUsageRule`, `TenantUsageHistoryRow`, `UsageBarDisplay`, `UsageBarData`, `LimitBannerProps`.

## Client-Side Tools (Model Interface Protocol)

Enable the assistant to call functions in your application:

```tsx
import { ChatDrawer, ModelInterfaceTool } from '@devicai/ui';

// Define client-side tools
const tools: ModelInterfaceTool[] = [
  {
    toolName: 'get_user_location',
    schema: {
      type: 'function',
      function: {
        name: 'get_user_location',
        description: 'Get the current user geographic location',
        parameters: {
          type: 'object',
          properties: {},
        },
      },
    },
    callback: async () => {
      return new Promise((resolve, reject) => {
        navigator.geolocation.getCurrentPosition(
          (pos) => resolve({
            latitude: pos.coords.latitude,
            longitude: pos.coords.longitude,
          }),
          (err) => reject(new Error(err.message))
        );
      });
    },
  },
  {
    toolName: 'get_current_page',
    schema: {
      type: 'function',
      function: {
        name: 'get_current_page',
        description: 'Get the current page URL and title',
        parameters: {
          type: 'object',
          properties: {},
        },
      },
    },
    callback: async () => ({
      url: window.location.href,
      title: document.title,
      pathname: window.location.pathname,
    }),
  },
  {
    toolName: 'navigate_to_page',
    schema: {
      type: 'function',
      function: {
        name: 'navigate_to_page',
        description: 'Navigate the user to a specific page',
        parameters: {
          type: 'object',
          properties: {
            path: {
              type: 'string',
              description: 'The path to navigate to',
            },
          },
          required: ['path'],
        },
      },
    },
    callback: async ({ path }) => {
      window.location.href = path;
      return { success: true, navigatedTo: path };
    },
  },
];

function App() {
  return (
    <ChatDrawer
      assistantId="my-assistant"
      modelInterfaceTools={tools}
      onToolCall={(toolName, params) => {
        console.log(`Tool called: ${toolName}`, params);
      }}
    />
  );
}
```

### Interactive Response Widgets

Instead of a `callback`, a tool can provide a `responseWidget` that renders
a React component in the chat UI. The user interacts with the widget and
the widget calls `submit(response)` to produce the tool response that is
sent back to the model. Use this when the tool response depends on
user input (confirmation, selection, rating, custom form, etc.).

Two render modes are supported:

- `render: 'inline'` — the widget is rendered inside the message thread
  at the position of the tool call. The chat input is disabled while
  inline widgets are pending.
- `render: 'input'` — the widget replaces the chat input area until it
  is submitted or cancelled. Only one `input` widget is shown at a time;
  additional calls are queued.

```tsx
import {
  ChatDrawer,
  ModelInterfaceTool,
  ResponseWidgetProps,
} from '@devicai/ui';

// Inline confirmation widget
function ConfirmationWidget({ params, submit, cancel }: ResponseWidgetProps) {
  return (
    <div>
      <p>{params.action}</p>
      <button onClick={() => submit({ confirmed: true })}>Confirm</button>
      <button onClick={() => submit({ confirmed: false })}>Reject</button>
      <button onClick={() => cancel?.('Dismissed by user')}>Dismiss</button>
    </div>
  );
}

// Input-replacement rating widget
function RatingWidget({ params, submit, cancel }: ResponseWidgetProps) {
  const [value, setValue] = useState(0);
  return (
    <div>
      <span>Rate: {params.topic}</span>
      {[1, 2, 3, 4, 5].map((n) => (
        <button key={n} onClick={() => setValue(n)}>{n <= value ? '★' : '☆'}</button>
      ))}
      <button disabled={!value} onClick={() => submit({ rating: value })}>Submit</button>
      <button onClick={() => cancel?.('Skipped')}>Skip</button>
    </div>
  );
}

const tools: ModelInterfaceTool[] = [
  {
    toolName: 'ask_user_confirmation',
    schema: {
      type: 'function',
      function: {
        name: 'ask_user_confirmation',
        description: 'Ask the user to confirm a destructive action',
        parameters: {
          type: 'object',
          properties: {
            action: { type: 'string', description: 'Action to confirm' },
          },
          required: ['action'],
        },
      },
    },
    responseWidget: { render: 'inline', component: ConfirmationWidget },
  },
  {
    toolName: 'ask_user_rating',
    schema: {
      type: 'function',
      function: {
        name: 'ask_user_rating',
        description: 'Ask the user to rate a topic from 1 to 5',
        parameters: {
          type: 'object',
          properties: {
            topic: { type: 'string' },
          },
          required: ['topic'],
        },
      },
    },
    responseWidget: { render: 'input', component: RatingWidget },
  },
];
```

`ResponseWidgetProps` the component receives:

| Prop | Type | Description |
|------|------|-------------|
| `toolCall` | `ToolCall` | The full tool call from the model (id, name, arguments) |
| `params` | `any` | Parsed `toolCall.function.arguments` |
| `submit` | `(response: any) => void` | Send the tool response to the model. Response is passed as the tool call result |
| `cancel` | `(reason?: string) => void` | Cancel the tool call. Sends an error response so the model can continue |

Rules:

- A tool must define either `callback` or `responseWidget`, not both.
- While any inline widgets are pending, the chat input is disabled.
- While an `input` widget is pending, it replaces the default input area.
- `submit` and `cancel` are one-shot — after either is called, the widget
  is removed and its tool response is sent to the model. Polling resumes
  automatically once all pending widgets resolve.

Low-level access via `useDevicChat`:

```tsx
const {
  pendingWidgetCalls,
  submitWidgetResponse,
  cancelWidgetCall,
} = useDevicChat({ assistantId, modelInterfaceTools: tools });
```

## Require Tool Use to Finish

An assistant can be configured to end every turn by calling a tool instead of
replying with plain text. This solves assistants that run their tool calls and
then trail off without producing a final answer.

### Configuration (platform UI)

It is a property of the assistant, not of the drawer — there is nothing to pass
to `<ChatDrawer>`. In the Devic platform:

**Assistant → Context Management → "Require Tool Use to Finish"**

| Setting | Meaning |
| --- | --- |
| Toggle | Enables the mode for this assistant |
| Finish tool | Which tool ends the turn. Defaults to the built-in `finish_execution`; any of the assistant's own tools can be selected instead |

> ⚠️ **This mode forces the model to call a tool in order to finish.** The
> backend sends `tool_choice: 'required'`, so the model *cannot* end a turn with
> plain text — every turn must terminate in a tool call. Consequences worth
> weighing before enabling it:
>
> - **Every turn spends an extra tool call**, which adds tokens and one more
>   round trip of latency to each answer.
> - **Smaller / weaker models struggle with it.** If the model cannot reliably
>   satisfy `tool_choice: 'required'`, the turn keeps calling tools until the
>   chat message limit kicks in. Test with your target model before rolling out.
> - **It only applies to assistants**, never to agent threads.
> - If the configured finish tool ends up unavailable (excluded by the
>   `enabledTools` whitelist, or its tool group / server disabled), the backend
>   falls back to `finish_execution` so the model always has a way out.

### Behaviour with the default `finish_execution`

The tool takes a single `message` argument holding the final answer (Markdown
supported). **The model emits only the tool call, no text of its own.** The
backend intercepts it, copies `message` into the assistant message's
`content.message`, and emits the tool response itself. So the reply you render
was produced by the tool, not typed by the model — `contentSource:
'finish_tool'` on the message says exactly that, and is absent on replies the
model wrote itself.

The resulting turn looks like this:

```jsonc
{
  "role": "assistant",
  "content": { "message": "The capital of France is Paris." }, // lifted by the backend
  "contentSource": "finish_tool",
  "tool_calls": [{ "function": { "name": "finish_execution",
                                 "arguments": "{\"message\":\"The capital of France is Paris.\"}" } }]
}
// followed by a synthetic tool response ("Execution finished.") that closes the
// tool_call/tool_result pair. It is never rendered.
```

devic-ui renders it as a **normal assistant bubble** — no tool activity line for
`finish_execution`, since it is how the answer is delivered rather than an action
the assistant took (≥ 0.35.0; earlier versions rendered the reply a second time
below the bubble, as a tool activity line). Nothing to configure:

```tsx
// The assistant has requiredToolUse enabled — the drawer needs no extra props.
<ChatDrawer assistantId="my-assistant" apiKey={apiKey} />
```

Use `contentSource` if you want to label the bubble as tool-produced in your own
UI; a custom `assistantMessageRenderer` receives the full message and can read
it:

```tsx
<ChatDrawer
  assistantId="my-assistant"
  options={{
    assistantMessageRenderer: ({ message, content }) => (
      <>
        <Markdown>{content}</Markdown>
        {message.contentSource === 'finish_tool' && <Badge>via finish tool</Badge>}
      </>
    ),
  }}
/>
```

### Behaviour with your own finish tool

The selected tool runs normally and then ends the turn. Two differences from the
default that bite in practice:

- **Nothing is promoted to a bubble.** Only `finish_execution` has its `message`
  copied to `content.message`, so the visible reply is whatever text the model
  emitted alongside the tool call — often nothing. Render the call yourself with
  `toolRenderers` (see [Custom Tool Renderers](#custom-tool-renderers)) if it
  should show up in the thread.
- **The model never sees the tool's result.** The turn ends right after the tool
  runs, so its output cannot influence the answer. Use it for delivery/side
  effects, not for something the model must read back.

### Combining with client-side widgets

This mode and Model Interface widgets compose without any extra wiring: the model
calls your MIP tool (the turn pauses and the widget renders), you submit the
response, the turn resumes, and the model then calls the finish tool to close.
They never collide — `finish_execution` is a backend built-in and never enters
the client-side tool path.

## Custom Chat UI with Hooks

Build a completely custom chat interface:

```tsx
import { useDevicChat } from '@devicai/ui';

function CustomChat() {
  const {
    messages,
    isLoading,
    status,
    error,
    sendMessage,
    clearChat,
  } = useDevicChat({
    assistantId: 'my-assistant',
    onMessageReceived: (message) => {
      console.log('New message:', message);
    },
    onError: (error) => {
      console.error('Chat error:', error);
    },
  });

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    const message = formData.get('message') as string;
    if (message.trim()) {
      sendMessage(message);
      e.currentTarget.reset();
    }
  };

  return (
    <div className="custom-chat">
      <div className="messages">
        {messages.map((msg) => (
          <div key={msg.uid} className={`message ${msg.role}`}>
            <strong>{msg.role}:</strong>
            <p>{msg.content.message}</p>
          </div>
        ))}
        {isLoading && <div className="loading">Thinking...</div>}
        {error && <div className="error">{error.message}</div>}
      </div>

      <form onSubmit={handleSubmit}>
        <input
          name="message"
          placeholder="Type a message..."
          disabled={isLoading}
        />
        <button type="submit" disabled={isLoading}>
          Send
        </button>
      </form>

      <button onClick={clearChat}>Clear Chat</button>
    </div>
  );
}
```

## File Uploads

Enable file attachments in chat. Files are uploaded to the Devic API (`POST /api/v1/files/upload`) by default, which returns a download URL that is sent along with the message.

```tsx
<ChatDrawer
  assistantId="document-assistant"
  options={{
    enableFileUploads: true,
    allowedFileTypes: {
      images: true,
      documents: true,
      audio: false,
      video: false,
    },
    maxFileSize: 10 * 1024 * 1024, // 10MB
  }}
/>
```

### Custom File Upload Handler

Replace the default upload with your own implementation using `onFileUpload`. It receives the raw `File` objects and must return `ChatFile[]` with `downloadUrl` populated:

```tsx
import { ChatDrawer, ChatFile } from '@devicai/ui';

<ChatDrawer
  assistantId="document-assistant"
  options={{ enableFileUploads: true }}
  onFileUpload={async (files: File[]): Promise<ChatFile[]> => {
    // Upload to your own storage (S3, Firebase, etc.)
    const results = await Promise.all(
      files.map(async (file) => {
        const formData = new FormData();
        formData.append('file', file);
        const res = await fetch('/api/my-upload', { method: 'POST', body: formData });
        const { url } = await res.json();
        return {
          name: file.name,
          downloadUrl: url,
          fileType: file.type.startsWith('image/') ? 'image' : 'document',
        } as ChatFile;
      })
    );
    return results;
  }}
/>
```

## Speech-to-Text (Voice Input)

Add a microphone to the prompt box that records audio, transcribes it via the
Devic `/whisper` endpoint, and fills the input for review before sending.

```tsx
<ChatDrawer
  assistantId="my-assistant"
  options={{
    enableSpeechToText: true,
    speechLanguage: 'es', // optional ISO-639-1 hint
  }}
/>
```

By default the recording **auto-stops on silence** (`speechAutoStop`) and
transcribes itself. You can also enable a **hands-free loop** (`speechHandoff`,
≥ 0.20.0): the mic records, auto-sends after a short cancellable countdown
(`speechHandoffSendDelayMs`), and re-opens once the assistant replies — until a
silent turn or any interaction ends it.

You can also drive transcription from a `customPromptBox` (via the
`transcribeAudio` prop, which accepts a binary or a URL) or build a fully custom
recorder with the `useSpeechRecording` hook.

For the full guide — default UI flow, auto-stop, hands-free mode, custom prompt
box integration, the `useSpeechRecording` hook and direct client usage — see
[speech-to-text.md](speech-to-text.md).

## Display Modes

ChatDrawer supports two display modes via the `mode` prop:

### Drawer Mode (default)

Renders as an overlay panel with a floating trigger button. Can be toggled open/closed.

```tsx
<ChatDrawer
  mode="drawer"
  assistantId="my-assistant"
  options={{
    position: 'right',
    defaultOpen: false,
    zIndex: 1000,
  }}
/>
```

### Inline Mode

Renders embedded in the page layout, always visible, no trigger button or toggle behavior.

```tsx
<ChatDrawer
  mode="inline"
  assistantId="my-assistant"
  options={{
    width: 400,
    borderRadius: 12,
  }}
/>
```

## Resizable Drawer

Enable drag-to-resize with width constraints:

```tsx
<ChatDrawer
  assistantId="my-assistant"
  options={{
    resizable: true,
    width: 400,
    minWidth: 300,
    maxWidth: 800,
    position: 'right', // resize handle appears on the opposite edge
  }}
/>
```

## Custom Rendering

### Custom Loading Indicator

```tsx
<ChatDrawer
  assistantId="my-assistant"
  options={{
    loadingIndicator: <MySpinner />,
  }}
/>
```

### Markdown rendering

Both **assistant and user** message bubbles render their text through
`markdown-to-jsx` (≥ 0.30.0; before that, only assistant messages did).
This means user input is interpreted as markdown — `**bold**`, `_italic_`,
`# headings`, `- lists`, fenced code blocks, links and tables all render.
Links, inline code and blockquotes are styled to stay legible on the colored
user bubble, and single-line messages keep their original bubble height
(outer paragraph margins are collapsed).

If your users paste content with stray markdown characters (`*`, `_`, `#`,
`1.`) be aware it will be formatted. To opt out — or to swap in your own
renderer (a different markdown library, plain text, a rich component) — pass a
[custom bubble renderer](#custom-message-bubble-renderers) per role, or use a
`customPromptBox` for the input side.

Tables get a horizontally scrollable wrapper (`.markdown-table`); the markdown
overrides are applied identically for both roles.

### Custom message bubble renderers

Inject a component that **replaces the built-in markdown renderer** for the
text content of a bubble, per role (≥ 0.31.0):

| Option | Type | Description |
|--------|------|-------------|
| `userMessageRenderer` | `MessageBubbleRenderer` | Replaces markdown for **user** bubbles |
| `assistantMessageRenderer` | `MessageBubbleRenderer` | Replaces markdown for **assistant** bubbles |

```tsx
import { ChatDrawer, MessageBubbleRendererProps } from '@devicai/ui';

<ChatDrawer
  assistantId="my-assistant"
  options={{
    // Raw, unformatted user text (opt out of markdown for user messages)
    userMessageRenderer: ({ content }) => <span>{content}</span>,
    // Your own renderer for assistant messages
    assistantMessageRenderer: ({ content, message }) => (
      <MyRichMarkdown text={content} raw={message} />
    ),
  }}
/>
```

The renderer receives `MessageBubbleRendererProps`:

| Prop | Type | Description |
|------|------|-------------|
| `message` | `ChatMessage` | The full message being rendered |
| `content` | `string` | Text to render. For user messages the `Elemento referenciado: …` reference prefix is already stripped, so this is the user's actual text |
| `role` | `'user' \| 'assistant'` | Resolved role of the bubble |
| `references` | `string[]` | Reference labels parsed out of the message (user messages only) |

Notes:

- The renderer only replaces the bubble's **text content**. Reference chips
  (above the bubble), file attachments and the voice playback control are still
  rendered by the library around the returned node.
- It is only invoked when there is text to render; a message with only
  attachments behaves as before.
- Without a renderer, the default markdown rendering applies (see
  [Markdown rendering](#markdown-rendering)).

Exported types: `MessageBubbleRenderer`, `MessageBubbleRendererProps`.

### Reference chip node

The chip that represents an `AIElementWrapper` reference is exported as a
standalone node, `ReferenceChip`, so you can render the same chip in custom
prompt boxes or custom message bubbles. It is the exact component used
internally by `ChatInput` (active references, removable) and `ChatMessages`
(read-only chips above a sent user bubble).

```tsx
import { ReferenceChip } from '@devicai/ui';

// Removable chip in an input area
<ReferenceChip
  label="Acme Corp"
  variant="input"
  onRemove={() => removeReference(ref.id)}
/>

// Read-only chip above a message bubble
<ReferenceChip label="Acme Corp" variant="message" />
```

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `label` | `string` | *required* | Text shown in the chip (quoted automatically) |
| `variant` | `'input' \| 'message'` | `'input'` | `input` = prompt-box styling; `message` = inside a sent bubble |
| `onRemove` | `() => void` | — | When set, renders a × button (meaningful for `input`) |
| `icon` | `ReactNode` | corner-down-right arrow | Override the default reference glyph |
| `className` | `string` | — | Extra class appended to the chip root |

Exported types: `ReferenceChipProps`, `ReferenceChipVariant`.

### Custom Prompt Box

Replace the entire default input area with a custom React component. The component receives `sendMessage`, `stop`, `isLoading`, and `newConversation` props so it can fully drive the conversation.

```tsx
import { ChatDrawer, CustomPromptBoxProps } from '@devicai/ui';

function MyPromptBox({ sendMessage, stop, isLoading, newConversation }: CustomPromptBoxProps) {
  const [text, setText] = useState('');

  const handleSend = () => {
    if (!text.trim()) return;
    sendMessage(text.trim());
    setText('');
  };

  return (
    <div style={{ display: 'flex', gap: 8, padding: 8 }}>
      <button onClick={newConversation} title="New conversation">+</button>
      <input
        value={text}
        onChange={(e) => setText(e.target.value)}
        onKeyDown={(e) => e.key === 'Enter' && handleSend()}
        placeholder="Ask anything..."
        disabled={isLoading}
        style={{ flex: 1 }}
      />
      {isLoading ? (
        <button onClick={stop}>Stop</button>
      ) : (
        <button onClick={handleSend} disabled={!text.trim()}>Send</button>
      )}
    </div>
  );
}

<ChatDrawer
  assistantId="my-assistant"
  options={{
    customPromptBox: (props) => <MyPromptBox {...props} />,
  }}
/>
```

The `CustomPromptBoxProps` interface:

| Prop | Type | Description |
|------|------|-------------|
| `sendMessage` | `(message: string, files?: File[], meta?: { transcriptId?: string }) => void` | Send a message (optionally with file attachments). Pass `meta.transcriptId` to link a speech-to-text transcript |
| `transcribeAudio` | `(audio: Blob \| string, options?: { language?, messageUid?, chatUid? }) => Promise<WhisperTranscriptionResponse>` | Transcribe a binary (Blob/File) or a download URL via `/whisper`. See [speech-to-text.md](speech-to-text.md) |
| `stop` | `() => void` | Stop the current assistant processing |
| `isLoading` | `boolean` | Whether the assistant is currently processing / polling |
| `newConversation` | `() => void` | Clear the current conversation and start a new one |
| `references` | `AIReference[]` | Active references created by AIElementWrapper components |
| `removeReference` | `(id: string) => void` | Remove a single reference by id |
| `clearReferences` | `() => void` | Clear all references |

### Custom Send Button

The click handler is managed by an overlay, so the node doesn't need to handle click events.

```tsx
<ChatDrawer
  assistantId="my-assistant"
  options={{
    sendButtonContent: <MyCustomIcon />,
  }}
/>
```

### Custom Tool Renderers

Replace the default tool call summary with custom UI per tool name:

```tsx
<ChatDrawer
  assistantId="my-assistant"
  options={{
    toolRenderers: {
      search_products: (input, output) => (
        <ProductGrid products={output.results} query={input.query} />
      ),
    },
    toolIcons: {
      search_products: <SearchIcon />,
    },
  }}
/>
```

### Tool Groups (Grouped Tool Call Rendering)

Group consecutive tool calls into a single unified renderer. Useful for rendering sequences of related tool calls (e.g., terminal commands + file reads) as a cohesive UI block.

```tsx
import { ChatDrawer, ToolGroupCall, ToolGroupConfig } from '@devicai/ui';

const toolGroups: ToolGroupConfig[] = [
  {
    tools: ['run_terminal_command', 'read_sandbox_file'],
    renderer: (calls: ToolGroupCall[]) => (
      <div className="terminal-trace">
        {calls.map((call) => (
          <div key={call.toolCallId} className="trace-entry">
            <code>{call.name}</code>
            <pre>{JSON.stringify(call.input, null, 2)}</pre>
            {call.output && <pre className="output">{JSON.stringify(call.output)}</pre>}
          </div>
        ))}
      </div>
    ),
  },
];

<ChatDrawer
  assistantId="my-assistant"
  options={{
    toolGroups,
    // toolRenderers still works for non-grouped tools
    toolRenderers: {
      search_web: (input, output) => <SearchResult query={input.query} results={output} />,
    },
  }}
/>
```

Tool groups work in all three components: `ChatDrawer`, `AICommandBar`, and `AIGenerationButton`. When consecutive tool calls match the same group's `tools` array, they are accumulated and passed as a single array to the group's `renderer`. Non-matching tools render individually as before (using `toolRenderers` or default rendering).

The `segmentToolCalls` utility is also exported for custom implementations:

```tsx
import { segmentToolCalls, ToolGroupCall, ToolGroupConfig } from '@devicai/ui';

const segments = segmentToolCalls(calls, toolGroups);
// Returns: Array<{ type: 'group', config, calls } | { type: 'single', call, index }>
```

## Theming

### Using Props

All color and typography properties can be set via `options`:

```tsx
<ChatDrawer
  assistantId="my-assistant"
  options={{
    color: '#6366f1',                    // Primary color
    backgroundColor: '#ffffff',          // Drawer background
    textColor: '#1e293b',               // Text color
    secondaryBackgroundColor: '#f8fafc', // Input/selector background
    borderColor: '#e2e8f0',             // Border color
    userBubbleColor: '#6366f1',         // User message bubble
    userBubbleTextColor: '#ffffff',     // User message text
    assistantBubbleColor: '#f1f5f9',    // Assistant message bubble
    assistantBubbleTextColor: '#1e293b',// Assistant message text
    sendButtonColor: '#6366f1',         // Send button background
    fontFamily: '"Inter", sans-serif',  // Font override
  }}
/>
```

### Using CSS Variables

Override the default theme by setting CSS variables:

```css
/* your-styles.css */
:root {
  --devic-primary: #6366f1;        /* Primary color */
  --devic-primary-hover: #4f46e5;  /* Primary hover */
  --devic-primary-light: #eef2ff;  /* Light primary background */
  --devic-bg: #ffffff;             /* Background */
  --devic-bg-secondary: #f8fafc;   /* Secondary background */
  --devic-text: #1e293b;           /* Text color */
  --devic-text-secondary: #64748b; /* Secondary text */
  --devic-text-muted: #94a3b8;     /* Muted text */
  --devic-border: #e2e8f0;         /* Border color */
  --devic-shadow: 0 10px 40px rgba(0, 0, 0, 0.1);
  --devic-radius: 12px;            /* Border radius */
  --devic-radius-sm: 6px;
  --devic-radius-lg: 20px;
}
```

### Using the color Option

Quick color customization:

```tsx
<ChatDrawer
  assistantId="my-assistant"
  options={{
    color: '#6366f1', // Sets primary color
  }}
/>
```

## Controlled Mode

Control the drawer state externally:

```tsx
function App() {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <>
      <button onClick={() => setIsOpen(true)}>
        Open Chat
      </button>

      <ChatDrawer
        assistantId="my-assistant"
        isOpen={isOpen}
        onOpen={() => setIsOpen(true)}
        onClose={() => setIsOpen(false)}
      />
    </>
  );
}
```

## Continuing Existing Conversations

Load and continue a previous chat:

```tsx
function App() {
  // Get chatUid from URL, localStorage, or your backend
  const existingChatUid = 'previous-chat-uid';

  return (
    <ChatDrawer
      assistantId="my-assistant"
      chatUid={existingChatUid}
      onChatCreated={(newChatUid) => {
        // Save the new chat UID for future reference
        localStorage.setItem('lastChatUid', newChatUid);
      }}
    />
  );
}
```

## Event Callbacks

Handle various chat events:

```tsx
<ChatDrawer
  assistantId="my-assistant"
  onMessageSent={(message) => {
    // Track user messages
    analytics.track('chat_message_sent', {
      messageLength: message.content.message?.length,
    });
  }}
  onMessageReceived={(message) => {
    // Track assistant responses
    analytics.track('chat_message_received', {
      hasToolCalls: !!message.tool_calls?.length,
    });
  }}
  onToolCall={(toolName, params) => {
    // Track tool usage
    analytics.track('chat_tool_called', { toolName });
  }}
  onError={(error) => {
    // Report errors
    errorReporting.capture(error);
  }}
  onChatCreated={(chatUid) => {
    // Store chat reference
    saveChatReference(chatUid);
  }}
  onOpen={() => {
    // Track drawer open
    analytics.track('chat_opened');
  }}
  onClose={() => {
    // Track drawer close
    analytics.track('chat_closed');
  }}
/>
```

## API Client Direct Usage

For advanced use cases, use the API client directly:

```tsx
import { DevicApiClient } from '@devicai/ui';

const client = new DevicApiClient({
  apiKey: 'your-api-key',
  baseUrl: 'https://api.devic.ai',
});

// List available assistants
const assistants = await client.getAssistants();

// Send a message (async mode)
const { chatUid } = await client.sendMessageAsync('assistant-id', {
  message: 'Hello!',
  tenantId: 'tenant-123',
  metadata: { source: 'web-app' },
});

// Poll for response
const checkResponse = async () => {
  const result = await client.getRealtimeHistory('assistant-id', chatUid);

  if (result.status === 'completed') {
    return result.chatHistory;
  } else if (result.status === 'error') {
    throw new Error('Processing failed');
  } else if (result.status === 'handed_off') {
    // Assistant delegated to a subagent
    // result.handedOffSubThreadId contains the subthread ID
    console.log('Handed off to subthread:', result.handedOffSubThreadId);
    // Continue polling until the handoff completes and processing resumes
  }

  // Continue polling
  await new Promise(r => setTimeout(r, 1000));
  return checkResponse();
};

const messages = await checkResponse();
```

## Server-Side Rendering (SSR)

The library is SSR-compatible. Ensure you only render the ChatDrawer on the client:

```tsx
// Next.js example
import dynamic from 'next/dynamic';

const ChatDrawer = dynamic(
  () => import('@devicai/ui').then(mod => mod.ChatDrawer),
  { ssr: false }
);

function Page() {
  return (
    <div>
      <h1>My Page</h1>
      <ChatDrawer assistantId="my-assistant" />
    </div>
  );
}
```

## TypeScript Support

All types are exported for TypeScript users:

```tsx
import type {
  // Chat types
  ChatMessage,
  ChatDrawerProps,
  ChatDrawerOptions,
  ChatDrawerHandle,
  SuggestedMessage,
  MessageBubbleRenderer,
  MessageBubbleRendererProps,

  // AICommandBar types
  AICommandBarProps,
  AICommandBarOptions,
  AICommandBarHandle,
  AICommandBarCommand,
  CommandBarResult,
  ToolCallSummary,

  // AIGenerationButton types
  AIGenerationButtonProps,
  AIGenerationButtonOptions,
  AIGenerationButtonHandle,
  AIGenerationButtonMode,
  GenerationResult,

  // Tool types
  ModelInterfaceTool,
  ModelInterfaceToolSchema,
  ResponseWidgetProps,
  ResponseWidgetConfig,
  PendingWidgetCall,
  ToolGroupCall,
  ToolGroupConfig,

  // Hook types
  UseDevicChatOptions,
  UseDevicChatResult,
  UseSpeechRecordingOptions,
  UseSpeechRecordingResult,
  SpeechRecordingStatus,

  // API types
  RealtimeChatHistory,  // Includes status (with 'handed_off') and handedOffSubThreadId
  RealtimeStatus,       // 'processing' | 'completed' | 'error' | 'waiting_for_tool_response' | 'handed_off'
  AssistantSpecialization, // Includes tenantIntegrations { enabled, count }
  WhisperTranscriptionResponse,
  DevicApiClientConfig,
  TenantSessionToken,   // { token, expiresAt?, expiresIn? }

  // Assistant info cache (0.42.0)
  UseAssistantInfoOptions,
  AssistantInfoState,

  // Connected apps
  IntegrationsModalProps,
  IntegrationsLauncherProps,
  IntegrationsHintProps,
  IntegrationLogoProps,
  IntegrationsState,
  IntegrationsScope,
  UseIntegrationsOptions,

  // Feedback types
  FeedbackSubmission,
  FeedbackEntry,
  FeedbackTheme,

  // Agent/Handoff types
  ThreadStateTagProps,
  StateConfig,
  HandoffSubagentWidgetProps,
  AgentThreadDto,
  AgentTaskDto,
  AgentDto,
  HandOffToolResponse,

  // Reference chip types
  ReferenceChipProps,
  ReferenceChipVariant,
} from '@devicai/ui';

// Import the AgentThreadState enum (value export)
import { AgentThreadState, segmentToolCalls, useSpeechRecording } from '@devicai/ui';

// Sessions, assistant info and connected apps (value exports)
import {
  createSharedSession,
  useAssistantInfo,
  forgetAssistant,
  useIntegrations,
  IntegrationsModal,
  IntegrationsLauncher,
  IntegrationsHint,
  IntegrationLogo,
} from '@devicai/ui';

// Use types in your code
const chatOptions: ChatDrawerOptions = {
  position: 'right',
  width: 400,
  welcomeMessage: 'Hello!',
};

const commandBarOptions: AICommandBarOptions = {
  shortcut: 'cmd+k',
  placeholder: 'Ask AI...',
};

const generationOptions: AIGenerationButtonOptions = {
  mode: 'modal',
  modalTitle: 'Generate with AI',
};

const handleMessage = (message: ChatMessage) => {
  console.log(message.content.message);
};

const handleCommandResult = (result: CommandBarResult) => {
  console.log('Chat UID:', result.chatUid);
  console.log('Tool calls:', result.toolCalls.length);
  console.log('Response:', result.message.content);
};

const handleGenerationResult = (result: GenerationResult) => {
  console.log('Generated content:', result.message.content.message);
  console.log('Tool calls executed:', result.toolCalls);
};
```

## ChatDrawer Props Reference

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `mode` | `'drawer' \| 'inline'` | `'drawer'` | Display mode: overlay drawer or embedded inline |
| `assistantId` | `string` | *required* | Assistant identifier |
| `chatUid` | `string` | — | Existing chat UID to continue conversation |
| `options` | `ChatDrawerOptions` | — | Display and behavior options (see below) |
| `enabledTools` | `string[]` | — | Tools enabled from assistant's configured tool groups |
| `modelInterfaceTools` | `ModelInterfaceTool[]` | — | Client-side tools for model interface protocol |
| `tenantId` | `string` | — | Tenant ID (overrides provider) |
| `tenantMetadata` | `TenantMetadata` | — | Tenant metadata `{ name, email, imageUrl, … }` (overrides provider) |
| `subtenantId` | `string` | — | Subtenant (end-user) ID (overrides provider). Falls back to `subtenantMetadata.id` / `tenantMetadata.userId` |
| `subtenantMetadata` | `SubtenantMetadata` | — | Subtenant metadata `{ id, name, email, imageUrl }` (overrides provider) |
| `tags` | `string[]` | — | Conversation tags (top-level, distinct from metadata). Merged/deduped with the provider's `tags` and per-message tags. See [Conversation Tags](#conversation-tags) |
| `showUsageBar` | `boolean \| 'onDemand'` | `false` | Usage bar above the input (`true` / `'onDemand'` toggle). Needs `tenantId` + configured limits |
| `usageBarMetric` | `'tokens' \| 'cost'` | — | Restrict the usage bar to one metric |
| `usageBarDisplay` | `UsageBarDisplay` | `{ showPercent, showAllRules }` | `{ showValues?, showPercent?, showAllRules? }` |
| `customUsageBar` | `(data: UsageBarData) => ReactNode` | — | Render a custom usage bar (receives `{ rules, tierId, loading }`) |
| `hideLimitBanner` | `boolean` | `false` | Suppress the built-in usage-limit banner |
| `limitBannerRenderer` | `(limit: TenantLimitExceeded) => ReactNode` | — | Custom usage-limit banner (input stays disabled while active) |
| `apiKey` | `string` | — | API key (overrides provider) |
| `baseUrl` | `string` | — | Base URL (overrides provider) |
| `isOpen` | `boolean` | — | Controlled open state (drawer mode only) |
| `className` | `string` | — | Additional CSS class |
| `onMessageSent` | `(message) => void` | — | Fires when user sends a message |
| `onMessageReceived` | `(message) => void` | — | Fires when assistant responds |
| `onToolCall` | `(toolName, params) => void` | — | Fires when a tool is called |
| `onError` | `(error) => void` | — | Fires on error |
| `onChatCreated` | `(chatUid) => void` | — | Fires when a new chat is created |
| `onOpen` | `() => void` | — | Fires when drawer opens |
| `onClose` | `() => void` | — | Fires when drawer closes |
| `onFileUpload` | `(files: File[]) => Promise<ChatFile[]>` | — | Custom file upload handler. Replaces default Devic API upload. Must return ChatFile[] with downloadUrl |
| `onConversationChange` | `(chatUid) => void` | — | Fires when active conversation changes |

## ChatDrawerOptions Reference

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `position` | `'left' \| 'right'` | `'right'` | Drawer position |
| `width` | `number \| string` | `'100%'` | Drawer width (px number or CSS string) |
| `defaultOpen` | `boolean` | `false` | Whether drawer starts open |
| `resizable` | `boolean` | `false` | Enable drag-to-resize handle |
| `minWidth` | `number` | `300` | Minimum width when resizable (px) |
| `maxWidth` | `number` | `800` | Maximum width when resizable (px) |
| `zIndex` | `number` | `1000` | Z-index for the drawer |
| `borderRadius` | `number \| string` | `0` | Border radius for the container |
| `style` | `CSSProperties` | — | Additional inline styles |
| `title` | `string \| ReactNode` | `'Chat'` | Header title |
| `showAvatar` | `boolean` | `false` | Show assistant image next to title |
| `welcomeMessage` | `string` | — | Welcome message shown at start |
| `suggestedMessages` | `(string \| SuggestedMessage)[]` | — | Quick action suggestions. Accepts plain strings or objects with `content` (ReactNode) and `message` (string to send on click) |
| `inputPlaceholder` | `string` | `'Type a message...'` | Input placeholder text |
| `showToolTimeline` | `boolean` | `true` | Show tool execution timeline |
| `enableFileUploads` | `boolean` | `false` | Enable file attachments |
| `allowedFileTypes` | `AllowedFileTypes` | — | Filter by file type (images, documents, audio, video) |
| `maxFileSize` | `number` | `10485760` | Max file size in bytes (10MB) |
| `enableSpeechToText` | `boolean` | `false` | Show a microphone in the prompt box for voice input via `/whisper`. See [speech-to-text.md](speech-to-text.md) |
| `speechLanguage` | `string` | — | ISO-639-1 language hint for speech-to-text (e.g. `'es'`, `'en'`) |
| `speechAutoStop` | `boolean` | `true` | Auto-confirm the recording after a short silence (once speech is detected) |
| `speechAutoStopCountdownMs` | `number` | `1000` | Duration of the auto-stop circular countdown |
| `speechHandoff` | `boolean` | `false` | Hands-free conversation loop (mic → auto-send → re-listen). ≥ 0.20.0. See [speech-to-text.md](speech-to-text.md#hands-free-handoff-mode) |
| `speechHandoffSendDelayMs` | `number` | `1000` | Hands-free: delay from transcription ready to auto-send. ≥ 0.21.0 |
| `color` | `string` | `'#1890ff'` | Primary theme color |
| `backgroundColor` | `string` | — | Drawer background color |
| `textColor` | `string` | — | Text color |
| `secondaryBackgroundColor` | `string` | — | Input/selector background color |
| `borderColor` | `string` | — | Border color |
| `userBubbleColor` | `string` | — | User message bubble background |
| `userBubbleTextColor` | `string` | — | User message bubble text |
| `assistantBubbleColor` | `string` | — | Assistant message bubble background |
| `assistantBubbleTextColor` | `string` | — | Assistant message bubble text |
| `sendButtonColor` | `string` | — | Send button background color |
| `fontFamily` | `string` | — | Font family override |
| `loadingIndicator` | `ReactNode` | — | Custom loading spinner |
| `sendButtonContent` | `ReactNode` | — | Custom send button content |
| `toolRenderers` | `Record<string, (input, output) => ReactNode>` | — | Custom tool call renderers by tool name |
| `toolIcons` | `Record<string, ReactNode>` | — | Custom tool call icons by tool name |
| `showFeedback` | `boolean` | `true` | Show thumbs up/down feedback buttons on assistant messages |
| `handoffWidgetRenderer` | `(props: { thread, agent, elapsedSeconds, isTerminal }) => ReactNode` | — | Custom renderer for the HandoffSubagentWidget (replaces default UI) |
| `toolGroups` | `ToolGroupConfig[]` | — | Group consecutive tool calls under a single renderer |
| `customPromptBox` | `(props: CustomPromptBoxProps) => ReactNode` | — | Replace the default input area with a custom component. Receives `sendMessage`, `transcribeAudio`, `stop`, `isLoading`, `newConversation` and reference helpers |
| `userMessageRenderer` | `MessageBubbleRenderer` | — | Replace the built-in markdown renderer for **user** message bubbles. Receives `{ message, content, role, references }` |
| `assistantMessageRenderer` | `MessageBubbleRenderer` | — | Replace the built-in markdown renderer for **assistant** message bubbles. Receives `{ message, content, role, references }` |
| `showIntegrationsButton` | `boolean` | `true` | Allow the connected-apps control in the header. Shows only when the assistant offers apps. See [connected-apps.md](connected-apps.md) |
| `integrationsLabel` | `string` | `'Connected apps'` | Tooltip / accessible name of that control |
| `maxIntegrationLogos` | `number` | `6` | App logos before the rest are counted in a `+N` box |
| `showIntegrationsHint` | `boolean` | `true` | Dismissible strip above the composer naming the connected apps. Ignored when the button is off |
| `integrationsHintLabel` | `string` | — | Text on that strip. Unset: "Connect your apps" / "Explore connected apps" |

## Message Feedback

Both ChatDrawer and AICommandBar support message feedback (thumbs up/down with optional comments). Feedback is submitted to the Devic API and associated with the chat.

### ChatDrawer Feedback

Feedback buttons appear on assistant messages by default. Users can click thumbs up/down and optionally add a comment via a modal.

```tsx
<ChatDrawer
  assistantId="my-assistant"
  options={{
    showFeedback: true, // default: true
  }}
/>
```

### AICommandBar Feedback

When `showResultCard` is enabled, feedback buttons appear below the response. The feedback UI automatically adapts to the command bar's theme.

```tsx
<AICommandBar
  assistantId="my-assistant"
  options={{
    showResultCard: true,
    // Feedback inherits theme from these options:
    backgroundColor: '#1f2937',
    textColor: '#f9fafb',
    borderColor: '#374151',
  }}
/>
```

### Feedback Theming

The feedback modal and action buttons automatically inherit theme colors from the parent component. For custom implementations, you can pass a `FeedbackTheme` object:

```tsx
interface FeedbackTheme {
  backgroundColor?: string;      // Modal background
  textColor?: string;            // Primary text
  textMutedColor?: string;       // Muted/secondary text
  secondaryBackgroundColor?: string; // Button backgrounds, hover states
  borderColor?: string;          // Modal borders
  primaryColor?: string;         // Primary action color
  primaryHoverColor?: string;    // Primary button hover
}
```

### Feedback API

Feedback is automatically submitted to the Devic API using these endpoints:

- `POST /api/v1/assistants/:identifier/chats/:chatUid/feedback` - Submit feedback
- `GET /api/v1/assistants/:identifier/chats/:chatUid/feedback` - Get feedback entries

You can also use the API client directly:

```tsx
import { DevicApiClient, FeedbackSubmission } from '@devicai/ui';

const client = new DevicApiClient({ apiKey: 'your-api-key' });

// Submit feedback
const feedback: FeedbackSubmission = {
  messageId: 'message-uid',
  feedback: true,  // true = positive, false = negative
  feedbackComment: 'Very helpful response!',
  feedbackData: { category: 'accuracy' },
};

await client.submitChatFeedback('assistant-id', 'chat-uid', feedback);

// Get all feedback for a chat
const entries = await client.getChatFeedback('assistant-id', 'chat-uid');
```

### Feedback Types

```tsx
import type {
  FeedbackSubmission,
  FeedbackEntry,
  FeedbackTheme,
} from '@devicai/ui';

// Submission payload
interface FeedbackSubmission {
  messageId: string;
  feedback?: boolean;              // true = positive, false = negative
  feedbackComment?: string;        // Optional comment
  feedbackData?: Record<string, any>; // Custom metadata
}

// Response from API
interface FeedbackEntry {
  _id: string;
  requestId: string;
  chatUID?: string;
  feedback?: boolean;
  feedbackComment?: string;
  feedbackData?: Record<string, any>;
  creationTimestamp: string;
  lastEditTimestamp?: string;
}
```

## AICommandBar Component

A floating command bar (similar to Spotlight/Command Palette) for quick AI interactions. It provides a minimal input interface that processes messages, shows tool execution progress, and displays results in a compact card.

### Basic Usage

```tsx
import { AICommandBar } from '@devicai/ui';

function App() {
  return (
    <AICommandBar
      assistantId="my-assistant"
      options={{
        placeholder: 'Ask AI...',
        shortcut: 'cmd+k',
      }}
      onResponse={({ message, toolCalls }) => {
        console.log('Response:', message.content);
      }}
    />
  );
}
```

### Fixed Position with Keyboard Shortcut

```tsx
<AICommandBar
  assistantId="support-assistant"
  options={{
    position: 'fixed',
    fixedPlacement: { bottom: 20, right: 20 },
    shortcut: 'cmd+j',
    placeholder: 'Ask AI about your data...',
    showShortcutHint: true,
  }}
/>
```

### Integration with ChatDrawer

Hand off conversations to the full ChatDrawer after getting a quick answer:

```tsx
import { useRef } from 'react';
import { AICommandBar, ChatDrawer, ChatDrawerHandle } from '@devicai/ui';

function App() {
  const drawerRef = useRef<ChatDrawerHandle>(null);

  return (
    <>
      <AICommandBar
        assistantId="my-assistant"
        onExecute="openDrawer"
        chatDrawerRef={drawerRef}
        options={{
          shortcut: 'cmd+k',
          showResultCard: false, // Don't show result since drawer opens
        }}
      />

      <ChatDrawer
        ref={drawerRef}
        assistantId="my-assistant"
      />
    </>
  );
}
```

### Command History

Command history is enabled by default. Users can:
- Press **Arrow Up/Down** to navigate through previous prompts
- Use the `/history` command to see the history list
- Click on a history item to reuse it

```tsx
<AICommandBar
  assistantId="my-assistant"
  options={{
    enableHistory: true,           // default: true
    maxHistoryItems: 50,           // default: 50
    historyStorageKey: 'my-app-command-history', // localStorage key
    showHistoryCommand: true,      // adds /history command
  }}
/>
```

### Commands System

Define slash commands that trigger predefined messages:

```tsx
<AICommandBar
  assistantId="my-assistant"
  options={{
    commands: [
      {
        keyword: 'summarize',
        description: 'Summarize the current page',
        message: 'Please summarize the content of this page.',
        icon: <SummarizeIcon />,
      },
      {
        keyword: 'translate',
        description: 'Translate selected text',
        message: 'Translate the following text to Spanish: ',
      },
      {
        keyword: 'explain',
        description: 'Explain like I\'m five',
        message: 'Explain this concept in simple terms: ',
      },
    ],
  }}
/>
```

When the user types `/`, a dropdown shows available commands. Arrow keys navigate, Enter selects, Tab autocompletes.

### Custom Tool Rendering

Display tool calls with custom icons and renderers:

```tsx
<AICommandBar
  assistantId="my-assistant"
  options={{
    toolIcons: {
      search_database: <DatabaseIcon />,
      fetch_weather: <WeatherIcon />,
    },
    toolRenderers: {
      search_database: (input, output) => (
        <div className="custom-result">
          Found {output.count} results for "{input.query}"
        </div>
      ),
    },
  }}
/>
```

### Theming

```tsx
<AICommandBar
  assistantId="my-assistant"
  options={{
    color: '#6366f1',              // Primary color (spinner, badges)
    backgroundColor: '#ffffff',    // Bar background
    textColor: '#1f2937',          // Text color
    borderColor: '#e5e7eb',        // Border color
    borderRadius: 12,              // Border radius (px or string)
    fontFamily: 'Inter, sans-serif',
    fontSize: 14,
    padding: '12px 16px',
    boxShadow: '0 4px 20px rgba(0, 0, 0, 0.1)',
    animationDuration: 200,        // ms
  }}
/>
```

### Controlled Visibility

```tsx
function App() {
  const [isVisible, setIsVisible] = useState(false);
  const commandBarRef = useRef<AICommandBarHandle>(null);

  return (
    <>
      <button onClick={() => commandBarRef.current?.toggle()}>
        Toggle Command Bar
      </button>

      <AICommandBar
        ref={commandBarRef}
        assistantId="my-assistant"
        isVisible={isVisible}
        onVisibilityChange={setIsVisible}
        onOpen={() => console.log('Opened')}
        onClose={() => console.log('Closed')}
      />
    </>
  );
}
```

### Using the Hook Directly

For custom UI implementations:

```tsx
import { useAICommandBar, formatShortcut } from '@devicai/ui';

function CustomCommandBar() {
  const {
    isVisible,
    open,
    close,
    toggle,
    inputValue,
    setInputValue,
    inputRef,
    focus,
    isProcessing,
    currentToolSummary,
    toolCalls,
    result,
    error,
    history,
    showingHistory,
    showingCommands,
    filteredCommands,
    submit,
    reset,
    handleKeyDown,
  } = useAICommandBar({
    assistantId: 'my-assistant',
    options: { shortcut: 'cmd+k' },
    onResponse: (result) => console.log(result),
  });

  if (!isVisible) return null;

  return (
    <div className="my-command-bar">
      {isProcessing ? (
        <span>{currentToolSummary || 'Processing...'}</span>
      ) : (
        <input
          ref={inputRef}
          value={inputValue}
          onChange={(e) => setInputValue(e.target.value)}
          onKeyDown={handleKeyDown}
          placeholder="Ask AI..."
        />
      )}
    </div>
  );
}
```

## AICommandBar Props Reference

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `assistantId` | `string` | *required* | Assistant identifier |
| `apiKey` | `string` | — | API key (overrides provider) |
| `baseUrl` | `string` | — | Base URL (overrides provider) |
| `tenantId` | `string` | — | Tenant ID (overrides provider) |
| `tenantMetadata` | `TenantMetadata` | — | Tenant metadata |
| `tags` | `string[]` | — | Conversation tags (merged/deduped with the provider's). See [Conversation Tags](#conversation-tags) |
| `options` | `AICommandBarOptions` | — | Display and behavior options |
| `isVisible` | `boolean` | — | Controlled visibility state |
| `onVisibilityChange` | `(visible: boolean) => void` | — | Fires when visibility changes |
| `onExecute` | `'openDrawer' \| 'callback'` | `'callback'` | What to do on completion |
| `chatDrawerRef` | `RefObject<ChatDrawerHandle>` | — | Ref to ChatDrawer (for openDrawer mode) |
| `onResponse` | `(result: CommandBarResult) => void` | — | Fires on completion (callback mode) |
| `modelInterfaceTools` | `ModelInterfaceTool[]` | — | Client-side tools |
| `onSubmit` | `(message: string) => void` | — | Fires when user submits |
| `onToolCall` | `(toolName, params) => void` | — | Fires when a tool is called |
| `onError` | `(error: Error) => void` | — | Fires on error |
| `onOpen` | `() => void` | — | Fires when bar opens |
| `onClose` | `() => void` | — | Fires when bar closes |
| `className` | `string` | — | Additional CSS class |

## AICommandBarOptions Reference

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `position` | `'inline' \| 'fixed'` | `'inline'` | Positioning mode |
| `fixedPlacement` | `{ top?, right?, bottom?, left? }` | — | Position offsets for fixed mode |
| `shortcut` | `string` | — | Keyboard shortcut (e.g., `'cmd+k'`, `'ctrl+j'`) |
| `showShortcutHint` | `boolean` | `true` | Show shortcut badge in bar |
| `placeholder` | `string` | `'Ask AI...'` | Input placeholder |
| `icon` | `ReactNode` | Sparkles icon | Custom icon for idle state |
| `width` | `number \| string` | `400` | Bar width |
| `maxWidth` | `number \| string` | `'100%'` | Maximum width |
| `zIndex` | `number` | `9999` | Z-index |
| `showResultCard` | `boolean` | `true` | Show result card on completion |
| `resultCardMaxHeight` | `number \| string` | `300` | Max height for result card |
| `processingMessage` | `string` | `'Processing...'` | Fallback message during processing |
| `color` | `string` | `'#3b82f6'` | Primary color |
| `backgroundColor` | `string` | `'#ffffff'` | Background color |
| `textColor` | `string` | `'#1f2937'` | Text color |
| `borderColor` | `string` | `'#e5e7eb'` | Border color |
| `borderRadius` | `number \| string` | `12` | Border radius |
| `fontFamily` | `string` | System fonts | Font family |
| `fontSize` | `number \| string` | `14` | Font size |
| `padding` | `number \| string` | `'12px 16px'` | Bar padding |
| `boxShadow` | `string` | Light shadow | Box shadow |
| `animationDuration` | `number` | `200` | Animation duration (ms) |
| `toolRenderers` | `Record<string, (input, output) => ReactNode>` | — | Custom tool renderers |
| `toolIcons` | `Record<string, ReactNode>` | — | Custom tool icons |
| `enableHistory` | `boolean` | `true` | Enable command history |
| `maxHistoryItems` | `number` | `50` | Max history items to store |
| `historyStorageKey` | `string` | `'devic-command-bar-history'` | localStorage key |
| `commands` | `AICommandBarCommand[]` | — | Slash commands |
| `showHistoryCommand` | `boolean` | `true` | Add built-in /history command |
| `toolGroups` | `ToolGroupConfig[]` | — | Group consecutive tool calls under a single renderer |

## AICommandBarHandle Reference

Methods exposed via ref:

| Method | Description |
|--------|-------------|
| `open()` | Open the command bar |
| `close()` | Close the command bar |
| `toggle()` | Toggle visibility |
| `focus()` | Focus the input |
| `submit(message?: string)` | Submit a message |
| `reset()` | Reset state (clear input, result, errors) |

## AIGenerationButton Component

A button component for triggering AI generation with three configurable interaction modes. Useful for "Generate with AI" buttons in forms, editors, and other UI contexts.

### Basic Usage

```tsx
import { AIGenerationButton } from '@devicai/ui';

function App() {
  return (
    <AIGenerationButton
      assistantId="my-assistant"
      options={{
        mode: 'modal',
        modalTitle: 'Generate Content',
        placeholder: 'Describe what you want to generate...',
      }}
      onResponse={({ message }) => {
        console.log('Generated:', message.content.message);
      }}
    />
  );
}
```

### Interaction Modes

#### Direct Mode

Sends a predefined prompt immediately when clicked. Best for specific, predetermined actions.

```tsx
<AIGenerationButton
  assistantId="my-assistant"
  options={{
    mode: 'direct',
    prompt: 'Generate a product description based on the form data',
    label: 'Auto-Generate Description',
    loadingLabel: 'Generating...',
  }}
  onBeforeSend={(prompt) => {
    // Optionally modify the prompt before sending
    return `${prompt}\n\nProduct: ${productName}`;
  }}
  onResponse={({ message }) => setDescription(message.content.message)}
/>
```

#### Modal Mode (Default)

Opens a modal dialog for the user to enter a custom prompt.

```tsx
<AIGenerationButton
  assistantId="my-assistant"
  options={{
    mode: 'modal',
    modalTitle: 'Generate with AI',
    modalDescription: 'Describe what you want and the AI will generate it for you.',
    placeholder: 'E.g., Create a function that validates email addresses...',
    confirmText: 'Generate',
    cancelText: 'Cancel',
  }}
  onResponse={({ message }) => {
    // Handle the generated content
    setCode(message.content.message);
  }}
/>
```

#### Tooltip Mode

Shows a compact inline input next to the button. Good for quick prompts without modal interruption.

```tsx
<AIGenerationButton
  assistantId="my-assistant"
  options={{
    mode: 'tooltip',
    tooltipPlacement: 'bottom', // 'top' | 'bottom' | 'left' | 'right'
    tooltipWidth: 350,
    placeholder: 'What should I generate?',
  }}
  onResponse={handleGeneration}
/>
```

### Button Styling

```tsx
<AIGenerationButton
  assistantId="my-assistant"
  options={{
    // Button variant
    variant: 'primary', // 'primary' | 'secondary' | 'outline' | 'ghost'

    // Button size
    size: 'medium', // 'small' | 'medium' | 'large'

    // Label and icon
    label: 'Generate with AI',
    hideLabel: false, // Set true for icon-only button
    icon: <CustomSparkleIcon />, // Custom icon
    hideIcon: false,

    // Loading state
    loadingLabel: 'Generating...',
  }}
/>
```

### Theming

```tsx
<AIGenerationButton
  assistantId="my-assistant"
  options={{
    color: '#6366f1',           // Primary color
    backgroundColor: '#ffffff',  // Button background (for secondary/outline variants)
    textColor: '#1f2937',        // Text color
    borderColor: '#e5e7eb',      // Border color
    borderRadius: 8,             // Border radius
    fontFamily: 'Inter, sans-serif',
    fontSize: 14,
    zIndex: 10000,               // Z-index for modal/tooltip
    animationDuration: 200,      // Animation duration in ms
  }}
/>
```

### Custom Button Content

Use `children` to completely customize the button appearance:

```tsx
<AIGenerationButton
  assistantId="my-assistant"
  options={{ mode: 'modal' }}
  onResponse={handleResponse}
>
  <span className="my-custom-button">
    <SparkleIcon /> Generate Code
  </span>
</AIGenerationButton>
```

### Programmatic Control

Use ref to control the component programmatically:

```tsx
import { useRef } from 'react';
import { AIGenerationButton, AIGenerationButtonHandle } from '@devicai/ui';

function Editor() {
  const buttonRef = useRef<AIGenerationButtonHandle>(null);

  const handleKeyboardShortcut = (e: KeyboardEvent) => {
    if (e.metaKey && e.key === 'g') {
      buttonRef.current?.open(); // Open modal/tooltip
    }
  };

  const generateDirectly = async () => {
    const result = await buttonRef.current?.generate('Generate a summary');
    if (result) {
      console.log('Generated:', result.message.content.message);
    }
  };

  return (
    <AIGenerationButton
      ref={buttonRef}
      assistantId="my-assistant"
      options={{ mode: 'modal' }}
      onResponse={handleResponse}
    />
  );
}
```

### Using the Hook Directly

For completely custom UI implementations:

```tsx
import { useAIGenerationButton } from '@devicai/ui';

function CustomGenerateButton() {
  const {
    isOpen,
    isProcessing,
    inputValue,
    setInputValue,
    error,
    result,
    inputRef,
    open,
    close,
    generate,
    reset,
    handleKeyDown,
  } = useAIGenerationButton({
    assistantId: 'my-assistant',
    onResponse: (result) => console.log('Generated:', result),
    onError: (error) => console.error('Error:', error),
  });

  return (
    <div>
      <button onClick={() => open()}>
        {isProcessing ? 'Generating...' : 'Generate'}
      </button>

      {isOpen && (
        <div className="custom-modal">
          <textarea
            ref={inputRef}
            value={inputValue}
            onChange={(e) => setInputValue(e.target.value)}
            onKeyDown={handleKeyDown}
            placeholder="Describe what to generate..."
          />
          <button onClick={() => generate()} disabled={isProcessing}>
            {isProcessing ? 'Working...' : 'Generate'}
          </button>
          <button onClick={close}>Cancel</button>
          {error && <p className="error">{error.message}</p>}
        </div>
      )}
    </div>
  );
}
```

## AIGenerationButton Props Reference

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `assistantId` | `string` | *required* | Assistant identifier |
| `apiKey` | `string` | — | API key (overrides provider) |
| `baseUrl` | `string` | — | Base URL (overrides provider) |
| `tenantId` | `string` | — | Tenant ID (overrides provider) |
| `tenantMetadata` | `TenantMetadata` | — | Tenant metadata |
| `tags` | `string[]` | — | Conversation tags (merged/deduped with the provider's). See [Conversation Tags](#conversation-tags) |
| `options` | `AIGenerationButtonOptions` | — | Display and behavior options |
| `modelInterfaceTools` | `ModelInterfaceTool[]` | — | Client-side tools |
| `onResponse` | `(result: GenerationResult) => void` | — | Fires on successful generation |
| `onBeforeSend` | `(prompt: string) => string \| undefined` | — | Modify prompt before sending |
| `onError` | `(error: Error) => void` | — | Fires on error |
| `onStart` | `() => void` | — | Fires when processing starts |
| `onOpen` | `() => void` | — | Fires when modal/tooltip opens |
| `onClose` | `() => void` | — | Fires when modal/tooltip closes |
| `disabled` | `boolean` | `false` | Disable the button |
| `className` | `string` | — | Additional CSS class for button |
| `containerClassName` | `string` | — | CSS class for container |
| `children` | `ReactNode` | — | Custom button content |
| `theme` | `FeedbackTheme` | — | Theme for modal/tooltip |

## AIGenerationButtonOptions Reference

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `mode` | `'direct' \| 'modal' \| 'tooltip'` | `'modal'` | Interaction mode |
| `prompt` | `string` | — | Predefined prompt (required for direct mode) |
| `placeholder` | `string` | `'Describe what you want to generate...'` | Input placeholder |
| `modalTitle` | `string` | `'Generate with AI'` | Modal title |
| `modalDescription` | `string` | — | Modal description text |
| `confirmText` | `string` | `'Generate'` | Confirm button text |
| `cancelText` | `string` | `'Cancel'` | Cancel button text |
| `tooltipPlacement` | `'top' \| 'bottom' \| 'left' \| 'right'` | `'top'` | Tooltip position |
| `tooltipWidth` | `number \| string` | `300` | Tooltip width |
| `variant` | `'primary' \| 'secondary' \| 'ghost' \| 'outline'` | `'primary'` | Button variant |
| `size` | `'small' \| 'medium' \| 'large'` | `'medium'` | Button size |
| `icon` | `ReactNode` | Sparkles icon | Custom button icon |
| `hideIcon` | `boolean` | `false` | Hide button icon |
| `label` | `string` | `'Generate with AI'` | Button label |
| `hideLabel` | `boolean` | `false` | Hide button label (icon-only) |
| `loadingLabel` | `string` | `'Generating...'` | Label during processing |
| `color` | `string` | `'#3b82f6'` | Primary color |
| `backgroundColor` | `string` | — | Background color |
| `textColor` | `string` | — | Text color |
| `borderColor` | `string` | — | Border color |
| `borderRadius` | `number \| string` | `8` | Border radius |
| `fontFamily` | `string` | System fonts | Font family |
| `fontSize` | `number \| string` | `14` | Font size |
| `zIndex` | `number` | `10000` | Z-index for overlays |
| `animationDuration` | `number` | `200` | Animation duration (ms) |
| `toolRenderers` | `Record<string, (input, output) => ReactNode>` | — | Custom tool call renderers by tool name |
| `toolIcons` | `Record<string, ReactNode>` | — | Custom tool icons by tool name |
| `processingMessage` | `string` | `'Processing...'` | Message shown during processing |
| `toolGroups` | `ToolGroupConfig[]` | — | Group consecutive tool calls under a single renderer |

## AIGenerationButtonHandle Reference

Methods exposed via ref:

| Method | Description |
|--------|-------------|
| `generate(prompt?: string)` | Trigger generation (returns Promise with result) |
| `open()` | Open modal/tooltip (for modal and tooltip modes) |
| `close()` | Close modal/tooltip |
| `reset()` | Reset component state |
| `isProcessing` | Boolean indicating if processing |

## AIElementWrapper Component

The `AIElementWrapper` wraps any React node and turns it into an AI-queryable element. When the user activates the floating trigger, the wrapper either:

- Shows an inline floating tooltip with the assistant's answer (`behavior: 'inline'`), or
- Pushes the wrapped element as a "reference" into the active `ChatDrawer` and opens the drawer (`behavior: 'drawer'`).

The trigger and inline tooltip are rendered through a React portal anchored via `position: fixed`, so they always render above other UI (including the `ChatDrawer`) regardless of stacking context.

### Quick Example

```tsx
import { DevicProvider, ChatDrawer, AIElementWrapper } from '@devicai/ui';

function App() {
  return (
    <DevicProvider apiKey="devic-xxx">
      {/* Inline behavior: hover the underlined value, click the pill,
          and the AI's answer appears in a floating tooltip. */}
      Última actualización:{' '}
      <AIElementWrapper
        label="Última actualización"
        data={{ value: 'hace un momento' }}
        behavior="inline"
        assistantId="my-assistant"
        getPrompt={({ data, label }) =>
          `Explica qué significa "${label}" siendo ${data?.value}.`
        }
      >
        <strong>hace un momento</strong>
      </AIElementWrapper>

      {/* Drawer behavior: pulling the trigger pushes a reference chip into
          the ChatDrawer's prompt and opens the drawer. */}
      <AIElementWrapper
        label="Acme Corp"
        data={{ country: 'ES', employees: 350 }}
        behavior="drawer"
        options={{ showOn: 'hover', triggerLabel: 'Preguntar al chat' }}
      >
        <CompanyCard company={acme} />
      </AIElementWrapper>

      {/* The drawer auto-registers itself in the DevicProvider so the
          wrapper above can open it. No drawer ref or prop needed. */}
      <ChatDrawer assistantId="my-assistant" mode="inline" />
    </DevicProvider>
  );
}
```

### Behaviors

| Behavior | What happens on activation |
|----------|----------------------------|
| `inline` | Sends the prompt built by `getPrompt({ data, label })` (or `Cuéntame más sobre: <label>` by default) and renders the answer in a floating tooltip near the wrapped element. Requires `assistantId`. |
| `drawer` | Adds an `AIReference` to the `DevicProvider` registry and calls `openDrawer()`. The reference appears as a chip in the ChatDrawer's input area; the ChatDrawer prefixes the next user message with `Elemento referenciado: "<labels>"\n\n<msg>` and the user-message render shows a chip widget instead of the raw prefix. |

### `showOn` modes

The trigger pill can appear under different conditions:

| `showOn` | Behavior |
|----------|----------|
| `'hover'` (default) | Visible while the wrapper or the trigger itself is hovered. Hover survives the gap between wrapper and pill (200 ms grace period). |
| `'click'` | Visible only while the inline tooltip is open. |
| `'always'` | Always visible. |
| `'select'` | Visible only while the user has selected text inside the wrapper. The trigger anchors to the selection rectangle, not the wrapper. In `behavior: 'drawer'` mode, the selected text replaces the `label` of the reference. |

Only one wrapper across the page can show its trigger at a time; later activations transparently hide previously active wrappers via a singleton registry.

### Anatomy of the floating UI

- **Trigger pill**: portal-rendered, `position: fixed`, anchored to the wrapper bounding rect (or the selection rect for `showOn: 'select'`).
- **Inline tooltip** (only `behavior: 'inline'`): portal-rendered, anchored to the same rect, contains a header with the `label` and a body with the assistant's response. While processing, shows a spinner with "Pensando…".

### `AIElementWrapperProps`

| Prop | Type | Description |
|------|------|-------------|
| `label` | `string` | Short identifier for the element. Used as chip text in drawer mode and as default-prompt fallback. |
| `data` | `Record<string, any>` | Optional structured data, passed to `getPrompt` and stored on the reference. |
| `referenceContent` | `React.ReactNode` | Optional rich content stored on the reference (drawer mode). |
| `behavior` | `'inline' \| 'drawer'` | Default `'inline'`. |
| `trigger` | `React.ReactNode` | Replaces the default sparkles+label pill. Click handler is attached automatically. |
| `options` | `AIElementWrapperOptions` | Display options (see below). |
| `assistantId` | `string` | Required when `behavior='inline'`. |
| `getPrompt` | `(args: { data?: any; label: string }) => string` | Builds the prompt (inline mode). |
| `apiKey` / `baseUrl` / `tenantId` / `tenantMetadata` | overrides for the `DevicProvider`. |
| `modelInterfaceTools` | `ModelInterfaceTool[]` | Client-side tools (inline mode). |
| `inlineRenderer` | `(message: ChatMessage) => React.ReactNode` | Custom renderer for the inline answer. |
| `onActivate` / `onInlineResponse` / `onError` | Callbacks. |
| `className` / `style` | Wrapper styling. |
| `children` | `React.ReactNode` | The element being wrapped. |

### `AIElementWrapperOptions`

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `showOn` | `'hover' \| 'click' \| 'always' \| 'select'` | `'hover'` | When the trigger pill appears. |
| `triggerPlacement` | `'top' \| 'bottom' \| 'left' \| 'right'` | `'bottom'` | Trigger position relative to the anchor. |
| `tooltipPlacement` | same | `'bottom'` | Inline tooltip position. |
| `tooltipWidth` | `number \| string` | `360` | Inline tooltip width. |
| `triggerLabel` | `string` | `'Preguntar a IA'` | Text inside the default pill. |
| `highlightOnInteract` | `boolean` | `true` | Whether to highlight the wrapped content while interacting. |
| `zIndex` | `number` | `2_147_483_000` | z-index of the portal-rendered overlays. |
| `triggerBorderRadius` | `number \| string` | `999` | Border radius of the default pill. |
| `color` | `string` | — | Primary color of the trigger gradient and tooltip accents. |
| `defaultInlinePrompt` | `string` | — | Used when `getPrompt` is not provided. |

### `AIElementWrapperHandle`

| Method | Description |
|--------|-------------|
| `activate()` | Programmatically click the trigger. |
| `close()` | Close the inline tooltip (no-op for drawer mode). |

### Reference handling on `ChatDrawer`

When `behavior='drawer'` is used, the wrapper integrates with the `ChatDrawer` through the `DevicProvider` (no extra prop wiring needed):

1. The wrapper calls `addReference({ label, content, data })` on the provider and `openDrawer()`.
2. The active `ChatDrawer` reads `references[]` from the provider context and:
   - Renders chip widgets above the textarea (or passes them to a custom prompt box).
   - On send, prefixes the user message with `Elemento referenciado: "<labels>"\n\n<message>`.
   - Clears the references after sending.
3. When the user message renders in the bubble, the prefix is parsed out and shown as `ReferenceChip` nodes (`variant="message"`) above the bubble — the bubble itself only displays the user's actual message text, **rendered as markdown** (see [Markdown rendering](#markdown-rendering)). The LLM still receives the full prefixed content.

The same `ReferenceChip` component is used for the active references inside the input area (`variant="input"`, with a remove button) and for the read-only chips above sent user bubbles, so both look identical. It is exported for reuse (see [Reference chip node](#reference-chip-node)).

### Custom prompt box integration

`CustomPromptBoxProps` exposes the references so a custom input can render or strip them itself. You can reuse the exported `ReferenceChip` to match the default look:

```tsx
import { ReferenceChip, CustomPromptBoxProps } from '@devicai/ui';

function MyPromptBox({
  sendMessage,
  references,
  removeReference,
  isLoading,
}: CustomPromptBoxProps) {
  return (
    <div>
      {references.map((r) => (
        <ReferenceChip
          key={r.id}
          label={r.label}
          variant="input"
          onRemove={() => removeReference(r.id)}
        />
      ))}
      <TextInput
        onSubmit={(text) => sendMessage(text)}
        disabled={isLoading}
      />
    </div>
  );
}
```

### `DevicContext` extensions

The `DevicProvider` now exposes references and drawer registration in the context value:

| Field | Description |
|-------|-------------|
| `references: AIReference[]` | Active references created by `AIElementWrapper` instances. |
| `addReference(ref): string` | Adds a reference, returns its generated id. |
| `removeReference(id)` / `clearReferences()` | Reference management. |
| `registerDrawer(handle): unregister` | Internal — `ChatDrawer` calls this on mount to make itself reachable to `openDrawer()`. |
| `openDrawer()` | Opens the registered `ChatDrawer`. |

### Notes

- The wrapper does not require a `DevicProvider` to work in `'inline'` mode (you can pass `apiKey`/`baseUrl` directly), but `'drawer'` mode does need a provider so the wrapper can locate the registered `ChatDrawer`.
- The trigger pill carries its own copy of the CSS variables (`--devic-aiwrap-color`, etc.) because portals do not inherit custom properties from the wrapper's stacking context.
- The `'select'` mode also adapts the prompt: if `getPrompt` is not provided, the prompt becomes `Cuéntame más sobre: "<selected text>"`.

## Subagent Handoff System

The library supports assistant-to-subagent handoff, where an assistant delegates work to a specialized agent. During handoff, the chat input is automatically disabled and a widget displays the subagent's progress in real time.

### How It Works

1. The assistant calls a `hand_off_subagent` tool, which creates a subthread on the backend
2. The realtime polling response status changes to `handed_off` with a `handedOffSubThreadId` field
3. `useDevicChat` detects the `handed_off` status, stops main polling, and sets `handedOff: true` with the subthread ID
4. ChatInput is disabled with a "Waiting for subagent to complete" notice
5. A `HandoffSubagentWidget` renders inline in the tool timeline, polling the subthread every 5s for status, tasks progress, and summary
6. A background handoff poll checks the realtime endpoint every 5s to detect when the parent thread is no longer in `handed_off` state
7. When the subthread reaches a terminal state (completed, failed, terminated), the widget calls `onHandoffCompleted` which clears handoff state and resumes main polling to pick up the parent thread's continuation

### Automatic Handoff in ChatDrawer

When using `ChatDrawer`, handoff is handled automatically. No extra configuration is needed — the widget appears inline when a `hand_off_subagent` tool call is detected, and the input is disabled until completion.

```tsx
<ChatDrawer
  assistantId="my-assistant"
  // Handoff works out of the box
/>
```

### Custom Handoff Widget

Replace the default handoff widget UI using `handoffWidgetRenderer`:

```tsx
<ChatDrawer
  assistantId="my-assistant"
  options={{
    handoffWidgetRenderer: ({ thread, agent, elapsedSeconds, isTerminal }) => (
      <div className="my-custom-handoff">
        <span>{agent?.name || 'Subagent'} is working...</span>
        {thread?.tasks && (
          <span>
            {thread.tasks.filter(t => t.completed).length}/{thread.tasks.length} tasks
          </span>
        )}
        {isTerminal && <span>Done!</span>}
      </div>
    ),
  }}
/>
```

### HandoffSubagentWidget (Standalone)

Use `HandoffSubagentWidget` directly for custom chat UIs:

```tsx
import { HandoffSubagentWidget } from '@devicai/ui';

<HandoffSubagentWidget
  subThreadId="thread-abc-123"
  onCompleted={() => console.log('Subagent finished')}
  renderWidget={({ thread, agent, elapsedSeconds, isTerminal }) => (
    <MyCustomWidget thread={thread} agent={agent} />
  )}
/>
```

### HandoffSubagentWidget Props Reference

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `subThreadId` | `string` | *required* | The subthread ID to monitor |
| `onCompleted` | `() => void` | — | Called when subthread reaches a terminal state |
| `apiKey` | `string` | — | API key (overrides provider) |
| `baseUrl` | `string` | — | Base URL (overrides provider) |
| `renderWidget` | `(props: { thread, agent, elapsedSeconds, isTerminal }) => ReactNode` | — | Custom renderer replacing the entire widget |

### useDevicChat Handoff Fields

When building custom UIs with `useDevicChat`, the hook exposes handoff state:

```tsx
import { useDevicChat } from '@devicai/ui';

function CustomChat() {
  const {
    messages,
    sendMessage,
    handedOff,              // true when a handoff is active
    handedOffSubThreadId,   // subthread ID being monitored
    onHandoffCompleted,     // callback for HandoffSubagentWidget
    // ...other fields
  } = useDevicChat({ assistantId: 'my-assistant' });

  return (
    <div>
      {/* Render messages, detect hand_off_subagent tool calls */}
      {handedOff && handedOffSubThreadId && (
        <HandoffSubagentWidget
          subThreadId={handedOffSubThreadId}
          onCompleted={onHandoffCompleted}
        />
      )}

      <input disabled={handedOff} placeholder="Type a message..." />
    </div>
  );
}
```

| Field | Type | Description |
|-------|------|-------------|
| `handedOff` | `boolean` | Whether the assistant has handed off to a subagent (set when realtime status is `handed_off`) |
| `handedOffSubThreadId` | `string \| null` | The active subthread ID (from `RealtimeChatHistory.handedOffSubThreadId`) |
| `onHandoffCompleted` | `() => void` | Callback that clears handoff state and resumes main polling when subagent finishes |
| `status` | `RealtimeStatus \| 'idle'` | Current status — includes `'handed_off'` when a handoff is active |

## ThreadStateTag Component

A standalone component that displays the current state of an agent thread as a colored tag with an interactive dropdown for actions (explain, pause, resume, approve, complete).

### Basic Usage

```tsx
import { ThreadStateTag, AgentThreadState } from '@devicai/ui';

<ThreadStateTag
  state={AgentThreadState.PROCESSING}
  threadId="thread-abc-123"
  agentName="Research Agent"
/>
```

### Interactive Actions

When `interactive` is true (default), clicking the tag opens a dropdown with context-specific actions:

- **Explain Thread**: AI-generated explanation of what the thread has done (with typing animation)
- **Pause/Resume**: Pause a processing thread or resume a paused one
- **Review Approval**: Approve or reject a thread waiting for approval
- **Complete Manually**: Admin action to manually complete/fail/terminate a thread

```tsx
<ThreadStateTag
  state={AgentThreadState.PAUSED_FOR_APPROVAL}
  threadId="thread-abc-123"
  agentName="Deployment Agent"
  showAdminActions={true}
  onActionComplete={(info) => {
    if (info === 'WAITING_FOR_RESPONSE_EXPIRED') {
      console.log('Response window expired');
    }
    refreshThread();
  }}
/>
```

### Display-Only Mode

Disable the dropdown for read-only contexts:

```tsx
<ThreadStateTag
  state={thread.state}
  threadId={thread._id}
  agentName="My Agent"
  interactive={false}
/>
```

### Thread States

The `AgentThreadState` enum defines all 12 possible states:

| State | Tag Color | Description |
|-------|-----------|-------------|
| `QUEUED` | Gold | Waiting to start |
| `PROCESSING` | Blue (processing) | Actively running |
| `COMPLETED` | Green | Successfully finished |
| `FAILED` | Red | Failed with error |
| `TERMINATED` | Red | Manually terminated |
| `PAUSED` | Gold | Paused by user or system |
| `PAUSED_FOR_APPROVAL` | Gold | Waiting for approval |
| `APPROVAL_REJECTED` | Red | Approval was rejected |
| `WAITING_FOR_RESPONSE` | Purple | Waiting for external input |
| `PAUSED_FOR_RESUME` | Gold | Paused and waiting to resume |
| `HANDED_OFF` | Blue | Delegated to subagent(s) |
| `GUARDRAIL_TRIGGER` | Red | Guardrail violation |

### ThreadStateTag Props Reference

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `state` | `AgentThreadState \| string` | *required* | Current thread state |
| `threadId` | `string` | *required* | Thread ID for API actions |
| `agentName` | `string` | *required* | Agent name shown in modals |
| `showIcon` | `boolean` | `true` | Show icon next to state text |
| `customIcon` | `ReactNode` | — | Replace the default state icon |
| `pausedReason` | `string` | — | Reason for pause (shown in tooltip) |
| `approvalRejectedMessage` | `string` | — | Message when approval rejected |
| `finishReason` | `string` | — | Reason the thread finished |
| `onActionComplete` | `(info?) => void` | — | Callback after actions (explain, pause, approve, etc.) |
| `pauseUntil` | `number` | — | Timestamp until which thread is paused |
| `subthreadCount` | `number` | — | Number of parallel subthreads (shown in handed_off tooltip) |
| `showAdminActions` | `boolean` | `false` | Show admin actions (complete manually) |
| `apiKey` | `string` | — | API key (overrides provider) |
| `baseUrl` | `string` | — | Base URL (overrides provider) |
| `interactive` | `boolean` | `true` | Enable dropdown on click |

## Agent API Client Methods

The `DevicApiClient` includes methods for managing agent threads:

```tsx
import { DevicApiClient } from '@devicai/ui';

const client = new DevicApiClient({ apiKey: 'your-api-key' });

// Get thread details (optionally with tasks)
const thread = await client.getThreadById('thread-id', true);

// Get agent details
const agent = await client.getAgentDetails('agent-id');

// Get AI explanation of thread execution
const explanation = await client.explainAgentThread('thread-id');

// Pause or resume a thread
await client.pauseResumeThread('thread-id', 'paused');
await client.pauseResumeThread('thread-id', 'queued');

// Handle approval (approve/reject with optional retry and message)
await client.handleThreadApproval('thread-id', true, false, 'Looks good');

// Manually complete a thread
await client.completeThread('thread-id', 'completed');

// Get full chat content (used after handoff completes)
const messages = await client.getChatHistoryContent('assistant-id', 'chat-uid');
```

## Troubleshooting

### Chat not loading

1. Verify your API key is correct
2. Check the browser console for errors
3. Ensure the assistant identifier exists

### Styles not applied

1. Make sure you imported the CSS: `import '@devicai/ui/styles.css'`
2. Check for CSS conflicts with your application styles
3. Try increasing specificity or using CSS variables

### Tools not being called

1. Verify the tool schema matches OpenAI function calling format
2. Check that `toolName` matches `function.name` in schema
3. Ensure the assistant has been configured to use client-side tools

### Handoff widget not appearing

1. Ensure the assistant is configured with a `hand_off_subagent` tool on the backend
2. The widget renders inside the tool timeline — verify `showToolTimeline` is not set to `false`
3. Check that the API key has permission to access agent thread endpoints
4. Verify the realtime response returns `status: 'handed_off'` with `handedOffSubThreadId` — the handoff detection relies on this
5. Check console for `[useDevicChat] Handoff state set:` logs to confirm the subthread ID is being received

### File uploads not working

1. Enable file uploads in options: `enableFileUploads: true`
2. Check allowed file types configuration
3. Verify file size is within limits (default 10MB in UI, 25MB max at API)
4. If using the default upload (no `onFileUpload`), ensure the API key has access to `POST /api/v1/files/upload`
5. If using a custom `onFileUpload`, ensure it returns `ChatFile[]` with valid `downloadUrl` values

## Support

For issues and feature requests, visit the [GitHub repository](https://github.com/devic-ai/devic-ui).
