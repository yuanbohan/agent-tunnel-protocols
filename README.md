# agent-tunnel-protocols

Canonical cross-repository protocol documents for Agent Tunnel.

This repository is the source of truth for protocol decisions shared by the
mobile companion, Relay/server implementation, and Go tunnel/daemon
implementation. Implementation repositories keep local mirrors, tests, and
operational notes, but protocol-level compatibility decisions should point back
here.

## Connectivity Protocols

- [Protocol](docs/protocol.md) defines the current
  daemon-to-mobile QUIC session transport, frame registry, JSON payload
  families, session metadata boundary, transport security invariants, and
  compatibility expectations.

## Repository Rules

- Keep fixtures synthetic and non-secret. Do not commit real terminal captures,
  credentials, private keys, tunnel tokens, device fingerprints, private paths,
  or user input.
- Protocol-breaking changes must update the owning document and describe the
  compatibility line or protocol version transition.
- Relay remains content-opaque for terminal bytes and fallback QUIC packets
  unless a future protocol decision explicitly changes that boundary.
