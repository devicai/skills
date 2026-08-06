# Connected Apps (devic-ui)

Where the **end user** connects their **own** third-party accounts — Gmail,
Slack, HubSpot — to the assistant they are talking to. Never the workspace-wide
accounts an administrator connected, and never another tenant's.

Requires `@devicai/ui` ≥ 0.38.0; the no-request-when-there-is-nothing behaviour
described below since 0.42.0.

## Prerequisites

- The assistant (or its environment) has **tenant integrations** enabled and
  lists the apps on offer. Nothing outside that list is connectable.
- The credential allows `/api/v1/tenant-integrations/*` — included in the
  `devic-ui` API key preset, and in what a tenant session may do.
- A tenant is resolved (`tenantId` on the provider or the drawer, or proven by a
  tenant session).

## In the drawer

Nothing to wire. The header grows a stack of the apps' own logos when there is
something behind it:

```tsx
<ChatDrawer
  assistantId="support-bot"
  tenantId="acme-corp"
  subtenantId="user-456"
  options={{
    showIntegrationsButton: true,       // default
    integrationsLabel: 'Connected apps',
    maxIntegrationLogos: 6,             // the rest are counted in a +N box
    showIntegrationsHint: true,         // default
    integrationsHintLabel: 'Connect your apps',
  }}
/>
```

| Option | Type | Default | Description |
|---|---|---|---|
| `showIntegrationsButton` | `boolean` | `true` | Allow the header control. `false` keeps it out even when apps are offered |
| `integrationsLabel` | `string` | `'Connected apps'` | Tooltip and accessible name |
| `maxIntegrationLogos` | `number` | `6` | Logos shown before a `+N` box; fewer if the header is narrow |
| `showIntegrationsHint` | `boolean` | `true` | The strip above the composer that says in words what the logos imply. Ignored when the button is off |
| `integrationsHintLabel` | `string` | — | Unset it reads "Connect your apps", and "Explore connected apps" once something is connected |

Connected apps come first in the stack and unconnected ones are dimmed, so it
doubles as the status. The hint strip can be closed, and the dismissal is
remembered per assistant and tenant/subtenant: the same end user is never told
twice, the next one still is.

### Assistants that offer nothing cost nothing

`GET /api/v1/assistants/{id}` returns `tenantIntegrations: { enabled, count }`,
and the drawer is already making that call for its header. So:

- `enabled: false` → no listing request at all, no control.
- `enabled: true` → `count` sizes a placeholder of dimmed circles while the
  listing arrives, instead of the control appearing a moment later and pushing
  the title sideways.
- **field absent** (older deployment) → read as *cannot tell*: the listing is
  requested and the control appears if there is anything behind it.

Before 0.42.0 every page load asked for the listing just to be refused.

## Standalone

Your own trigger, anywhere:

```tsx
<IntegrationsModal
  isOpen={open}
  onClose={() => setOpen(false)}
  assistantId="support-bot"
  tenantId="acme-corp"        // falls back to the provider's
  subtenantId="user-456"      // falls back to the provider's
  onChange={(apps) => console.log(apps.filter((a) => a.connected))}
/>
```

To put the same stack somewhere else — or to know whether an assistant offers
anything before rendering your own control — use the pieces. The listing is
loaded once and shared, so a launcher and a modal never ask twice:

```tsx
const apps = useIntegrations({ assistantId, tenantId, subtenantId, enabled: isOpen });

{apps.offered && <MyButton count={apps.integrations.length} />}
<IntegrationsLauncher state={apps} onClick={open} dark />
<IntegrationsModal isOpen={isOpen} onClose={close} state={apps} {...scope} />
```

`useIntegrations` returns `IntegrationsState`:

| Field | Type | Description |
|---|---|---|
| `integrations` | `Integration[]` | Offered apps, each with this tenant's own accounts |
| `loading` | `boolean` | A request is in flight |
| `error` | `string \| null` | — |
| `offered` | `boolean` | Whether this assistant offers apps at all. **False before the first answer**, so anything hanging off it stays hidden rather than appearing and disappearing |
| `settled` | `boolean` | True once an answer or a refusal has arrived |
| `refresh` | `(dropCache?: boolean) => Promise<void>` | Re-read the listing |
| `client` / `scope` | | The client and resolved scope used |

`enabled` is what keeps a widget nobody opens from costing a request — the
drawer passes its own open state.

Exported: `IntegrationsModal`, `IntegrationsLauncher`, `IntegrationsHint`,
`IntegrationLogo`, `useIntegrations`. Types: `IntegrationsModalProps`,
`IntegrationsLauncherProps`, `IntegrationsHintProps`, `IntegrationLogoProps`,
`IntegrationsState`, `IntegrationsScope`, `UseIntegrationsOptions`.

## Theming

Opened from the drawer it inherits the drawer's colours and font. Standalone,
pass them yourself — both dialogs render through a portal, so nothing cascades
into them on its own:

```tsx
<IntegrationsModal
  ...
  theme={{ backgroundColor: '#1a1a1a', textColor: '#e6e6e6', color: '#e8833a',
           secondaryBackgroundColor: '#0f0f0f', borderColor: '#333' }}
/>
```

## Behaviour worth knowing

**Connecting** opens the provider's consent screen in a pop-up and refreshes as
soon as it closes. If the browser blocks the pop-up, the authorisation URL is
offered as a link instead.

**One account per app.** A tenant connecting an app again is switching account,
not adding one: the previous account is retired server-side. Two accounts for
the same app would be indistinguishable — the run has to pick one, and the end
user cannot see which.

**A 4xx means "not for you", not "something broke".** A disabled catalogue, a
missing tenant and an assistant that does not exist are all answered the same
way, deliberately, and the UI simply shows nothing.

## `useAssistantInfo`

The hook behind the header, exported because a host building its own needs the
same answer:

```tsx
import { useAssistantInfo, forgetAssistant } from '@devicai/ui';

const { assistant, settled } = useAssistantInfo({
  assistantId: 'support-bot',
  client,                            // DevicApiClient
  baseUrl: 'https://api.devic.ai',
  credential: apiKey ?? 'session',   // separates accounts in the cache key
  enabled: true,
});

if (settled && assistant?.tenantIntegrations?.enabled) { ... }
```

Fetched at most once per assistant, however many components ask — the **promise**
is cached, so a second caller arriving mid-request waits for the first. A
failure resolves to `assistant: null` with `settled: true`.

Gate on `settled`, never on `assistant`: a null before it has settled only means
"not yet".

`forgetAssistant(baseUrl, assistantId, credential?)` drops the cached answer.
Needed after changing the assistant from your own admin UI; a page that only
chats never calls it.

## Related

- [tenant-sessions.md](tenant-sessions.md) — proving the tenant whose accounts
  these are
- The `devic-api` skill — the `/api/v1/tenant-integrations` endpoints and
  `tenantIntegrations` on the assistant
