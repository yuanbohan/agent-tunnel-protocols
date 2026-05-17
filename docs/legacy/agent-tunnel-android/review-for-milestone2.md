---
title: "Milestone 2 review: direct + fallback daemon transport"
type: review
date: 2026-05-12
scope:
  - agent-tunnel-android
  - agent-tunnel
---

# Milestone 2 Review: Direct + Fallback Daemon Transport

> Legacy status: historical Android milestone review. This file contains stale point-in-time findings and is not current protocol or implementation authority. Use `docs/status/implementation-matrix.md` and current SSOT docs for current status.


## Review Goal

This review compares the current Android branch against the intended connectivity architecture:

- Android pairs with a computer/daemon and stores trust locally.
- Relay remains a control plane plus blind fallback packet relay.
- The actual session data path is daemon-owned: `session_index`, previews, terminal snapshots, live bytes, input, and resize move over the daemon transport, not Relay session APIs.
- Direct path is attempted first over UDP.
- Fallback path carries the same QUIC/TLS transport over a Relay WebSocket tunnel.
- QUIC/TLS uses ALPN `tunnel-conn/1` and pinned device identity; direct and fallback differ only by packet carrier.

Overall status at the time of this review: the server side was much closer to the target architecture than Android. Android had useful pairing, trust, frame protocol, reducer, carrier, and UI projection scaffolding, but the real milestone-2 path was not yet closed: Android did not establish native QUIC or run a direct/fallback daemon connection in production.

## High-Confidence Incorrect Implementations

### 1. Android SAS computation does not match server/docs

This is the most important correctness bug. Android adds a SAS domain prefix:

- `app/src/main/java/ai/diaro/agentunnel/domain/pairing/PairingTranscript.kt:60`
- `app/src/main/java/ai/diaro/agentunnel/domain/pairing/PairingTranscript.kt:61`

Server SAS canonicalization length-prefixes only four fields and has no prefix:

- `/Users/yuanbo/workspace/github.com/agent-tunnel/internal/connectivity/pairing/sas.go:35`
- `/Users/yuanbo/workspace/github.com/agent-tunnel/internal/connectivity/pairing/sas.go:38`

The golden vectors already prove the mismatch. Server expects `696700`, `626209`, `670900` in `/Users/yuanbo/workspace/github.com/agent-tunnel/internal/connectivity/pairing/sas_test.go:26`, `:34`, `:42`, while Android tests claim upstream vectors are `180870`, `087648`, `682679` in `app/src/test/java/ai/diaro/agentunnel/domain/pairing/PairingSasTest.kt:15`, `:22`, `:29`.

Impact: real pairing can submit and forward a response, but the user-visible SAS on Android and daemon will differ. That makes #132/#135 pairing confirmation fail in the real human verification step, or worse, trains tests around the wrong contract.

Fix: make Android SAS canonicalization byte-for-byte identical to server and update Android golden vectors to the server vectors.

### 2. Android fallback tunnel URL uses the wrong path

Android builds fallback tunnel URLs as `/api/connectivity/tunnel/ws`:

- `app/src/main/java/ai/diaro/agentunnel/data/relay/RelayService.kt:762`
- `app/src/main/java/ai/diaro/agentunnel/data/relay/RelayService.kt:764`
- `app/src/test/java/ai/diaro/agentunnel/data/relay/RelayServiceUrlTest.kt:63`

Server registers the route at `/connectivity/tunnel/ws`, without `/api`:

- `/Users/yuanbo/workspace/github.com/agent-tunnel/internal/relay/handler/new.go:112`

The daemon also dials `/connectivity/tunnel/ws`:

- `/Users/yuanbo/workspace/github.com/agent-tunnel/internal/tunnel/daemon/connectivity_connector.go:536`

Impact: Android fallback tunnel connection will hit the wrong endpoint and fail before any QUIC packet can be relayed.

Fix: change Android `buildRelayTunnelUrl` and test expectations to `/connectivity/tunnel/ws`.

### 3. Android private candidate filtering accepts loopback and documentation networks by default

Android treats loopback and documentation addresses as allowed private candidates:

- `app/src/main/java/ai/diaro/agentunnel/domain/transport/TransportCandidateModels.kt:68`
- `app/src/main/java/ai/diaro/agentunnel/domain/transport/TransportCandidateModels.kt:72`
- `app/src/main/java/ai/diaro/agentunnel/domain/transport/TransportCandidateModels.kt:73`

Server only allows loopback/test documentation networks when explicit options are enabled:

- `/Users/yuanbo/workspace/github.com/agent-tunnel/internal/connectivity/direct/candidate.go:11`
- `/Users/yuanbo/workspace/github.com/agent-tunnel/internal/connectivity/direct/candidate.go:88`
- `/Users/yuanbo/workspace/github.com/agent-tunnel/internal/connectivity/direct/candidate.go:91`

