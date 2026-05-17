# Connectivity 端到端流程

## 状态

本文是 Android companion、Relay/server、Go tunnel/daemon 共享的端到端流程 SSOT。

配套图文说明在 [draws](draws/README.md)：

- [0. Computer List](draws/00-computer-list.md)
- [1. Pairing](draws/01-pairing.md)
- [2. Session List And Preview](draws/02-session-list-preview.md)
- [3. Direct And Relay](draws/03-direct-relay-data-flow.md)
- [4. Detail Input](draws/04-detail-input.md)

更细的协议文档：

- [API](api.md)
- [Architecture](architecture.md)
- [Pairing](pairing.md)
- [Relay Control Plane](relay-control-plane.md)
- [Daemon Transport Protocol](protocol.md)

## 参与模块

- Android companion (`agent-tunnel-android`)：app login、client identity、安全本地存储、pairing UI、trusted computer list、direct-first connection manager、Relay fallback packet carrier、session list、preview、session detail。
- Go tunnel/daemon/Relay/STUN (`agent-tunnel`)：Relay HTTP/WebSocket control plane、Binding-only STUN、local daemon、local broker、pairing state、trusted client roster、direct UDP accept path、fallback packet tunnel、daemon transport。
- Protocol SSOT (`agent-tunnel-protocols`)：跨仓库协议、安全、数据流和兼容性决策。

## 0. 怎么看到 Computer List

Computer list 不是单纯从 Relay 拉一个 durable list。它是三类状态的交集：

- Android 本地 `TrustedComputerStore`
- Relay live presence
- account policy

Android 本地的 trusted computer record 用 protected storage 保存，并按这些字段隔离：

- `account_id`
- `relay_base_url`
- `computer_id`
- daemon fingerprint

Relay 只保存 live visibility。一个 daemon 对某个 app session 可见，需要同时满足：

1. App 使用 app bearer token 登录。
2. Relay 里的 app session 已绑定 `client_fingerprint`。
3. Daemon 使用同一 account 下的 agent token 连接。
4. Daemon 已注册 `GET /connectivity/computer/ws`。
5. Daemon 的 `trusted_clients` roster 包含该 app 的 `client_fingerprint`，或者刚通过 `pair_completed` 建立了 live visibility。

Android 打开：

```text
GET /api/connectivity/ws
Authorization: Bearer <app-access-token>
```

第一帧发送：

```json
{"type":"app_register","protocol_version":2}
```

Relay 返回 `computer_snapshot`，之后推送 `computer_visible`、`computer_removed`、`client_revoked`。

Android 不能只信 Relay presence。UI projection 会 join：

- protected local trusted records
- Relay realtime computers，按 `(computer_id, computer_fingerprint)` 匹配
- account policy (`free` / `pro`)
- 每台 computer 的 daemon connection state

本地 trusted 但 Relay 不可见，就显示 offline。Relay 可见但本地没有 trusted record，不会显示成 trusted computer。只有两边都匹配后，Android 才会为这台 computer 启动 daemon connection。

## 1. Pairing 详细流程

Pairing 的目标是把一个 Android app installation 绑定到一个 daemon computer identity。Relay 负责传递消息和提供 authenticated account context，但不是 trust root。

高层流程：

1. 用户在电脑上执行 `tunnel pair`。
2. Daemon 通过 Relay `pair_invitation_reserve` 预留一个短期 correlation。
3. Relay 返回 `pair_invitation_reserved`，包含 authenticated `account_id` 和 correlation。
4. Daemon 创建 version `2` signed invitation，写入本地 invitation state，并打印 QR/paste payload。
5. Android 导入 invitation，验证 version、expiry、account id、daemon fingerprint、daemon Ed25519 signature。
6. Android 读取或生成自己的 Ed25519 client identity，签名 Android response transcript。
7. Android 调 `POST /api/pairing/responses` 提交 response。
8. Relay 验证 app bearer session、`account_id`、`client_fingerprint`，然后以 `pair_response_forward` 转发给 daemon。
9. Daemon 验证 Android Ed25519 signature，计算 6 位 SAS，并把 response 存为 pending。
10. 两端显示相同 SAS。
11. 用户在 daemon CLI 上确认 SAS。
12. Daemon 持久化 trusted Android client，并向 Relay 发送 `pair_completed`。
13. Relay 向匹配的 app peer 推 `computer_visible`。
14. Android 等到 paired computer visible 后，才把 daemon trust record 写入本地 trusted store，并清理 pending pairing。

关键安全点：

- 长期 device identity 和 pairing signature 使用 Ed25519。
- `fingerprint = SHA-256(raw_ed25519_public_key)`，lowercase hex。
- SAS 使用 `SHA-256` 计算，输入是 length-prefixed daemon key、client key、invitation id、nonce；取 digest 前 4 字节 big-endian uint32 后 `mod 1_000_000`，显示为 6 位数字。
- Pairing 不产生长期 symmetric secret，只产生 pinned peer identities。
- 后续 transport 用 pinned identity 认证新的 TLS 1.3 handshake，session key 每次连接都是新的。
- Relay 可以拒绝或延迟消息，但不能改 signed transcript field；否则 signature 会失败。
- SAS 必须由用户 out-of-band 对比，不能让 Relay 自动比较。

## 2. Pairing 后怎么看 Session List 和 Recent Output Preview

Relay 不是 session authority。Relay 可以请求某台 online computer launch session，但 mobile companion 的 session rows 和 preview 来自 daemon transport。

已经 trusted 且 visible 的 computer：

