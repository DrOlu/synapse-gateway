# Changelog

Distribution packages track the gateway version-for-version. Highlights of
recent gateway releases:

## 1.7.12 — the distribution fix

- Gateway releases now auto-publish to **PyPI and npm** (the `released`
  event trigger gap is closed; a daily drift-guard catches any future skew)
- The **fallback `py3-none-any` wheel** reaches PyPI — musl/Alpine, BSD and
  32-bit hosts install with guidance instead of failing
- Checksum verification hardened (binary-mode entries, filenames with
  spaces, missing entries are hard failures)
- npm publisher secret corrected (`SYNAPSE_NPM_TOKEN`)

## 1.7.6 — packaged baseline

- PyPI + npm packaging of the gateway + agentd binaries with per-platform
  wheels, launcher entry points, and fallback guidance
- Mesh bridge surface: NATS event mesh / Synapse bridge, verification modes,
  registry (JetStream KV / broadcast), durable mailbox, gated skill
  invocation, signed task webhooks
