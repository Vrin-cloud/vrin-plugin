# vrin-plugin

Ambient Vrin retrieval for Claude Code. When you ask a nuanced question about your product, architecture, or past decisions, Claude silently calls your Vrin knowledge base via the `vrin` CLI and grounds its answer in what you've ingested.

No MCP server registration. No per-turn token tax. Just a skill that tells Claude *when* to reach for Vrin.

## Install

**Prerequisite:** install the Vrin CLI and sign in once.

```bash
pip install vrin          # or: npm install -g @vrin/cli
vrin login                # one-time browser auth
```

Then inside Claude Code:

```
/plugin marketplace add vrin-cloud/vrin-plugin
/plugin install vrin
```

That's it. Try asking:

- *"Why did we choose X over Y for the retrieval pipeline?"*
- *"What's our position on the RAG-platform framing?"*
- *"How do our enterprise differentiators compare to \<competitor\>?"*

The `vrin-context` skill auto-fires on questions like these, runs `vrin query` via Bash, and grounds the answer in your ingested docs. Trivial prompts (typos, renames, generic coding help) don't trigger it.

## Commands

| Command | What it does |
|---|---|
| `/vrin-context <question>` | Force a Vrin retrieval on-demand — useful when auto-trigger misses |
| `/vrin-status` | Check auth + backend health |

## Optional: deterministic trigger rule

If you want Vrin to fire on *every* question about your domain (not just the ones Claude classifies as nuanced), paste this into your project's `CLAUDE.md`:

> *For any question about our product direction, architecture, past decisions, competitor positioning, or internal domain knowledge, call `vrin query "<question>" --json` via Bash first and ground your answer in the returned facts.*

Opt-in. Skip if you want Claude's own judgment to decide.

## Using Vrin in other agents

This plugin is Claude Code specific. For Cursor, Windsurf, Claude Desktop, or ChatGPT plugins, install the MCP server instead:

```bash
pip install vrin-mcp-server
```

See [docs.vrin.cloud/mcp](https://docs.vrin.cloud/mcp) for per-client setup.

## Uninstall

```
/plugin uninstall vrin
```

Leaves your `vrin login` credentials alone — manage those via `vrin logout`.

## License

MIT — see [LICENSE](./LICENSE).
