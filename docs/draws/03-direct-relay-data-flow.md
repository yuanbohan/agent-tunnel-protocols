# 3. Direct And Relay Data Flow

Direct 和 Relay fallback 都承载同一套 daemon transport。区别只在 QUIC packet 怎么到达对端。

## Direct-first path selection

```mermaid
flowchart TD
  classDef mobile fill:#e8f3ff,stroke:#2f6fbd,color:#0f2742
  classDef relay fill:#fff4db,stroke:#b66b00,color:#3b2600
  classDef daemon fill:#eaf8ef,stroke:#2b8a4b,color:#102f1c
  classDef decision fill:#f3ecff,stroke:#7557c2,color:#26164a
  classDef success fill:#eaf8ef,stroke:#2b8a4b,color:#102f1c

  A[Android picks trusted online computer]:::mobile
  B[Generate attempt_id]:::mobile
  C[STUN Binding<br/>discover public UDP addr]:::relay
  D[Collect private UDP candidates]:::mobile
  E[Send rendezvous_open<br/>over /api/connectivity/ws]:::relay
  F[Relay forwards rendezvous_hint<br/>to daemon]:::relay
  G[Daemon replies with daemon candidates<br/>and opens direct UDP accept]:::daemon
  H{QUIC/TLS direct succeeds<br/>before timeout?}:::decision
  I[Use direct path<br/>pinned daemon transport]:::success
  J[Request relay_tunnel_request<br/>fallback_reason + diagnostics]:::relay
  K[Relay sends relay_tunnel_ready<br/>side-specific tokens]:::relay
  L[Both sides connect<br/>/connectivity/tunnel/ws]:::relay
  M[Run same QUIC/TLS daemon transport<br/>over encrypted packet tunnel]:::success

  A --> B --> C --> D --> E --> F --> G --> H
  H -->|yes| I
  H -->|no / skipped / failed| J --> K --> L --> M
```

## Direct data flow

```mermaid
sequenceDiagram
  autonumber
  participant App as Android
  participant Relay as Relay realtime
  participant Stun as STUN
  participant Daemon as tunnel daemon

  App->>Stun: Binding request
  Stun-->>App: observed public UDP address
  App->>Relay: rendezvous_open(attempt_id, candidates)
  Relay->>Daemon: rendezvous_hint(actor=client, candidates, client_fingerprint)
  Daemon->>Relay: rendezvous_hint(actor=daemon, candidates)
  Relay->>App: rendezvous_hint(actor=daemon, candidates)
  App->>Daemon: UDP QUIC Initial packets direct
  Daemon->>App: QUIC/TLS handshake direct
  App->>Daemon: verify daemon cert public key against pairing pin
  Daemon->>App: verify client cert public key against trusted roster
  Daemon->>Relay: direct_session_open(attempt_id)
  App->>Daemon: daemon transport frames<br/>hello, preview_subscribe, input, resize
  Daemon->>App: daemon transport frames<br/>session_index, preview, snapshot, live_bytes
```

Relay 在 direct path 里只看到 rendezvous hints 和 authorization lifecycle event。Relay 看不到 QUIC payload。

## Relay fallback data flow

```mermaid
sequenceDiagram
  autonumber
  participant App as Android
  participant Relay as Relay
  participant Daemon as tunnel daemon

  App->>Relay: relay_tunnel_request(attempt_id, computer_id, fallback_reason)
  Relay->>Relay: authorize by account + app session + client_fingerprint + daemon trusted roster
  Relay->>App: relay_tunnel_ready(actor=client, tunnel_token)
  Relay->>Daemon: relay_tunnel_ready(actor=daemon, tunnel_token)
  App->>Relay: GET /connectivity/tunnel/ws + client token
  Daemon->>Relay: GET /connectivity/tunnel/ws + daemon token
  App->>Relay: binary WebSocket message = encrypted QUIC packet
  Relay->>Daemon: same binary packet unchanged
  Daemon->>Relay: binary WebSocket message = encrypted QUIC packet
  Relay->>App: same binary packet unchanged
  App->>Daemon: inside QUIC/TLS: daemon transport frames
  Daemon->>App: inside QUIC/TLS: daemon transport frames
```

Fallback 的 Relay WebSocket 只承载 binary QUIC packets。Relay 不解析：

- QUIC。
- TLS。
- daemon transport frame。
- session metadata。
- preview。
- terminal snapshot/live bytes。
- input/resize。

## Carrier 对比

| 项目 | Direct | Relay fallback |
|---|---|---|
| Packet carrier | UDP between Android and daemon | WebSocket binary packets through Relay |
| Relay 可见内容 | rendezvous hints, lifecycle events | tunnel setup metadata, encrypted QUIC packets |
| TLS identity | pairing-pinned Ed25519 certs | same |
| ALPN | `tunnel-conn/1` | same |
| daemon transport frame registry | same | same |
| terminal plaintext | endpoint only | endpoint only |
| security model | pinned peer identity + fresh TLS keys | same |
| UX difference | lower latency when reachable | works when direct UDP cannot connect |

## 失败和降级

Android 可以进入 Relay fallback 的原因包括：

- direct skipped because no usable candidates。
- direct timeout。
- QUIC handshake failed。
- authorization superseded by newer attempt。
- network path temporarily unavailable。

Relay 会在这些情况清理 live state：

- attempt expired。
- app logout。
- password change revokes app sessions。
- agent token revoked。
- daemon disconnect。
- user deleted。
- trusted client revoked。
- newer attempt supersedes older attempt。

清理 live state 不代表 Relay 获得 terminal plaintext。已经建立的 direct transport 也要受 daemon-side authorization lifecycle 约束；daemon 收到 `direct_session_close` 或发现 authorization socket gone 后应关闭对应 direct transport。
