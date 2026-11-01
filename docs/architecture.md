# Architecture — the office-building model

## 1. One property first: "no model keys, ever"

The gateway is infrastructure. Infrastructure that holds LLM credentials is a
bounty target and an audit nightmare. So the rule is structural: **agents
hold their own provider keys**; the gateway holds identity (tokens), policy
(gates), and history (the ledger). A compromised gateway leaks *who asked
what* — never a key to your model account.

## 2. The building, part by part

| Building part | Package | Role |
|---|---|---|
| Sign-in desk | `internal/auth` | Gateway token or per-agent tokens, persisted in the agent DB, individually revocable |
| Staff directory | agent directory + `meshstate` | Every signed-in agent: name, id, live status — **addressable by name or capability** |
| Meeting rooms | `internal/session`, `chatwire` | Chat sessions; commands delivered to the owning agent's stream with prepare/start/deliver timeouts |
| Transport | `internal/transport` | WebSocket data planes: `/ws/v2/agent` (agents + desktops), `/ws/v2/terminal` (terminals) |
| Security gates | gates in the routing path | Capability checks **before** a tool call executes |
| Visitor log | audit trail | Who asked, which agent, which tool, what moved |
| Filing cabinet | `internal/db` | Local database: tokens, directory, sessions, approvals |

## 3. Sign-in and the two client shapes

Both shapes connect to **the same endpoint** (`/ws/v2/agent`) and are
indistinguishable server-side:

1. **Desktop** — a human-driven client with chat UI and tools
2. **agentd** — the headless worker: signs in, pulls remote chat turns,
   executes with real tool use, holds its own provider key and workdir

This is deliberate: *"From the gateway's point of view it is indistinguishable
from a desktop: a row in the local agent directory, addressed by name or
capability, behind the same gates and the same audit trail. No gateway
configuration changes to adopt it."*

## 4. One turn, end to end

```
Desktop: "how many P1 incidents this week?"
   │  artifact in (signed-in session)
   ▼
Gateway: gate check → deliver command to the owning agent's stream
   │
   ▼
Agent: picks a tool (its own LLM, its own key)
   │  tool call artifact
   ▼
Gateway: gate check on the tool call → allowed
   │
   ▼
Tool executes against real data → result artifact
   │
   ▼
Gateway: reply artifact back to the desktop
   └─ every hop is one row in the ledger
```

The gateway **never executes anything itself** and **never decodes a
payload it is not party to** — it is the referee, not a player.

## 5. The mesh — many buildings, one directory overlay

A single gateway is a closed building. The **mesh bridge** (disabled by
default) connects buildings so an edge can *advertise* what sits behind it:

- Your edge publishes its **local agent directory manifest** (name,
  description, capabilities) to a shared NATS event mesh
- Peers discover by capability; **the mesh routes to the edge, and the edge
  knows its own agents** — NATS never reaches past the front door
- Registry modes: `auto` (JetStream KV when the server offers it, else a
  broadcast window on plain NATS), `jetstream` (required), `broadcast`
- A **durable mailbox** (JetStream stream) holds skill invocations for an
  absent agent and delivers them when it returns

### Mesh verification model (defence in depth)

| Control | Flag family |
|---|---|
| Envelope signatures | `-mesh-verify-mode off\|prefer\|require` |
| Peer identity | sha256 fingerprints; trust-on-first-use (default on) or an explicit `-mesh-trusted-peers` list |
| Replay resistance | `-mesh-clock-skew` bounding envelope timestamps |
| Flooding | per-sender `-mesh-rate-limit-per-second` + burst cap; `-mesh-max-envelope-bytes` |
| Remote invocation | `-mesh-allow-remote-invoke`, `-mesh-require-verified-invoke` (default: refuse unverified callers), `-mesh-invoke-operations` allow-list |
| Version pinning | `-mesh-accepted-versions` |
| Escalation bounding | `-mesh-invoke-timeout`, `-mesh-task-max-runtime`, `-mesh-task-retention` |

Default posture: a peer cannot invoke anything behind your edge unless its
identity was **verified** (signed and trusted). Discovery is public;
invocation is not.

## 6. The boundary with neuralOS

A **neuralOS instance** is a verified data-answer runtime: caged probes,
refusal semantics, an audit ledger of its own. Served behind an agentd, it
becomes a mesh capability — peers delegate data questions and receive
either a verified answer or an honest refusal, with two correlated audit
trails (gateway ledger + instance audit). See [Integrate](integrate.md).

## 7. Failure posture

| Failure | Behaviour |
|---|---|
| NATS down / unconfigured | Not fatal; gateway serves; `/api/mesh/status` shows why |
| Agent disconnects | Directory row drops; mesh manifest refreshes; mailbox holds invocations if enabled |
| Provider key wrong | Fails in the agentd (its key, its problem) — gateway unaffected |
| Gateway restart | Sessions re-establish; agents re-sign-in; DB state persists |
