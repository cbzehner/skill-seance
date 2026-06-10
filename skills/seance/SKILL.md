---
name: seance
description: >-
  Search and summarize past local agent sessions, transcripts, and session logs.
  Use for recalling previous work, checking what happened in earlier sessions,
  exploring session history, finding prior decisions or commands, resurrecting
  abandoned work, or continuing something the user started before. Trigger when
  the user says "session logs", "past sessions", "what did we do before", "find
  the session", "resurrect", or asks to investigate prior agent behavior.
argument-hint: "[question or resurrect target]"
arguments:
  - query
license: MIT
effort: medium
allowed-tools: Bash Read Write Task
---

# Seance

Commune with your past sessions.

## Routing

| Input | Action |
|-------|--------|
| `/seance` | Quick list of recent sessions (instant, no LLM) |
| `/seance resurrect <target>` | Oracle with **action intent** → generates replan |
| `/seance <anything else>` | Oracle with **info intent** → explores and answers |

All paths except quick list use the same Oracle engine. The word "resurrect" signals action intent.

## When NOT to Use

- **Current session context** — if the answer is in this conversation, don't search old sessions
- **Git history questions** — use `git log` / `git blame` for code authorship and change history
- **Non-agent work** — seance only searches local agent session logs, not shell history or editor sessions

## Path Resolution

Detect which session store is available. Try both; use whichever has data.
If the current project has no local session directory and the user asks about
general working patterns, search all local agent sessions instead of returning
"No sessions found."

### Claude Code sessions

```
CLAUDE_PROJECT_DIR="$HOME/.claude/projects/$(printf '%s' "$PWD" | sed 's|/|-|g')"
INDEX_FILE="$CLAUDE_PROJECT_DIR/sessions-index.json"
CLAUDE_SESSIONS_ROOT="$HOME/.claude/projects"
```

Claude stores may be either indexed or raw JSONL:

- Indexed project store: `$CLAUDE_PROJECT_DIR/sessions-index.json`
- Raw project store: `$CLAUDE_PROJECT_DIR/*.jsonl`
- Raw global store: `$CLAUDE_SESSIONS_ROOT/**.jsonl`

Ignore `*/subagents/*.jsonl` for top-level session lists unless the user asks
about delegated agent work. Subagent logs are useful evidence, but including
them by default double-counts sessions.

### Codex sessions

Codex stores an index at `~/.codex/session_index.jsonl` and full rollouts at
`~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl`, cwd-tagged in each rollout's
`session_meta` line. When this backend is selected, read
[references/codex-backend.md](references/codex-backend.md) for event shapes,
the quick-list script, and search recipes.

### Resolution order

1. If `$INDEX_FILE` exists and has entries → use Claude Code store
2. Else if `$CLAUDE_PROJECT_DIR/*.jsonl` exists → scan current-project Claude JSONL files
3. Else if the user asks about broad patterns, history, or "how we work" and
   `$CLAUDE_SESSIONS_ROOT` exists → scan all top-level Claude JSONL files
4. Else if `$CODEX_SESSIONS_DIR` exists → scan Codex JSONL files
5. Else → "No sessions found"

Set `SESSION_BACKEND` to `claude_index`, `claude_jsonl`, `claude_global_jsonl`,
or `codex` so downstream steps know which format to parse.

---

## Quick List

When user runs `/seance` with no arguments:

**Claude Code backend:**
```bash
if [ -f "$INDEX_FILE" ]; then
  jq -r '.entries | sort_by(.modified) | reverse | .[:10][] |
    "\(.modified | .[0:10])  \(.sessionId | .[0:8])  \(.messageCount // 0 | tostring | if length < 3 then " " * (3 - length) + . else . end) msgs  \(.summary // .firstPrompt // "No summary" | .[0:55])"' \
    "$INDEX_FILE" 2>/dev/null
else
  find "${CLAUDE_PROJECT_DIR:-$HOME/.claude/projects}" -name '*.jsonl' -type f -print 2>/dev/null |
    grep -v '/subagents/' |
    while IFS= read -r f; do
      jq -R -r '
        fromjson?
        | select(.type=="user" and .parentUuid==null)
        | [
            (.timestamp // "" | .[0:10]),
            (.sessionId // "" | .[0:8]),
            (if (.message.content|type)=="string" then .message.content
             elif (.message.content|type)=="array" then ([.message.content[]? | .text? // empty] | join(" "))
             else "No summary" end | gsub("[\r\n\t]+"; " ") | .[0:55])
          ]
        | @tsv
      ' "$f" | head -1
    done |
    sort -r |
    head -10 |
    awk -F '\t' '{printf "%s  %s       msgs  %s\n", $1, $2, $3}'
fi
```

**Codex backend:** use the quick-list script in
[references/codex-backend.md](references/codex-backend.md).

