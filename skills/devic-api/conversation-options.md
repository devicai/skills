# Conversation names and reasoning effort

These fields belong to the public assistant API. Use them in normal message
requests, both synchronous and `?async=true`; payload-template mode treats the
webhook body as opaque and does not extract these options from it.

## Name a new conversation

`POST /api/v1/assistants/:identifier/messages` accepts `chatName`:

```json
{
  "message": "Review this proposal",
  "chatName": "Proposal review"
}
```

Use `chatName`, not `name`, in the message request. The saved history returns it
as `name`. Assistant create/update `name` is the assistant's display name, a
different setting. `chatUid` identifies a conversation; it is not its name.

- The name is saved only on creation, including when the caller provides a new
  `chatUid`, and before execution or input-delay queueing.
- An existing conversation keeps its name even if another `chatName` is sent.
- Outer whitespace is trimmed. Omission, `null` or blank strings keep automatic
  naming. Non-string values return HTTP 400.

## Discover compatible effort levels

`GET /api/v1/models` returns the current catalog in `data`, grouped by provider.
Each provider has `models` and an informational `reasoningEfforts` union; each
model has its own `reasoningEfforts` list. Choose from the **model's** list.
An empty list means no named effort selection; omission/null keeps defaults.
This catalog describes capabilities, not account credentials or provider credit.

Native values across models are `none`, `minimal`, `low`, `medium`, `high`,
`xhigh`, `max`. Do not assume every model supports every value or use an
invented numeric budget. Unknown/unsupported explicit levels return HTTP 400.
When changing model, choose a compatible effort or clear it with null.

## Select the scope

| Request field | Scope | Omitted | JSON `null` |
| --- | --- | --- | --- |
| Assistant create/PATCH `reasoningEffort` | Saved assistant default | Preserve on PATCH | Clear the explicit assistant effort |
| Message `conversationReasoningEffort` | Saved conversation default | Preserve the conversation setting | Inherit the assistant |
| Message `reasoningEffort` | This user turn, including tool iterations/resumptions | Inherit defaults | Inherit defaults |

Resolution is **message → conversation → assistant → adapter default**.
`none` is an explicit model level, not inheritance. The API accepts JSON `null`,
not the strings `"default"` or `"null"`. A subsequent message without an override
uses the saved conversation/assistant defaults again.

```json
{
  "message": "Review this proposal carefully",
  "chatName": "Proposal review",
  "conversationReasoningEffort": "low",
  "reasoningEffort": "high"
}
```

For a model supporting both levels, this turn uses high and following turns use
low. Include `chatUid` to continue an existing conversation. Send
`conversationReasoningEffort: null` with a message to restore assistant inheritance.

Assistant reads return `reasoningEffort`. Conversation history returns
`reasoningEffort` (saved setting) and `effectiveReasoningEffort` (last recorded
model call). Individual responses record `messageTokenUsage.reasoningEffort`.
An absent effective value means the existing adapter default was used.

Agent configuration can set `assistantSpecialization.reasoningEffort`; this does
not imply that the assistant message fields are accepted by agent-run endpoints.

Workspace assistant configuration separately accepts `reasoningEffort` on its
existing session-authenticated create/PATCH routes under
`/workspace-api/workspaces/:workspaceId/assistants`. Null removes the local
override, restoring source inheritance for linked assistants. Keep workspace
membership and revision checks; account API keys do not grant workspace access.

Check support in the target environment. Source merges, npm releases and backend
deployments are separate; do not infer production availability from a DEV check.
