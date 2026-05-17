# 0. Computer List

这个图回答：用户打开 Android app 后，为什么能看到某些 computers，为什么有些本地信任过的 computers 会显示 offline，以及为什么 Relay presence 不能单独算 trusted。

## 状态来源

```mermaid
flowchart TD
  classDef mobile fill:#e8f3ff,stroke:#2f6fbd,color:#0f2742
  classDef relay fill:#fff4db,stroke:#b66b00,color:#3b2600
  classDef daemon fill:#eaf8ef,stroke:#2b8a4b,color:#102f1c
  classDef decision fill:#f3ecff,stroke:#7557c2,color:#26164a
  classDef muted fill:#f5f5f5,stroke:#999,color:#333

  A[Android app starts]:::mobile
  B[Load Relay credentials<br/>and client Ed25519 identity]:::mobile
  C[Load TrustedComputerStore<br/>account_id + relay_base_url + computer_id + fingerprint]:::mobile
  D[Open GET /api/connectivity/ws<br/>Bearer app access token]:::relay
  E[Send app_register<br/>protocol_version = 2]:::mobile
  F[Relay checks app session<br/>and bound client_fingerprint]:::relay
  G[Daemons online on<br/>/connectivity/computer/ws]:::daemon
  H[Daemon registration includes<br/>trusted_clients roster]:::daemon
  I[Relay sends computer_snapshot<br/>then visible/removed/revoked events]:::relay
  J{Local trusted record<br/>matches Relay visible computer?}:::decision
  K[Show trusted computer online<br/>start daemon connection manager]:::mobile
  L[Show local trusted computer offline<br/>do not start transport]:::mobile
  M[Ignore as not locally trusted<br/>not a trusted computer row]:::muted
  P[Fetch GET /api/account/policy<br/>free/pro trusted computer limit]:::relay

  A --> B --> C
  A --> D --> E --> F --> I
  G --> H --> I
  C --> J
  I --> J
  P --> J
  J -->|yes| K
  J -->|local only| L
  J -->|Relay only| M
```

## Sequence

```mermaid
sequenceDiagram
  autonumber
  participant App as Android companion
  participant Store as Protected local store
  participant Relay as Relay
  participant Daemon as tunnel daemon

  App->>Store: read client identity and trusted computer records
  Daemon->>Relay: GET /connectivity/computer/ws + computer_register(trusted_clients)
  App->>Relay: GET /api/connectivity/ws + Bearer app token
  App->>Relay: app_register(protocol_version=2)
  Relay->>Relay: bind account_id + app_session_id + client_fingerprint
  Relay->>App: computer_snapshot(visible computers for this fingerprint)
  App->>Relay: GET /api/account/policy
  Relay->>App: tier free/pro
  App->>App: join local trust records with Relay live visibility
  App->>App: render online/offline trusted computer sections
```

## 读图要点

- Android 的 trusted computer list 是 local trust + Relay live visibility + account policy 的 projection。
- Relay 只知道当前 online 且由 daemon registration 报告 trusted roster 的 computers。Relay 不保存 durable trusted computer database。
- 本地有 trusted record 但 Relay 没有 visible event，UI 可以展示 offline。
- Relay 有 visible computer 但本地没有 trusted record，Android 不能把它当 trusted computer，因为 app installation 没有本地 pinned daemon identity。
- `GET /api/computers` 是 remote launch control plane list，只表示 `/device/ws` online launchable computers；它不能替代 trusted computer list。

## 安全边界

Android 启动 daemon transport 前，必须确认：

- local trusted record 的 `computer_id` 和 `computer_fingerprint` 匹配 Relay visible computer。
- app session 绑定的 `client_fingerprint` 与本地 client public key 对应。
- account 和 Relay base URL 没有混用。

这些检查防止同 account 下的非信任 daemon、旧 app installation、或者错误 Relay environment 被误投影成同一台 trusted computer。
