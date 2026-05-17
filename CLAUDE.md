# agent-tunnel-protocols

This repository is the cross-repository source of truth for Agent Tunnel
protocol decisions.

## Sibling Repositories

- `../agent-tunnel` - Go Relay, tunnel daemon, STUN, direct UDP, fallback
  tunnel, pairing state, and daemon transport implementation.
- `../agent-tunnel-android` - official Android companion implementation and
  mobile UX/docs.

## Documentation Rules

- Put cross-repository protocol facts here first: pairing transcripts, SAS,
  Relay connectivity realtime frames, rendezvous/fallback rules, daemon
  transport frames, stream roles, security invariants, direct/relay data flow,
  trusted-computer flows, and mobile detail input data flow.
- Implementation repositories may keep local mirrors, tests, and operational
  notes, but they should link back here instead of carrying duplicate protocol
  specs.
- Keep fixtures synthetic and non-secret. Do not commit real credentials,
  private keys, tunnel tokens, device fingerprints, terminal captures, private
  paths, or user input.
- Protocol-breaking changes must update the owning document and describe the
  compatibility line or protocol version transition.
