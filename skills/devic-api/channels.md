# Channels API

Attach the places your customers already write from — a WhatsApp number, a
Slack workspace — to the tenants you already have.

**Base path:** `/api/v1/channels`

## Why it exists

A message arriving from a channel names something the platform assigned: a
Slack workspace id, a WhatsApp contact id. None of those mean anything in your
product. You know your customer as `acme-corp`; Meta knows them as
`BSUID-8f21…`, and nothing connects the two.

This API is that connection. You issue a link naming one of your tenants, the
person opens it and connects themselves, and from then on every message from
that identity is attributed to that tenant — same `tenantId` as the rest of
your usage, costs and limits.

**The tenant is frozen into the link when it is made and never read from a
request.** That single fact is what makes the link safe to send through an
email, a ticket or a chat: whoever holds it decides *whether* to connect, never
*to whom*.

## The two shapes of consent

The channels do not connect the same way, and the difference is not cosmetic.

| | Slack | WhatsApp |
|---|---|---|
| What the person does | installs the bot in their workspace | sends one message |
| What proves it is them | the OAuth round trip | the message itself — WhatsApp authenticates the sender |
| What the link returns | `url` to a consent page | `url` to a page with a QR, a button and a code |
| What binds | the workspace | that person's WhatsApp, on that number |

On WhatsApp there is nothing to install and no consent screen. The link opens a
chat with a short code already written in the box; when the person sends it,
**that message is the consent**, and the assistant answers that same message
already knowing who they are.

---

## Issuing a link

```
POST /api/v1/channels/invites
```

Call it from your backend, with a server-side key. A credential that can issue
links can issue one naming any tenant of your account, which is why this route
is outside the `devic-ui` key preset and outside what a tenant session may
call.

### Request

| Field | Type | Required | Description |
|---|---|---|---|
| `tenantId` | string | ✅ | The customer this link belongs to. Frozen at creation |
| `channelAppId` | string | ✅ | The presence it connects to — a Slack bot, a WhatsApp app |
| `platform` | `slack` \| `whatsapp` | — | Defaults to `slack` |
| `accountId` | string | WhatsApp | Which of the app's numbers the person writes to. The code only works on that number |
| `subtenantId` | string | — | The person inside that customer. Without it the channel contact stands in for them — stable, and meaningless in your own product |
| `greeting` | string | — | WhatsApp: what the pre-written message says in front of the code. The person can edit it before sending, so treat it as a suggestion |
| `expiresInHours` | number | — | Default `168` (7 days), maximum `720` (30 days) |
| `maxBindings` | number | — | How many distinct identities may connect through it. `1` unless you say otherwise; more for a company whose staff each write in from their own phone. Max `20` |
| `requireApproval` | boolean | — | What connects lands as `pending` until somebody approves it in the console |
| `purpose` | string | — | One line shown on the page the person opens, under the title. It is **not** private, and it is never written into the WhatsApp message itself |

```bash
curl -X POST "https://api.devic.ai/api/v1/channels/invites" \
  -H "Authorization: Bearer devic-your-server-side-key" \
  -H "Content-Type: application/json" \
  -d '{
    "platform": "whatsapp",
    "tenantId": "acme-corp",
    "subtenantId": "user-4812",
    "channelAppId": "6aa188e756c1f65dc1ee815d",
    "accountId": "6aa189b0726ebfe037ce45ec",
    "greeting": "Hi, I would like to get set up",
    "expiresInHours": 72
  }'
```

### Response

```json
{
  "success": true,
  "data": {
    "id": "6aa2f1c0…",
    "platform": "whatsapp",
    "tenantId": "acme-corp",
    "expiresAtMs": 1789300000000,
    "url": "https://app.devic.ai/connect/workspace/9f3c…",
    "waUrl": "https://wa.me/19807055175?text=Hi%2C%20I%20would%20like%20to%20get%20set%20up%20DV-7K3M-9QX2",
    "joinCode": "DV-7K3M-9QX2"
  }
}
```

