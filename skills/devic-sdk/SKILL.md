---
name: devic-sdk
description: "@devicai/sdk reference — the official Devic TypeScript/JavaScript SDK for Node backends. Use when calling the Devic API from server-side code: chatting with assistants, running agents, managing tool servers, documents, skills, integrations and triggers, and above all minting tenant sessions so a browser never carries an API key."
---

# @devicai/sdk

The official Devic SDK for TypeScript and JavaScript. **Runs on your server** —
it takes an API key, and an API key must not reach a browser.

```bash
npm install @devicai/sdk
```

```ts
import { Devic } from '@devicai/sdk';

const devic = new Devic({ apiKey: process.env.DEVIC_API_KEY! });

const reply = await devic.assistants.chat('support-bot', 'where is my order?');
```

## The one idea

`devic.*` speaks for your **workspace**. `devic.auth(tenantId)` speaks for **one
of your customers** inside it.

```ts
// As the workspace: what an operator configures.
await devic.assistants.list();
await devic.toolServers.create({ /* … */ });
await devic.projects.list();

// On behalf of a customer: what an end user does.
const acme = devic.auth('acme-corp', 'user-456');
await acme.assistants.chat('support-bot', 'where is my order?');
await acme.assistants.conversations.list('support-bot');
await acme.integrations.list('support-bot');
await acme.usage.get();
```

Everything through a scope carries that customer's identity, so it cannot be
left off one call by accident — the failure with no symptom: the message goes
through, the answer looks right, and the conversation is filed under your
workspace instead of under your customer.

And what a customer must not do is simply not reachable from there. There is no
`acme.toolServers`, no `acme.projects`, no `acme.documents`.

## Surface

On `devic`:

| | |
|---|---|
| `devic.assistants` | `list` `get` `create` `update` `delete` `chat` `chatAsync` `respondToTools`, and `conversations.{list,get,live,search,stop,feedback}` |
| `devic.agents` | `list` `get` `create` `update` `delete`, `runs.{start,list,get,update,approve,reject,complete,pause,resume,evaluate,evaluation,feedback}`, `costs.{daily,monthly}` |
| `devic.toolServers` | servers, their `definition`, and `tools.{list,get,add,update,delete,test}` |
| `devic.projects` | projects, their `runs`, `conversations`, `stats` and `costs` |
| `devic.documents` | knowledge documents, versions, folders |
| `devic.skills` | the skill catalogue, install and uninstall |
| `devic.integrations` | the app catalogue and the **workspace's** connected accounts |
| `devic.triggers` | starting agents and assistants from app events |
| `devic.tenantSessions` | minting tokens that prove which customer is calling |

On `devic.auth(tenantId, subtenantId?)`:

| | |
|---|---|
| `.assistants` / `.agents` | the same, with the customer filled in |
| `.integrations` | the apps **that customer** connected for themselves |
| `.usage` | their limits and what they have consumed |
| `.session()` | the credential for their browser |

`chat` accepts a bare string or the full message DTO:

```ts
await acme.assistants.chat('support-bot', 'hola');
await acme.assistants.chat('support-bot', {
  message: 'hola',
  chatUid: 'existing-conversation',
  tags: ['support'],
  metadata: { source: 'web' },
});
```

Anything not wrapped yet is reachable on `devic.client`, the HTTP client
underneath.

## Tenant sessions — the main reason to install this

Your page needs a credential. Sending it your API key makes the tenant a claim:
the key is readable by anyone who opens the network tab, so anyone can say they
are any of your customers.

Mint a session instead, from your server, taking the identity from **your own
login** and never from the request body:

```ts
app.post('/api/devic-session', requireLogin, async (req, res) => {
  const session = await devic
    .auth(req.user.organisationId, req.user.id)
    .session();                       // { token, expiresAt, expiresIn, … }

  res.json(session);
});
```

Then in your page, with [`@devicai/ui`](https://www.npmjs.com/package/@devicai/ui):

```tsx
<DevicProvider
  getTenantSession={async () => {
    const r = await fetch('/api/devic-session', { credentials: 'include' });
    return r.json();
  }}
  onSessionExpired={() => location.assign('/login')}
>
  <ChatDrawer assistantId="support-bot" />
</DevicProvider>
```

A session lasts an hour by default (60 s min, 12 h max via
`session({ ttlSeconds })`), is confined to what an end user does, and dies with
the API key that minted it. It cannot create assistants, read your costs, or
reach another customer — whatever the page asks for.

### Without a renewal endpoint

Mint it inside your own login with a matching lifetime and put it where the page
can read it:

```ts
const { token } = await devic.auth(user.org, user.id).session({ ttlSeconds: 8 * 3600 });
res.cookie('devic_session', token, { sameSite: 'lax' });
```

The trade is only the window if a token is stolen — the same one your session
cookie already accepts. Do set `onSessionExpired`: there is nothing to renew
from, so without it the widget stops answering at the exact moment the user's
login has expired too.

### Making it compulsory

All of this is a convention until the key cannot do anything else. In the
console, an API key has an identity mode:

| Mode | What the key can do |
|---|---|
| `open` (default) | Anything it is allowed, for whichever tenant it declares beside itself |
| `signed` | Mint tenant sessions, and nothing else. Every other `/api/v1` call with the key alone answers `401` |

```ts
const devic = new Devic({ apiKey: process.env.DEVIC_API_KEY! });

await devic.auth('acme-corp', 'user-456').session();   // the one thing it can do
await devic.assistants.list();                         // 401 — and that is the point
```

So a `signed` key is single-purpose: minting sessions in front of a browser.
Anything else your server does — provisioning assistants, reading costs, running
agents — needs a second key left on `open`. Two keys, two jobs.

The issuer also refuses a browser outright: a request carrying an `Origin`
header, or made with a key that has `allowedDomains`, gets `403`. And a session
cannot mint another session.

## Errors

Every failure is a `DevicApiError` with the status code and the API's own
message:

```ts
import { DevicApiError } from '@devicai/sdk';

try {
  await devic.auth('acme-corp').assistants.chat('support-bot', 'hola');
} catch (e) {
  if (e instanceof DevicApiError && e.statusCode === 429) {
    // the tenant is over its usage limit
  }
  throw e;
}
```

## Configuration

```ts
new Devic({
  apiKey: process.env.DEVIC_API_KEY!,
  baseUrl: 'https://api.devic.ai',        // default
  source: 'sdk',                          // how the API files this traffic
});
```

`source` only matters when building a tool on top of this package and wanting
its usage counted apart from your own.

For callers whose credential is a token that expires rather than a static API
key — an internal service passing a user's access token:

```ts
new Devic({
  apiKey: accessToken,
  refreshToken: () => renew(),              // called after a 401, then retried once
  shouldRefreshProactively: () => isExpired(accessToken),
});
```

## Which tool for what

| | |
|---|---|
| `@devicai/sdk` | Server-side code in Node/TypeScript. Typed, and the tenant scope is enforced by construction |
| `@devicai/ui` | React components in the browser. Never holds an API key when sessions are used |
| `@devicai/cli` | Terminal and scripts. Same API, no code |
| REST directly | Any other language — see the `devic-api` skill |

## Related skills

- `devic-api` — the endpoints underneath, and `tenant-sessions.md` for the full
  session rules (allowed routes, refusal codes, identity pinning)
- `devic-ui` — the browser half, `getTenantSession` and connected apps
- `devic-cli` — the same operations from a terminal
