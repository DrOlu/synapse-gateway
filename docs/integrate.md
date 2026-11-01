# Integrate

The gateway is the switchboard; this page is about what plugs into it.

## Pair with a neuralOS instance (verified data answers)

A **neuralOS instance** is a verified data-answer runtime: the model picks
from a closed menu of probes, arguments are caged, weak matches are refused,
and every query is audited. Served behind an agentd, it becomes a mesh
capability — peers delegate data questions and get verified answers *or an
honest refusal*.

```bash
# 1. serve the instance with token auth
neuralosd serve --instance-dir ./finacle-users --port 8877 \
                --auth-token "$NEURALOSD_TOKEN"

# 2. expose it as the agent's toolbox
neuralosd openapi ./finacle-users      # OpenAPI + MCP manifest

# 3. attach the agentd whose agent uses that toolbox
synapse-agentd -gateway ws://127.0.0.1:3000/ws/v2/agent \
  -agent-id finacle-users-agent \
  -provider-key "$LLM_KEY" -workdir /srv/agentd-work
# agent config: MCP server → http://127.0.0.1:8877
```

What crosses the mesh afterwards: **verified answers or typed refusals** —
never database credentials, never invented numbers. Two correlated audit
trails record the whole chain (gateway ledger + instance audit).

Why this pairing matters:

| Problem | How the pair answers it |
|---|---|
| "Can another team query our data with an LLM?" | Only a menu of caged questions is exposed; no DB access |
| Hallucinated figures in agent workflows | Worst case is a typed refusal — an invented number is structurally excluded |
| Compliance forensics | Gateway ledger + instance audit reconstruct question → probe → data |
| Data residency | The instance never leaves your premises; only answers cross the mesh |

## Webhooks — know when mesh tasks finish

Async tasks a peer creates on your edge can notify your systems:

```bash
synapse-gateway -mesh-enabled … \
  -mesh-task-webhook https://ops.example.com/hooks/task-done \
  -mesh-task-executor-webhook https://ops.example.com/hooks/task-input
```

Notifications are **signed** and POSTed on completion (or when a task you
execute pauses for input — the task and its question are the payload).
Verify signatures with the edge identity before acting on them.

## MCP and AI products

The gateway's directory + gates are agent-agnostic. Any MCP-capable client
or AI product can consume an edge's capabilities; the neuralOS pairing above
is the pattern for making those capabilities hallucination-proof.

## Multi-organisation meshes

1. Each org runs its own gateway (data stays behind each edge)
2. Share one NATS server (or cluster) as the discovery mesh
3. Each edge: `-mesh-enabled` + credentials + `-mesh-capabilities`
4. Verification posture for regulated meshes:
   `-mesh-verify-mode require -mesh-trust-on-first-use=false
   -mesh-trusted-peers sha256:…,sha256:…`
5. Bound escalation: `-mesh-require-verified-invoke` (default on),
   `-mesh-invoke-operations`, `-mesh-invoke-timeout`,
   `-mesh-skill-proxy-targets`

Discovery is public by design; **invocation is not**.
