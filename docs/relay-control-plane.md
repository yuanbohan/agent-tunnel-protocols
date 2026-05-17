# Relay Connectivity Control Plane

## Status

This document is the cross-repository source of truth for the Relay-owned
connectivity control plane used by the official mobile companion and Go daemon.

Relay owns authentication, account context, live daemon presence,
pairing-response routing, direct rendezvous hint exchange, fallback tunnel
authorization, and opaque fallback packet forwarding. Relay does not own
session discovery, previews, terminal bytes, input, resize, or interactive
authorization.

## Endpoint Inventory

App endpoints:

- `GET /api/connectivity/ws`
- `POST /api/pairing/responses`
- `GET /api/account/policy`
- `POST /api/computers/:computerID/sessions`

Daemon endpoints:

- `GET /connectivity/computer/ws`

Fallback packet tunnel:

- `GET /connectivity/tunnel/ws`

The old connectivity realtime aliases are not part of this compatibility line.

## App Authentication

App-facing Relay endpoints use:

```text
Authorization: Bearer <app-access-token>
```

The app access token is opaque to clients. Relay stores app session ownership
server-side and binds that app session to the app client fingerprint supplied
at login/refresh time.

Relay uses `(account_id, app_session_id, client_fingerprint)` as the app-side
identity for:

- connectivity realtime registration
- pairing response submission
- trusted computer visibility
- rendezvous attempts
- fallback tunnel requests
- account policy reads

If an app session has no bound client fingerprint, Relay must reject
connectivity realtime.

## Daemon Authentication

Daemon connectivity realtime uses:

```text
Authorization: Bearer <agent-token>
```

Relay binds the daemon socket to the account that owns the agent token.
Daemon-local trust remains daemon-owned; Relay only receives the current live
trusted roster during `computer_register` and later `pair_completed` /
`client_revoked` events.

## Shared JSON Envelope

Realtime messages are JSON objects with:

- `type`
- optional `protocol_version`
- optional `request_id`
- event-specific fields

Current realtime protocol version:

```text
2
```

Peers must tolerate unknown event types where the local implementation can
safely ignore them. Relay should answer unsupported app commands with an
`error` frame rather than closing a valid app socket.

## App Realtime Socket

The app opens:

```text
GET /api/connectivity/ws
Authorization: Bearer <app-access-token>
```

The first app frame must be:

```json
{"type":"app_register","protocol_version":2}
```

Relay then sends:

- `computer_snapshot`
- later `computer_visible`
- later `computer_removed`
- later `client_revoked`
- direct rendezvous and fallback events addressed to that app session

`computer_snapshot` fields:

- `type`: `computer_snapshot`
- `computers`: array of `ConnectivityComputer`

`ConnectivityComputer` fields:

- `computer_id`
- `display_name`
- `platform_family`
- `platform_id`
- `computer_public_key`
- `computer_fingerprint`
- `tunnel_version`

Relay computes visible computers by matching the authenticated app account and
client fingerprint against the live daemon trusted roster.

## Daemon Realtime Socket

The daemon opens:

```text
GET /connectivity/computer/ws
Authorization: Bearer <agent-token>
```

The first daemon frame must be `computer_register`:

```json
{
  "type": "computer_register",
  "protocol_version": 2,
  "computer": {
    "computer_id": "dev_abcd1234",
    "display_name": "Work Mac",
    "platform_family": "macos",
    "platform_id": "macos",
    "computer_public_key": "<hex-ed25519-public-key>",
    "computer_fingerprint": "<hex-sha256-public-key>",
    "tunnel_version": "v0.1.0"
  },
  "trusted_clients": [
    {
      "fingerprint": "<client-fingerprint>",
      "display_name": "Pixel"
    }
  ],
  "direct_sessions": [
    {
      "attempt_id": "<attempt-id>",
      "client_fingerprint": "<client-fingerprint>"
    }
  ]
}
```

Relay uses this to rebuild live visibility after daemon reconnect. Relay does
not persist `trusted_clients` durably.

## Pairing Event Family

Daemon to Relay:

- `pair_invitation_reserve`
- `pair_completed`
- `client_revoked`

Relay to daemon:

- `pair_invitation_reserved`
- `pair_response_forward`
- `error`

App to Relay:

- `POST /api/pairing/responses`

Relay to app:

- `computer_visible`
- `client_revoked`
- `computer_removed`

Pairing transcript and SAS rules are defined in [pairing.md](pairing.md).

## Direct Rendezvous Event Family

Direct attempts exchange UDP candidate hints through Relay realtime.

