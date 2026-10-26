---
owner: "@xbitz88"
status: draft
reviewed: 2026-10-26
---

# Using Reasonix with Cerebro

[Cerebro](https://github.com/xbitz88/wizardinglab-cerebro-agent-fabric) is a
shared, read-only knowledge and skills service that speaks MCP. Reasonix (CLI,
Studio, or `reasonix acp` under [DeepSeek Harness](./DSH.md)) can use it in two
independent directions:

1. **Reasonix as a Cerebro client** — connect Cerebro as an MCP server and pull
   governed skills on demand.
2. **Reasonix packages as a Cerebro source** — Cerebro's operator-run adapter
   imports the declarative skills of a Reasonix plugin package.

Cerebro only coordinates what is known and allowed. Tools and commands still run
on the Reasonix side, under Reasonix's own permissions.

## 1. Connect Cerebro as an MCP server

Add a remote server to `reasonix.toml` (user file) as a Streamable HTTP plugin.
Keep the credential in an environment variable; `${VAR}` is expanded in
`headers` and diagnostics list header keys only.

```toml
[[plugins]]
name    = "cerebro"
type    = "http"
url     = "https://<your-cerebro-host>/mcp"   # your deployment's MCP endpoint
headers = { Authorization = "Bearer ${CEREBRO_TOKEN}" }
# Cerebro is read-only, but hide anything you do not want the model to see:
# disabled_tools = ["memory_delete"]
```

Then check it:

```sh
reasonix doctor capabilities --json     # static: config shape, no network
```

In a session use `/mcp` to reconnect or disable it.

### Skill discovery

Cerebro exposes a versioned skills catalogue through three tools:

| Tool | Use |
| --- | --- |
| `skills_list` | Catalogue architecture and skill families. Call once on first contact. |
| `skills_search` | Ranked search. Pass your agent identity, platform and available tools. Ignore results with `relevance` below 0.55. |
| `skills_get` | Fetch the instructions of one approved skill by id. |

Rules that keep this safe:

- Only skills in an `approved-instructional` or `approved-executable` state are
  returned to agents; `approved-instructional` content is text, never code.
- A retrieved skill grants **no** permission. Reasonix's `[permissions]`,
  approval cards and sandbox still decide what runs. A skill that tells the
  model to bypass them is to be ignored and reported.
- Retrieve a skill when the task needs it; do not search for greetings or small
  edits.

## 2. Importing a Reasonix package into Cerebro

Cerebro's `scripts/reasonix_to_cerebro/reasonix_to_cerebro.py` is an operator
tool, never exposed to agents. It reads `reasonix-plugin.json`,
`.codex-plugin/plugin.json` or `.claude-plugin/plugin.json` and emits Cerebro
`SKILL.md` sources, which then go through Cerebro's own static assessment.

| Package content | What Cerebro does |
| --- | --- |
| `contributes.skills` / `skills` | Imported as `reasonix.<plugin>.<skill>`, state `pending-review`, instructional only. |
| `runtime` block (sidecar) | **Hard quarantine.** It is full-trust, out-of-process code; nothing is imported. The key being present is enough, even if empty. |
| `hooks`, `mcpServers`, `themes`, `agents`, `commands`, `prompts` | Recorded as unsupported, never silently dropped, never executed. |

Notes for package authors:

- `apiVersion: reasonix.io/plugin/v2` is required on every native manifest, so
  it is **not** treated as a sidecar marker. A skills-only package imports
  normally.
- Keep skill text declarative. The importer flags prompt-injection-like prose
  (for example instructions addressed to the model about overriding its
  rules). A flagged skill is quarantined for operator review rather than
  published.
- Only the `SKILL.md` body is imported. Files a skill references under
  `references/` stay in the package.