# Relay API

## 状态

本文是 Agent Tunnel 当前 public Relay API 的跨仓库 SSOT。

已在 2026-05-17 对照 `agent-tunnel/internal/relay/handler/new.go` 验证 public endpoint inventory。原 `agent-tunnel/docs/api.md` 的详细实现文档已经迁到这里；实现仓库只在 README/AGENTS/CLAUDE 保留导航和 implementation entry points。

Relay API 负责 authentication、account policy、computer launch control plane、pairing response forwarding、connectivity realtime、direct rendezvous、fallback tunnel setup。它不是 mobile session data plane，不提供 session list、session detail、terminal attach、terminal frame replay、preview storage 或 input/resize API。

相关文档：

- [Architecture](architecture.md)
- [End-To-End Flows](end-to-end-flows.md)
- [Pairing](pairing.md)
- [Relay Control Plane](relay-control-plane.md)
- [Daemon Transport Protocol](protocol.md)

## Endpoint 清单

Public app HTTP：

- `GET /healthz`
- `POST /api/auth/register`
- `POST /api/auth/login`
- `POST /api/auth/refresh`
- `POST /api/auth/logout`
- `POST /api/auth/password/change`
- `GET /api/account/policy`
- `GET /api/agent-tokens`
- `POST /api/agent-tokens`
- `DELETE /api/agent-tokens/:tokenID`
- `GET /api/computers`
- `POST /api/computers/:computerID/sessions`
- `POST /api/pairing/responses`

Realtime 和 packet tunnel：

- `GET /api/connectivity/ws`
- `GET /connectivity/computer/ws`
- `GET /connectivity/tunnel/ws`
- `GET /agent/ws`
- `GET /device/ws`

Operator routes 位于 local operator surface，不属于 public `/api/` contract。

Local-only/internal routes 不属于 public Relay API contract：

- `GET /internal/version`：local-only version probe，受 Relay local-only middleware 保护。

旧 endpoint 不属于当前 compatibility line：

- Relay session list / stop / attach / frame routes
- `GET /api/devices`
- `POST /api/devices/:id/launch`
- `GET /api/connectivity/app/ws`
- `GET /connectivity/daemon/ws`
- older global update websocket

## Base URL

Hosted relay 默认是：

```text
https://agentunnel.cn
```

Dev/prod Android flavor 可以选择不同 package name，但当前默认 Relay 都指向同一个 hosted base URL。具体 build flavor 的 base URL 是实现仓库配置，不是 protocol 字段。

## Token Types

| Token | 使用方 | 用途 |
|---|---|---|
| App access token | Android companion 或其他 app client | app-facing `/api/...` route 和 `GET /api/connectivity/ws` 的 `Authorization: Bearer` |
| App refresh token | app client | `POST /api/auth/refresh` body，用来 rotate app session |
| Agent token | `tunnel`、`tunnel daemon` | `/agent/ws`、`/device/ws`、`/connectivity/computer/ws` 的 `Authorization: Bearer` |
| Fallback tunnel token | app side 和 daemon side | `GET /connectivity/tunnel/ws` 的 one-time bearer token |

Fallback tunnel token 是 short-lived、single-use、side-specific，并绑定 attempt、account、app session、client fingerprint、target computer 和 actor type。

## JSON Rules

App-facing JSON endpoint 使用 strict JSON decode：

- body 最大 1 MiB。
- unknown fields 会被拒绝。
- 第一个 JSON value 后不能有 trailing data。
- malformed JSON 或 schema mismatch 通常返回 `400` + `code: 1001`。

Realtime WebSocket frame 是 JSON object：

- 必须有 `type`。
- 可带 `protocol_version`、`request_id` 和 event-specific fields。
- 当前 connectivity realtime protocol version 是 `2`。
- receiver 要 tolerate unknown event types 或 unknown fields；Relay 对 unsupported app command 返回 `error` frame。

所有 JSON timestamp 都是 Unix timestamp seconds。

## Unified Response Envelope

App-facing HTTP JSON response 使用统一 envelope：

```json
{
  "code": 0,
  "message": "success",
  "body": {}
}
```

规则：

- `code == 0` 表示成功。
- `code != 0` 表示业务失败。
- 业务失败时 `body` 是 `null`。
- HTTP status 仍表达 transport outcome，例如 `400`、`401`、`403`、`404`、`500`、`503`。

常用 code：

