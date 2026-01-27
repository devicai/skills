---
name: devic-ui
description: Devic UI is a react component library to integrate AI UI components like chats and agents executions handler directly in your code base connected to devicai API
---

# Devic UI Integration Guide

This guide explains how to integrate the `@devicai/ui` library into your React application to add AI assistant chat capabilities.

## Prerequisites

- Node.js 20+
- React 17+ application
- Devic API key (obtain from Devic dashboard)

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
  tenantId="global-tenant-id"
  tenantMetadata={{ organizationId: 'org-123' }}
>
  <ChatDrawer
    assistantId="support-assistant"
    tenantId="specific-tenant-override"  // Overrides provider
    tenantMetadata={{
      userId: 'user-456',
      plan: 'enterprise'
    }}
  />
</DevicProvider>
```

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

Enable file attachments in chat:

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
  ChatMessage,
  ChatDrawerProps,
  ChatDrawerOptions,
  ModelInterfaceTool,
  ModelInterfaceToolSchema,
  UseDevicChatOptions,
  UseDevicChatResult,
  RealtimeChatHistory,
  AssistantSpecialization,
} from '@devicai/ui';

// Use types in your code
const options: ChatDrawerOptions = {
  position: 'right',
  width: 400,
  welcomeMessage: 'Hello!',
};

const handleMessage = (message: ChatMessage) => {
  console.log(message.content.message);
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
| `tenantMetadata` | `Record<string, any>` | — | Tenant metadata (overrides provider) |
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
| `suggestedMessages` | `string[]` | — | Quick action suggestions |
| `inputPlaceholder` | `string` | `'Type a message...'` | Input placeholder text |
| `showToolTimeline` | `boolean` | `true` | Show tool execution timeline |
| `enableFileUploads` | `boolean` | `false` | Enable file attachments |
| `allowedFileTypes` | `AllowedFileTypes` | — | Filter by file type (images, documents, audio, video) |
| `maxFileSize` | `number` | `10485760` | Max file size in bytes (10MB) |
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

### File uploads not working

1. Enable file uploads in options: `enableFileUploads: true`
2. Check allowed file types configuration
3. Verify file size is within limits

## Support

For issues and feature requests, visit the [GitHub repository](https://github.com/devic-ai/devic-ui).
