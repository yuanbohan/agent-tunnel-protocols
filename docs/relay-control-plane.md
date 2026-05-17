# Relay Connectivity Control Plane

## 状态

本文是 official mobile companion 和 Go daemon 使用的 Relay-owned connectivity control plane SSOT。

Relay 负责 authentication、account context、live daemon presence、pairing-response routing、direct rendezvous hint exchange、fallback tunnel authorization、opaque fallback packet forwarding。

Relay 不负责 session discovery、preview、terminal bytes、input、resize、interactive authorization。

流程图见：

- [Computer List](draws/00-computer-list.md)
- [Direct And Relay](draws/03-direct-relay-data-flow.md)

## Endpoint 清单

App endpoints：

- `GET /api/connectivity/ws`
- `POST /api/pairing/responses`
- `GET /api/account/policy`
- `POST /api/computers/:computerID/sessions`

Daemon endpoints：

- `GET /connectivity/computer/ws`

Fallback packet tunnel：

- `GET /connectivity/tunnel/ws`

旧 connectivity realtime aliases 不属于当前 compatibility line：

- `GET /api/connectivity/app/ws`
- `GET /connectivity/daemon/ws`

## App Authentication

App-facing Relay endpoints 使用：

```text
Authorization: Bearer <app-access-token>
```

App access token 对 client 是 opaque。Relay 在 server-side 存 app session ownership，并把 app session 绑定到 login/refresh 时提供的 `client_fingerprint`。

Relay 用 `(account_id, app_session_id, client_fingerprint)` 作为 app-side identity，用于：

- connectivity realtime registration
- pairing response submission
- trusted computer visibility
- rendezvous attempts
- fallback tunnel requests
- account policy reads

如果 app session 没有 bound client fingerprint，Relay 必须拒绝 connectivity realtime。

## Daemon Authentication

Daemon connectivity realtime 使用：

```text
Authorization: Bearer <agent-token>
```

Relay 把 daemon socket 绑定到 agent token 所属 account。Daemon-local trust 仍由 daemon 持有；Relay 只在 `computer_register` 和后续 `pair_completed` / `client_revoked` 中接收当前 live trusted roster。

## Shared JSON Envelope

Realtime message 是 JSON object：

- `type`
- optional `protocol_version`
- optional `request_id`
- event-specific fields

当前 realtime protocol version：

```text
2
```

Peer 在安全可忽略时要 tolerate unknown event types。Relay 对 unsupported app commands 应返回 `error` frame，而不是关闭一个有效 app socket。

## App Realtime Socket

App 打开：

```text
GET /api/connectivity/ws
Authorization: Bearer <app-access-token>
```

第一帧必须是：

```json
{"type":"app_register","protocol_version":2}
```

Relay 随后发送：

- `computer_snapshot`
- later `computer_visible`
- later `computer_removed`
- later `client_revoked`
- 发给该 app session 的 direct rendezvous 和 fallback events

`computer_snapshot` fields：

- `type`: `computer_snapshot`
- `computers`: array of `ConnectivityComputer`

`ConnectivityComputer` fields：

- `computer_id`
- `display_name`
- `platform_family`
- `platform_id`
- `computer_public_key`
- `computer_fingerprint`
- `tunnel_version`

Relay 通过 authenticated app account + client fingerprint 和 live daemon trusted roster 匹配，计算 visible computers。

## Daemon Realtime Socket

Daemon 打开：

```text
GET /connectivity/computer/ws
Authorization: Bearer <agent-token>
```

第一帧必须是 `computer_register`：

```json
{
  "type": "computer_register",
  "protocol_version": 2,
  "computer": {
    "computer_id": "dev_abcd1234",
    "display_name": "Work Mac",
    "platform_family": "macos",
    "platform_id": "macos",
    "computer_public_key": "<hex-ed25519-public-key>",
    "computer_fingerprint": "<hex-sha256-public-key>",
    "tunnel_version": "v0.1.0"
  },
  "trusted_clients": [
    {
      "fingerprint": "<client-fingerprint>",
      "display_name": "Pixel"
    }
  ],
  "direct_sessions": [
    {
      "attempt_id": "<attempt-id>",
      "client_fingerprint": "<client-fingerprint>"
    }
  ]
}
```

Relay 使用这个 registration 在 daemon reconnect 后重建 live visibility。Relay 不 durable persist `trusted_clients`。

## Pairing Event Family

Daemon to Relay：

- `pair_invitation_reserve`
- `pair_completed`
- `client_revoked`

Relay to daemon：

- `pair_invitation_reserved`
- `pair_response_forward`
- `error`

App to Relay：

- `POST /api/pairing/responses`

Relay to app：

- `computer_visible`
- `client_revoked`
- `computer_removed`

Pairing transcript 和 SAS 规则见 [pairing.md](pairing.md)。

## Direct Rendezvous Event Family

Direct attempts 通过 Relay realtime 交换 UDP candidate hints。

App 发送 `rendezvous_open`：

```json
{
  "type": "rendezvous_open",
  "request_id": "req-1",
  "attempt_id": "attempt-1",
  "computer_id": "dev_abcd1234",
  "public_udp_addr": "203.0.113.10:50000",
  "private_udp_addrs": ["10.0.0.5:50000"]
}
```

