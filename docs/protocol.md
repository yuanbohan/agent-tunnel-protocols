# Daemon Transport Protocol

## Status

This document is the cross-repository source of truth for the current
daemon-to-mobile connectivity transport. It defines the protocol surface shared
by the official mobile companion and the Go tunnel/daemon implementation.

Implementation repositories may keep local mirrors, implementation notes, and
tests, but compatibility decisions for the frame registry, payload families,
transport security invariants, and session metadata boundary should be made
here first.

## Ownership Boundary

The daemon transport carries mobile-visible session state and interactive
terminal traffic after a mobile client has selected and connected to one
trusted computer.

It owns:

- daemon-to-mobile session roster synchronization
- preview subscription and preview snapshots
- interactive session grant/denial/release
- terminal snapshot and live-byte streams
- mobile-to-daemon input and resize messages
- transport path diagnostics

It does not own:

- Relay app authentication
- Relay pairing transport
- Relay computer presence
- Relay rendezvous or fallback tunnel authorization
- account tier policy
- classic Relay session list/detail/attach APIs
- local daemon broker mechanics that are not mobile-visible

Relay may route encrypted fallback QUIC packets, but it must not parse this
session transport or derive terminal/session semantics from it.

## Transport Security

The transport uses QUIC with TLS 1.3. Both daemon and mobile endpoints present
self-signed Ed25519 certificates whose SubjectPublicKeyInfo is the paired device
public key.

Implementations must enforce:

- mutual certificate presentation
- peer public-key pinning against the paired device identity
- ALPN `tunnel-conn/1`
- no QUIC 0-RTT use for terminal input or session traffic
- fresh transport session keys per connection

Certificate chain validation is not the trust root for this transport. Pairing
establishes the trusted public keys; TLS proves possession and derives fresh
session keys.

TLS 1.3 chooses the concrete ephemeral key exchange group, HKDF traffic-key
derivation, and AEAD packet protection through the QUIC/TLS implementation.
This compatibility line requires TLS 1.3, Ed25519 endpoint identities,
certificate/public-key pinning, ALPN `tunnel-conn/1`, no 0-RTT, and fresh
per-connection traffic keys; it does not require one fixed TLS cipher suite.

## Path Modes And Packet Carriers

The daemon transport is path-agnostic above QUIC. Direct and Relay fallback
use the same TLS identity checks, control stream, interactive streams, frame
registry, and JSON payload families.

Direct path:

- QUIC packets are sent over UDP between Android and daemon.
- Relay realtime carries short-lived rendezvous hints.
- STUN is used only to discover observed public UDP addresses.
- Relay may assist discovery but cannot decrypt daemon transport payloads.

Relay fallback path:

- Relay realtime issues side-specific, short-lived, single-use tunnel tokens.
- Android and daemon redeem those tokens at `GET /connectivity/tunnel/ws`.
- The fallback WebSocket carries binary QUIC packets only.
- Relay forwards encrypted packets unchanged and must not parse session frames.

Relay fallback is a carrier for encrypted QUIC packets, not a terminal/session
protocol. Any behavior difference between direct and Relay fallback should be
limited to availability and diagnostics, not trust or plaintext access.

## Protocol Version

`hello.protocol_version` is a single integer. The current daemon transport
version is:

```text
2
```

Version `2` uses JSON payloads for typed control frames. A future CBOR,
protobuf, or otherwise non-JSON encoding profile requires a new protocol
version or an explicit compatibility-line decision. Implementations must not
silently reinterpret version `2` as a different payload encoding.

There is no in-band downgrade negotiation in this revision. If a peer sends a
different version, the receiver closes the session with
`protocol_version_mismatch`.

## Stream Model

One daemon connection uses:

- one long-lived bidirectional control stream opened by the mobile client
- zero or more daemon-initiated unidirectional interactive streams

