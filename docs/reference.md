# Reference

Every flag, both commands. Each flag has an environment twin — gateway flags
`LIVEAGENT_GATEWAY_*`, agentd flags `LIVEAGENT_AGENTD_*`.

## `synapse-gateway`

### Core

| Flag | Env | Default | Meaning |
|---|---|---|---|
| `-token` | `LIVEAGENT_GATEWAY_TOKEN` | *(empty)* | Gateway authentication token |
| `-agent-db` | `LIVEAGENT_GATEWAY_AGENT_DB` | auto path | Per-agent token SQLite DB (auto-created) |
| `-http-addr` | `LIVEAGENT_GATEWAY_HTTP_ADDR` | auto | HTTP listen address |
| `-tls-cert` / `-tls-key` | — | *(empty)* | TLS certificate / private key |
| `-max-agent-connections` | `…_MAX_AGENT_CONNECTIONS` | built-in bound | Max concurrent desktop agent connections |
| `-max-browser-connections` | `…_MAX_BROWSER_CONNECTIONS` | built-in bound | Max concurrent browser connections |
| `-max-terminal-connections` | `…_MAX_TERMINAL_CONNECTIONS` | built-in bound | Max concurrent terminal data-plane connections |

### Chat timing

| Flag | Default | Meaning |
|---|---|---|
| `-request-timeout` | 2m | Non-streaming API call timeout |
| `-chat-prepare-timeout` | 2s | Pre-submit desktop agent liveness probe |
| `-chat-delivery-timeout` | 5s | Delivering an accepted command to the agent stream |
| `-chat-start-timeout` | 5s | Waiting for a delivered request to start |

### Mesh bridge (all optional; bridge is disabled by default)

| Flag | Default | Meaning |
|---|---|---|
| `-mesh-enabled` | false | Enable the NATS event mesh / Synapse bridge |
| `-mesh-url` | *(empty)* | NATS server URL |
| `-mesh-agent-id` | auto | Unique id across the mesh (fixed once the identity file exists) |
| `-mesh-name` | "ReactorPro Gateway" | Manifest display name |
| `-mesh-capabilities` | *(empty)* | Comma-separated capabilities this edge advertises |
| `-mesh-identity-path` | auto | Mesh identity file (minted on first use) |
| `-mesh-token` / `-mesh-user` + `-mesh-password` / `-mesh-creds-file` | *(empty)* | NATS auth (creds file takes precedence) |
| `-mesh-verify-mode` | prefer | Inbound envelope verification: `off` / `prefer` / `require` |
| `-mesh-clock-skew` | built-in | Allowed envelope timestamp drift |
| `-mesh-trusted-peers` | *(empty)* | Comma-separated `sha256:<hex>` peer fingerprints trusted without first-use learning |
| `-mesh-trust-on-first-use` | true | Record a peer fingerprint on its first verified message |
| `-mesh-max-envelope-bytes` | built-in cap | Max inbound mesh envelope size |
| `-mesh-rate-limit-per-second` / `-mesh-rate-limit-burst` | 50 / 100 | Per-sender inbound rate limit (0 disables) |
| `-mesh-accepted-versions` | *(any)* | Comma-separated protocol versions to accept |
| `-mesh-registry` | auto | Discovery registry: `auto` / `jetstream` / `broadcast` |
| `-mesh-registry-bucket` | built-in | JetStream KV bucket holding one manifest per edge |
| `-mesh-registry-ttl` | 3× heartbeat | Registry entry validity without a heartbeat |
| `-mesh-mailbox` | false | JetStream mailbox for invocations to an absent agent |
| `-mesh-mailbox-stream` | built-in | Stream backing the mailbox |
| `-mesh-mailbox-max-age` / `-mesh-mailbox-max-msgs` | 7 days / bound | Mailbox retention |
| `-mesh-skills-enabled` | true | Serve built-in mesh skills (ping, describe, status, gated invoke) |
| `-mesh-skills` | *(all)* | Comma-separated subset of built-in skills |
| `-mesh-allow-remote-invoke` | true | Allow peers to invoke operations on local desktop agents |
| `-mesh-require-verified-invoke` | true | Refuse invocation from unverified caller identity |
| `-mesh-invoke-operations` | task | Comma-separated operations a peer may invoke |
| `-mesh-skill-proxy-targets` | *(empty)* | Local harness agent ids the skillproxy skill may reach (empty disables) |
| `-mesh-skill-proxy-timeout` | mesh default | Bound on one proxied request |
| `-mesh-invoke-timeout` | 3m | How long one remote invocation may run |
| `-mesh-task-max-runtime` | 30m | Max runtime of one async mesh task |
| `-mesh-task-retention` | 7 days | Finished-task queryability window |
| `-mesh-task-webhook` | *(empty)* | Default signed webhook for task completion (per-task notifyUrl wins) |
| `-mesh-task-executor-webhook` | *(empty)* | Signed webhook when a task executed for a peer pauses for input |
| `-mesh-approval-history` | built-in bound | Decided mesh approvals retained in memory + disk |

### HTTP endpoints

| Endpoint | Auth | Meaning |
|---|---|---|
| `GET /healthz` | open | Name-only liveness |
| `GET /api/health` | token | Deeper health check |
| `GET /api/mesh/status` | token | Mesh bridge state and failure reason |
| `POST /api/messages` (+ `/api/*`) | token | Chat / session APIs |
| `GET /ws/v2/agent` | token | Agent + desktop sign-in WebSocket |
| `GET /ws/v2/terminal` | token | Terminal data-plane WebSocket |
| `/t/<slug>/…` | per-slug | Tunnelled browser sessions |

## `synapse-agentd`

Core flags (every one has a `LIVEAGENT_AGENTD_*` env twin):

| Flag | Meaning |
|---|---|
| `-gateway` | Gateway WebSocket URL (`ws(s)://host:3000/ws/v2/agent`) |
| `-agent-id` | Directory name — addressable by name or capability |
| `-token` | Gateway or per-agent token |
| `-provider-url` | LLM API base (the agentd holds its own key by design) |
| `-provider-key` | LLM API key |
| `-provider-model` | Model id |
| `-workdir` | Working directory for execution |

## Distributions

| Channel | Notes |
|---|---|
| PyPI `synapse-gateway` | Per-platform wheels + a fallback `py3-none-any` wheel whose launcher prints binary guidance on unsupported platforms |
| npm `@hyperspaceng/synapse-gateway` | Launcher package + per-platform optional dependencies |
| GitHub releases | Standalone binaries + `SHA256SUMS` (verified by the packaging pipeline) |
