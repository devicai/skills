# Conversation names and reasoning effort

Check `devic --version` and `devic assistants chat --help` when an option is
unknown. `--chat-name` was added in CLI 0.26.0; effort flags and model discovery
were added in 0.27.0. A merged source change does not establish npm publication
or backend support in the target environment.

## Name a conversation

```bash
devic assistants chat "$ASSISTANT_ID" -m "Hello" --chat-name "Customer onboarding"
```

This sends `chatName` when creating a conversation, including with a caller's new
`--chat-uid`. Existing conversations keep their name. Outer whitespace is trimmed;
omitted or blank names keep automatic naming. The saved history field is `name`.
`assistants create/update --name` names the assistant, not the conversation.

## Find model-compatible effort levels

```bash
devic models list
devic models list --provider open-ai --model gpt-6-luna
```

Filters match exact provider/model identifiers. Use each model's `reasoningEfforts`
list; the provider union is informational. An empty model list of efforts means
default behavior only. Supported native values across models are `none`,
`minimal`, `low`, `medium`, `high`, `xhigh`, `max`; model support varies.

## Set and reset effort

```bash
# Saved assistant default (also available on assistants create):
devic assistants update "$ASSISTANT_ID" --reasoning-effort high

# Save low for the conversation, use high only for this turn:
devic assistants chat "$ASSISTANT_ID" -m "Review carefully" \
  --chat-name "Proposal review" \
  --conversation-reasoning-effort low --reasoning-effort high

# Restore assistant inheritance on an existing conversation:
devic assistants chat "$ASSISTANT_ID" --chat-uid "$CHAT_UID" -m "Continue" \
  --conversation-reasoning-effort default

# Clear the saved assistant effort:
devic assistants update "$ASSISTANT_ID" --reasoning-effort default
```

Resolution is message → conversation → assistant → adapter default. Omitting
flags preserves saved settings. `default` (also `null`) sends JSON null to restore
inheritance; `none` is an explicit model level. Unsupported model/level combinations
return an API error; select a compatible level or reset it when changing model.

The flags work with default async polling, `--no-wait` (sync), and `--detach`
(async without polling). `--from-json` uses `chatName`, `reasoningEffort` and
`conversationReasoningEffort`. Explicit flags override their JSON fields, even
for resets. JSON effort resets use native null, not `"default"` or `"null"`.

`assistants get` shows the saved effort; `assistants chats get` shows the saved
conversation effort and last effective effort. JSON history also contains
per-response `messageTokenUsage.reasoningEffort`.

These are the public assistant commands. Do not invent `--workspace` or apply
the message flags to agent-run commands. Agent assistant defaults may be set
using `assistantSpecialization.reasoningEffort` in agent JSON configuration.