Present as:
```
DATE        ID        MSGS  SUMMARY
2026-01-30  e0f76681   46   Fix Missing AssuranceConfig on Organization Creation
2026-01-29  656f96e1   11   Mercury child org tests hidden permissions
...

Try: "/seance what was I working on?" or "/seance resurrect <id>"
```

---

## Oracle

Oracle handles both exploration (`/seance <question>`) and resurrection (`/seance resurrect <target>`). Same engine, different intent.

### Intent Detection

| Input | Intent | Oracle Output |
|-------|--------|---------------|
| `/seance what was session X about?` | Info | Summary, explanation |
| `/seance why does auth keep breaking?` | Info | Analysis, patterns |
| `/seance resurrect abc123` | Action | Replan to continue work |
| `/seance resurrect #42` | Action | PR status + next steps |
| `/seance resurrect feature-branch` | Action | Branch analysis + completion plan |

**Resurrect targets:**
- Session ID (or prefix): `resurrect abc123`
- PR number: `resurrect #42` or `resurrect PR:42`
- Branch name: `resurrect feature-branch`

### Time Ambiguity Check

For info-intent questions, check if time scope is ambiguous:

| Question | Action |
|----------|--------|
| "What was I working on?" | Ask: "Recent, last month, or all time?" |
| "What was I working on last week?" | Proceed (explicit) |
| "Why does auth keep breaking?" | Search all (no time constraint) |

### Dispatch Oracle Subagent

**Claude Code indexed store:** Use a Task subagent with the Claude session index.
```
Task:
  subagent_type: "general-purpose"
  model: "haiku"
  prompt: [see oracle-prompt.md — substitute $INDEX_FILE, $SESSION_DIR, <USER_INPUT>, and <INFO|ACTION>]
```

**Claude raw JSONL / Codex / other hosts:** Run the oracle inline or as a
lightweight subprocess. Prefer structured `jq` extraction over raw grep so base
instructions, encrypted reasoning, token counts, and tool-output noise do not
dominate results.

**Claude JSONL search recipe:**
```bash
query='<USER_QUERY>'
root="${CLAUDE_PROJECT_DIR:-$HOME/.claude/projects}"
find "$root" -name '*.jsonl' -type f -print 2>/dev/null |
  grep -v '/subagents/' |
  while IFS= read -r f; do
    jq -R -r --arg q "$query" --arg f "$f" '
      fromjson?
      | select(.type=="user" or .type=="assistant")
      | {
          ts: (.timestamp // ""),
          id: (.sessionId // ""),
          role: (.message.role // .type),
          cwd: (.cwd // ""),
          text: (
            if (.message.content|type)=="string" then .message.content
            elif (.message.content|type)=="array" then ([.message.content[]? | .text? // empty] | join(" "))
            else "" end
          )
        }
      | select((.text | ascii_downcase) | contains($q | ascii_downcase))
      | "\($f)\t\(.ts[0:10])\t\(.id[0:8])\t\(.role)\t\(.text | gsub("[\r\n\t]+"; " ") | .[0:240])"
    ' "$f"
  done
```

**Codex search:** use the search and ID-prefix recipes in
[references/codex-backend.md](references/codex-backend.md).

The oracle's job is the same across backends: scan sessions, find relevant
content, format the response, cite dates/session IDs, and scrub secrets before
showing excerpts.

The oracle prompt template contains: available jq queries for session exploration, output JSON schemas for both info and action intents, and constraints on scope/citations. Read it before dispatching.

### Processing Response

**For INFO intent:**

```
## What I Found

<answer>

_Searched <total> sessions from <date_range>_

### Relevant Sessions

| Date | ID | Summary |
|------|-----|---------|
...

### Evidence

<citations>

### You might also ask...

<follow_up>
```

**For ACTION intent (resurrect):**

```
## Resurrecting: <target>

### Original Goal
<original_goal>

### What Was Done
<what_was_done as bullets>

### Current State
<status>
- Files: <files_touched>
- Blockers: <blockers>

### Plan to Continue
<plan as numbered steps>

Ready to start? Just say "go" or ask me to adjust the plan.
```

---

## Error Handling

**No sessions:**
```
No sessions found for this project.
Try running from a directory where you've used this agent before.
```

**Target not found:**
```
Couldn't find <target>.
Run `/seance` to see available sessions, or check the PR/branch exists.
```

**Invalid JSON from subagent:**
```
I explored your sessions but had trouble structuring the results:

<raw response>
```

---

## Secrets Scrubbing

When displaying session content:

```bash
sed -E \
  -e 's/([A-Za-z_]*(KEY|TOKEN|SECRET|PASSWORD|API_KEY)[^=]*[=:][[:space:]]*)[A-Za-z0-9_\-]{16,}/\1[REDACTED]/gi' \
  -e 's/Bearer [A-Za-z0-9_\-\.]{20,}/Bearer [REDACTED]/g' \
  -e 's/sk-[A-Za-z0-9]{32,}/sk-[REDACTED]/g' \
  -e 's/ghp_[A-Za-z0-9]{36}/ghp_[REDACTED]/g'
```
