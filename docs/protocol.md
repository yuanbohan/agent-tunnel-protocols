# Daemon Transport Protocol

## 状态

本文是 daemon-to-mobile connectivity transport 的跨仓库 SSOT。它定义 official mobile companion 和 Go tunnel/daemon 共享的 protocol surface。

实现仓库可以保留 local mirror、implementation notes 和 tests，但 frame registry、payload families、transport security invariants、session metadata boundary 的兼容性决策要先在这里更新。

相关流程图：

- [Direct And Relay](draws/03-direct-relay-data-flow.md)
- [Detail Input](draws/04-detail-input.md)

## Ownership Boundary

Daemon transport 在 mobile client 已选择并连接到一台 trusted computer 后，承载 mobile-visible session state 和 interactive terminal traffic。

它负责：

- daemon-to-mobile session roster synchronization
- preview subscription 和 preview snapshots
- interactive session grant / denial / release
- terminal snapshot 和 live-byte streams
- mobile-to-daemon input 和 resize messages
- transport path diagnostics

它不负责：

- Relay app authentication
- Relay pairing transport
- Relay computer presence
- Relay rendezvous 或 fallback tunnel authorization
- account tier policy
- classic Relay session list/detail/attach APIs
- 不属于 mobile-visible contract 的 local daemon broker mechanics

Relay 可以 route encrypted fallback QUIC packets，但不得解析 session transport 或从中推导 terminal/session semantics。

## Transport Security

Transport 使用 QUIC with TLS 1.3。Daemon 和 mobile endpoints 都展示 self-signed Ed25519 certificates；certificate 的 SubjectPublicKeyInfo 是 pairing 时建立的 device public key。

实现必须 enforce：

- mutual certificate presentation
- peer public-key pinning against paired device identity
- ALPN `tunnel-conn/1`
- terminal input 和 session traffic 不使用 QUIC 0-RTT
- 每次 connection 都生成 fresh transport session keys

Certificate chain validation 不是 trust root。Pairing 建立 trusted public keys；TLS 证明 private key possession 并派生 fresh session keys。

TLS 1.3 的具体 ephemeral key exchange group、HKDF traffic-key derivation、AEAD packet protection 由 QUIC/TLS implementation 协商。本 compatibility line 要求 TLS 1.3、Ed25519 endpoint identities、certificate/public-key pinning、ALPN `tunnel-conn/1`、no 0-RTT、fresh per-connection traffic keys；不固定一个 TLS cipher suite。

## Path Modes And Packet Carriers

Daemon transport 在 QUIC 之上 path-agnostic。Direct 和 Relay fallback 使用相同 TLS identity checks、control stream、interactive streams、frame registry、JSON payload families。

Direct path：

- QUIC packets 直接走 Android 和 daemon 之间的 UDP。
- Relay realtime 只承载 short-lived rendezvous hints。
- STUN 只用于发现 observed public UDP address。
- Relay 可辅助 discovery，但不能解密 daemon transport payload。

Relay fallback path：

- Relay realtime 发 side-specific、short-lived、single-use tunnel tokens。
- Android 和 daemon 用 token 连接 `GET /connectivity/tunnel/ws`。
- Fallback WebSocket 只承载 binary QUIC packets。
- Relay 原样转发 encrypted packets，不解析 session frames。

Relay fallback 是 encrypted QUIC packets 的 carrier，不是 terminal/session protocol。Direct 和 Relay fallback 的差异应限于可用性和 diagnostics，不应影响 trust 或 plaintext access。

## Protocol Version

`hello.protocol_version` 是单个 integer。当前 daemon transport version：

```text
2
```

Version `2` 对 typed control frames 使用 JSON payload。未来 CBOR、protobuf 或其他 non-JSON encoding profile 需要新的 protocol version 或明确 compatibility-line decision。实现不得把 version `2` 静默解释成其他 encoding。

本版本没有 in-band downgrade negotiation。Peer version 不匹配时，receiver 使用 `protocol_version_mismatch` 关闭 session。

## Stream Model

一个 daemon connection 使用：

- 一个 mobile client 打开的 long-lived bidirectional control stream
- 零个或多个 daemon-initiated unidirectional interactive streams

Control stream 承载 session metadata、preview、interactive control、input、resize、path diagnostics、error frames。

每个 interactive stream 属于一次 granted interactive session lifetime，只承载这个 session 的 snapshot boundary frames、snapshot chunks、live terminal bytes，方向是 daemon -> mobile。

同一个 session 在当前版本不支持多个 concurrent interactive attach lifetimes。

## Frame Wire Format

所有 control 或 interactive stream frame 使用：

```text
[1-byte type] [varint payload_length] [payload bytes]
```

