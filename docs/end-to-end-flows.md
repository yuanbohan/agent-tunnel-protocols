# Connectivity End-To-End Flows

## Status

This document is the cross-repository source of truth for the user-visible
connectivity flows shared by the Android companion, Relay/server
implementation, and Go tunnel/daemon implementation.

It explains how the protocol documents fit together:

- pairing and long-lived trust: [pairing.md](pairing.md)
- Relay realtime, presence, rendezvous, and fallback setup:
  [relay-control-plane.md](relay-control-plane.md)
- daemon-to-mobile QUIC session transport: [protocol.md](protocol.md)

Implementation repositories may keep local notes, tests, and operational
guides, but protocol and data-flow decisions should point back here.

## Actors And Repositories

- Android companion (`agent-tunnel-android`): app login, client identity,
  secure local stores, pairing UI, trusted-computer list, direct-first
  connection manager, Relay fallback packet carrier, session list, previews,
  and session detail UI.
- Go tunnel/daemon/Relay/STUN (`agent-tunnel`): Relay HTTP/WebSocket control
  plane, Binding-only STUN service, local tunnel daemon, local broker,
  pairing state, trusted client roster, direct UDP accept path, fallback packet
  tunnel, and daemon session transport.
- Protocol SSOT (`agent-tunnel-protocols`): cross-repository protocol,
  security, and compatibility decisions.

## 0. Trusted Computer List

The trusted computer list is the intersection of local app trust, Relay-visible
daemon presence, and account policy.

Android maintains a protected local `TrustedComputerStore` scoped by:

- `account_id`
- `relay_base_url`
- `computer_id`
- daemon fingerprint

Relay maintains live-only visibility. A daemon becomes visible to one app
session only when all of these are true:

1. The app is authenticated with a Relay app bearer token.
2. The app session is bound server-side to the app client fingerprint.
3. The daemon is authenticated with an agent token owned by the same account.
4. The daemon registered on `GET /connectivity/computer/ws`.
5. The daemon registration contains a trusted client roster that includes the
   app client's fingerprint, or a valid `pair_completed` event created that
   live visibility grant.

Android opens `GET /api/connectivity/ws`, sends:

```json
{"type":"app_register","protocol_version":2}
```

Relay replies with `computer_snapshot` and later emits `computer_visible`,
`computer_removed`, and `client_revoked`.

The Android list projection does not treat Relay presence as durable trust.
It joins:

- local trusted records from protected storage
- Relay realtime computers matched by `(computer_id, computer_fingerprint)`
- account policy (`free` or `pro`)
- per-computer daemon connection state

Local trust without Relay presence is shown as offline. Relay presence without
matching local trust is not enough to create a trusted-computer row. Once a
trusted computer is both locally trusted and Relay-visible, Android may start a
daemon connection for that computer.

## 1. Pairing Flow

Pairing binds one Android app installation to one daemon computer identity.
Relay transports messages but is not the trust authority.

High-level flow:

1. The user runs `tunnel pair` on the computer.
2. The daemon reserves a short-lived pairing correlation through Relay using
   `pair_invitation_reserve`.
3. Relay replies with `pair_invitation_reserved`, including the authenticated
   account id and correlation id.
4. The daemon creates a version `2` signed invitation, persists invitation state
   locally, and prints a QR/paste payload.
5. Android imports the invitation and verifies version, expiry, account id,
   daemon fingerprint, and the daemon Ed25519 signature.
6. Android loads or creates its app client Ed25519 identity, signs the Android
   response transcript, and submits it to Relay with
   `POST /api/pairing/responses`.
7. Relay verifies the app bearer session, checks that the submitted
   `account_id` and `client_fingerprint` match the authenticated app session,
   and forwards the response to the daemon as `pair_response_forward`.
8. The daemon verifies the Android Ed25519 signature, computes the 6-digit SAS,
   and stores the response as pending.
9. Both sides display the same SAS derived from the pairing transcript.
10. The user confirms the SAS on the daemon CLI.
11. The daemon persists the trusted Android client and sends `pair_completed`
    to Relay.
12. Relay emits `computer_visible` to matching app peers.
13. Android waits until that paired computer is visible, then persists the
    daemon trust record locally and clears pending pairing state.

Detailed canonical transcripts, signatures, SAS, persistence, and revocation
rules are defined in [pairing.md](pairing.md).

Security summary:

- Long-lived device identity and pairing signatures use Ed25519.
- Public-key fingerprints are `SHA-256(raw_ed25519_public_key)` encoded as
  lowercase hex.
- The SAS is `SHA-256` over length-prefixed daemon key, client key,
  invitation id, and nonce, truncated to a 6-digit decimal code by taking the
  first four digest bytes as a big-endian integer modulo `1_000_000`.
- Pairing does not create a long-lived symmetric secret. It creates pinned peer
  identities that later authenticate fresh TLS 1.3 handshakes.
- Relay can withhold or forward pairing messages, but it cannot rewrite signed
  transcript fields without breaking signatures.
- A network-side SAS comparison is not trusted; the user compares the code
  out-of-band.

## 2. Session List And Recent Output Preview

Relay is not the session authority. It may launch a session on an online
computer, but the official mobile companion receives session rows and previews
from the selected daemon transport.

For an already trusted and visible computer:

