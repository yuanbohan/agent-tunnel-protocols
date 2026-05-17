# agent-tunnel-protocols

这个仓库是 Agent Tunnel 跨仓库 protocol decision 的 SSOT。

## Sibling 仓库

- `../agent-tunnel` - Go Relay、tunnel daemon、STUN、direct UDP、fallback tunnel、pairing state、daemon transport 实现。
- `../agent-tunnel-android` - official Android companion 实现和 mobile UX/docs。

## 文档规则

- 跨仓库 protocol facts 优先写在这里：pairing transcripts、SAS、Relay connectivity realtime frames、rendezvous/fallback rules、daemon transport frames、stream roles、security invariants、direct/relay data flow、trusted-computer flows、mobile detail input data flow。
- 实现仓库可以保留 implementation notes、tests、evidence、ops/fix docs 和 README/AGENTS/CLAUDE 导航，但不应该保留 current protocol mirror 或独立 pointer docs。Endpoint table、frame registry、SAS transcript、security invariant 等 protocol facts 只在这里维护。
- 文档以中文描述为主，但保留 protocol、Relay、daemon、client、fingerprint、QUIC、TLS、WebSocket、frame、payload 等英文术语。
- Fixture 必须是 synthetic 且不含 secret。不要提交真实 credential、private key、tunnel token、device fingerprint、terminal capture、private path 或 user input。
- Protocol-breaking change 必须更新 owning document，并说明 compatibility line 或 protocol version transition。