`payload_length` 是 payload byte length。

`payload_length` 使用 QUIC variable-length integer encoding：第一个 byte 的两个最高位表示总 integer length（`00` = 1 byte，`01` = 2 bytes，`10` = 4 bytes，`11` = 8 bytes），剩余 bits 表示 unsigned big-endian integer。

每个 frame 最大 payload size 是 1 MiB (`1 << 20`)。更大的 terminal snapshots 或 live output bursts 必须拆成多个 `snapshot_chunk` 或 `live_bytes` frames。Receiver 必须拒绝 declared payload length 超过 1 MiB 的 frame。

Typed control payload 使用 UTF-8 JSON。`snapshot_chunk` 和 `live_bytes` 承载 raw PTY bytes，不包 JSON。

Receiver 必须 tolerate unknown frame type：drop/ignore，不关闭整个 transport。Known JSON payload 内 unknown fields 也要 ignore。

Malformed frame、invalid varint、oversized payload、incomplete payload、known JSON frame 的 malformed JSON 仍然是 error。

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

未列出的 frame type values 在当前 compatibility line 中未分配。

## Control Stream Ordering

QUIC/TLS connection setup 后：

1. mobile 发送 `hello`
2. daemon 验证 `hello`
3. daemon 发送 `hello`
4. daemon 发送 `session_index`
5. daemon 可以发送 `path_state`
6. 只有在 `session_index` 之后，daemon 才能发送 session deltas、preview snapshots、interactive grant/denial controls

Mobile client 在 apply initial `session_index` 前，不得处理 session 或 preview frames。

## JSON Payload Families

下面字段列表定义 compatibility surface。Receiver 必须 ignore unknown JSON fields。

### `hello`

- `protocol_version`
- `actor_type`: `mobile` or `daemon`
- `client_fingerprint`
- `path_kind`: `direct` or `relay`

### `session_index`

- `sessions`: array of `SessionMetadata`

Daemon 发送所选 computer 当前全部 session set。如果 Relay-launched `tunnel run` 在 mobile transport 连接前已经注册到 local daemon，初始 `session_index` 必须包含它的 `session_id`。

### `session_upsert`

- `session`: `SessionMetadata`

Session 出现或变化时，daemon 发送完整 replacement metadata object。如果 Relay-launched `tunnel run` 在 mobile transport 已连接后注册，daemon 用 `session_upsert` 发布同一个 `session_id`。

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

Preview text 独立于 session metadata 传输，这样 list row 和 preview content 可以独立更新。

### `interactive_request`

- `session_id`
- `cols`
- `rows`

### `interactive_granted`

- `session_id`
- `interactive_stream_id`
- `cols`
- `rows`

`interactive_stream_id` 是 daemon-initiated unidirectional stream 的 QUIC stream id。该 stream 先发 `snapshot_begin`，然后零个或多个 `snapshot_chunk`，再发 `snapshot_end`，之后同一 interactive lifetime 的 `live_bytes` 继续走同一 stream。

### `interactive_denied`

- `session_id`
- `reason`

Receiver 必须 tolerate unknown `reason` values。当前 known values：

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

`path_state` 是 advisory。App connection manager 仍是 current path badge 的 authority。Daemon transport 只报告所用 path 的 diagnostics，不覆盖 app-side path selection 或 badge authority。

### `error`

- `code`
- `message` (optional)

Receiver 必须 tolerate unknown `code` values。

### `snapshot_begin`

在 `interactive_granted.interactive_stream_id` 指向的 daemon-initiated interactive stream 上发送。

- `session_id`
- `cols`
- `rows`

### `snapshot_chunk`

Raw PTY snapshot bytes。无 JSON wrapper。

### `snapshot_end`

在同一个 interactive stream 上发送。

- `session_id` (optional)
- `chunk_count` (optional)

### `live_bytes`

Snapshot completion 后的 raw PTY output bytes。无 JSON wrapper。

## Session Metadata

`SessionMetadata` 是 `session_index` 和 `session_upsert` 使用的 daemon transport session payload shape。

当前字段：

- `session_id`
- `label`
- `command_preview`
- `cwd`
- `git_branch`
- `started_at`
- `updated_at`
- `online`

所选 computer transport 已经把 metadata scope 限制在一个 trusted computer 内。当前版本用 `session_id` 足够把 Relay launch correlation result 匹配到后续 daemon transport state。

`SessionMetadata` 不得携带：

- terminal snapshot bytes
- live terminal bytes
- preview text payloads，例如 `preview` 或 `latest_preview`
- account tier、entitlement、subscription、policy fields
- Relay-only launch correlation fields
- direct/fallback path authority fields
