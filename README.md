> **Moved:** this skill now lives in [cbzehner/skills](https://github.com/cbzehner/skills) under `skills/seance/`. This repo is archived and read-only.

# Seance

Search and summarize past local agent sessions. Use it to recover prior decisions, abandoned work, commands that were run, or context that only exists in transcripts.

## Use It For

- Finding what happened in earlier sessions
- Resurrecting work that stopped midstream
- Checking old agent logs before repeating an investigation

## Install

Clone the repo and run the installer:

```bash
git clone https://github.com/cbzehner/skill-seance.git
cd skill-seance
./install.sh all
```

Install targets:

- `./install.sh claude` installs to `~/.claude/skills/seance`
- `./install.sh codex` installs to `~/.codex/skills/seance`
- `./install.sh agents` installs to `~/.agents/skills/seance`
- `./install.sh opencode` installs to `~/.config/opencode/skills/seance`
- `./install.sh all --copy` copies files instead of symlinking

Manual install works too: symlink or copy `skills/seance` into your agent's skills directory.

## Agent Support

This repo uses the plain `skills/seance/SKILL.md` layout. Claude Code and Codex also get small plugin manifests at `.claude-plugin/plugin.json` and `.codex-plugin/plugin.json`.

Other agents can read the same `SKILL.md` file. If a host does not support a frontmatter field or tool name, ignore that field and follow the workflow text.

## Layout

```text
.claude-plugin/plugin.json
.codex-plugin/plugin.json
install.sh
skills/seance/SKILL.md
README.md
LICENSE
```

## Public Notes

These repos are public. Keep private repo names, secrets, customer data, raw logs, cookies, and absolute filesystem paths out of examples.

## License

MIT