Relay 把 client-origin `rendezvous_hint` 转发给 daemon，并附上 authenticated `client_fingerprint`。

Daemon 回复 daemon-origin `rendezvous_hint`：

```json
{
  "type": "rendezvous_hint",
  "request_id": "req-1",
  "attempt_id": "attempt-1",
  "computer_id": "dev_abcd1234",
  "client_fingerprint": "<client-fingerprint>",
  "actor": "daemon",
  "public_udp_addr": "198.51.100.20:50000",
  "private_udp_addrs": ["10.0.0.8:50000"],
  "expires_at": 1777478400
}
```

任意一侧可发送 `rendezvous_close`。Direct QUIC accept 成功后，daemon 发送 `direct_session_open`，Relay 记录 direct 已赢得该 attempt。授权撤销时，Relay 可以发送 `direct_session_close` 给 daemon，让绑定该 live authorization 的 direct transport 尽快关闭。

Rendezvous rules：

- `attempt_id` 由 app 每次 connection attempt 生成。
- hints 是 short-lived；当前实现默认 30 seconds。
- 同一 app session + computer 的新 attempt 会 supersede 老 attempt。
- candidate list 必须有上限。
- private candidate addresses 必须限制在 private、link-local 或明确 test-allowed ranges。
- endpoint 和 Relay 在 forwarding/probing 前都应拒绝 loopback、unspecified、multicast、broadcast、unexpected public IP、以及非 test mode 下的 documentation ranges。只做 `ip:port` parse 不足以满足当前安全规则。

Relay 可以 route candidate hints，但不得从中推导 terminal/session semantics。

## Fallback Relay Tunnel Event Family

App 在 direct skipped、failed 或 timeout 后发送 `relay_tunnel_request`：

```json
{
  "type": "relay_tunnel_request",
  "request_id": "req-2",
  "attempt_id": "attempt-1",
  "computer_id": "dev_abcd1234",
  "fallback_reason": "direct_timeout",
  "direct_setup_latency_ms": 3000
}
```

Relay 只在 authenticated app account + bound client fingerprint 当前对 online daemon 有 pairing-derived visibility 时授权 fallback。

Relay 发送 side-specific `relay_tunnel_ready`：

```json
{
  "type": "relay_tunnel_ready",
  "request_id": "req-2",
  "attempt_id": "attempt-1",
  "computer_id": "dev_abcd1234",
  "client_fingerprint": "<client-fingerprint>",
  "actor": "client",
  "tunnel_token": "<single-use-token>",
  "fallback_reason": "direct_timeout",
  "direct_setup_latency_ms": 3000
}
```

Daemon 收到另一枚 token，`actor: "daemon"`。

Tunnel-token rules：

- 每侧一枚 token。
- short-lived。
- single-use。
- 绑定 attempt id、account、app session、client fingerprint、target computer、actor identity、actor type。
- 在 expiry、disconnect、logout、token revocation、user deletion、daemon disconnect、superseding attempt、trusted-device revocation 时失效。

## Fallback Packet Tunnel

两端使用各自 token 连接：

```text
GET /connectivity/tunnel/ws
Authorization: Bearer <single-use-token>
```

WebSocket 只承载 binary messages。每个 binary message 是一个 opaque encrypted QUIC packet。Relay 按同一 attempt 关联 client/daemon endpoints，并原样转发 binary packets。

Relay 必须在 text message、invalid token、token reuse、authorization revocation、peer disconnect 或 tunnel expiry 时关闭 tunnel。

## Account Policy Surface

Relay 通过 authenticated app API 暴露 account policy。当前 mobile product policy 使用：

- `free`: app 可以保留 1 台 active trusted computer
- `pro`: app 可以保留最多 10 台 active trusted computers

Relay 不发 per-session grants，不决定哪条 daemon session row 可打开，也不在当前 compatibility line 给 daemon 发送 tier policy。

## Relay Must Not Carry

Relay realtime 和 fallback tunnel payload 不得承载 plaintext：

- session list
- recent output preview text
- terminal snapshot bytes
- live terminal bytes
- mobile input
- resize payloads
- daemon-side interactive grants
- daemon session metadata from QUIC transport

这些属于 [protocol.md](protocol.md) 定义的 pinned daemon transport。

## Failure 语义

Relay failure 会影响：

- sign-in / token refresh
- new pairing
- 当前 trusted-computer visibility updates
- new direct rendezvous
- new fallback tunnel setup
- mobile-created launch requests
- account policy refresh

Relay failure 不让 Relay 获得读取 daemon transport payload 的能力。现有 direct transport 可以继续，直到自身 path 关闭，或授权撤销通过其他机制传达到 endpoint。

Revocation 的 authoritative source 是 daemon-local trusted roster。Daemon revoke 必须立即关闭本地 direct/fallback transports 和 interactive ownership；Relay notification 是 best-effort，用来清理 live visibility、fallback token 和 rendezvous state。Relay offline 或 notification failure 不会恢复被 revoke client 的 daemon transport 权限；下一次 daemon registration 会重新发布过滤后的 trusted roster。