The control stream carries session metadata, preview, interactive control,
input, resize, path diagnostics, and error frames. Each interactive stream
belongs to one granted interactive session lifetime and carries only that
session's snapshot boundary frames, snapshot chunks, and live terminal bytes
from daemon to mobile.

The same session does not support multiple concurrent interactive attach
lifetimes in this revision.

## Frame Wire Format

Every control or interactive stream frame uses:

```text
[1-byte type] [varint payload_length] [payload bytes]
```

`payload_length` is the byte length of the payload.

`payload_length` uses the QUIC variable-length integer encoding: the two most
significant bits of the first byte encode the total integer length (`00` = 1
byte, `01` = 2 bytes, `10` = 4 bytes, `11` = 8 bytes), and the remaining bits
encode an unsigned big-endian integer value.

The maximum payload size is 1 MiB (`1 << 20`) per frame. Senders must split
larger terminal snapshots or live output bursts across multiple
`snapshot_chunk` or `live_bytes` frames. Receivers must reject frames whose
declared payload length is larger than 1 MiB.

Typed control payloads are JSON encoded as UTF-8. `snapshot_chunk` and
`live_bytes` carry raw PTY bytes and are not JSON wrapped.

Receivers must tolerate unknown frame type values by dropping or ignoring them
without closing the whole transport. Receivers must ignore unknown JSON fields
inside known JSON payloads.

Malformed frames, invalid varints, oversized payloads, incomplete payloads, and
malformed JSON for known JSON frame families remain errors.

## Frame Type Registry

| Type | Name | Payload |
|---|---|---|
| `0x01` | `hello` | JSON |
| `0x02` | `session_index` | JSON |
| `0x03` | `preview_subscribe` | JSON |
| `0x04` | `session_upsert` | JSON |
| `0x05` | `session_gone` | JSON |
| `0x06` | `preview_unsubscribe` | JSON |
| `0x07` | `preview_snapshot` | JSON |
| `0x08` | `interactive_request` | JSON |
| `0x09` | `interactive_granted` | JSON |
| `0x0a` | `interactive_denied` | JSON |
| `0x0b` | `interactive_release` | JSON |
| `0x0c` | `input_text` | JSON |
| `0x0d` | `input_key` | JSON |
| `0x0e` | `resize` | JSON |
| `0x0f` | `path_state` | JSON |
| `0x10` | `snapshot_begin` | JSON |
| `0x11` | `snapshot_chunk` | raw bytes |
| `0x12` | `live_bytes` | raw bytes |
| `0x13` | `snapshot_end` | JSON |
| `0x7f` | `error` | JSON |

Frame type values not listed here are unassigned in this compatibility line.

## Control Stream Ordering

After QUIC/TLS connection setup:

1. mobile sends `hello`
2. daemon validates `hello`
3. daemon sends `hello`
4. daemon sends `session_index`
5. daemon may send `path_state`
6. only after `session_index` may daemon send session deltas, preview
   snapshots, or interactive grant/denial controls

Mobile clients must not process session or preview frames before applying the
initial `session_index`.

## JSON Payload Families

The field lists below define the compatibility surface. Receivers must ignore
unknown JSON fields.

### `hello`

- `protocol_version`
- `actor_type`: `mobile` or `daemon`
- `client_fingerprint`
- `path_kind`: `direct` or `relay`

### `session_index`

- `sessions`: array of `SessionMetadata`

The daemon sends the full current session set known to the selected computer.
If a Relay-launched `tunnel run` has registered with the local daemon before
the mobile transport connects, the resulting `session_id` must be present in
the initial `session_index`.

### `session_upsert`

- `session`: `SessionMetadata`

The daemon sends a full replacement metadata object when a session appears or
changes. If a Relay-launched `tunnel run` registers after the mobile transport
is already connected, the daemon publishes that same `session_id` as a
`session_upsert`.

### `session_gone`

- `session_id`

### `preview_subscribe`

- `session_id`

### `preview_unsubscribe`

- `session_id`

