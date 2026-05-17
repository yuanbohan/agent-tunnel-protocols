# Architecture

## 状态

本文是 Agent Tunnel 当前 cross-repository architecture 的 SSOT。

已在 2026-05-17 从原 `agent-tunnel/docs/architecture.md` 迁入，并按 direct-first + Relay fallback 的当前实现重新校准。实现仓库可以保留本地 package map、测试入口、implementation notes 和运维细节，但 shared system boundary 要指向这里。

## System Shape

Agent Tunnel 由四个主要运行面组成：

- Android companion：登录 Relay、保存 client identity 和 trusted computer records、发起 pairing、展示 trusted computer/session list、连接 daemon transport、渲染 terminal、发送 input。
- Go `tunnel run`：真正启动本地 CLI command，拥有 PTY、local terminal、terminal mirror、daemon broker registration、Relay `/agent/ws` registration。
- Go `tunnel daemon`：机器本地后台 runtime，拥有 local control socket、broker socket、tmux workspace、device identity、connectivity identity、trusted client roster、direct UDP listener、Relay fallback packet tunnel client。
- Relay + PostgreSQL + STUN：认证和 live control plane。Relay 保存 durable auth/account/operator 数据，维护 live presence/correlation/rendezvous/fallback state；STUN 只做 Binding-only UDP address discovery。

核心原则：

- Terminal session plaintext authority 是 `tunnel run` + local daemon broker + pinned daemon transport。
- Relay 是 authentication/control/routing plane，不是 terminal data plane。
- Pairing 建立 pinned public-key trust。后续每次 direct 或 fallback connection 都重新做 QUIC/TLS 1.3 handshake，并生成 fresh traffic keys。
- Direct 和 Relay fallback 的差别只是 packet carrier，不改变 daemon transport protocol、不改变 trust root、不让 Relay 看到 plaintext。

## Runtime Graph

```text
local computer
┌──────────────────────────────────────────────────────────────────────┐
│                              tunnel run                              │
│  PATH command -> PTY child -> session hub -> terminal mirror          │
│        │              │                 │                            │
│        │              │                 └─ broker snapshots/output    │
│        │              └─ local terminal                              │
│        └─ /agent/ws register + launch_ready                          │
└──────────────────────────────┬───────────────────────────────────────┘
                               │ local broker socket
┌──────────────────────────────▼───────────────────────────────────────┐
│                            tunnel daemon                             │
│  control socket, broker roster/cache, tmux workspace, pairing state   │
│  /device/ws, /connectivity/computer/ws, direct UDP, fallback QUIC      │
└──────────────┬───────────────────────────────┬───────────────────────┘
               │                               │ pinned QUIC/TLS
               │ Relay control plane           │ direct UDP or fallback packets
               ▼                               ▼
┌──────────────────────────────┐        ┌──────────────────────────────┐
│ Relay + PostgreSQL + STUN    │        │ Android companion             │
│ auth, account policy,        │        │ trust store, computer list,    │
│ presence, launch, pairing,   │        │ session list, preview, detail, │
│ rendezvous, fallback tokens  │        │ terminal rendering, input      │
└──────────────────────────────┘        └──────────────────────────────┘
```

## Relay Ownership

Relay owns：

- user account、invite code、app session、agent token、account policy、operator audit 的 durable state。
- app bearer auth 和 agent bearer auth。
- app session 与 `client_fingerprint` 的 server-side binding。
- `/api/computers` online computer list。
- `POST /api/computers/:computerID/sessions` launch request correlation。
- `/agent/ws` live session ownership 和 launch-ready correlation。
- `/device/ws` online computer routing 和 launch request delivery。
- `/api/connectivity/ws` app realtime peers。
- `/connectivity/computer/ws` daemon connectivity peers。
- pairing response correlation 和 forwarding。
- direct rendezvous hints forwarding。
- short-lived fallback tunnel token issue 和 validation。
- `/connectivity/tunnel/ws` opaque binary packet forwarding。
- token/user revocation 时断开 live peers 并清理 live state。