Impact: Android can advertise non-production candidates in normal operation. Loopback is especially dangerous because it can create misleading direct-attempt behavior and bad production diagnostics.

Fix: align Android defaults with server: allow private and link-local only by default; gate loopback/documentation addresses behind explicit test/debug options.

### 4. Android transport runtime assumes already-framed messages, but QUIC control is a byte stream

`DaemonTransportConnection` exposes `incomingControlFrames: Flow<ByteArray>`:

- `app/src/main/java/ai/diaro/agentunnel/data/transport/DaemonTransportConnection.kt:6`
- `app/src/main/java/ai/diaro/agentunnel/data/transport/DaemonTransportConnection.kt:7`

`DaemonTransportRuntime` decodes each emitted `ByteArray` as exactly one frame:

- `app/src/main/java/ai/diaro/agentunnel/data/transport/DaemonTransportRuntime.kt:71`
- `app/src/main/java/ai/diaro/agentunnel/data/transport/DaemonTransportRuntime.kt:72`

Server writes and reads frames on QUIC streams using stream I/O:

- `/Users/yuanbo/workspace/github.com/agent-tunnel/internal/tunnel/daemon/connectivity_transport.go:71`
- `/Users/yuanbo/workspace/github.com/agent-tunnel/internal/tunnel/daemon/connectivity_transport.go:592`
- `/Users/yuanbo/workspace/github.com/agent-tunnel/internal/tunnel/daemon/connectivity_transport.go:609`

Impact: this is fine for fake JVM tests, but it is not a production QUIC stream boundary. A real stream can deliver partial frames or multiple frames in one read. If the native bridge exposes reads directly, Android will mis-decode or drop data.

Fix: the native/Kotlin QUIC bridge needs a stream-frame reader that buffers bytes and emits decoded frames only after `[type][varint length][payload]` is complete. It should also reject trailing bytes only when they are genuinely invalid, not because a stream read boundary was arbitrary.

### 5. Trusted computer connection state is keyed by `computerId` only

Presence matching uses `(computerId, daemonFingerprint)`:

- `app/src/main/java/ai/diaro/agentunnel/domain/connectivity/TrustedComputerHomeProjector.kt:117`
- `app/src/main/java/ai/diaro/agentunnel/domain/connectivity/TrustedComputerHomeProjector.kt:120`
- `app/src/main/java/ai/diaro/agentunnel/domain/connectivity/TrustedComputerHomeProjector.kt:127`

But connection snapshots are looked up by only `computerId`:

- `app/src/main/java/ai/diaro/agentunnel/domain/connectivity/TrustedComputerHomeProjector.kt:132`

The coordinator also stores/publishes by `computerId`:

- `app/src/main/java/ai/diaro/agentunnel/data/transport/TrustedComputerConnectionCoordinator.kt:31`
- `app/src/main/java/ai/diaro/agentunnel/data/transport/TrustedComputerConnectionCoordinator.kt:37`

Impact: after daemon key rotation, stale trust records, or duplicate computer IDs, a connection state can attach to the wrong trusted record. That is exactly the identity boundary this milestone is trying to make explicit.

Fix: key daemon connection snapshots by composite identity: account/relay base URL as needed plus `(computerId, daemonFingerprint)`.

## Major Missing Work

### Android #138 is mostly scaffold, not a production transport

The Android branch has a useful native-load facade and protocol runtime, but the Rust bridge only exposes version/ALPN:

- `app/src/main/java/ai/diaro/agentunnel/data/transport/quic/QuicNativeBridge.kt:56`
- `app/src/main/java/ai/diaro/agentunnel/data/transport/quic/QuicNativeBridge.kt:60`
- `app/src/main/rust/agentunnel_quic/src/lib.rs:8`
- `app/src/main/rust/agentunnel_quic/src/lib.rs:18`

There is no native QUIC connection, packet ingest/flush loop, stream open/read/write API, timer handling, or AndroidKeystore signing callback. The project docs accurately record this as partial evidence:

- `docs/connectivity/implementation/step-05-fallback-transport-validation.md:111`
- `docs/connectivity/evidence/2026-05-12-quic-direct-fallback.md:29`
- `docs/connectivity/evidence/2026-05-12-quic-direct-fallback.md:38`

Until this exists, Android cannot prove either direct or fallback QUIC/TLS.

### Android production wiring does not start a daemon transport manager

`AppContainer` only creates a `TrustedComputerConnectionCoordinator`:

- `app/src/main/java/ai/diaro/agentunnel/AppContainer.kt:73`

