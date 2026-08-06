# Tenant Sessions (devic-ui)

How a page proves **which of your customers** is using it, without shipping a
credential that can claim to be any of them.

Requires `@devicai/ui` ≥ 0.41.0 (`createSharedSession` since 0.41.1).

## The problem with `apiKey` in a browser

```tsx
<DevicProvider apiKey="devic-xxx" tenantId="acme-corp">
```

Both values are in your bundle. Anyone who opens the network tab can read the
key, change `acme-corp` to a competitor's identifier, and get their
conversations back. `allowedDomains` on the key narrows *where* it works from,
never *who* is behind it.

## `getTenantSession`

Your backend — the only place that knows who is logged in — mints a short-lived
token that carries the tenant in a signed claim. The page never sees an API key.

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

| Prop | Type | Description |
|---|---|---|
| `getTenantSession` | `(force?: boolean) => Promise<string \| TenantSessionToken>` | Returns a session token. Called again on its own before expiry, and with `force: true` after the API refuses the token in hand |
| `onSessionExpired` | `() => void` | The session is dead and could not be replaced — `getTenantSession` handed back the same expired token |
| `apiKey` | `string` | **Optional** once `getTenantSession` is supplied. Omit it: a page using sessions has no reason to carry one |

`TenantSessionToken` is `{ token, expiresAt?, expiresIn? }` — `expiresAt` in
epoch **ms**, `expiresIn` in seconds. Returning the bare token string also
works: the expiry is then read out of the JWT itself.

`force` matters. A client passes it after the API refused the token it just
used, which means the cached answer is known to be dead. Handing the same token
back turns a recoverable `401` into a widget that never speaks again — so a
custom implementation must not ignore the argument.

## The server half

```ts
// DEVIC_API_KEY is a server-side key. Never the one in your bundle.
app.post('/api/devic-session', requireLogin, async (req, res) => {
  const r = await fetch('https://api.devic.ai/api/v1/tenant-sessions', {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${process.env.DEVIC_API_KEY}`,
      'Content-Type': 'application/json',
    },
    // From YOUR session, never from the request body.
    body: JSON.stringify({
      tenantId: req.user.organisationId,
      subtenantId: req.user.id,
      ttlSeconds: 3600,              // default; 60s min, 12h max
    }),
  });
  res.json(await r.json());
});
```

With [`@devicai/sdk`](https://www.npmjs.com/package/@devicai/sdk) it is one
line — `await devic.auth(orgId, userId).session()`. See the `devic-sdk` skill.

The issuer refuses a browser: a request carrying an `Origin` header, or made
with a key that has `allowedDomains` configured, gets `403`. A session cannot
mint another session either.

## Without a renewal endpoint

You do not have to expose one. Mint the session inside your own login, give it a
lifetime matching your session, and put it where the page can read it:

```tsx
<DevicProvider
  getTenantSession={async () => readCookie('devic_session')}
  onSessionExpired={() => location.assign('/login')}
>
```

The trade is only the window if a token is stolen — the same one your own
session cookie already accepts. A stolen session still cannot act as another
tenant or reach past what an end user does.

**Do set `onSessionExpired` in this mode.** There is nothing to renew from, so
without it the widget simply stops answering, at the exact moment the user's own
login has expired too.

The cookie has to be readable by JavaScript, so it is exposed to XSS; injecting
the token into the page at render time carries the same risk without sending it
on every request to your own domain.

## One session for the whole tree

Every component builds its own API client, and each dedupes only its own
renewals — so a drawer, a command bar and a modal would ask your backend for
three tokens before anyone had said a word.

`DevicProvider` shares one between everything below it; nothing to configure.

Outside a provider — a bare `DevicApiClient`, or your own code wanting the token
the widgets are using — `createSharedSession` gives the same behaviour: one
in-flight request, reused until close to expiry, re-fetched on `force`.

```tsx
import { createSharedSession, DevicApiClient } from '@devicai/ui';

const session = createSharedSession(() =>
  fetch('/api/devic-session', { credentials: 'include' }).then((r) => r.json())
);

const client = new DevicApiClient({
  baseUrl: 'https://api.devic.ai',
  getTenantSession: session,
  onSessionExpired: () => location.assign('/login'),
});
```

## What a session may do

Chatting, its own conversations, attachments, dictation, its own core memory,
its own limits, its own connected apps. Nothing that configures the workspace —
creating assistants, reading the account's costs, touching agents' setup — even
when the API key that minted it could.

The tenant in the token is imposed on path, query string and body. A request
naming another tenant answers `403`; a request naming none has the right one
written in. So `tenantId` props become redundant under a session: passing the
matching value is harmless, passing a different one fails.

For the full route list, the refusal codes and the API key's `signed` mode, see
the `devic-api` skill's `tenant-sessions.md`.

## Making it compulsory

The page is only as safe as the key it stopped shipping. In the Devic console,
put the key that mints the sessions in **`signed`** mode: minting becomes the
only thing it can do, and every other `/api/v1` call with the key alone answers
`401`.

That key belongs on your server. With sessions, the bundle carries no key at
all.

> Switching a key that is already in a bundle to `signed` takes the page down
> immediately. Mint a second key, move the page to `getTenantSession`, and
> revoke the old one afterwards.

## Migration checklist

1. Create a second API key for your server, in `signed` mode.
2. Add the minting endpoint (or mint inside your login and set a cookie).
3. Replace `apiKey` with `getTenantSession` on `DevicProvider`, and add
   `onSessionExpired`.
4. Drop `tenantId` / `subtenantId` props that only restated what the token
   proves — keep them only where they still match.
5. Revoke the old browser key. Until you do, it still works.