Relay does not own：

- app-facing session list/detail/stop/attach endpoints。
- terminal emulation。
- durable transcript history。
- preview generation。
- snapshot generation。
- terminal input translation。
- daemon broker roster/cache。
- offline trusted-computer inventory。
- trusted client durable roster。
- direct candidate semantics。
- fallback QUIC packet semantics。

## Tunnel Run Ownership

`tunnel run <command>` owns one actual local terminal session。

它负责：

- resolve command from `PATH`。
- 创建 PTY child 并保持 launching terminal interactive。
- local terminal raw mode 和 resize。
- local session hub fanout。
- terminal mirror。
- 向 local daemon broker 注册 session metadata、latest preview、coalesced snapshot、live output。
- 用 `/agent/ws` 向 Relay 注册 live session。
- mobile launch 时发送 `launch_ready`。
- 把 daemon transport `input_text`、`input_key` 翻译成 PTY bytes。

`tunnel run` 启动前有两个 gate：

1. local daemon broker registration 必须成功。
2. Relay `/agent/ws` registration 必须在 startup wait window 内成功。

如果任一 gate 失败，`tunnel run` 在 terminal setup 和 child startup 之前退出。启动后 Relay 断线不会中断本地 terminal work；`/agent/ws` connector 会 backoff retry。

## Tunnel Daemon Ownership

`tunnel daemon` 是一台 computer 的本地 coordinator。

它负责：

- daemon lifecycle：`start`、`status`、`stop`、`doctor`。
- local control socket：本机 `tunnel session list`、`tunnel session stop`、workspace 管理。
- broker socket：接收 `tunnel run` metadata、preview、snapshot、live output、input routing。
- broker roster/cache：当前 mobile-visible session set。
- tmux workspace：mobile-created launch 的 dedicated workspace。
- device identity：用于 `/device/ws` launch surface。
- connectivity identity：Ed25519 daemon public key，pairing 和 daemon transport pinning 使用。
- pairing invitation、pending response、SAS confirmation、trusted client roster、revocation。
- Relay `/device/ws` connector。
- Relay `/connectivity/computer/ws` connector。
- direct UDP rendezvous listener。
- fallback QUIC-over-WebSocket packet tunnel。
- pinned QUIC/TLS daemon transport accept path。
- launch validation、busy state、launch health、last failure reason。

它不负责：

- durable transcript archive。
- account-wide session sharing。
- Relay auth storage。
- terminal semantic parsing。
- payment/subscription policy。

## Android Companion Ownership

Android companion 负责 mobile user experience 和 client-side security storage。

它负责：

- app login/register/refresh/logout。
- `client_fingerprint` 绑定 app session。
- client Ed25519 identity。
- pending pairing state。
- trusted computer records。
- account policy read。
- Relay connectivity realtime。
- trusted computer list projection。
- direct-first connection manager。
- Relay fallback packet tunnel carrier。
- daemon transport `hello`、session list、preview subscribe、interactive attach、input、release。
- terminal snapshot/live bytes replay and rendering。
- diagnostics and debug export。

Android companion 的 trusted-computer projection 和 daemon transport eligibility 是两个不同边界。Projection 可以使用本地 durable trust 和 retained realtime visibility 来避免 UI 在 lifecycle seam 中闪空；daemon transport eligibility 必须额外要求当前 `/api/connectivity/ws` connected，才能发送 rendezvous 或 fallback tunnel control-plane commands。

Android 不负责：

- 启动本地 PTY。
- 作为 SSH server/client。
- 把 terminal output 当 chat message 处理。
- 直接信任 Relay visible computer 而跳过本地 trusted store。
- 在 Relay fallback 中读取 plaintext。

## PostgreSQL Ownership

PostgreSQL 是 Relay durable data authority：

- users。
- invite codes。
- password digests。
- app sessions 和 refresh/access token state。
- app session client fingerprint binding。
- account subscription tier。
- agent tokens。
- operator audit events。