Search shows `DaemonConnectionManager`, `DaemonTransportRuntime`, `DirectUdpPacketCarrier`, `RelayTunnelPacketCarrier`, `openRelayTunnelPacketSocket`, and `loadOrCreateTransportIdentitySource` are only used by tests or as isolated classes. There is no production path that:

- reacts to eligible trusted/visible computers,
- sends `rendezvous_open`,
- opens direct UDP or Relay tunnel carriers,
- creates a native QUIC connection,
- starts `DaemonTransportRuntime`,
- publishes real `DaemonConnectionSnapshot` updates.

Impact: the home projection can display daemon sessions if someone publishes snapshots, but nothing in production publishes them.

### Session list/detail was not yet fully daemon-routed

At the time of this review, trusted-computer daemon sessions were rendered only as rows inside a computer card:

- `app/src/main/java/ai/diaro/agentunnel/ui/list/SessionListScreen.kt:482`
- `app/src/main/java/ai/diaro/agentunnel/ui/list/SessionListScreen.kt:488`

Those rows were not clickable and did not route to daemon-backed session detail:

- `app/src/main/java/ai/diaro/agentunnel/ui/list/SessionListScreen.kt:500`
- `app/src/main/java/ai/diaro/agentunnel/ui/list/SessionListScreen.kt:504`

The detail route still constructed `SessionDetailViewModel(sessionId, container)` for the old route shape:

- `app/src/main/java/ai/diaro/agentunnel/MainActivity.kt:398`
- `app/src/main/java/ai/diaro/agentunnel/MainActivity.kt:409`

Impact: Steps #124/#125/#126 were not product-complete. Daemon-owned sessions did not yet have preview subscription, terminal attach, input, resize, or navigation.

### Direct path is only a primitive, not a real NAT traversal attempt

Android `DirectUdpPacketCarrier` sends every packet to the first candidate only:

- `app/src/main/java/ai/diaro/agentunnel/data/transport/carrier/DirectUdpPacketCarrier.kt:46`

Server has direct/STUN support and controlled simulator evidence, but Android still needs the real attempt loop: same-socket STUN, public/private candidates, rendezvous hint handling, multi-candidate probing, timeout/fallback sequencing, and diagnostics.

### Fallback opacity is not proven

Server Relay tunnel is shaped correctly: it redeems bearer tokens, pairs app/daemon actors, and forwards only binary messages:

- `/Users/yuanbo/workspace/github.com/agent-tunnel/internal/relay/handler/connectivity/tunnel_ws.go:38`
- `/Users/yuanbo/workspace/github.com/agent-tunnel/internal/relay/handler/connectivity/tunnel_ws.go:64`
- `/Users/yuanbo/workspace/github.com/agent-tunnel/internal/relay/handler/connectivity/tunnel_ws.go:135`
- `/Users/yuanbo/workspace/github.com/agent-tunnel/internal/relay/handler/connectivity/tunnel_ws.go:155`

But Android has not produced evidence that the Relay sees encrypted QUIC packets rather than session plaintext:

- `docs/connectivity/evidence/2026-05-12-quic-direct-fallback.md:36`

### Policy-unavailable behavior needs a product/security decision

The issue text says policy unavailable should fail closed, but Android currently allows the first local trust transaction when policy is unavailable:

- `app/src/main/java/ai/diaro/agentunnel/domain/connectivity/TrustedComputerPolicy.kt:99`
- `app/src/main/java/ai/diaro/agentunnel/domain/connectivity/TrustedComputerPolicy.kt:101`
- `app/src/test/java/ai/diaro/agentunnel/domain/connectivity/TrustedComputerPolicyTest.kt:64`
- `app/src/test/java/ai/diaro/agentunnel/domain/connectivity/TrustedComputerPolicyTest.kt:76`

This may be intentional if the intended interpretation is “fail closed only for additional/Pro multi-computer behavior.” If not, it is a policy bug. This should be resolved before treating #131/#136 as complete.

## Server-Side Status

Server has the main shape needed by milestone 2:

- Pairing invitation reservation happens through Relay before daemon emits an invitation: `/Users/yuanbo/workspace/github.com/agent-tunnel/internal/tunnel/daemon/runtime.go:260`.
- Android pairing responses are stored pending local daemon confirmation: `/Users/yuanbo/workspace/github.com/agent-tunnel/internal/tunnel/daemon/connectivity_connector.go:211`.
- Pair completion is sent only after daemon confirmation: `/Users/yuanbo/workspace/github.com/agent-tunnel/internal/tunnel/daemon/runtime.go:289`.
- Relay tunnel tokens are per-attempt/per-actor and rate-limited: `/Users/yuanbo/workspace/github.com/agent-tunnel/internal/relay/connectivity/registry.go:838`, `:848`, `:889`.
- Daemon fallback accepts the Relay WebSocket packet carrier and serves QUIC/TLS over it: `/Users/yuanbo/workspace/github.com/agent-tunnel/internal/tunnel/daemon/connectivity_connector.go:379`, `:382`.
- Daemon transport sends `hello`, `session_index`, and `path_state`: `/Users/yuanbo/workspace/github.com/agent-tunnel/internal/tunnel/daemon/connectivity_transport.go:105`, `:120`, `:123`.
- Daemon supports preview subscribe, interactive grant, input, resize, snapshot chunks, and live bytes: `/Users/yuanbo/workspace/github.com/agent-tunnel/internal/tunnel/daemon/connectivity_transport.go:283`, `:313`, `:345`, `:470`, `:595`.