| code | message |
|---|---|
| `0` | `success` |
| `1001` | `The request is invalid.` |
| `1002` | `Too many requests. Please try again later.` |
| `1003` | `The username is already taken.` |
| `1004` | `The password must be at least 6 characters.` |
| `1005` | `Invalid access code.` |
| `1006` | `This access code is invalid.` |
| `1007` | `This access code has expired.` |
| `1008` | `This access code has been disabled.` |
| `1009` | `This access code has already been used.` |
| `1010` | `The username is invalid.` |
| `1011` | `The username or password is invalid.` |
| `1012` | `The session is invalid.` |
| `1013` | `This agent token was not found.` |
| `1014` | `The user was not found.` |
| `1015` | `The session was not found or is offline.` |
| `1016` | `The request is unauthorized.` |
| `1017` | `The request is forbidden.` |
| `1018` | `The requested endpoint was not found.` |
| `1019` | `The HTTP method is not allowed for this endpoint.` |
| `1020` | `The client fingerprint is invalid.` |
| `2001` | `The service is temporarily unavailable.` |
| `2002` | `An unexpected internal error occurred.` |

## Auth Endpoints

### `POST /api/auth/register`

用 invite code 创建 account。

Auth：none。

Request：

```json
{
  "invite_code": "AB2C3D",
  "username": "alice",
  "password": "password123"
}
```

Validation：

- `username` trim + lowercase，至少 4 个字符。
- `username` 只允许 `a-z`、`0-9`、`_`、`-`、`.`。
- `password` 至少 6 个字符。
- `invite_code` trim + uppercase，必须是 6 位，字符集为 `23456789ABCDEFGHJKMNPQRSTUVWXYZ`。
- 失败注册会按 remote IP throttle。

Success：`201 Created`

```json
{
  "code": 0,
  "message": "success",
  "body": {
    "user_id": 1,
    "username": "alice"
  }
}
```

常见错误：`1001` invalid request、`1010` invalid username、`1004` invalid password、`1005` malformed invite code、`1006` unknown invite code、`1007` expired invite code、`1008` disabled invite code、`1009` used invite code、`1003` username taken、`1002` rate limit。

### `POST /api/auth/login`

用 username/password 创建 app session。

Auth：none。

Request：

```json
{
  "username": "alice",
  "password": "password123",
  "client_fingerprint": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
}
```

`client_fingerprint` 对 legacy client 是 optional，但 connectivity-capable client 必须发送。它是 client Ed25519 public key 的 SHA-256 fingerprint，64 hex chars，大小写都可以。当前兼容旧 key `device_fingerprint` 作为 alias。

Success：`200 OK`

```json
{
  "code": 0,
  "message": "success",
  "body": {
    "access_token": "<access-token>",
    "refresh_token": "<refresh-token>",
    "expires_in": 86400,
    "token_type": "Bearer",
    "account_id": "123"
  }
}
```

Notes：

- `expires_in` 当前最多 86400 seconds。
- app session absolute lifetime 当前是 90 days。
- fingerprint-bound session refresh 时必须使用同一个 fingerprint。

### `POST /api/auth/refresh`

用 refresh token rotate app session tokens。

Auth：none。

Request：

```json
{
  "refresh_token": "<refresh-token>",
  "client_fingerprint": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
}
```

如果原始 session 绑定了 fingerprint，refresh 必须带同一个 `client_fingerprint`。旧 key `device_fingerprint` 仍是 alias。

Success：`200 OK`

```json
{
  "code": 0,
  "message": "success",
  "body": {
    "access_token": "<new-access-token>",
    "refresh_token": "<new-refresh-token>",
    "expires_in": 86400,
    "token_type": "Bearer",
    "account_id": "123"
  }
}
```

Client 必须原子替换 access token 和 refresh token。Refresh token 每次成功 refresh 后延长 30 days，但不会超过 app session 90 days absolute lifetime。

### `POST /api/auth/logout`

撤销当前 app session。

Auth：app access token。

Success：`200 OK`

```json
{
  "code": 0,
  "message": "success",
  "body": null
}
```

Logout 会关闭该 app session 的 live connectivity peers。它不会断开 owning agent session。

### `POST /api/auth/password/change`

修改当前用户密码，并撤销该 user 的所有 app sessions。

Auth：app access token。

Request：

```json
{
  "current_password": "password123",
  "new_password": "betterpass456"
}
```

Success：`200 OK`，`body: null`。

所有被撤销 app session 的 connectivity peers 会断开。Agent sessions 不会因为 app password change 自动断开。

## Agent Token Endpoints

### `GET /api/agent-tokens`

列出当前 user 的 agent tokens。

Auth：app access token。

Success：

```json
{
  "code": 0,
  "message": "success",
  "body": [
    {
      "id": "agt_123",
      "name": "MacBook",
      "created_at": 1775376000,
      "last_used_at": 1775377000,
      "revoked_at": 1775378000
    }
  ]
}
```

