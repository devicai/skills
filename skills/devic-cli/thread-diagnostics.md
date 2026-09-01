# Thread Diagnostics with jq

Recipes for inspecting agent execution threads — tool calls, errors, message search — without dumping the full thread into your context.

## ⚠️ Read this first

`devic agents threads get <threadId>` returns the **entire** thread, including every assistant message, tool call, and tool response. A non-trivial thread can be tens of thousands of tokens.

**Never pipe the raw output into your reasoning.** Always filter first:

```bash
# ❌ Bad — dumps the whole thread into context
devic agents threads get $TID -o json

# ✅ Good — filter on stdout, only the result enters context
devic agents threads get $TID -o json | jq '<filter>'
```

The native `--grep <pattern>` flag (on `threads get`) also filters client-side and is fine for quick lookups, but jq lets you extract exactly the shape you need.

For listing threads, use `--omit-content` to skip the bulk:

```bash
devic agents threads list $AID --omit-content --limit 50
```

---

## Thread JSON cheatsheet

`devic agents threads get` returns an `AgentThread` document. The structure the recipes below assume:

```jsonc
{
  "_id": "...",
  "agentId": "...",
  "state": "completed",                  // queued|processing|completed|failed|terminated|paused|paused_for_approval|approval_rejected|waiting_for_response|paused_for_resume|handed_off|guardrail_trigger
  "finishReason": "MAX_TOOL_CALLS_REACHED: 100 tool calls",
  "pausedReason": "Waiting for 1 async tool response(s)",
  "creationTimestampMs": 0,
  "finishTimestampMs": 0,
  "tokenUsage": { /* aggregated usage with cost */ },
  "pendingHandOffSubThreadIds": ["..."],
  "pendingAsyncToolCalls": [             // present when paused on async tools
    { "toolCallId": "call_...", "toolName": "...", "callbackUrl": "..." }
  ],
  "emptyContentRetries": { "attempts": 2, "exhausted": false },
  "guardrailResults": { /* present when state=guardrail_trigger */ },

  "threadContent": [
    {
      "uid": "msg-uuid",
      "role": "user" | "assistant" | "tool" | "developer",
      "timestamp": 0,
      "summary": "AI-generated short summary",        // optional
      "latencyMs": 0,                                  // LLM or tool latency
      "content": {
        "message": "the textual content",
        "data": { /* arbitrary structured payload */ },
        "files": [ ... ]
      },

      // role=assistant — populated when the LLM issued tool calls
      "tool_calls": [
        {
          "id": "call_abc123",
          "type": "function",
          "function": {
            "name": "search_contacts",
            "arguments": "{\"query\":\"acme\"}"        // JSON string, parse with `fromjson`
          }
        }
      ],

      // role=tool — links this response back to the assistant tool_call
      "tool_call_id": "call_abc123",

      // role=tool — present when the response was served from the tool-call cache
      "cacheInfo": { "cached": true, "cachedAt": 0, "ageMs": 0, "ttlSeconds": 60, "remainingMs": 0 },

      "messageTokenUsage": { /* per-message tokens */ },
      "messageCost": { /* per-message cost breakdown */ }
    }
  ]
}
```

Key relationships:

- A `role: "assistant"` message with `tool_calls` is **the request**.
- For every `tool_calls[].id` there is normally one matching `role: "tool"` message later in `threadContent` with `tool_call_id` set to that id (the response). If missing, the call is **pending**.
- `function.arguments` is always a JSON string — use `fromjson?` to parse, falling back to the raw string if the LLM returned malformed JSON.
- A subthread has `isSubthread: true`, `parentThreadId`, `parentAgentId`, `subThreadToolCallId`. The parent's `pendingHandOffSubThreadIds` lists which subthreads it is waiting on.

---

## Quick start: TID shortcut

All recipes assume the thread id is in `$TID`:

```bash
TID="<thread-id>"
```

For some recipes you'll also want the tool-call id in `$TCID` or a pattern in `$PATTERN`.

---

## Recipes

### List all tool calls with their arguments

The most common starting point when diagnosing what an agent did.

```bash
devic agents threads get $TID -o json | jq '
  [ .threadContent[]
    | select(.role=="assistant")
    | .tool_calls // []
    | .[]
    | { id, name: .function.name, args: (.function.arguments | fromjson? // .function.arguments) }
  ]
'
```

### Show one tool call's response by id

After picking an `id` from the previous recipe:

```bash
TCID="call_abc123"
devic agents threads get $TID -o json | jq --arg id "$TCID" '
  .threadContent[]
  | select(.role=="tool" and .tool_call_id==$id)
  | { uid, latencyMs, cached: .cacheInfo.cached, summary, content }
'
```

### Pair each tool call with its response

Single-pass view: every call alongside its result. Useful for spotting which call produced an error.