App sends `rendezvous_open`:

```json
{
  "type": "rendezvous_open",
  "request_id": "req-1",
  "attempt_id": "attempt-1",
  "computer_id": "dev_abcd1234",
  "public_udp_addr": "203.0.113.10:50000",
  "private_udp_addrs": ["10.0.0.5:50000"]
}
```

Relay forwards a client-origin `rendezvous_hint` to the daemon with the
authenticated `client_fingerprint`.

Daemon replies with a daemon-origin `rendezvous_hint`:

```json
{
  "type": "rendezvous_hint",
  "request_id": "req-1",
  "attempt_id": "attempt-1",
  "computer_id": "dev_abcd1234",
  "client_fingerprint": "<client-fingerprint>",
  "actor": "daemon",
  "public_udp_addr": "198.51.100.20:50000",
  "private_udp_addrs": ["10.0.0.8:50000"],
  "expires_at": 1777478400
}
```

Either side may send `rendezvous_close`. After direct QUIC accept succeeds,
the daemon sends `direct_session_open`. Relay records that direct won the
attempt. When authorization is revoked, Relay may send `direct_session_close`
to the daemon so direct transports tied to the revoked live authorization close
promptly.

Rendezvous rules:

- `attempt_id` is minted by the app per connection attempt
- hints are short-lived; current implementation default is 30 seconds
- a newer attempt for the same app session and computer supersedes older live
  attempt state
- candidate lists must be bounded
- private candidate addresses should be limited to private, link-local, or
  explicitly test-allowed ranges

Relay may route candidate hints, but it must not derive terminal/session
semantics from them.

## Fallback Relay Tunnel Event Family

App sends `relay_tunnel_request` after direct is skipped, fails, or times out:

```json
{
  "type": "relay_tunnel_request",
  "request_id": "req-2",
  "attempt_id": "attempt-1",
  "computer_id": "dev_abcd1234",
  "fallback_reason": "direct_timeout",
  "direct_setup_latency_ms": 3000
}
```

Relay authorizes fallback only when the authenticated app account and bound
client fingerprint currently have pairing-derived visibility to the online
daemon.

Relay sends side-specific `relay_tunnel_ready` frames:

```json
{
  "type": "relay_tunnel_ready",
  "request_id": "req-2",
  "attempt_id": "attempt-1",
  "computer_id": "dev_abcd1234",
  "client_fingerprint": "<client-fingerprint>",
  "actor": "client",
  "tunnel_token": "<single-use-token>",
  "fallback_reason": "direct_timeout",
  "direct_setup_latency_ms": 3000
}
```

The daemon receives a different token with `actor: "daemon"`.

Tunnel-token rules:

- one token per side
- short-lived
- single-use
- bound to attempt id, account, app session, client fingerprint, target
  computer, actor identity, and actor type
- invalidated on expiry, disconnect, logout, token revocation, user deletion,
  daemon disconnect, superseding attempts, or trusted-device revocation

## Fallback Packet Tunnel

Both sides redeem their tokens at:

```text
GET /connectivity/tunnel/ws
Authorization: Bearer <single-use-token>
```

The WebSocket carries binary messages only. Each binary message is one opaque
encrypted QUIC packet. Relay pairs the client and daemon endpoints for the same
attempt and forwards binary packets unchanged.

Relay must close the tunnel on text messages, invalid tokens, token reuse,
authorization revocation, peer disconnect, or tunnel expiry.

## Account Policy Surface

Relay exposes account policy to the app through authenticated app APIs. Current
mobile product policy uses:

- `free`: app may keep one active trusted computer
- `pro`: app may keep up to ten active trusted computers

Relay does not issue per-session grants, does not decide which daemon session
row can be opened, and does not send tier policy to daemons in this
compatibility line.

## Relay Must Not Carry

Relay realtime and fallback tunnel payloads must not carry plaintext:

- session list
- recent output preview text
- terminal snapshot bytes
- live terminal bytes
- mobile input
- resize payloads
- daemon-side interactive grants
- daemon session metadata from the QUIC transport

Those belong to the pinned daemon transport defined in [protocol.md](protocol.md).

## Failure Semantics

Relay failure can prevent:

- sign-in and token refresh
- new pairing
- current trusted-computer visibility updates
- new direct rendezvous
- new fallback tunnel setup
- mobile-created launch requests
- account policy refresh

Relay failure does not let Relay read daemon transport payloads. Existing
direct transports may continue until their own daemon/app path closes or
authorization revocation is delivered through another mechanism.
