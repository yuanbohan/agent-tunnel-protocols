# Security Model

本文把 Agent Tunnel 当前 SSOT 的安全边界集中到一个 threat model。详细 protocol shape 仍在各 owning docs 中维护。

## Assets

- Android client Ed25519 private key 和 trusted computer records。
- daemon Ed25519 private key 和 trusted client roster。
- daemon transport TLS session keys。
- terminal plaintext：snapshot、live bytes、input、resize、preview。
- app access/refresh token、agent token、fallback tunnel token。
- launch command、cwd、label。

## Trust Boundaries

- Pairing trust root 是 daemon public key 和 Android client public key。
- Relay 是 account auth、routing、presence、pairing forwarding、rendezvous、fallback token issuer；Relay 不是 terminal/session plaintext authority。
- Direct 和 Relay fallback 都必须运行同一套 pinned QUIC/TLS daemon transport。
- Relay fallback 只转发 encrypted QUIC packets。
- Local daemon trusted roster 是 client revocation 的 authoritative source。

## Attacker Model

| Attacker | 能力 | 必须保持的安全性质 |
|---|---|---|
| Malicious Relay | 改变/丢弃/重排 realtime events，拒绝 fallback，伪造 visibility timing | 不能解密 terminal plaintext，不能绕过 pinned daemon/client identity |
| Stolen app token | 调用 app-facing HTTP/realtime surfaces | 不应自动等价于 pairing private key；remote-command authority 必须被明确 threat-modeled |
| Compromised paired client | 使用合法 client key 发起 direct/fallback | 只能访问 daemon 授权给该 trusted client 的 computer/session scope；revocation 后失效 |
| Malicious daemon | 给 Android 发错误 metadata/terminal bytes | Android 只信 pairing-pinned daemon identity；account/Relay auth 不替代 daemon pin |
| Candidate abuse peer | 提交恶意 rendezvous candidates | Relay 和 endpoints 必须限制 address class，避免被用作 arbitrary UDP probe |

## Required Controls

- Pairing invitation 和 Android response 使用 Ed25519 signatures。
- SAS 使用 daemon key、client key、invitation id、nonce 的 length-prefixed SHA-256 transcript。
- QUIC/TLS certificate/public key 必须 pin 到 pairing 产出的 Ed25519 identities。
- TLS 1.3 session keys 每次 connection fresh；pairing 不保存 symmetric traffic keys。
- `0-RTT` 禁用。
- Fallback tunnel token 必须 short-lived、single-use、side-specific，并绑定 attempt/account/app session/client fingerprint/computer/actor。
- identity/pin/ALPN/protocol failure 必须 fail closed，不得降级成普通网络失败后继续 fallback。
- private rendezvous candidates 必须拒绝 loopback、unspecified、multicast、broadcast、unexpected public IP、以及非 test mode documentation ranges。

## Known Design Gates

### Launch Authority

当前实现的 `POST /api/computers/:computerID/sessions` 是 app access token + account scoped launch control plane。它没有 per-launch Ed25519 signature，也没有在 Relay handler 层强制 pairing-derived visibility。

这意味着 app access token 目前是 remote-command authority。后续如果要求 launch 必须绑定 pairing trust root，需要同步修改：

- Relay handler：按 `client_fingerprint` + live trusted roster 授权 target computer。
- Android：对 launch request 做 client-key proof，或至少只对 paired visible computer 发起。
- daemon：验证 request 关联的 client identity 后再执行 command。
- `docs/api.md` 和 implementation matrix。

### Pairing Completion

当前 Android 在 SAS 后等待 Relay-visible computer，再持久化 trusted record。由于 Relay 不是 trust root，长期更强的方案是：

- daemon 发送 signed `pair_completed` proof；或
- Android 把 trust 保持为 provisional，直到第一次 pinned daemon transport 成功。

在改实现之前，当前 SSOT 必须明确：transport pinning 仍保护 terminal plaintext，Relay visibility 只能影响 UX/availability，不能伪造 daemon private key。

### Revocation

Daemon-local revocation 是 authoritative：

- daemon 立即移除 trusted client 或标记 revoked。
- daemon 立即关闭该 client 的 direct/fallback transports 和 interactive ownership。
- Relay notification 是 best-effort。
- daemon 重新 registration 时发布过滤后的 trusted roster，清理 stale Relay visibility。
