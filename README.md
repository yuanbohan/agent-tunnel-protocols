# agent-tunnel-protocols

Agent Tunnel 跨仓库协议文档的 SSOT。

这个仓库保存 Android companion、Relay/server、Go tunnel/daemon 共享的协议、数据流、安全边界和兼容性决策。实现仓库可以保留本地实现说明、测试入口和运维记录，但 shared protocol 的事实要指向这里。

## 本地 sibling 仓库

跨仓库工作时，建议保持这些 checkout 并列：

- `../agent-tunnel` - Go Relay、tunnel daemon、STUN、direct UDP、fallback tunnel、pairing state、daemon transport 实现。
- `../agent-tunnel-android` - official Android companion 实现和 mobile UX/docs。

改动这里的 protocol 文档时，通常也需要在一个或两个实现仓库里同步实现、测试或本地导航。实现仓库不应该维护第二份 current protocol mirror。

## 文档入口

- [End-To-End Flows](docs/end-to-end-flows.md) - trusted computer list、pairing、session list、recent output preview、direct/relay、detail input、key storage、卸载后重新 pairing。
- [Draws](docs/draws/README.md) - 上面关键流程的 Mermaid 流程图和 sequence diagram。
- [Relay API](docs/api.md) - 当前 public Relay API、auth、endpoint、WebSocket、fallback tunnel、removed endpoints。
- [Architecture](docs/architecture.md) - `tunnel run`、daemon、Relay、PostgreSQL、local broker、mobile companion 的系统结构。
- [Pairing](docs/pairing.md) - Ed25519 pairing transcript、Relay forwarding、SAS、trust completion、persistence、revocation。
- [Relay Control Plane](docs/relay-control-plane.md) - Relay realtime auth、presence、pairing forwarding、direct rendezvous、fallback tunnel token、opaque packet forwarding。
- [Daemon Transport Protocol](docs/protocol.md) - daemon-to-mobile QUIC transport、frame registry、payload、stream、security invariants、direct/relay carrier boundary。
- [Security](docs/security.md) - trust boundary、attacker model、stolen token、malicious Relay、candidate abuse、revocation 和 fail-closed 规则。
- [Implementation Status](docs/status/implementation-matrix.md) - 当前 Go/Android 实现与 SSOT 的验证状态、已知 gap 和升级 gate。
- [Legacy](docs/legacy/README.md) - historical/superseded 设计，防止重新采用已经淘汰的 Relay session authority 等方案。

## 仓库规则

- Fixture 必须是 synthetic 且不包含 secret。不要提交真实 terminal capture、credential、private key、tunnel token、device fingerprint、private path 或 user input。
- Protocol-breaking change 必须更新负责该 surface 的文档，并说明 compatibility line 或 protocol version 迁移。
- Relay 对 terminal bytes 和 fallback QUIC packets 保持 content-opaque，除非未来明确修改 protocol boundary。
- Implementation repo 只保留 README/AGENTS/CLAUDE 的导航、implementation notes、tests、evidence、ops/fix docs；不要复制 endpoint table、frame registry、SAS transcript、security invariant 等 protocol facts。