Remaining server-side work is mostly validation and hardening rather than architecture:

- Process-level fallback test against a running Relay + daemon + mobile/client path.
- Android interop proof, not just Go simulator proof.
- Relay packet counters and diagnostics that prove opaque forwarding without logging sensitive payload.
- Production direct-path measurement across real networks/NATs.
- End-to-end revocation/teardown evidence while direct or fallback transports are active.

## Issue-by-Issue Status

### #131 Identity/Auth Policy Foundation

Mostly implemented on Android for local identity, trust storage, account policy, and cleanup. Remaining concerns:

- policy-unavailable semantics are ambiguous;
- native QUIC cannot yet use the identity for TLS signing;
- connection state should use composite daemon identity, not just computer ID.

### #132 Pairing Protocol Core

Partially implemented, but currently blocked by the SAS mismatch. Server and Android do not compute the same verification code.

### #133 QR Scan/Paste Import

The Android import/scan flow appears implemented at the product layer. I did not do a camera/device UX pass in this review. The more important blocker is downstream pairing correctness.

### #134 Relay Control Plane + REST Pairing Submit

Server route and forwarding path exist, and Android has submitter/use case plumbing. Needs end-to-end proof after SAS is fixed.

### #135 SAS Confirmation + Trusted Computer Store

Store and confirmation flow exist locally, but real confirmation is blocked by SAS mismatch. Also clarify policy-unavailable behavior.

### #136 Trusted Computer Home Integration

Home projection and UI shell exist. It is not connected to a real daemon transport manager, and daemon sessions are not navigable or terminal-capable.

### #137 Transport Frame Protocol Foundation

The frame registry/reducer work is directionally good. Remaining work:

- byte-stream framing boundary for real QUIC streams;
- interop fixtures shared with server;
- production limits/error handling once native bridge exists.

### #138 QUIC Direct/Fallback Transport MVP

Not complete as MVP. Current branch implements scaffolding and deterministic JVM tests, not real direct/fallback QUIC/TLS. The existing Android evidence doc already marks this as partial.

### #119-#128 Earlier Milestone Work

The current state covers parts of the design, identity, pairing, realtime presence, fallback foundation, daemon-card projection, and path badge concepts. It does not yet complete:

- #124 daemon-owned session list/previews as the primary list;
- #125 terminal detail over daemon transport;
- #126 real Direct/Relay path badge driven by active transport;
- #127 hardening/e2e gate;
- #128 milestone umbrella.

## Recommended Next Sequence

1. Fix hard contract mismatches first: Android SAS vectors, fallback tunnel URL, candidate sanitizer defaults.
2. Add a minimal Android native QUIC connection API: packet input/output, timers, control stream read/write, uni stream read, and TLS signing/pinning.
3. Build fallback-only first using Relay tunnel. Prove `hello -> session_index -> path_state` on emulator and physical device.
4. Add stream-frame buffering in Android and test partial/multiple frame reads.
5. Wire a production daemon connection manager from trusted + visible computers to `DaemonTransportRuntime`.
6. Only after fallback works, add direct UDP attempt orchestration and direct-to-relay fallback evidence.
7. Migrate session list/detail incrementally: daemon session roster, preview subscribe, terminal snapshot/live bytes, input, resize.
8. Add e2e gates: pairing, revocation, fallback opacity, direct success/fallback degradation, and terminal interaction.

## Go/No-Go For Milestone 2

Current implementation is not ready to call milestone 2 complete.

Go criteria should be:

- Android and server pass shared SAS golden vectors.
- Android can pair with a real daemon through Relay and daemon-side SAS confirmation.
- Android can establish pinned QUIC/TLS over Relay fallback to a real daemon.
- Relay fallback carries opaque QUIC packets only.
- Android can receive daemon `session_index` and at least one preview/terminal snapshot over the daemon transport.
- Direct attempt either succeeds on a controlled real network or falls back cleanly with diagnostics and path state.

Until those are true, the honest status is: protocol/control-plane foundation mostly exists; Android direct/fallback daemon transport is still under construction.
