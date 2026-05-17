# Connectivity Pairing 协议

## 状态

本文是 mobile client installation 和 computer daemon pairing 的跨仓库 SSOT。

Pairing 建立 durable pinned peer identities。它不创建长期 symmetric shared secret，也不让 Relay 成为 trust root。

流程图见 [draws/01-pairing.md](draws/01-pairing.md)。

## 目标

- 把一个 mobile client installation 绑定到一个 computer daemon identity。
- 保持 daemon 和 mobile endpoints 作为 trust roots。
- Relay 只负责 authenticated account context 和 message transport。
- 产出后续 QUIC/TLS daemon transport 使用的 pinned identities。
- 用 human-confirmed SAS 发现 network-side public-key substitution。

## Device Identity 模型

两端各自持有一个 persistent Ed25519 identity key pair。

Daemon：

- 本地保存 connectivity identity。
- 用 daemon private key 签名 invitation。
- 后续 daemon transport 使用 self-signed Ed25519 certificate；certificate 的 SubjectPublicKeyInfo 就是 daemon public key。

Mobile client：

- 使用 platform secure storage 保存 app client identity。
- 用 client private key 签名 pairing response。
- 后续 daemon transport 使用 self-signed Ed25519 certificate；certificate 的 SubjectPublicKeyInfo 就是 client public key。

Public-key fingerprint：

```text
fingerprint = lowercase_hex(SHA-256(raw_ed25519_public_key))
```

Display name 不是 trust identity。

## Protocol Version

当前 pairing protocol version：

```text
2
```

Version `2` 使用 Ed25519 public keys/signatures、SHA-256 fingerprints，以及下面定义的 canonical transcripts。

## Invitation

`tunnel pair` 创建一个短期、一次性的 invitation。当前 daemon invitation TTL 是 5 minutes。

JSON invitation fields：

- `version`
- `account_id`
- `computer_id`
- `computer_display_name`
- `computer_public_key`
- `computer_fingerprint`
- `invitation_id`
- `correlation_id`
- `nonce`
- `expires_at`
- `relay_base_url`
- `signature`

`computer_public_key` 是 32 bytes raw Ed25519 public key，lowercase hex 编码。`computer_fingerprint` 必须等于 `SHA-256(computer_public_key_raw_bytes)`。

Daemon signature 是 Ed25519 over canonical invitation transcript：

```text
domain = "tunnel-pairing-invitation-v1"

fields, in order:
- account_id
- computer_id
- computer_display_name
- normalized computer_fingerprint
- computer_public_key
- invitation_id
- correlation_id
- nonce
- relay_base_url
- expires_at
```

每个 transcript field 都通过 canonical encoder 做 length prefix。Receiver 必须验证 canonical transcript，不能依赖 JSON 原始字节顺序。

Mobile client 必须拒绝这些 invitation：

- `version != 2`
- required field 缺失
- invitation expired
- `account_id` 和当前 authenticated Relay account 不匹配
- `computer_public_key` 不是 32 bytes
- `computer_fingerprint` 和 public key 不匹配
- Ed25519 signature 失败

## Compact QR Form

实现可以把同一个 invitation 编成 compact QR payload：

```text
TP2:<base45-payload>
```

Compact payload 表达同样的 version、account-scoped context、computer id、invitation id、correlation id、nonce、expiry、computer public key、signature、display name。Compact import 可以从 authenticated app context 获取 `account_id` 和 `relay_base_url`。解码后的 invitation 必须按 JSON invitation 同样规则验证。

## Android Response

Mobile client 验证 invitation 后签名 Android response。

JSON response fields：

- `version`
- `account_id`
- `invitation_id`
- `correlation_id`
- `client_public_key`
- `client_fingerprint`
- `client_display_name`
- `signature`

`client_public_key` 是 32 bytes raw Ed25519 public key，lowercase hex 编码。`client_fingerprint` 必须等于 `SHA-256(client_public_key_raw_bytes)`。

Client signature 是 Ed25519 over canonical Android response transcript：

```text
domain = "tunnel-pairing-android-response-v1"

fields, in order:
- account_id
- invitation_id
- correlation_id
- normalized client_fingerprint
- client_public_key
- client_display_name
```

Daemon 必须拒绝这些 response：

