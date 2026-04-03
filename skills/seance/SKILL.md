---
name: seance
description: Commune with past local agent sessions. Use for recalling past work, exploring session history, finding previous sessions, or resurrecting abandoned work. Use this whenever the user asks about previous work, past sessions, what they were doing before, or wants to continue something they started earlier.
argument-hint: "[question or resurrect target]"
license: MIT
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

### Claude Code sessions

```
SESSION_DIR=!`echo "$HOME/.claude/projects/$(echo "$PWD" | sed 's|/|-|g')"`
INDEX_FILE=!`echo "$HOME/.claude/projects/$(echo "$PWD" | sed 's|/|-|g')/sessions-index.json"`
```

### Codex sessions

```
CODEX_SESSIONS_DIR=~/.codex/sessions
```

Codex stores sessions at `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl`. Each
file starts with a `session_meta` line containing `payload.cwd`. Filter for
entries where `payload.cwd` matches `$PWD`.

### Resolution order

1. If `$INDEX_FILE` exists and has entries → use Claude Code store
2. Else if `$CODEX_SESSIONS_DIR` exists → scan Codex JSONL files
3. Else → "No sessions found"

Set `SESSION_BACKEND` to `claude` or `codex` so downstream steps know which
format to parse.

---

## Quick List

When user runs `/seance` with no arguments:

**Claude Code backend:**
```bash
jq -r '.entries | sort_by(.modified) | reverse | .[:10][] |
  "\(.modified | .[0:10])  \(.sessionId | .[0:8])  \(.messageCount // 0 | tostring | if length < 3 then " " * (3 - length) + . else . end) msgs  \(.summary // .firstPrompt // "No summary" | .[0:55])"' \
  "$INDEX_FILE" 2>/dev/null || echo "No sessions found"
```

**Codex backend:**
```bash
# Scan recent Codex session files, filter by cwd, extract first user message
for f in $(ls -t ~/.codex/sessions/*/*/*.jsonl 2>/dev/null | head -20); do
  cwd=$(head -1 "$f" | jq -r '.payload.cwd // empty' 2>/dev/null)
  if [ "$cwd" = "$PWD" ]; then
    ts=$(head -1 "$f" | jq -r '.payload.timestamp // empty' | cut -c1-10)
    id=$(head -1 "$f" | jq -r '.payload.id // empty' | cut -c1-8)
    summary=$(grep -m1 '"type":"user_message"' "$f" | jq -r '.payload.content // "No summary"' | cut -c1-55)
    echo "$ts  $id  $summary"
  fi
done
```

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

**Claude Code:** Use a Task subagent with the Claude session index.
```
Task:
  subagent_type: "general-purpose"
  model: "haiku"
  prompt: [see oracle-prompt.md — substitute $INDEX_FILE, $SESSION_DIR, <USER_INPUT>, and <INFO|ACTION>]
```

**Codex / other hosts:** Run the oracle inline or as a lightweight subprocess.
Adapt the oracle prompt to search Codex JSONL files instead of the Claude
session index. The oracle's job is the same — scan sessions, find relevant
content, format the response — only the data source changes.

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