### `preview_snapshot`

- `session_id`
- `preview`
- `updated_at` (optional)

Preview text is delivered separately from session metadata so list rows and
preview content can update independently.

### `interactive_request`

- `session_id`
- `cols`
- `rows`

### `interactive_granted`

- `session_id`
- `interactive_stream_id`
- `cols`
- `rows`

`interactive_stream_id` is the QUIC stream id of the daemon-initiated
unidirectional stream that carries the interactive snapshot and live bytes.
That stream begins with `snapshot_begin`, then zero or more `snapshot_chunk`
frames, then `snapshot_end`; later `live_bytes` frames for the same interactive
lifetime follow on the same stream.

### `interactive_denied`

- `session_id`
- `reason`

Receivers must tolerate unknown `reason` values. Current known values are:

- `device_not_trusted`
- `session_unavailable`
- `daemon_busy`
- `unknown`

### `interactive_release`

- `session_id`

### `input_text`

- `session_id`
- `text`
- `submit` (optional)

### `input_key`

- `session_id`
- `key`

### `resize`

- `session_id`
- `cols`
- `rows`

### `path_state`

- `attempt_id` (optional)
- `path_kind`
- `fallback_reason` (optional)
- `direct_setup_latency_ms` (optional)
- `relay_setup_latency_ms` (optional)

`path_state` is advisory. The app connection manager remains the source of the
current path badge. Daemon transport reports diagnostics for the path that was
used; it does not override app-side path selection or badge authority.

### `error`

- `code`
- `message` (optional)

Receivers must tolerate unknown `code` values.

### `snapshot_begin`

Sent on the daemon-initiated interactive stream announced by
`interactive_granted.interactive_stream_id`.

- `session_id`
- `cols`
- `rows`

### `snapshot_chunk`

Raw PTY snapshot bytes. No JSON wrapper.

### `snapshot_end`

Sent on the same daemon-initiated interactive stream as the preceding
`snapshot_begin` and `snapshot_chunk` frames.

- `session_id` (optional)
- `chunk_count` (optional)

### `live_bytes`

Raw PTY output bytes after snapshot completion. No JSON wrapper.

## Session Metadata

`SessionMetadata` is the daemon transport session payload shape used by
`session_index` and `session_upsert`.

Current fields:

- `session_id`
- `label`
- `command_preview`
- `cwd`
- `git_branch`
- `started_at`
- `updated_at`
- `online`

The selected computer transport already scopes metadata to one trusted
computer. `session_id` is sufficient for matching Relay launch correlation
results to later daemon transport state in this revision.

`SessionMetadata` must not carry:

- terminal snapshot bytes
- live terminal bytes
- preview text payloads such as `preview` or `latest_preview`
- account tier, entitlement, subscription, or policy fields
- Relay-only launch correlation fields
- direct/fallback path authority fields

Preview text is delivered with `preview_snapshot`. Terminal bytes are delivered
on interactive streams. Tier and account policy remain outside the daemon
transport.

## Interactive Recovery

After reconnect, the mobile client reopens a fresh daemon transport, reapplies
desired preview subscriptions, and sends new `interactive_request` messages for
sessions it still wants to use interactively. The daemon responds with fresh
grant/denial results and new interactive streams.

This protocol does not provide missed-byte replay. Recovery is based on fresh
session metadata, fresh previews, and fresh interactive snapshots.

## Compatibility Notes

- Unknown JSON fields are ignored.
- Unknown frame types are dropped or ignored.
- Unknown `interactive_denied.reason` and `error.code` values are tolerated.
- Version `2` payloads are JSON. Non-JSON profiles require a new version or
  compatibility-line decision.
- The Relay fallback packet tunnel is opaque to this protocol.

## Fixture Hygiene

Any future fixtures for this protocol must be synthetic and non-secret.
Fixtures must not contain real credentials, private keys, tunnel tokens, device
fingerprints, terminal captures, private file paths, or user input.