1. Android 先尝试 direct daemon connection。
2. 如果 direct 不可用，再 fallback 到 Relay packet tunnel。
3. 无论 direct 还是 fallback，都建立同一套 pinned QUIC/TLS daemon transport。
4. Android 发 `hello`。
5. Daemon 验证 pinned client identity，回 `hello`。
6. Daemon 发送 `session_index`，包含这台 computer 当前全部 mobile-visible session metadata。
7. Android 渲染 computer section 和 session rows。
8. Android 只给可见 rows 发送 `preview_subscribe`。
9. Daemon 对 subscribed sessions 发送 `preview_snapshot`，之后通过 `session_upsert` / `session_gone` 同步变化。

Mobile-created launch：

1. Android 调 `POST /api/computers/:computerID/sessions`。
2. Relay 把 launch request 转给 online daemon。
3. Daemon 启动 tmux-backed `tunnel run`。
4. 新 `tunnel run` 注册 local broker 和 `/agent/ws`，并发送 `launch_ready`。
5. Relay 返回 `session_ready` 和 `session_id`。
6. Android 把这个 `session_id` 当 correlation key；真正可见 row 必须等 daemon transport 的 `session_index` 或 `session_upsert`。

List 底部区域叫 recent output preview，不是 chat message。Preview text 是独立 daemon transport payload，不能塞进 `SessionMetadata`。

## 3. Direct 和 Relay 的数据流向

Direct 和 Relay fallback 只差 QUIC 下面的 packet carrier。上层 daemon session protocol 完全一样。

Direct path：

```text
Android app
  -> Relay app realtime: rendezvous_open
  -> Relay forwards client candidate hint
  -> daemon sends daemon candidate hint
  -> Android + daemon send UDP probes
  -> QUIC/TLS 1.3 over direct UDP
  -> daemon transport frames
```

Relay fallback path：

```text
Android app
  -> Relay app realtime: relay_tunnel_request
  -> Relay issues side-specific single-use tunnel tokens
  -> Android opens /connectivity/tunnel/ws with client token
  -> daemon opens /connectivity/tunnel/ws with daemon token
  -> Relay forwards binary WebSocket messages as opaque QUIC packets
  -> QUIC/TLS 1.3 over packet tunnel
  -> daemon transport frames
```

Relay fallback 在 transport security 上不比 direct 弱：

- 两条路径都用相同 pinned peer identities。
- 两条路径都协商新的 TLS 1.3 session keys。
- Relay fallback 只转发 encrypted QUIC packets。
- Relay 不解析 QUIC、terminal bytes、preview、snapshot、input、resize、path badge 或 daemon session semantics。

Relay 可以影响可用性，例如不转发 rendezvous hints 或拒绝 fallback setup；但不能解密 direct 或 fallback 的 daemon transport payload。

## 4. 进入 Detail 后，移动端发送消息怎么走

Session detail attach 到一个已连接 trusted computer transport。

Attach 流程：

1. Android route identity 解出 `(computer_id, daemon_fingerprint, attempt_id, session_id)`。
2. Android 在 daemon transport control stream 发送 `interactive_request`，带 `session_id` 和初始 `cols` / `rows`。
3. Daemon broker 只给该 session lifetime 一个 interactive owner；如果不可用则 `interactive_denied`。
4. Grant 成功后，daemon 在 control stream 发 `interactive_granted`，并打开 daemon-initiated unidirectional interactive stream。
5. Interactive stream 发送 `snapshot_begin`、多个 `snapshot_chunk`、`snapshot_end`，之后发送 `live_bytes`。
6. Android 用 terminal pipeline 渲染 snapshot 和 live bytes。
7. Android 在 snapshot complete 后才启用 input。

Mobile input 永远不走 interactive stream，而是走 control stream：

- 普通输入、paste、IME commit、draft sync、submit：`input_text`
- special key：`input_key`
- protocol-level terminal geometry：`resize`
- 离开 detail：`interactive_release`

Daemon 收到 input 前会验证该 client 当前确实拥有 active interactive grant，然后才转给 local broker 和 PTY owner。

当前 Android detail 实现会在 `interactive_request` 里发送初始 geometry，并通过 control stream 发送 text/key input。Protocol 支持 `resize`，但不要声称 Android 已经发送 live geometry resize frame，除非 UI path 确实接上线并通过验证。

## 5. Key Storage、卸载 App、为什么要重新 Pairing

Android 使用 app-local protected storage：

- primary Ed25519 client identity：Android Keystore key pair
- fallback client signing identity：AndroidX `EncryptedSharedPreferences` 里的 encrypted seed
- trusted computers：按 account 和 relay base URL 隔离的 encrypted local store
- pending pairing：encrypted local store
- Relay credentials：encrypted local config store

Daemon 使用本地 private state：

- Ed25519 connectivity identity
- invitation records
- pending pairing responses
- trusted Android client roster
- revoked client records

Relay 不保存 durable trusted-client database，只保存 live auth/routing state，例如 app sessions、daemon registrations、pairing correlations、direct rendezvous attempts、direct session records、fallback tunnel tokens。

Android app 卸载后，系统会删除 app data 和该 app 的 Keystore entries。重新安装后的 app：

- 没有旧 client private key
- 会有新的 client public key 和 fingerprint
- 没有本地 trusted-computer records
- 没有 pending pairing records
- 没有 Relay app session tokens

Daemon 可能还保存旧 client fingerprint，但新 app 无法证明自己拥有旧 private key，也无法用旧 fingerprint 通过 daemon transport authentication。用户必须重新 sign in 和 pairing。

这是故意的安全边界。如果卸载/备份恢复后不需要原 private key 就能恢复 trust，就等于允许一个新安装伪装成旧 trusted client。
