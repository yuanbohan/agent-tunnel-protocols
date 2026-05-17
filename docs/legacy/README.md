# Legacy Designs

这个目录保存 historical/superseded 设计，目的是防止后续重新采用已经被 current SSOT 淘汰的方案。

这些文档不是 current protocol authority。当前事实以这些文档为准：

- [Relay API](../api.md)
- [Architecture](../architecture.md)
- [Pairing](../pairing.md)
- [Relay Control Plane](../relay-control-plane.md)
- [Daemon Transport Protocol](../protocol.md)
- [End-To-End Flows](../end-to-end-flows.md)
- [Security](../security.md)

## 读取规则

- Legacy 文档可以解释“为什么不要这么做”，不能作为新实现的 contract。
- 如果 legacy 文档和 current SSOT 冲突，current SSOT wins。
- 如果需要恢复一个 legacy 行为，必须先更新 current SSOT、security model 和 implementation status，再改实现。

## 当前收纳的 legacy 来源

来自 `agent-tunnel`：

- `agent-tunnel/connectivity/architecture.md` - 早期 direct session connectivity architecture mirror。
- `agent-tunnel/connectivity/contract.md` - 早期 Go implementation boundary。
- `agent-tunnel/connectivity/reference/*` - 早期 decision record、state machines、sequence flows、error catalog。
- `agent-tunnel/connectivity/ux/*` - 早期 Android/subscription UX reference。
- `agent-tunnel/connectivity/archive/2026-04-26-architect-review.md` - point-in-time architecture review。
- `agent-tunnel/tui-attach-flow.md` - 旧 Relay session viewing path。

来自 `agent-tunnel-android`：

- `agent-tunnel-android/review-for-milestone2.md` - Android milestone-2 point-in-time review。里面包含一些已经修复或失效的 findings，不能作为 current status。

## 明确淘汰的模式

- Relay 作为 plaintext terminal/session data plane。
- Relay session list/detail/attach/frame replay 作为 official mobile companion 的 session authority。
- `GET /api/devices`、`POST /api/devices/:id/launch`、`GET /api/connectivity/app/ws`、`GET /connectivity/daemon/ws` 作为当前 primary mobile protocol。
- `device_fingerprint` 作为新 client 的 primary field，而不是 `client_fingerprint`。
- Basic Auth/password persistence 作为 Android Relay config。
- Relay-visible event 单独作为 pairing trust completion proof。
- 只验证 `ip:port` 就转发/probe rendezvous private candidates。
