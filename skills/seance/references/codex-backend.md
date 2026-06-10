# Codex Backend

Load this when the resolution order lands on the Codex store (`SESSION_BACKEND=codex`).

```
CODEX_SESSIONS_DIR=~/.codex/sessions
CODEX_SESSION_INDEX=~/.codex/session_index.jsonl
```

Codex stores a lightweight index at `~/.codex/session_index.jsonl` and full
rollouts at `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl`.

Each rollout starts with a `session_meta` line containing `payload.id`,
`payload.timestamp`, and `payload.cwd`. Filter for entries where `payload.cwd`
matches `$PWD` unless the user asks to search all sessions.

## Event Shapes

- User turns: `type == "event_msg"` and `payload.type == "user_message"`, with
  text in `payload.message`
- User transcript items: `type == "response_item"`, `payload.type ==
  "message"`, `payload.role == "user"`, with text blocks in
  `payload.content[].text`
- Assistant updates: `type == "event_msg"` and `payload.type ==
  "agent_message"`, with text in `payload.message`
- Assistant transcript items: `type == "response_item"`, `payload.type ==
  "message"`, `payload.role == "assistant"`, with text blocks in
  `payload.content[].text`
- Shell/tool calls and outputs: `type == "response_item"` with `payload.type`
  such as `function_call`, `function_call_output`, `custom_tool_call`, or
  `custom_tool_call_output`

## Quick List

```bash
# Scan cwd-matched rollout files and extract the first user message.
find ~/.codex/sessions -name '*.jsonl' -type f -print 2>/dev/null |
  while IFS= read -r f; do
    meta=$(head -1 "$f")
    cwd=$(printf '%s\n' "$meta" | jq -r '.payload.cwd // empty' 2>/dev/null)
    [ "$cwd" = "$PWD" ] || continue
    ts=$(printf '%s\n' "$meta" | jq -r '.payload.timestamp // empty' | cut -c1-10)
    id=$(printf '%s\n' "$meta" | jq -r '.payload.id // empty' | cut -c1-8)
    msgs=$(jq -s -r '[.[] | select(.type=="event_msg" and (.payload.type=="user_message" or .payload.type=="agent_message"))] | length' "$f")
    summary=$(jq -r 'select(.type=="event_msg" and .payload.type=="user_message") | .payload.message // empty' "$f" |
      head -1 | tr '\n' ' ' | cut -c1-55)
    printf '%s  %s  %3s msgs  %s\n' "$ts" "$id" "$msgs" "${summary:-No summary}"
  done |
  sort -r |
  head -10
```

## Search Recipe

```bash
query='<USER_QUERY>'
find ~/.codex/sessions -name '*.jsonl' -type f -print 2>/dev/null |
  while IFS= read -r f; do
    meta=$(head -1 "$f")
    cwd=$(printf '%s\n' "$meta" | jq -r '.payload.cwd // empty' 2>/dev/null)
    [ "$cwd" = "$PWD" ] || continue
    jq -r --arg q "$query" --arg f "$f" '
      def text_blocks:
        if (.payload.content | type) == "array" then
          [.payload.content[]? | .text? // empty] | join(" ")
        else empty end;
      select(
        (.type=="event_msg" and (.payload.type=="user_message" or .payload.type=="agent_message"))
        or (.type=="response_item" and .payload.type=="message" and (.payload.role=="user" or .payload.role=="assistant"))
      )
      | {
          role: (.payload.role // if .payload.type=="user_message" then "user" else "assistant" end),
          text: (.payload.message // text_blocks // "")
        }
      | select((.text | ascii_downcase) | contains($q | ascii_downcase))
      | "\($f)\t\(.role)\t\(.text | gsub("[\r\n\t]+"; " ") | .[0:240])"
    ' "$f"
  done
```

For a specific Codex session ID prefix, find the rollout by `session_meta`:

```bash
id_prefix='<ID_PREFIX>'
find ~/.codex/sessions -name '*.jsonl' -type f -print 2>/dev/null |
  while IFS= read -r f; do
    id=$(head -1 "$f" | jq -r '.payload.id // empty' 2>/dev/null)
    case "$id" in "$id_prefix"*) printf '%s\n' "$f";; esac
  done
```
