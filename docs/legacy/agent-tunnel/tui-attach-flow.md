# TUI Session Transport

> Legacy status: historical copy from `agent-tunnel`. This file is not current protocol authority. Use `docs/api.md`, `docs/architecture.md`, `docs/pairing.md`, `docs/relay-control-plane.md`, `docs/protocol.md`, and `docs/end-to-end-flows.md` in this repository for current SSOT.


This document used to describe the deleted Relay session viewing path. The current product no longer exposes Relay session list, stop, or attach APIs.

Current behavior:

- `tunnel run` owns the PTY and terminal mirror.
- `tunnel run` registers with the local daemon broker before starting the user command.
- The daemon broker receives session metadata, latest preview, coalesced terminal snapshots, and live output bytes from the local `tunnel run` process.
- Trusted mobile clients receive session state and interactive terminal transport through daemon connectivity transport.
- Relay remains the auth, pairing, computer presence, launch-correlation, rendezvous, and opaque fallback packet relay.

For the current contracts, see:

- [docs/protocol.md](./protocol.md)
- [docs/architecture.md](./architecture.md)
- [docs/connectivity/protocol/transport.md](./connectivity/protocol/transport.md)