```bash
devic agents threads get $TID -o json | jq '
  .threadContent as $c
  | [ $c[]
      | select(.role=="assistant")
      | .tool_calls[]? as $tc
      | {
          id: $tc.id,
          name: $tc.function.name,
          args: ($tc.function.arguments | fromjson? // $tc.function.arguments),
          response: ( $c[] | select(.role=="tool" and .tool_call_id==$tc.id) | .content.message ) // null,
          responseSummary: ( $c[] | select(.role=="tool" and .tool_call_id==$tc.id) | .summary ) // null,
          latencyMs: ( $c[] | select(.role=="tool" and .tool_call_id==$tc.id) | .latencyMs ) // null
        }
    ]
'
```

### Filter tool calls by tool name

```bash
TOOL="send_email"
devic agents threads get $TID -o json | jq --arg name "$TOOL" '
  [ .threadContent[].tool_calls[]?
    | select(.function.name==$name)
    | { id, args: (.function.arguments | fromjson? // .function.arguments) }
  ]
'
```

### Find tool calls whose response looks like an error

Heuristic: matches common error keywords in the tool response. Adjust the regex when your tools use their own error vocabulary.

```bash
devic agents threads get $TID -o json | jq '
  [ .threadContent[]
    | select(.role=="tool")
    | select((.content.message // "") | test("error|failed|exception|forbidden|unauthori[sz]ed|not found|timeout|denied"; "i"))
    | { tool_call_id, latencyMs, snippet: (.content.message // "")[:200] }
  ]
'
```

Variant — only `content.data.error` fields (when tools return structured errors):

```bash
devic agents threads get $TID -o json | jq '
  [ .threadContent[]
    | select(.role=="tool" and (.content.data.error // .content.data.success==false))
    | { tool_call_id, error: .content.data.error // .content.data, latencyMs }
  ]
'
```

### Find tool calls with no response yet (pending)

```bash
devic agents threads get $TID -o json | jq '
  .threadContent as $c
  | [ $c[].tool_calls[]?
      | . as $tc
      | select( [ $c[] | select(.role=="tool" and .tool_call_id==$tc.id) ] | length == 0 )
      | { id: $tc.id, name: $tc.function.name }
    ]
'
```

### Regex search across all message content + tool arguments

This is the swap for "download the whole thread and grep it locally". Pattern is a JS-style regex (jq uses Oniguruma — close enough for most cases).

```bash
PATTERN="user@example\\.com"
devic agents threads get $TID -o json | jq --arg re "$PATTERN" '
  [ .threadContent[]
    | . as $m
    | (
        (.content.message // "" | test($re; "i"))
        or (.summary // "" | test($re; "i"))
        or ((.tool_calls // []) | any(.function.arguments // "" | test($re; "i")))
        or ((.tool_call_id // "") | test($re; "i"))
      )
      as $hit
    | select($hit)
    | {
        uid, role, timestamp,
        snippet: (
          .content.message
          // ( .tool_calls // [] | map(.function.arguments) | join(" | ") )
          // ""
        )[:200]
      }
  ]
'
```

For a case-sensitive match drop the `"i"` flag. The native `devic agents threads get $TID --grep "..."` also works for literal substring matches and prints the matching messages directly — use whichever is more ergonomic for the case at hand.

### Timeline view (compact one-line-per-message)

Useful as a 30-second overview of what the agent did. Combine with `column -t -s ';'` for alignment if your terminal supports it.

```bash
devic agents threads get $TID -o json | jq -r '
  .threadContent[]
  | [ .timestamp,
      .role,
      ( if .role=="assistant" and (.tool_calls // [] | length) > 0
        then "→ " + (.tool_calls | map(.function.name) | join(", "))
        elif .role=="tool" then "← " + (.tool_call_id // "")
        else (.content.message // "")[:60] end ),
      (.latencyMs // "")
    ]
  | @tsv
'
```

### Token usage per assistant turn

Spot which turn cost the most.

```bash
devic agents threads get $TID -o json | jq '
  [ .threadContent[]
    | select(.role=="assistant")
    | { uid, t: .timestamp, tokens: .messageTokenUsage, cost: .messageCost }
  ]
'
```

### Diagnose summary in one query

`state`, `finishReason`, error counts, last assistant message — the equivalent of a "what went wrong?" first look.

```bash
devic agents threads get $TID -o json | jq '
  {
    state, finishReason, pausedReason,
    emptyContentRetries, guardrailResults,
    pendingAsyncToolCalls,
    pendingHandOffSubThreadIds,
    totalMessages: (.threadContent | length),
    totalToolCalls: ( [ .threadContent[].tool_calls // [] | length ] | add // 0 ),
    pendingToolCalls: (
      .threadContent as $c
      | [ $c[].tool_calls[]? | . as $tc
          | select( [ $c[] | select(.role=="tool" and .tool_call_id==$tc.id) ] | length == 0 )
        ] | length
    ),
    failedToolCalls: (
      [ .threadContent[]
        | select(.role=="tool")
        | select((.content.message // "") | test("error|failed|exception|forbidden|unauthori[sz]ed|not found|timeout|denied"; "i"))
        | { tool_call_id, snippet: (.content.message // "")[:160] }
      ]
    ),
    lastAssistantMessage: (
      .threadContent
      | map(select(.role=="assistant" and ((.content.message // "") | length) > 0))
      | last
      | .content.message[:500]
    ),
    lastUserMessage: (
      .threadContent
      | map(select((.role=="user" or .role=="developer") and ((.content.message // "") | length) > 0))
      | last
      | .content.message[:500]
    )
  }
'
```

