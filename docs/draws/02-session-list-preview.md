# 2. Session List And Preview

Pairing 之后，Android 上的 session list 和 recent output preview 来自 daemon transport，不来自 Relay session API。

## 数据流

```mermaid
flowchart TD
  classDef mobile fill:#e8f3ff,stroke:#2f6fbd,color:#0f2742
  classDef relay fill:#fff4db,stroke:#b66b00,color:#3b2600
  classDef daemon fill:#eaf8ef,stroke:#2b8a4b,color:#102f1c
  classDef terminal fill:#101820,stroke:#4a5568,color:#f7fafc

  A[Android SessionList]:::mobile
  B[TrustedComputerConnectionCoordinator]:::mobile
  C[DaemonConnectionManager<br/>direct first, relay fallback]:::mobile
  D[Relay<br/>presence/rendezvous/fallback only]:::relay
  E[tunnel daemon<br/>broker roster/cache]:::daemon
  F[tunnel run<br/>PTY owner + terminal mirror]:::terminal

  A --> B --> C
  C <--> D
  C <== "pinned QUIC/TLS daemon transport" ==> E
  F -->|metadata, preview,<br/>snapshot, live output| E
  E -->|session_index / upsert / gone| A
  A -->|preview_subscribe visible rows only| E
  E -->|preview_snapshot| A
```

## Sequence

```mermaid
sequenceDiagram
  autonumber
  participant App as Android SessionList
  participant Conn as Daemon connection manager
  participant Relay as Relay
  participant Daemon as tunnel daemon
  participant Run as tunnel run

  App->>Conn: online trusted computer selected for connection
  Conn->>Relay: direct rendezvous or fallback setup
  Conn->>Daemon: establish pinned QUIC/TLS daemon transport
  App->>Daemon: hello(protocol_version=2, path_kind)
  Daemon->>App: hello(protocol_version=2)
  Run->>Daemon: broker registration + metadata + preview cache
  Daemon->>App: session_index(sessions[])
  App->>App: render computer section and session rows
  App->>Daemon: preview_subscribe(session_id) for visible rows
  Daemon->>App: preview_snapshot(session_id, preview, updated_at)
  Run->>Daemon: metadata or preview changes
  Daemon->>App: session_upsert / session_gone / preview_snapshot
```

## 关键事实

- Relay 没有 current session list API。
- Relay 不保存 preview，也不从 terminal output 里生成 preview。
- `session_index` 是 daemon 对当前 computer mobile-visible session set 的完整同步。
- `session_upsert` 是完整 replacement metadata object，不是 partial patch。
- `preview_snapshot` 和 session metadata 分开传，避免 metadata 改动和 preview 文本互相阻塞。
- Android 只订阅 visible rows 的 preview，减少 daemon transport traffic。

## SessionMetadata 边界

`SessionMetadata` 当前包含：

- `session_id`
- `label`
- `command_preview`
- `cwd`
- `git_branch`
- `started_at`
- `updated_at`
- `online`

它不包含：

- terminal snapshot bytes。
- live terminal bytes。
- preview text payload。
- account tier。
- Relay launch correlation fields。
- direct/fallback path authority。

## Launch 后如何收敛到 list

Mobile launch 是两段式：

```mermaid
sequenceDiagram
  autonumber
  participant App as Android app
  participant Relay as Relay
  participant Daemon as tunnel daemon
  participant Run as new tunnel run

  App->>Relay: POST /api/computers/:id/sessions(command,cwd,label)
  Relay->>Daemon: launch request over /device/ws
  Daemon->>Run: start tmux workspace session<br/>tunnel run --launch-source mobile
  Run->>Daemon: broker registration
  Run->>Relay: /agent/ws register + launch_ready
  Relay->>App: session_ready(session_id)
  Daemon->>App: session_upsert(session_id) over daemon transport
  App->>App: match session_id and show row
```

`session_ready` 只说明 Relay launch correlation 完成。Android 仍要等 daemon transport 的 `session_upsert` 或下一次 `session_index` 才能显示真正可交互的 session row。

## Preview 的 UX 名称

UI 中这个区域应叫 `recent output preview`。它是 daemon broker 提供的 bounded latest preview，不是 chat message，也不是 Relay last message。