**All three are returned once, here.** Only their hashes are stored, so a link
that is lost is reissued, never recovered — and reading the link back later
answers without them.

Which one to hand over:

- **`url`** unless you have a reason not to. It is a page showing the QR, the
  button that starts the chat and the code, with a line saying what sending the
  message does. The QR matters more than it sounds: invitations are read on a
  laptop and answered from a phone.
- **`waUrl`** when you know they are already on the phone — one tap, no page in
  between.
- **`joinCode`** for somebody who already has your number saved and would
  rather just type it. It is found anywhere inside whatever they write, so
  "Hi, they gave me DV-7K3M-9QX2, thanks" works.

---

## Reading your links

```
GET /api/v1/channels/invites?tenantId=acme-corp&status=pending
```

| Query | Description |
|---|---|
| `tenantId` | Only this customer's |
| `appId` | Only this presence's |
| `status` | `pending`, `completed`, `expired`, `revoked` |

Without their tokens or codes, which are not stored. **`bindings` is how you
tell whether a customer ever used theirs** — it lists what connected, and when.

A link is `completed` once it has bound everything `maxBindings` allowed.

---

## Withdrawing a link

```
POST /api/v1/channels/invites/{id}/revoke
```

**What it has already bound stays bound.** Revoking withdraws the link, not the
mapping it made — unmapping is a separate and deliberate act, done from the
console.

That asymmetry is on purpose, and it is the same reason there is no route
behind the public page that disconnects anything: "somebody forwarded the link
and connected" is annoying and reversible; "somebody forwarded the link and
left the customer disconnected" is an incident.

---

## Who reaches you through which channel

```
GET /api/v1/channels/identities?tenantId=acme-corp
```

One row per identity that has been mapped or merely seen.

| Query | Description |
|---|---|
| `tenantId` | Only this customer's |
| `platform` | `slack`, `whatsapp`, `email` |
| `status` | see below |

| Status | Meaning |
|---|---|
| `active` | Mapped and answering |
| `pending` | Bound through a link that asked for approval; does not resolve yet |
| `unbound` | Seen in traffic, mapped to nobody. Exists so you can see who is talking without an owner |
| `blocked` | Explicitly refused. Its messages are dropped |

A row's `key` is `platform:identifier[:subIdentifier]` — `whatsapp:<number>` for
a whole line, `whatsapp:<number>:<contact>` for one person on it. **The more
specific row wins**, so a line dedicated to one customer and an individual
mapped inside it can both exist and mean what they look like they mean.

---

## What happens on the way in

Once an identity is mapped, an arriving message resolves like this:

| Mapping | `tenantId` | `subtenantId` |
|---|---|---|
| The contact is mapped | your tenant | your subtenant, or the contact when the link named none |
| Only the number is mapped | your tenant | the contact — it is the only thing telling the staff apart |
| Nothing is mapped | the contact itself | *(empty)* |

The third row is the default and is right for an open support line, where
whoever writes *is* the customer. An entity that only answers people you have
registered can be set to say so instead — the person gets one fixed sentence,
in the language of their own calling code, and no run starts. That is a setting
on the entity's channel binding, in the console.

---

## Getting `channelAppId` and `accountId`

Both come from the console — **Configuration → Channels** — where the presence
is created and the number connected or bought. Connecting a WhatsApp number
goes through Meta's own signup, so it is not something an API call can do for
you; you do it once and reuse the ids.

They are also on every row of `GET /api/v1/channels/identities` for a channel
that has already seen traffic.

---

## Errors

| Code | Meaning |
|---|---|
| `400` | A WhatsApp link with no `accountId`, or a number that has not finished connecting |
| `404` | `CHANNEL_APP_NOT_FOUND`, or `WHATSAPP_NUMBER_NOT_FOUND` for a number that is not on that app |
| `401` | Missing key, or a tenant session trying to call this |

## Related

- [tenants.md](tenants.md) — the tenants and subtenants these links attach to
- [tenant-sessions.md](tenant-sessions.md) — why a route that can name any tenant stays on your server
