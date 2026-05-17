# agent-tunnel-protocols

Canonical cross-repository protocol documents for Agent Tunnel.

This repository is the source of truth for protocol decisions shared by the
mobile companion, Relay/server implementation, and Go tunnel/daemon
implementation. Implementation repositories keep local mirrors, tests, and
operational notes, but protocol-level compatibility decisions should point back
here.

## Sibling Implementation Repositories

For local cross-repository work, keep these sibling checkouts together:

- `../agent-tunnel` - Go Relay, tunnel daemon, STUN, direct UDP, fallback
  tunnel, pairing state, and daemon transport implementation.
- `../agent-tunnel-android` - official Android companion implementation and
  mobile UX/docs.

Protocol changes here should usually be paired with implementation or pointer
updates in one or both implementation repositories.

## Connectivity Protocols

- [End-to-End Flows](docs/end-to-end-flows.md) explains the current trusted
  computer, pairing, session list, preview, direct/relay, detail input, and
  key-storage flows across Android, Relay, and the daemon.
- [Pairing](docs/pairing.md) defines the Ed25519 pairing transcripts, Relay
  pairing transport boundary, SAS algorithm, trust completion, persistence,
  and revocation semantics.
- [Relay Control Plane](docs/relay-control-plane.md) defines Relay realtime
  auth, presence, pairing forwarding, direct rendezvous, fallback tunnel token
  setup, and opaque packet forwarding.
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
