# Connectivity Pairing Protocol

## Status

This document is the cross-repository source of truth for pairing one mobile
client installation with one computer daemon.

Pairing establishes durable pinned peer identities. It does not create a
long-lived symmetric shared secret and it does not make Relay a trust root.

## Goals

- bind one mobile client installation to one computer daemon identity
- keep the daemon and mobile endpoints as the trust roots
- use Relay only for authenticated account context and message transport
- produce pinned identities for later QUIC/TLS daemon transport connections
- make network key substitution detectable through a human-confirmed SAS

## Device Identity Model

Each endpoint owns one persistent Ed25519 identity key pair.

Daemon:

- stores its connectivity identity in daemon-local state
- signs invitations
- authenticates later daemon transports with a self-signed Ed25519
  certificate whose SubjectPublicKeyInfo is the daemon public key

Mobile client:

- stores its app client identity in platform secure storage
- signs pairing responses
- authenticates later daemon transports with a self-signed Ed25519
  certificate whose SubjectPublicKeyInfo is the client public key

The public-key fingerprint is:

```text
fingerprint = lowercase_hex(SHA-256(raw_ed25519_public_key))
```

Display names are not trust identities.

## Protocol Version

Current pairing protocol version:

```text
2
```

Version `2` uses Ed25519 public keys and signatures, SHA-256 fingerprints, and
the canonical transcripts below.

## Invitation

`tunnel pair` creates one short-lived, one-time invitation. The current
daemon invitation TTL is 5 minutes.

JSON invitation fields:

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

`computer_public_key` is 32 raw Ed25519 public-key bytes encoded as lowercase
hex. `computer_fingerprint` must equal
`SHA-256(computer_public_key_raw_bytes)`.

The daemon signature is Ed25519 over the canonical invitation transcript:

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

Each transcript string or byte field is length-prefixed by the implementation
canonical encoder. Receivers must verify against the canonical transcript, not
against raw JSON byte order.

The mobile client must reject an invitation when:

- `version` is not `2`
- a required field is missing
- the invitation is expired
- `account_id` does not match the authenticated Relay account
- `computer_public_key` is not 32 bytes
- `computer_fingerprint` does not match the public key
- the Ed25519 signature fails

## Compact QR Form

Implementations may encode the same invitation as a compact QR payload:

```text
TP2:<base45-payload>
```

The compact payload is an encoding of the same version, account-scoped context,
computer id, invitation id, correlation id, nonce, expiry, computer public key,
signature, and display name. Account id and Relay base URL may come from the
authenticated app context for compact imports. The decoded invitation must
validate identically to the JSON invitation before it is trusted.

## Android Response

The mobile client signs an Android response after validating the invitation.

JSON response fields:

- `version`
- `account_id`
- `invitation_id`
- `correlation_id`
- `client_public_key`
- `client_fingerprint`
- `client_display_name`
- `signature`

`client_public_key` is 32 raw Ed25519 public-key bytes encoded as lowercase
hex. `client_fingerprint` must equal
`SHA-256(client_public_key_raw_bytes)`.

The client signature is Ed25519 over the canonical Android response transcript:

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

The daemon must reject the response when:

- `version` is not `2`
- `account_id`, `invitation_id`, or `correlation_id` differs from the
  invitation record
- the invitation is missing, expired, or already consumed
- `client_public_key` is not 32 bytes
- `client_fingerprint` does not match the public key
- the Ed25519 signature fails

## Relay Pairing Transport

Relay participates in pairing transport but not trust establishment.

Daemon reservation:

1. daemon connects to `GET /connectivity/computer/ws`
2. daemon sends `pair_invitation_reserve`
3. Relay reserves a live correlation for the authenticated daemon account
4. Relay replies with `pair_invitation_reserved`

Mobile response submission:

```text
POST /api/pairing/responses
Authorization: Bearer <app-access-token>
```

Relay must validate:

- app bearer token is valid
- app session has a bound client fingerprint
- submitted `account_id` matches the authenticated app account
- submitted `client_fingerprint` matches the app session fingerprint
- correlation id exists, is live, and belongs to a daemon under the same
  account

Relay then forwards the signed response to the daemon as
`pair_response_forward`. Relay must not modify signed fields.

## SAS

SAS means Short Authentication String. It lets the human operator detect
network-side public-key substitution.

Both sides compute the SAS from exactly these inputs, in order:

1. `computer_public_key` as 32 raw bytes
2. `client_public_key` as 32 raw bytes
3. `invitation_id` as UTF-8 bytes
4. `nonce` as raw bytes

Each input is encoded as:

```text
u16be(length) || bytes
```

The algorithm is:

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

This gives about 20 bits of human-confirmed MITM resistance. It is not a
password, a shared secret, or a network-verifiable token.

Relay must not auto-compare SAS values. The SAS exists specifically because
Relay is not trusted to prove that both endpoints saw the same keys.

## Trust Completion

Pairing is complete only after SAS confirmation.

Daemon completion:

1. daemon stores the verified response as pending
2. daemon CLI shows the SAS and asks the operator to enter/confirm it
3. if the SAS matches, daemon marks the invitation consumed
4. daemon persists the Android client public key/fingerprint as trusted
5. daemon sends `pair_completed` to Relay

Mobile completion:

1. Android stores a pending pairing record after response submission
2. Android shows the SAS to the user
3. Android waits for Relay realtime to report the paired computer as visible
4. Android persists the trusted computer record locally
5. Android clears pending pairing state

If either side reports mismatch, expiry, account mismatch, invalid signature,
or missing pending state, pairing fails closed. Invitations are one-time use.

## Local Persistence

Daemon stores, under daemon-local state with private permissions:

- Ed25519 connectivity identity
- invitation records
- pending pairing responses
- trusted Android clients
- revoked Android clients

Android stores, in app-local protected storage:

- Ed25519 client identity
- pending pairing records
- trusted computer records
- Relay app session credentials

Relay stores only live pairing correlations and live visibility/routing state.
Relay is not the durable trusted-client database.

## Revocation

The daemon is authoritative for trust revocation.

After `tunnel pair revoke <fingerprint>`:

- daemon marks/removes the trusted client locally
- daemon rejects future daemon transport handshakes from that fingerprint
- daemon closes active connections and interactive ownership for that client
- daemon notifies Relay with `client_revoked`
- Relay removes derived live visibility and closes affected rendezvous/fallback
  state
- the mobile client must stop treating the computer as trusted

## Transport Consequence

Pairing output is:

- a pinned daemon identity on the mobile client
- a pinned mobile client identity on the daemon
- local trust records on both endpoints

Later direct or Relay-fallback daemon transports use those pinned public keys
to authenticate a fresh TLS 1.3 handshake. The symmetric traffic keys are
fresh per transport connection and are not stored by pairing.