- `version != 2`
- `account_id`、`invitation_id`、`correlation_id` 和 invitation record 不匹配
- invitation missing、expired、already consumed
- `client_public_key` 不是 32 bytes
- `client_fingerprint` 和 public key 不匹配
- Ed25519 signature 失败

## Relay Pairing Transport

Relay 参与 pairing transport，但不建立 trust。

Daemon reservation：

1. daemon 连接 `GET /connectivity/computer/ws`
2. daemon 发送 `pair_invitation_reserve`
3. Relay 为 authenticated daemon account 预留 live correlation
4. Relay 返回 `pair_invitation_reserved`

Mobile response submission：

```text
POST /api/pairing/responses
Authorization: Bearer <app-access-token>
```

Relay 必须验证：

- app bearer token 有效
- app session 绑定了 client fingerprint
- submitted `account_id` 匹配 authenticated app account
- submitted `client_fingerprint` 匹配 app session fingerprint
- correlation id 存在、未过期，并且属于同 account 下的 daemon

Relay 然后以 `pair_response_forward` 把 signed response 转发给 daemon。Relay 不得修改 signed fields。

## SAS

SAS 是 Short Authentication String，用来让人发现 network-side public-key substitution。

两端用完全相同的输入计算 SAS，顺序固定：

1. `computer_public_key` raw 32 bytes
2. `client_public_key` raw 32 bytes
3. `invitation_id` UTF-8 bytes
4. `nonce` raw bytes

每个输入编码为：

```text
u16be(length) || bytes
```

算法：

```text
canonical = lp(computer_public_key)
         || lp(client_public_key)
         || lp(invitation_id_utf8)
         || lp(nonce)

digest  = SHA-256(canonical)
short   = first 4 digest bytes interpreted as big-endian uint32
sas     = short mod 1_000_000
display = sas as zero-padded 6 decimal digits
```

它大约提供 20 bits 的 human-confirmed MITM resistance。它不是 password、shared secret 或 network-verifiable token。

Relay 不能自动比较 SAS，因为 SAS 存在的原因正是 Relay 不被信任来证明两端看到了相同 key。

## Trust Completion

SAS 确认前，pairing 不算完成。

Daemon completion：

1. daemon 把 verified response 存成 pending
2. daemon CLI 展示 SAS 并要求 operator 输入/确认
3. SAS 匹配后，daemon 标记 invitation consumed
4. daemon 持久化 Android client public key/fingerprint 为 trusted
5. daemon 向 Relay 发送 `pair_completed`

Mobile completion：

1. Android response submission 后保存 pending pairing record
2. Android 向用户展示 SAS
3. Android 等 Relay realtime 报告 paired computer visible
4. Android 持久化 trusted computer record
5. Android 清理 pending pairing state

任何一侧出现 mismatch、expiry、account mismatch、invalid signature、missing pending state，都必须 fail closed。Invitation 一次性使用。

## Local Persistence

Daemon 使用 daemon-local private state 保存：

- Ed25519 connectivity identity
- invitation records
- pending pairing responses
- trusted Android clients
- revoked Android clients

Android 使用 app-local protected storage 保存：

- Ed25519 client identity
- pending pairing records
- trusted computer records
- Relay app session credentials

Relay 只保存 live pairing correlations 和 live visibility/routing state。Relay 不是 durable trusted-client database。

## Revocation

Daemon 是 trust revocation 的 authority。

执行 `tunnel pair revoke <fingerprint>` 后：

- daemon 本地标记/移除 trusted client
- daemon 拒绝该 fingerprint 的未来 daemon transport handshake
- daemon 关闭该 client 的 active connections 和 interactive ownership
- daemon 用 `client_revoked` 通知 Relay
- Relay 移除 derived live visibility，并关闭相关 rendezvous/fallback state
- mobile client 必须停止把该 computer 当作 trusted

## Transport Consequence

Pairing 的产物：

- mobile client 上的 pinned daemon identity
- daemon 上的 pinned mobile client identity
- 两端本地 trust records

后续 direct 或 Relay-fallback daemon transport 使用这些 pinned public keys 认证新的 TLS 1.3 handshake。Symmetric traffic keys 每个 transport connection 都重新生成，不由 pairing 持久保存。
