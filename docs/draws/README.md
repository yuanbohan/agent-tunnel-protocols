# Draws

这里集中放 Agent Tunnel connectivity 的流程图。文字版 SSOT 在上一层文档里；这里用 Mermaid 把关键路径画出来，方便快速建立全局模型。

建议阅读顺序：

1. [0. Computer List](00-computer-list.md)
2. [1. Pairing](01-pairing.md)
3. [2. Session List And Preview](02-session-list-preview.md)
4. [3. Direct And Relay](03-direct-relay-data-flow.md)
5. [4. Detail Input](04-detail-input.md)

这些图的边界：

- 图里的 Relay 是 authentication/control/routing plane。
- 图里的 daemon transport 是 pinned QUIC/TLS session。
- Relay fallback 图中经过 Relay 的内容是 encrypted QUIC packets，不是 terminal plaintext。
- Android 本地 storage 和 daemon 本地 storage 都是 trust boundary 的一部分，不是 UI cache。

## 总览

```mermaid
flowchart LR
  classDef mobile fill:#e8f3ff,stroke:#2f6fbd,color:#0f2742
  classDef relay fill:#fff4db,stroke:#b66b00,color:#3b2600
  classDef daemon fill:#eaf8ef,stroke:#2b8a4b,color:#102f1c
  classDef danger fill:#ffe9e9,stroke:#c43d3d,color:#4a1111

  A[Android companion<br/>client identity<br/>trusted store]:::mobile
  R[Relay<br/>auth, presence, pairing transport,<br/>rendezvous, fallback tokens]:::relay
  D[tunnel daemon<br/>trusted roster, broker cache,<br/>direct UDP, fallback QUIC]:::daemon
  T[tunnel run<br/>PTY owner<br/>terminal mirror]:::daemon
  N[Not Relay-owned<br/>session list, preview,<br/>terminal bytes, input]:::danger

  A <--> R
  D <--> R
  D <--> T
  A <== "pinned QUIC/TLS<br/>direct UDP or relay packet carrier" ==> D
  R -. "must not parse" .-> N
```