### Bulk grep across many threads

Find which threads of an agent contain a string, without paying the bytes for content you don't need:

```bash
PATTERN="target@email.com"
devic agents threads list $AID --omit-content -o json | jq -r '.threads[]._id' | while read t; do
  count=$(devic agents threads get "$t" -o json \
    | jq --arg re "$PATTERN" '[.threadContent[] | select((.content.message // "") | test($re; "i"))] | length')
  [ "$count" -gt 0 ] && echo "$t: $count match(es)"
done
```

### Inspect a subthread chain

When a thread is `handed_off`, follow into its subthreads:

```bash
devic agents threads get $TID -o json | jq '{ state, pending: .pendingHandOffSubThreadIds, history: .handOffSubThreadIds }'

# Then for each subthread:
for sub in $(devic agents threads get $TID -o json | jq -r '.pendingHandOffSubThreadIds[]?'); do
  echo "--- $sub ---"
  devic agents threads get "$sub" -o json | jq '{ state, finishReason, totalMessages: (.threadContent | length) }'
done
```

---

## Diagnostic workflows

Step-by-step playbooks composing the recipes above. Follow these in order — each step narrows the search.

### Workflow A — A thread failed and I don't know why

1. **Get the diagnose summary** (recipe: *Diagnose summary in one query*). Read `state`, `finishReason`, and the `failedToolCalls` array first.
2. If `finishReason` starts with `MAX_TOOL_CALLS_REACHED`: jump to **Workflow C** (tool loop).
3. If `state` is `guardrail_trigger`: inspect `guardrailResults` and the last assistant message — the model tripped a safety rule.
4. If `failedToolCalls` is non-empty: grab the first `tool_call_id` and use *Show one tool call's response by id* to read the full error. Then *List all tool calls with their arguments* and search for the same tool name to see if the failure was a one-off or systemic.
5. If `failedToolCalls` is empty but `state=failed`: read `lastAssistantMessage` from the diagnose query. The model usually explains what blocked it in plain text.
6. Still unclear? Run the **Timeline view** to scan the full sequence — look for unexpected loops or skipped steps.

### Workflow B — The agent did something it shouldn't have

When the output is wrong but the thread completed cleanly.

1. **Timeline view** to see the full sequence at a glance.
2. *List all tool calls with their arguments* to see what the agent actually decided to invoke. Compare against what you expected.
3. *Regex search across all message content + tool arguments* with the suspicious value (an email, an id, a domain) to find every place it appears in the thread. Often the agent picked up the value from an unexpected source.
4. Read the final `assistantMessage`s (filter by role) to see the agent's stated reasoning before the wrong action.

### Workflow C — Suspected tool loop / agent stuck

1. *List all tool calls with their arguments* and look at the `name` distribution: `... | jq 'group_by(.name) | map({name: .[0].name, count: length}) | sort_by(-.count)'`.
2. If one tool dominates, check whether the same arguments are repeating: `... | jq '[.[] | select(.name=="<tool>") | .args]'`.
3. If yes, the tool's response is probably not advancing the agent's state. *Show one tool call's response by id* on the latest few calls — look for empty/identical responses.

### Workflow D — Looking for a specific value across threads

When you don't know which thread contains the data.

1. `devic agents threads list $AID --omit-content -o json | jq '.threads[] | {id: ._id, state, t: .creationTimestampMs}'` to scan candidate threads cheaply.
2. *Bulk grep across many threads* with your pattern to narrow down to the matching threads.
3. On each match, *Regex search across all message content + tool arguments* to see the exact context.

### Workflow E — Subthread / handoff debugging

1. Diagnose the parent thread first. If `state=handed_off`, read `pendingHandOffSubThreadIds`.
2. *Inspect a subthread chain* to see the state of each child.
3. Diagnose the failing subthread using **Workflow A** — they have the same shape as parent threads.
4. To trace which assistant message triggered the handoff: search the parent's `tool_calls` for the `hand_off_subagent` tool name.

---

## When jq is not enough

Reach for a real script (Node, Python) when:

- You need to cross-reference multiple threads with non-trivial joins.
- You need to compute aggregate statistics across hundreds of threads.
- The thread is large enough that jq is slow — at that point downloading once to a file and processing locally is faster:

  ```bash
  devic agents threads get $TID -o json > /tmp/thread.json
  jq '...' /tmp/thread.json
  ```

For everything else, jq is faster to compose than writing a script and keeps the output out of your context window until you actually need it.