List 以 `created_at` newest-first。`last_used_at` 和 `revoked_at` 不存在时省略。Revoked token 仍可列出。

### `POST /api/agent-tokens`

创建一个 agent token，供 `tunnel` 和 `tunnel daemon` 使用。

Auth：app access token。

Request：

```json
{
  "name": "MacBook"
}
```

Success：`201 Created`

```json
{
  "code": 0,
  "message": "success",
  "body": {
    "id": "agt_123",
    "name": "MacBook",
    "created_at": 1775376000,
    "token": "<plaintext-agent-token>"
  }
}
```

Plaintext `token` 只在创建时返回。Client 必须立即保存。

### `DELETE /api/agent-tokens/:tokenID`

撤销当前 user 名下的 agent token。

Auth：app access token。

Success：`200 OK`，`body: null`。

Relay 会立即断开该 token 对应的 live `/agent/ws`、`/device/ws` 和 `/connectivity/computer/ws` peers，并清理相关 live launch、presence、rendezvous、fallback state。

## Account Policy

### `GET /api/account/policy`

读取 account policy tier。

Auth：app access token。

Success：

```json
{
  "code": 0,
  "message": "success",
  "body": {
    "account_id": "123",
    "tier": "free"
  }
}
```

当前 `tier`：

- `free`：最多 1 台 active trusted computer。
- `pro`：最多 10 台 active trusted computers。

这个 response 不包含 active computer selection、session permission、preview entitlement 或 per-session policy。Daemon transport 和 session list 不读取 Relay tier。

## Computer Launch API

### `GET /api/computers`

列出当前 user 在线的 launchable computers。

Auth：app access token。

Success：

```json
{
  "code": 0,
  "message": "success",
  "body": [
    {
      "computer_id": "dev_abcd1234",
      "display_name": "Yuanbo's MacBook Pro",
      "platform_family": "macos",
      "platform_id": "macos",
      "launch_health": "healthy"
    }
  ]
}
```

Important：

- 这个 endpoint 是 launch control plane list，不是 trusted computer list 的唯一来源。
- 返回的 computer 必须当前 `/device/ws` online。
- 返回结果 user-scoped，不包含其他 account 的 computer。
- 它不返回 offline computer，不返回 session rows，不返回 terminal data。
- Android 的 trusted computer list 还需要 join 本地 `TrustedComputerStore` 和 connectivity realtime presence。

### `POST /api/computers/:computerID/sessions`

请求一台 online computer 在 daemon-managed tmux workspace 中启动新的 `tunnel run <command>`。

Auth：app access token。

当前实现说明：这个 endpoint 目前按 authenticated account + online `/device/ws` computer 授权。它没有要求 per-launch Ed25519 signature，也没有在 handler 层强制 pairing-derived visibility。这个能力等价于 app access token 持有者可以请求该 account 下 online computer 启动 command。见 [Security](security.md) 的 remote-command authority gate；如果产品要求 launch 只能来自 paired client，必须先调整实现，再把本段改成 paired-client operation。

Request：

```json
{
  "command": "codex --profile prod",
  "cwd": "/repo",
  "label": "api-fix"
}
```

Fields：

- `command` required。
- `cwd` required，目标 machine 上的 working directory。
- `label` optional，会传入新 session metadata。

成功 launch：

```json
{
  "code": 0,
  "message": "success",
  "body": {
    "request_id": "dev_abcd1234-150405.000000000",
    "status": "session_ready",
    "session_id": "sess-1"
  }
}
```

失败 launch：

```json
{
  "code": 0,
  "message": "success",
  "body": {
    "request_id": "dev_abcd1234-150405.000000000",
    "status": "failed",
    "reason": "launch_timeout"
  }
}
```

Known `reason`：

- `device_offline`
- `busy`
- `command_not_allowed`
- `tmux_not_found`
- `session_start_failed`
- `tunnel_not_found`
- `path_not_found`
- `launch_timeout`

`session_ready` 的含义是新 `tunnel run` 已经完成 local daemon broker registration、Relay `/agent/ws` registration、PTY process startup，并发送 `launch_ready`。Official Android companion 仍必须等待 daemon transport 通过 `session_index` 或 `session_upsert` 发布同一个 `session_id`，才能渲染 session row/detail。

Relay launch API 不 auto-attach，也不返回 terminal bytes。

## Pairing Response API

### `POST /api/pairing/responses`

提交 Android signed pairing response。

Auth：fingerprint-bound app access token。

Request：