1. Android starts a direct-first daemon connection.
2. The chosen path establishes the daemon transport defined in
   [protocol.md](protocol.md).
3. Android sends daemon transport `hello`.
4. The daemon validates the pinned client identity and sends daemon `hello`.
5. The daemon sends `session_index` with the full current session metadata for
   that computer.
6. Android renders one computer section with daemon session rows.
7. Android subscribes only to visible row previews with `preview_subscribe`.
8. The daemon sends `preview_snapshot` for subscribed sessions and later
   `session_upsert` / `session_gone` deltas as local broker state changes.

For a mobile-created launch:

1. Android asks Relay to launch with `POST /api/computers/:computerID/sessions`.
2. Relay routes the request to the online daemon.
3. The daemon starts a tmux-backed `tunnel run` and waits for local broker plus
   Relay launch readiness.
4. Relay returns `session_ready` with a `session_id`.
5. Android treats that `session_id` as a correlation key only. The visible row
   must still arrive from daemon transport `session_index` or `session_upsert`.

The list preview area is a recent output preview, not a chat message. Preview
text is a separate daemon transport payload and must not be embedded inside
`SessionMetadata`.

## 3. Direct And Relay Data Flow

Direct and Relay fallback use the same daemon session protocol. Only the packet
carrier below QUIC changes.

Direct path:

```text
Android app
  -> Relay app realtime: rendezvous_open
  -> Relay forwards client candidate hint
  -> daemon sends daemon candidate hint
  -> Android + daemon send UDP probes
  -> QUIC/TLS 1.3 over direct UDP
  -> daemon transport frames
```

Relay fallback path:

```text
Android app
  -> Relay app realtime: relay_tunnel_request
  -> Relay issues single-use side-specific tunnel tokens
  -> Android opens /connectivity/tunnel/ws with client token
  -> daemon opens /connectivity/tunnel/ws with daemon token
  -> Relay forwards binary WebSocket messages as opaque QUIC packets
  -> QUIC/TLS 1.3 over packet tunnel
  -> daemon transport frames
```

Relay fallback is not less secure than direct at the daemon transport layer:

- both paths use the same pinned peer identities
- both paths negotiate fresh TLS 1.3 session keys
- Relay fallback forwards encrypted QUIC packets only
- Relay must not parse QUIC, terminal bytes, previews, snapshots, input,
  resize, path badges, or daemon session semantics

Relay can degrade availability by withholding rendezvous hints or refusing
fallback setup. Relay cannot decrypt direct or fallback daemon transport
payloads.

## 4. Session Detail And Mobile Input

Opening session detail attaches to one daemon session through the already
connected trusted-computer transport.

The attach flow is path-agnostic:

1. Android decodes the route identity into `(computer_id, daemon_fingerprint,
   attempt_id, session_id)`.
2. Android sends `interactive_request` on the daemon transport control stream
   with the target `session_id` and initial terminal `cols` / `rows`.
3. The daemon broker grants exactly one interactive owner for that session
   lifetime or denies with a stable reason such as `session_unavailable` or
   `daemon_busy`.
4. On grant, the daemon sends `interactive_granted` on the control stream and
   opens a daemon-initiated unidirectional interactive stream.
5. The interactive stream sends `snapshot_begin`, zero or more
   `snapshot_chunk` frames, `snapshot_end`, and then `live_bytes`.
6. Android renders snapshot and live bytes through the terminal pipeline.
7. Android enables input only after the snapshot completes.

Mobile input never travels on the interactive stream. It is sent on the control
stream:

- normal typing, pasted text, IME committed text, draft sync, and submit use
  `input_text`
- special keys use `input_key`
- protocol-level resize uses `resize`
- release uses `interactive_release`

The daemon validates that the sender currently owns an active interactive grant
for that session before routing input to the local broker and PTY owner.

Current Android detail behavior sends the initial geometry in
`interactive_request` and routes text/key input through the control stream.
The protocol supports `resize`; implementation repositories should not claim
live Android geometry updates are emitted unless that UI path is wired in code.

## 5. Key Storage, App Uninstall, And Re-Pairing

Android stores the client identity and trust state in app-local protected
storage:

- primary client Ed25519 identity: Android Keystore key pair
- fallback client signing identity: encrypted seed in AndroidX
  `EncryptedSharedPreferences`
- trusted computers: encrypted local store scoped by account and relay base URL
- pending pairing: encrypted local store
- Relay credentials: encrypted local config store

The daemon stores local connectivity identity and pairing state under
daemon-local state, with private file permissions:

- daemon Ed25519 connectivity identity
- invitation records
- pending pairing responses
- trusted Android client roster
- revoked client records

Relay does not keep the durable trusted-client database. It keeps live
authorization/routing state derived from authenticated app sessions, daemon
registrations, pairing correlations, direct rendezvous attempts, direct session
records, and fallback tunnel tokens.

When the Android app is uninstalled, Android removes app data and the app's
Keystore entries. The reinstalled app therefore has:

- no previous client private key
- a new client public key and fingerprint
- no local trusted-computer records
- no pending pairing records
- no Relay app session tokens

The daemon may still have the old client fingerprint in its trusted roster, but
the reinstalled app cannot prove possession of that old private key and cannot
authenticate as that old fingerprint. The user must sign in and pair again.

This is intentional. Restoring trust without the original device private key
would let a reinstall or backup restore impersonate a previously trusted
client installation.
