# Tenant Sessions API

A short-lived bearer token that **proves** which of your customers is calling,
instead of letting the caller declare it.

**Base path:** `/api/v1/tenant-sessions`

## Why it exists

Everywhere else in this API, the tenant travels beside the key as a parameter —
`?tenantId=acme-corp`, `{"tenantId": "acme-corp"}`, `/tenant-usage/acme-corp`.
That is fine for a key that never leaves your servers, because you are the only
one holding it.

In a browser it is not. The key is in the bundle, so anyone who opens the
network tab can read it, put a different tenant beside it, and get that tenant's
conversations back. Allowed domains narrow *where* a key works from, never *who*
is behind it.

A tenant session replaces the claim with a fact:

| | API key alone | Tenant session |
|---|---|---|
| Who says the tenant | the caller, per request | your backend, once, inside a signed token |
| Can it act as another tenant | yes, by editing the request | no — the token is signed |
| What it reaches | everything the key is allowed | only what an end user does (see below) |
| Lifetime | until revoked | 1 h by default, 12 h maximum |

## Issuing a session

```
POST /api/v1/tenant-sessions
```

**Call this from your backend**, with a server-side API key, taking the identity
from **your own login** — never from the request body of the page asking for it.
A page that could mint its own session could mint one for any tenant, which is
exactly the impersonation this replaces.

### Request

| Field | Type | Required | Description |
|---|---|---|---|
| `tenantId` | string | ✅ | The identity this session proves |
| `subtenantId` | string | — | The end user inside that tenant |
| `ttlSeconds` | number | — | Lifetime. Default `3600`, min `60`, max `43200` (12 h). Out-of-range values are **clamped**, not refused |

```bash
curl -X POST "https://api.devic.ai/api/v1/tenant-sessions" \
  -H "Authorization: Bearer devic-your-server-side-key" \
  -H "Content-Type: application/json" \
  -d '{
    "tenantId": "acme-corp",
    "subtenantId": "user-456",
    "ttlSeconds": 3600
  }'
```

### Response

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "tokenType": "tenant-session",
  "tenantId": "acme-corp",
  "subtenantId": "user-456",
  "expiresIn": 3600,
  "expiresAt": 1754500000000
}
```

`expiresAt` is epoch **milliseconds**; `expiresIn` is seconds from now. Hand the
whole object to your page — [`@devicai/ui`](https://www.npmjs.com/package/@devicai/ui)
accepts it as-is and renews before expiry.

### Refusals

| Status | When | What it means |
|---|---|---|
| `400` | no `tenantId` | The session has no identity to prove |
| `400` | `ttlSeconds` is not a number | Out-of-range numbers are clamped; non-numbers are refused |
| `403` | the request carries an `Origin` header | It came from a browser. Only a server may mint |
| `403` | the API key has `allowedDomains` configured | That key is a browser key; use a server-side one |
| `403` | the bearer is itself a tenant session | A session cannot mint another one — otherwise the expiry could be extended forever by whoever holds the token |

The `Origin` check and the `allowedDomains` check are independent, and either
one alone refuses the call.

## Using a session

Send it exactly like an API key:

```bash
curl "https://api.devic.ai/api/v1/assistants/support-bot/chats" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

Two rules then apply, and both matter — the first says which doors exist, the
second says which room you end up in.

### 1. Surface: what a session may call

Deny by default. A route not on this list answers `401`, including routes added
to the API after your token was minted.

| Area | Allowed |
|---|---|
| The assistant | `GET /assistants`, `GET /assistants/tags`, `GET /assistants/{id}` |
| Talking | `POST /assistants/{id}/messages`, `POST /assistants/chats`, `GET /assistants/{id}/chats`, `GET /assistants/{id}/chats/{uid}`, `.../realtime`, `.../search`, `POST .../tool-response`, `POST .../stop`, `GET`+`POST .../feedback` |
| Agent runs | `GET /agents/{id}`, `GET /agents/threads/{id}`, `.../search`, `POST .../messages`, `.../approval`, `.../complete`, `.../pause`, `.../resume`, `GET`+`POST .../feedback` |
| Attachments & dictation | `POST /files/upload`, `POST /whisper`, `GET /whisper/{id}` |
| Own core memory | `GET`+`POST /memory/assistants/{id}/core`, `PATCH`+`DELETE /memory/assistants/{id}/core/{entryId}` |
| Own limits | `GET /tenant-usage/{tenantId}`, `.../history`, `.../subtenants/{id}`, `.../subtenants/{id}/history` |
| Own connected apps | `GET /tenant-integrations`, `GET /tenant-integrations/{app}/accounts`, `POST /tenant-integrations/{app}/connect`, `POST /tenant-integrations/refresh`, `DELETE /tenant-integrations/accounts/{id}` |

