# Cookbook — worked recipes

## Recipe 1: Single host, one agent (5 minutes)

```bash
pip install synapse-gateway
synapse-gateway                                    # terminal 1
synapse-agentd -gateway ws://127.0.0.1:3000/ws/v2/agent \
  -agent-id solo -provider-key "$LLM_KEY" \
  -provider-url https://api.openai.com/v1 \
  -provider-model gpt-5 -workdir ~/agentd          # terminal 2
```

Open the desktop, sign in, talk to `solo`. Done — the audit ledger already
has your first entries.

## Recipe 2: A fleet of headless workers

One gateway, many agentds, each a directory row:

```bash
# on N servers, the same three lines with different -agent-id and -workdir
synapse-agentd -gateway ws://gw.internal:3000/ws/v2/agent \
  -agent-id agentd-finance-1 -token "$FINANCE_AGENT_TOKEN" \
  -provider-key "$LLM_KEY" -workdir /srv/work/finance
```

Issue **per-agent tokens** so each worker is individually revocable. The
gateway needs zero per-worker configuration — that is the point.

## Recipe 3: Cross-organisation mesh (regulated posture)

Org A (bank) and Org B (processor) want agents to discover each other and
run gated tasks — without exposing data or granting unverified access.

```bash
# shared NATS server (plain is fine; JetStream unlocks the KV registry +
# durable mailbox)
# Org A edge:
synapse-gateway -mesh-enabled -mesh-url nats://mesh.example.com:4222 \
  -mesh-agent-id bank-edge -mesh-name "Bank Edge" \
  -mesh-capabilities "incidents,ledger,reporting" \
  -mesh-creds-file /etc/reactorpro/mesh.creds \
  -mesh-verify-mode require -mesh-trust-on-first-use=false \
  -mesh-trusted-peers "sha256:<orgB-fingerprint>" \
  -mesh-require-verified-invoke \
  -mesh-invoke-operations task -mesh-invoke-timeout 10m
# Org B edge: mirrored, trusting Org A's fingerprint
```

Result: B discovers A's capabilities on the mesh; invocation requires a
verified signature from a pinned fingerprint; timeouts bound every call.

## Recipe 4: Pair with a neuralOS instance (verified data answers)

```bash
neuralosd serve --instance-dir ./incidents --port 8877 --auth-token "$T"
neuralosd openapi ./incidents
# attach an agentd whose agent uses the instance as its toolbox
```

Now *"how many P1 incidents this week?"* on the mesh returns a verified
count — or a typed refusal — with two correlated audit trails. See
[Integrate](integrate.md).

## Recipe 5: Durable mailbox for off-duty agents

Agents that are offline should not lose mesh invocations:

```bash
synapse-gateway -mesh-enabled -mesh-url nats://mesh:4222 \
  -mesh-mailbox -mesh-mailbox-max-age 48h
```

Invocations for an absent agent are retained in the JetStream stream and
delivered when it signs back in.

## Recipe 6: Signed task webhooks into your ticketing system

```bash
synapse-gateway -mesh-enabled … \
  -mesh-task-webhook https://itsm.example.com/hooks/mesh-task \
  -mesh-task-executor-webhook https://itsm.example.com/hooks/mesh-input
```

Your hook receives a **signed** POST on completion (or when a task executed
for a peer pauses for input). Verify the edge signature, then open or update
the ticket programmatically.

## Recipe 7: Zero-trust mesh hardening checklist

```bash
-mesh-verify-mode require \
-mesh-trust-on-first-use=false \
-mesh-trusted-peers sha256:…,sha256:… \
-mesh-require-verified-invoke \
-mesh-invoke-operations task \
-mesh-invoke-timeout 10m \
-mesh-rate-limit-per-second 20 \
-mesh-accepted-versions v1 \
-mesh-encryption require
```

`-mesh-encryption require` seals every cross-org payload to the receiving
edge's X25519 key (advertised in the mesh manifest) — the broker routes
ciphertext only. Combine with `verify-mode require` + pinned peers for the
full zero-trust posture.

Discovery is public; **invocation is verified, allow-listed, time-bounded
and rate-limited**.

## Recipe 8: Upgrades with zero downtime

The gateway and agentd are single static files:

1. Upload the new binaries beside the old ones
2. Swap only when no process runs (`tasklist`/`pgrep` first)
3. Agents re-sign-in automatically; sessions re-establish
4. The mesh manifest refreshes itself
