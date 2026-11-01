# Quick Start & Operations

Step-by-step: install, run the gateway, attach agents, secure it, and go to
production. Every command is copy-paste ready.

## 1. Install

=== "pip (recommended)"

    ```bash
    pip install synapse-gateway
    # installs two commands: synapse-gateway + synapse-agentd
    ```

    Platform wheels: linux amd64/arm64 (manylinux2014), darwin amd64/arm64,
    windows amd64. On platforms without a native wheel (musl/Alpine, BSD,
    32-bit) the **fallback wheel** installs and its launcher prints where to
    grab a binary.

=== "npm"

    ```bash
    npm i -g @hyperspaceng/synapse-gateway
    # optional platform packages are pulled in automatically
    ```

=== "standalone binary"

    Grab a single file from
    [ReactorPro releases](https://github.com/DrOlu/ReactorPro/releases)
    (`reactorpro-gateway-<os>-<arch>` + `reactorpro-agentd-<os>-<arch>`,
    SHA256SUMS provided and verified by the packaging pipeline).

=== "Docker / source"

    The gateway lives in `crates/agent-gateway` in the
    [ReactorPro repo](https://github.com/DrOlu/ReactorPro) (Go); a Dockerfile
    and `gateway-docker` workflow are provided upstream.

## 2. Run the gateway

```bash
synapse-gateway
# → serving on http://127.0.0.1:3000
#   /healthz            liveness (name-only)
#   /api/health         deeper health check
#   /ws/v2/agent        agent sign-in WebSocket
#   /ws/v2/terminal     terminal data-plane WebSocket
#   /api/mesh/status    mesh bridge status (see §5)
```

Every flag has an environment twin (`LIVEAGENT_GATEWAY_*`). The essentials:

```bash
synapse-gateway \
  -token "$GATEWAY_TOKEN" \                # sign-in token (or LIVEAGENT_GATEWAY_TOKEN)
  -http-addr 0.0.0.0:3000 \                # bind address
  -agent-db /var/lib/reactorpro/agents.db  # per-agent token store (auto-created)
```

Optional TLS: `-tls-cert` / `-tls-key`.

Connection ceilings (protect the building from crowding):
`-max-agent-connections`, `-max-browser-connections`,
`-max-terminal-connections`.

## 3. Attach an agentd (the headless worker)

```bash
synapse-agentd -gateway ws://127.0.0.1:3000/ws/v2/agent \
  -agent-id agentd-server-1 \
  -token "$GATEWAY_TOKEN" \
  -provider-url https://api.openai.com/v1 \
  -provider-key "$OPENAI_API_KEY" \
  -provider-model gpt-5 \
  -workdir /srv/agentd-work
```

Key properties (from the source):

- **Same WebSocket as the desktop** — to the gateway an agentd is a row in
  the same directory, behind the same gates, in the same audit trail
- **The agentd holds its own provider key** — the gateway keeps its
  "no model keys, ever" property
- Serves remote chat turns **with real tool use, several at a time**
- Every flag has an environment twin (`LIVEAGENT_AGENTD_*`)

## 4. Token auth

Agents and browsers authenticate with a gateway token or per-agent tokens
(auto-created in the agent DB). Browsers use `LIVEAGENT_GATEWAY_TOKEN` at
sign-in; issue per-agent tokens for headless workers so each one is
individually revocable.

`/healthz` is name-only liveness and is intentionally open; `/api/health`
is the deeper check. Bind non-loopback only with a token set, behind TLS,
or behind your own proxy.

## 5. The mesh bridge (optional — off by default)

One gateway is standalone: WebSocket + local SQLite, nothing connects
anywhere. NATS enters **only** for cross-organisation discovery:

```bash
synapse-gateway \
  -mesh-enabled \
  -mesh-url nats://mesh.example.com:4222 \
  -mesh-agent-id bank-lagos-edge \
  -mesh-name "Bank Lagos Edge" \
  -mesh-capabilities "incidents,finacle,reporting" \
  -mesh-creds-file /etc/reactorpro/mesh.creds     # NKey/JWT (or -mesh-token / -mesh-user + -mesh-password)
```

- Failure to connect is **not fatal** — the gateway keeps serving; the
  reason is on `/api/mesh/status`
- Discovery registry: `auto` (JetStream KV when offered, else broadcast on
  a plain NATS server), or force `jetstream` / `broadcast`
- Verification: inbound mesh envelopes can be signature-verified —
  `-mesh-verify-mode off|prefer|require`, peer fingerprint
  **trust-on-first-use** (on by default) or an explicit
  `-mesh-trusted-peers sha256:…` list, clock-skew bounding, per-sender rate
  limits, and envelope size caps
- **Durable mailbox**: `-mesh-mailbox` keeps skill invocations for an absent
  agent in a JetStream stream and delivers them when it returns
- Skills served to peers are gated: ping, describe, status, and the
  **gated invoke** — `-mesh-allow-remote-invoke`,
  `-mesh-require-verified-invoke`, `-mesh-invoke-operations`,
  `-mesh-skill-proxy-targets`, `-mesh-invoke-timeout` bound exactly what a
  peer may reach and for how long
- Async **tasks** a peer creates can run up to `-mesh-task-max-runtime`,
  stay queryable for `-mesh-task-retention`, and notify signed
  **webhooks** on completion (`-mesh-task-webhook` /
  `-mesh-task-executor-webhook`)

## 6. Operations

- **Run under a service manager** (systemd/launchd/Windows service) — the
  binary is a single static file; upgrades are binary swaps
- **Back up the agent DB** (`-agent-db` path) — it holds tokens and names
- **Watch `/api/mesh/status`** in monitoring if the bridge is enabled
- Chat/timing knobs (`-request-timeout`, `-chat-prepare-timeout`,
  `-chat-delivery-timeout`, `-chat-start-timeout`) have sane defaults; raise
  only for slow providers
