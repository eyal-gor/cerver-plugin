# cerver

The Claude Code plugin for [cerver](https://cerver.ai) — a session-layer API for AI apps and agents.

## What you get

Installing this plugin loads the `cerver` skill into every Claude Code session. With it Claude can:

- **Remember** — recall what you (or sibling agents on the same account) did in past sessions; sessions are append-only transcripts that survive restarts.
- **Run code** — POST to a sandboxed compute (your local relay, Vercel sandboxes, E2B, etc.) without leaving Claude Code.
- **Fetch secrets** — pull API keys from your Infisical vault via `secret_fetch(name)` without ever printing or storing the raw value.

See [`skills/cerver/SKILL.md`](skills/cerver/SKILL.md) for the full skill body.

## Install

```bash
# Add this marketplace
/plugin marketplace add eyal-gor/cerver-plugin

# Install the plugin
/plugin install cerver
```

After that, every Claude Code session loads the `cerver` skill automatically. You'll also need a cerver account — `curl -fsSL https://cerver.ai/install.sh | bash` sets up the CLI, an API token, and (optionally) registers your machine as compute.

## Without the plugin

The skill body is hosted at https://cerver.ai/skill.md and can be:

- Dropped into any agent system prompt (Codex, Grok, etc.)
- Loaded with `WebFetch https://cerver.ai/skill.md` mid-session
- Installed manually: `curl https://cerver.ai/skill.md > ~/.claude/skills/cerver/SKILL.md`

The plugin path is the most ergonomic for Claude Code users; the URL is the cross-tool fallback.

## License

MIT