PostgreSQL 不保存：

- terminal transcript。
- terminal snapshot。
- session preview。
- trusted client roster。
- offline computer inventory。
- direct rendezvous attempts。
- fallback tunnel tokens。

## STUN Ownership

STUN 是 Binding-only address discovery service。

它负责：

- 收到 UDP Binding request。
- 返回 observed public UDP address。

它不负责：

- TURN relay。
- credential auth。
- terminal data。
- trust establishment。
- pairing。
- session routing。

Direct path 只用 STUN 帮助 Android 和 daemon 形成 UDP candidates。Relay realtime 转发 hints；实际 QUIC/TLS traffic 不经过 Relay。

## Launch Lifecycle

Mobile-created launch 流程：

1. Daemon 通过 `/device/ws` 注册 online launchable computer。
2. App 调 `GET /api/computers` 获得当前 online computers。
3. App 调 `POST /api/computers/:computerID/sessions`，提交 `cwd`、`command`、optional `label`。
4. Relay 把 launch request 转发给 online daemon，并创建 request correlation。
5. Daemon 检查 busy state、tmux availability、`cwd`、command policy、`tunnel` availability。
6. Daemon 在 dedicated tmux workspace 启动 `tunnel run --launch-source mobile --launch-request-id <id> <command>`。
7. 新 `tunnel run` 注册 local broker、注册 `/agent/ws`、启动 PTY，并发送 `launch_ready`。
8. Relay 匹配 `launch_ready` 和 pending request，返回 `session_ready` + `session_id`。
9. Android 把 `session_id` 当 correlation key，等待 daemon transport 通过 `session_index` 或 `session_upsert` 发布同一个 session。
10. Android 渲染 session row，之后可订阅 preview 或进入 detail。

`session_ready` 不等于 terminal attach completed。Relay 不 auto-attach，不返回 terminal bytes。

## Trusted Computer Lifecycle

Trusted computer 是 Android local trust record 与 Relay live visibility 的交集。

Pairing 完成后：

- Daemon 持久化 trusted Android client fingerprint/public key。
- Android 持久化 trusted daemon computer id/fingerprint/public key。
- Relay 只保存 live visibility derived state。

App 启动时：

1. Android 读取 local trusted records。
2. Android 打开 `/api/connectivity/ws`。
3. Relay 发送 `computer_snapshot`。
4. Android 按 `(account_id, relay_base_url, computer_id, computer_fingerprint)` join local trust 和 Relay visibility。
5. Join 成功的 computer 进入 trusted computer list。
6. 本地有 trust 但 Relay 不可见，则显示 offline。
7. Relay 可见但本地无 trust，不显示成 trusted computer。

卸载 Android app 会删除 app-local protected storage，所以 client private key、pending pairing、trusted computer records、Relay credentials 都消失。重新安装后即便 Relay 或 daemon 仍有旧 fingerprint 的记录，新 app installation 也没有对应 private key，必须重新 pairing。

## Connectivity Lifecycle

已经 trusted 且 online 的 computer 进入 daemon transport 建连流程：

1. Android 为本次 attempt 生成 `attempt_id`。
2. Android 先走 direct-first：STUN 发现 public UDP address，收集 private UDP candidates。
3. Android 通过 Relay app realtime 发送 `rendezvous_open`。
4. Relay 把 client hints 转发给 daemon。
5. Daemon 回 daemon hints，并打开 direct UDP accept path。
6. Android 尝试直接 QUIC/TLS handshake。
7. 双方 certificate public key 必须匹配 pairing pin。
8. Direct 成功后进入 daemon transport `hello`、`session_index`。
9. Direct skipped/failed/timeout 时，Android 请求 `relay_tunnel_request`。
10. Relay 发 side-specific `relay_tunnel_ready`。
11. Android 和 daemon 分别连接 `/connectivity/tunnel/ws`。
12. 双方在 WebSocket binary packet carrier 上运行同一套 QUIC/TLS daemon transport。