All paths are relative to `/api/v1`. Wildcards consume **one segment**, so
`GET /assistants/{id}` grants nothing nested underneath it.

Deliberately absent — a token that reaches a browser must not do these even when
the key that minted it can:

- Creating, updating or deleting anything: assistants, agents, tool servers,
  documents, projects.
- The account's costs and its price plans (`GET /tenant-usage/tiers` is refused
  explicitly, since `tiers` would otherwise look like a tenant id).
- Tenant administration (`/tenant-admin/*`) and tenant management
  (`/tenants/*`).
- The workspace's own connected accounts (`/integrations/*`) and triggers.
- Minting another session.

### 2. Data: which tenant the call lands on

The identity in the token is **imposed** on every request — path, query string
and body alike, since the same field is spelled in all three across this API.

- **Says nothing** → the value is written in. A listing with no filter returns
  the caller's own data, not everyone's.
- **Says someone else** → `403 This session is for tenant "acme-corp" and
  cannot act as "globex".` It is not silently rewritten: an honest bug and a
  real attempt look identical from here, and both deserve to be seen.

A session minted for a tenant with no `subtenantId` may look at any of its own
subtenants — they are its own. One minted **for** a subtenant is held to it.

Conversations and threads are checked by ownership too: touching one that
belongs to another tenant answers `403 This chat belongs to someone else.` A
conversation that does not exist stays a `404` — answering `403` would reveal
which ids exist.

## Making sessions compulsory: the API key identity mode

Everything above is still optional until the key that mints the sessions cannot
do anything else. Every API key has an identity mode, set when you create it or
from **API Keys → restrictions** in the console:

| Mode | What the key can do |
|---|---|
| `open` (default) | Anything it is allowed, for whichever tenant it declares beside itself |
| `signed` | `POST /api/v1/tenant-sessions`, and nothing else. Every other `/api/v1` call with the key alone answers `401` |

```
401 This API key only acts through a tenant session. Mint one with
POST /api/v1/tenant-sessions and send that as the bearer token.
```

The mode is checked before anything else, so it also covers routes an
allowlist would not have known about. It applies to the **key**; calls carrying
a session issued by it are unaffected.

Which makes a `signed` key a single-purpose key: minting sessions in front of a
browser. Anything else your server does — provisioning assistants, reading
costs, running agents — needs a second key left on `open`.

> Switching an existing key to `signed` breaks every direct call it was making,
> immediately. Mint a new key for the sessions, move the page over, and revoke
> the old one afterwards.

## End to end

```ts
// ── Your server ──────────────────────────────────────────────────────
// DEVIC_API_KEY is a `signed` key: it can do this and nothing else.
app.post('/api/devic-session', requireLogin, async (req, res) => {
  const r = await fetch('https://api.devic.ai/api/v1/tenant-sessions', {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${process.env.DEVIC_API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      tenantId: req.user.organisationId,   // from YOUR session,
      subtenantId: req.user.id,            // never from the body
    }),
  });
  res.json(await r.json());
});
```

```tsx
// ── Your page: no API key anywhere ───────────────────────────────────
<DevicProvider
  getTenantSession={() =>
    fetch('/api/devic-session', { credentials: 'include' }).then((r) => r.json())
  }
  onSessionExpired={() => location.assign('/login')}
>
  <ChatDrawer assistantId="support-bot" />
</DevicProvider>
```

With [`@devicai/sdk`](https://www.npmjs.com/package/@devicai/sdk) the server half
is one line — `await devic.auth(orgId, userId).session()`. See the `devic-sdk`
skill.

### Without a renewal endpoint

You do not have to expose one. Mint the session inside your own login with a
`ttlSeconds` matching your session, put it where the page can read it, and set
`onSessionExpired` so the widget says something when both expire together:

```ts
const { token } = await issueSession({ tenantId: org, subtenantId: user, ttlSeconds: 8 * 3600 });
res.cookie('devic_session', token, { sameSite: 'lax' });
```

The trade is only the window if a token is stolen — the same one your own
session cookie already accepts. What matters is untouched: a stolen session
still cannot act as another tenant or reach past what an end user does.

## Related

- [tenants.md](tenants.md) — tenants, subtenants, usage and limits
- [assistants.md](assistants.md) — `tenantIntegrations` on an assistant, and the
  connected-apps configuration whose `identityMode: 'signed'` refuses unsigned
  callers outright
- The `devic-ui` skill — `getTenantSession` in the browser
- The `devic-sdk` skill — minting sessions from a Node backend
