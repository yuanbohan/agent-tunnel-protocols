# Implementation Status Matrix

本文记录 current SSOT 与 Go/Android 实现的对齐状态。它不是 protocol spec；spec 仍由 `docs/api.md`、`docs/architecture.md`、`docs/pairing.md`、`docs/relay-control-plane.md`、`docs/protocol.md` 和 `docs/end-to-end-flows.md` 拥有。

更新时间：2026-05-17。

| Surface | Current status | Evidence / gate |
|---|---|---|
| Relay public endpoint inventory | Mostly matched | `agent-tunnel/internal/relay/handler/new.go` 已对照；`GET /internal/version` 是 local-only route，不属于 public contract。 |
| Pairing transcript / SAS | Matched | Go 和 Android 使用 Ed25519，fingerprint = lowercase SHA-256(raw public key)，SAS golden vectors 一致。 |
| Trusted computer list | Matched | Android join local `TrustedComputerStore` + Relay visibility + account policy；Relay 不持久化 trusted roster。 |
| Session list / recent output preview | Matched | Session rows 和 preview 来自 daemon transport，不来自 Relay session API。 |
| Relay fallback | Matched | Android/daemon 通过 `/connectivity/tunnel/ws` 交换 opaque encrypted QUIC packets。 |
| Direct-first | Implemented with evidence caveats | Direct path selection and diagnostics exist；direct success proof 应以 `job_connected path_kind=direct` 或等价 evidence 为 gate。 |
| Session detail input | Matched | Android 通过 daemon transport 发送 `input_text` / `input_key`；Relay 不承载 input。 |
| Initial terminal geometry | Matched | Android 在 `interactive_request` 发送 initial cols/rows。 |
| Live terminal resize | Protocol-supported, Android not wired | `resize` frame 在 protocol/daemon side 存在；Android `onTerminalGeometryChanged` 当前不能声称已发送 live resize。 |
| Launch authority | Design gate open | 当前 launch 是 app token + account scoped remote-command authority；paired-client launch authorization 需要后续实现和 SSOT update。 |
| Pairing completion proof | Design gate open | 当前 Android 用 Relay visibility 完成 trust persistence；更强模型是 daemon-signed completion proof 或 first pinned transport success。 |
| Rendezvous candidate validation | Design gate open | SSOT 已要求 address-class filtering；实现仍需确保 Relay 和 endpoints 不只做 `ip:port` parse。 |

## Verification Guidance

Go focused checks：

```bash
go test ./internal/connectivity/... ./internal/protocol ./internal/relay/... ./internal/tunnel/daemon
```

Android focused checks：

```bash
./gradlew :app:testDevDebugUnitTest --tests "ai.diaro.agentunnel.*Pairing*Test"
./gradlew :app:testDevDebugUnitTest --tests "ai.diaro.agentunnel.*Connectivity*Test"
./gradlew :app:testDevDebugUnitTest --tests "ai.diaro.agentunnel.ui.session.SessionDetail*Test"
```
