# Stacksmith agent skill

Helpers that teach AI coding agents to **author `.stacksmith.yml` files** and **control a
running Stacksmith app** through its MCP server. Two front-ends share one body of knowledge:

- `claude/stacksmith/` is a Claude **Agent Skill** (a `SKILL.md` plus `references/`).
- `codex/AGENTS.md` is a self-contained guidance file for **Codex CLI**.

Both rely on the Stacksmith MCP server for the actual control and inspection. The MCP server
ships inside the app at `/Applications/Stacksmith.app/Contents/Helpers/stacksmith-mcp`. The
skill is the knowledge layer on top of it.

## Install for Claude Code

Copy the skill folder into your skills directory.

```bash
# Global (available in every project):
cp -R claude/stacksmith ~/.claude/skills/stacksmith

# Or per-project (committed alongside a repo):
mkdir -p .claude/skills
cp -R claude/stacksmith .claude/skills/stacksmith
```

Then register the MCP server so the control and inspection tools exist. Add to your MCP config
(for example `~/.claude.json` or a project `.mcp.json`):

```json
{
  "mcpServers": {
    "stacksmith": {
      "command": "/Applications/Stacksmith.app/Contents/Helpers/stacksmith-mcp"
    }
  }
}
```

The skill triggers on its own whenever you mention Stacksmith, a `.stacksmith.yml` file, or ask
to define, run, or diagnose a local stack. You do not need to invoke it manually.

## Install for Codex CLI

Place `AGENTS.md` where Codex will read it, then register the MCP server.

```bash
# Per-project (Codex reads AGENTS.md from the working directory upward):
cp codex/AGENTS.md ./AGENTS.md

# Or global, for every project:
mkdir -p ~/.codex
cp codex/AGENTS.md ~/.codex/AGENTS.md

# Register the MCP server:
codex mcp add stacksmith -- \
  /Applications/Stacksmith.app/Contents/Helpers/stacksmith-mcp
codex mcp get stacksmith
```

If your project already has an `AGENTS.md`, append the contents of `codex/AGENTS.md` to it
rather than overwriting.

## Enable the tools in the app

Both setups need the in-app toggles, because the MCP server only does what the app permits:

1. Open Stacksmith and select a project.
2. Settings → MCP → enable **read-only inspection**.
3. Enable **component control** separately (off by default) to allow start, stop, restart, and
   run-job.

## What the agent can do

- Write and edit `.stacksmith.yml` from a description of your stack, using the exact schema
  (apps, workers, jobs, tunnels, links; health checks; dependency-aware ordering; port and URL
  interpolation).
- Validate the loaded config and explain issues.
- Inspect status, per-component logs, and the deterministic diagnosis.
- Start, stop, restart components, and run one-shot jobs, one component at a time, after
  confirming any change that touches local state.

## Compatibility note

This skill describes the schema and tools as of the bundled MCP server it ships with. Keep it
version-matched to the app: when the config schema or the MCP tools change, update
`claude/stacksmith/references/` and `codex/AGENTS.md` together (the Codex file duplicates the
essentials on purpose, so edit both).

## Publishing to users

The canonical source lives in the Stacksmith repo so it stays in sync with the MCP server. To
distribute it to users, publish a copy to the public
[stacksmith-releases](https://github.com/getstacksmith/stacksmith-releases) repo (or attach a
zip to a release):

```bash
# from a checkout of stacksmith-releases:
cp -R /path/to/Stacksmith/integrations/stacksmith-agent-skill ./agent-skill
git add agent-skill && git commit -m "Add Stacksmith agent skill" && git push
```

Then point users at it from the release notes or the website docs.
