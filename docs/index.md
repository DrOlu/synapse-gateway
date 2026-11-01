# synapse-gateway

**The verified agent mesh runtime.** A single native binary that turns any
server into an **office building for AI agents**: agents sign in with tokens,
find each other in a **directory addressed by name or capability**, and work
behind **security gates** with a **full audit ledger**. It ships with
**agentd**, a headless agent worker that signs into the building and does
real work — indistinguishable from a desktop client to the gateway.

One property from the source, worth putting first: the gateway keeps its
**"no model keys, ever"** property. LLM credentials belong to the agents that
execute — never to the infrastructure they connect through.

## Quick Start

```bash
# one command, everything included
pip install synapse-gateway

# open the building
synapse-gateway
# → serving on http://127.0.0.1:3000  (/healthz, /api/*, /ws/v2/agent)

# hire the headless employee
synapse-agentd -gateway ws://127.0.0.1:3000/ws/v2/agent \
  -agent-id agentd-server-1 \
  -provider-url https://api.openai.com/v1 -provider-key sk-... \
  -provider-model gpt-5 -workdir /srv/agentd-work
```

Also available: `npm i -g @hyperspaceng/synapse-gateway`, or standalone
binaries (linux amd64/arm64, darwin amd64/arm64, windows amd64) from
[ReactorPro releases](https://github.com/DrOlu/ReactorPro/releases).

## Why it's different

| Typical agent infra | synapse-gateway |
|---|---|
| Agents hardcode each other's addresses | Agents are **discovered in a directory** by name or capability |
| Model keys live in the orchestrator | **"No model keys, ever"** — executors hold their own |
| Every tool call trusted | Calls pass **gates** before execution |
| Failures and data movement invisible | Everything lands in an **audit ledger** |
| Needs a message broker to start | **Zero dependencies** — WebSocket + local DB; the NATS mesh bridge is optional and off by default |
| Headless = configuration marathon | agentd is **indistinguishable from a desktop** to the gateway — no config changes to adopt it |

## What's new in 1.7.12

- Distribution fixed: gateway releases now publish to **PyPI and npm
  automatically** (the `released` event trigger gap is closed, with a daily
  drift-guard so package skew can never hide again)
- The **fallback `py3-none-any` wheel** now actually reaches PyPI — platforms
  without a native wheel (musl/Alpine, BSD, 32-bit) install cleanly and
  print guidance instead of failing
- Checksum verification hardened: `SHA256SUMS` parsing covers binary-mode
  `*file` entries and filenames with spaces; an asset missing from the sums
  is a hard failure, never a silent skip
- npm publishing token corrected and verified (`SYNAPSE_NPM_TOKEN`)

## The two commands

| Command | What it is |
|---|---|
| `synapse-gateway` | The **building**: sign-in desk (token auth), staff directory, meeting rooms (chat sessions), security gates, visitor log (audit), local DB |
| `synapse-agentd` | The **employee**: signs into a gateway over the same WebSocket the desktop uses, serves remote chat turns with real tool use — several at a time — holding its own LLM key and its own work directory |

## Explore

- [Quick Start](usage.md) — install, run, attach, secure
- [Architecture](architecture.md) — the office-building model, gate pipeline, mesh routing
- [Reference](reference.md) — every flag and endpoint, both commands
- [Integrate](integrate.md) — neuralOS instances, MCP, webhooks, mesh pairing
- [Cookbook](cookbook.md) — worked recipes: single host, fleet, cross-org mesh
- [Changelog](changelog.md) — release history
