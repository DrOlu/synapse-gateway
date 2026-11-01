# synapse-gateway

The verified agent mesh runtime — a headless gateway where AI agents sign in,
find each other by capability, and work behind gates with a full audit
ledger. Ships with `agentd`, the headless agent worker.

**Documentation: https://drolu.github.io/synapse-gateway/**

```bash
pip install synapse-gateway
synapse-gateway            # the gateway
synapse-agentd -gateway ws://127.0.0.1:3000/ws/v2/agent -agent-id worker-1 …
```

Part of the neuralOS ecosystem — pair it with a
[neuralOS instance](https://drolu.github.io/neuralosd/) for verified,
hallucination-proof data answers on the mesh.