Direct 和 Relay fallback 的 daemon transport 完全相同：

- 同一个 ALPN：`tunnel-conn/1`。
- 同一个 pinned Ed25519 certificate model。
- 同一个 frame registry。
- 同一个 session metadata、preview、interactive/input payload。
- 同一个 no plaintext to Relay invariant。

## Session List And Preview Lifecycle

Session list 不来自 Relay。

1. Daemon transport 建立后，Android 发 `hello`。
2. Daemon 验证 client pinned identity，回 `hello`。
3. Daemon 发送 `session_index`。
4. Android 用 `session_index` 渲染 per-computer session rows。
5. Android 只对 visible rows 发送 `preview_subscribe`。
6. Daemon 从 broker cache 发送 `preview_snapshot`。
7. 后续 session 变化使用 `session_upsert` 或 `session_gone`。
8. preview 变化继续用 `preview_snapshot`。

Relay 在这个流程里只负责保持 app/daemon authorization 和 path setup，不读取 session metadata 或 preview。

## Session Detail And Input Lifecycle

进入 detail：

1. Android 发送 `interactive_request`，包含 `session_id`、`cols`、`rows`。
2. Daemon 检查 trusted client、session availability、interactive ownership。
3. Daemon 返回 `interactive_granted` 或 `interactive_denied`。
4. Granted 后 daemon 打开 unidirectional interactive stream。
5. Daemon 发送 `snapshot_begin`、多个 `snapshot_chunk`、`snapshot_end`。
6. Snapshot 后继续发送 `live_bytes`。
7. Android terminal pipeline replay snapshot/live bytes。
8. Android 发送 `input_text` 和 `input_key` 到 control stream。
9. Daemon broker 把 input route 到 owning `tunnel run`。
10. `tunnel run` 翻译 PTY bytes 并写入 PTY。
11. PTY output 回到 broker，再经 interactive stream 发给 Android。

当前 Android 行为：initial terminal geometry 在 `interactive_request` 里发送；protocol 支持 `resize` frame，但 live Android geometry update 只有在 UI path wiring 完成后才能描述为实现。

## Package And Module Map

Go implementation entry points：

- `cmd/tunnel`：local CLI。
- `cmd/relay`：Relay server。
- `internal/tunnel/session/`：PTY lifecycle、local terminal、session hub、terminal mirror。
- `internal/tunnel/connector/`：`/agent/ws` connector。
- `internal/tunnel/daemon/`：daemon control、broker、pairing、tmux workspace、connectivity。
- `internal/protocol/`：Relay-facing wire types 和 daemon transport payload types。
- `internal/relay/auth/`：auth、app sessions、agent tokens。
- `internal/relay/device/`：`/device/ws` live computer routing 和 launch correlation。
- `internal/relay/session/`：`/agent/ws` live session ownership。
- `internal/relay/connectivity/`：pairing visibility、rendezvous、fallback tunnel state。
- `internal/relay/handler/`：Gin router、REST handlers、WebSockets。
- `internal/relay/store/postgres/`：PostgreSQL persistence。

Android implementation entry points：

- `data/relay/`：Relay HTTP/WebSocket DTOs and service。
- `data/config/` 和 related stores：Relay credentials、client identity、trusted computers。
- `domain/sessions/`：daemon transport attach runtime。
- `ui/list/`：trusted computer/session list projection。
- `ui/session/`：detail screen and input orchestration。
- `ui/terminal/`：terminal replay/rendering。

## Deployment Shape

Production Relay deployment currently uses Docker Compose：

- PostgreSQL container。
- Relay HTTP/WebSocket container。
- Binding-only STUN container。
- nginx handles HTTP/WebSocket edge; STUN is exposed directly on UDP `3478`。

Fresh PostgreSQL volumes initialize from `deploy/postgres/latest.sql` in `agent-tunnel`。Existing production database migration remains operator-managed manual SQL unless product boundary changes。