```json
{
  "version": 2,
  "account_id": "123",
  "invitation_id": "pair_abcd1234",
  "correlation_id": "corr_abcd1234",
  "client_public_key": "<hex-ed25519-public-key>",
  "client_fingerprint": "<hex-sha256-public-key>",
  "client_display_name": "Pixel",
  "signature": "<hex-ed25519-signature>"
}
```

Relay checks：

- app access token valid。
- app session 绑定了 `client_fingerprint`。
- request `account_id` 匹配 authenticated account。
- request `client_fingerprint` 匹配 authenticated app session fingerprint。
- `correlation_id` live、未过期，且属于同 account 的 daemon。

Relay 不验证 Ed25519 pairing signature。Daemon 才是 transcript verifier。Relay 只把已通过 Relay-level checks 的 response 作为 `pair_response_forward` 转发给 daemon。

Success：

```json
{
  "code": 0,
  "message": "success",
  "body": {
    "status": "forwarded"
  }
}
```

Pairing transcript、SAS 和 trust completion 见 [Pairing](pairing.md)。

## Connectivity WebSockets

### `GET /api/connectivity/ws`

App connectivity realtime socket。

Auth：fingerprint-bound app access token。

Client first frame：

```json
{
  "type": "app_register",
  "protocol_version": 2
}
```

Relay initial frame：

```json
{
  "type": "computer_snapshot",
  "computers": []
}
```

Relay 后续可以发送：

- `computer_visible`
- `computer_removed`
- `client_revoked`
- app-facing `rendezvous_hint`
- app-facing `rendezvous_close`
- app-facing `relay_tunnel_ready`
- `error`

App 可以发送：

- `rendezvous_open`
- `rendezvous_close`
- `relay_tunnel_request`

Connectivity app socket 不承载 pairing response body、session list、preview、snapshot、live bytes、input、resize 或 QUIC packets。

### `GET /connectivity/computer/ws`

Daemon connectivity realtime socket。

Auth：agent bearer token。

Daemon first frame：

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

Daemon 可以发送：

- `pair_invitation_reserve`
- `pair_completed`
- `client_revoked`
- daemon-origin `rendezvous_hint`
- `direct_session_open`
- `direct_session_close`

Relay 可以发送：

- `pair_invitation_reserved`
- `pair_response_forward`
- daemon-facing `rendezvous_hint`
- `rendezvous_close`
- daemon-facing `relay_tunnel_ready`
- `direct_session_close`
- `error`

Relay 根据 daemon registration 里的 `trusted_clients` 重建 live visibility。Relay 不把 trusted client roster 持久化为 durable database。

### `GET /connectivity/tunnel/ws`

Fallback packet tunnel socket。

Auth：one-time fallback tunnel token。

```text
GET /connectivity/tunnel/ws
Authorization: Bearer <single-use-token>
```

WebSocket 只接受 binary messages。每个 binary message 是一个 opaque encrypted QUIC packet。Relay 通过同一个 attempt 关联 app side 和 daemon side endpoint，并原样转发 binary packets。

Relay 不解析 QUIC，不读取 TLS plaintext，不理解 daemon transport frame，不读取 terminal bytes、preview、input 或 resize。

## Retained Agent And Device WebSockets

这些 WebSocket 仍是当前实现的一部分，但不是 official mobile companion 的 session data plane。不要把它们理解成可恢复的 Relay session list/detail/attach protocol。

### `GET /agent/ws`

`tunnel run` 使用 agent token 连接，注册 one running session，并在 mobile launch path 中发送 `launch_ready`。Relay 用它完成 launch correlation，但不对 app 暴露 session list/detail/attach API。

### `GET /device/ws`

`tunnel daemon` 使用 agent token 连接，注册 launchable computer presence，并接收 `POST /api/computers/:computerID/sessions` 转发的 launch request。它是 launch control plane，不承载 trusted computer transport 的 terminal data。

`/device/ws` 和 `/connectivity/computer/ws` 都由 daemon 使用，但职责不同：

- `/device/ws`：online computer list + remote launch request/result。
- `/connectivity/computer/ws`：pairing visibility + direct rendezvous + fallback authorization。

## Security Boundary

Relay 可以认证 account、转发 signed transcript、路由 live event、授权短期 fallback token。Relay 不能成为 terminal/session plaintext authority。

Client 必须把这些事实当成 security invariant：

- Terminal plaintext 只出现在 pinned daemon transport 端点内。
- Relay fallback tunnel 只转发 encrypted QUIC packets。
- Pairing trust root 是 daemon public key 和 Android client public key，不是 Relay。
- App session fingerprint 是 Relay auth binding，不能替代 Ed25519 pairing signature。
- `session_id` 从 launch API 返回后只是 correlation key；真正可渲染状态来自 daemon transport。
