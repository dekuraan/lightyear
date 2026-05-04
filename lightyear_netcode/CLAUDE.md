# lightyear_netcode — Claude guidance

## Purpose
Pure-Rust implementation of the netcode.io protocol (Glenn Fiedler / mas-bandwidth) sitting between an unreliable byte transport (UDP, WebTransport datagrams, etc.) and Lightyear's connection layer. Provides token-based authentication, ChaCha20-Poly1305 encryption, replay protection, and keep-alives. Defines a stronger `Connected` than `RawClient` — Linked is necessary but not sufficient; the netcode handshake must complete first.

## Key public API
- `NetcodeClient` component (`client_plugin.rs:32`) — `#[require(Link, lightyear_connection::client::Client)]`, `#[require(Disconnected)]`. Wraps `crate::client::Client<()>`.
- `NetcodeServer` + `TokenUserData` (`server_plugin.rs:20-25`).
- `NetcodeConfig` — separate types for client and server, both in their respective `prelude::client` / `prelude::server` modules.
- `Authentication` (`auth.rs:24`) — `Token(ConnectToken)`, `Manual { server_addr, client_id, private_key, protocol_id }`, or `None` (waiting for token from backend).
- `ConnectToken` / `ConnectTokenBuilder` (`token.rs`) — 2048-byte signed/encrypted blob the web backend hands the client.
- `Key` + `generate_key()` / `try_generate_key()` (`crypto.rs`) — 32-byte server private key.
- Plugins: `NetcodeClientPlugin` (`client_plugin.rs`), `NetcodeServerPlugin` (`server_plugin.rs`).

## Constants (`src/lib.rs:112-126`)
- `MAC_BYTES = 16`, `MAX_PKT_BUF_SIZE = 1300`, `MAX_PACKET_SIZE = 1200`.
- `CONNECTION_TIMEOUT_SEC = 15`, `PACKET_SEND_RATE_SEC = 1.0/10.0` (10 Hz keepalives).
- `PRIVATE_KEY_BYTES = 32`, `USER_DATA_BYTES = 256`, `CONNECT_TOKEN_BYTES = 2048`.
- `NETCODE_VERSION = b"NETCODE 1.02\0"` (wire-protocol version stamp).
- Default `NetcodeConfig`: `num_disconnect_packets: 10`, `keepalive_packet_send_rate: 1/10`, `client_timeout_secs: 3`, `token_expire_secs: 30` (`client_plugin.rs:66`).

## Feature flags
- `default = ["std", "client", "server"]`.
- `client` / `server` — gate the respective plugin module + feature in `lightyear_connection`. The `client` feature pulls in `aeronet_io` (peer addr handling); `server` pulls in `rand` (token nonces).
- `std`, `trace`.

## Notable internals
- `chacha20poly1305` is the AEAD cipher (no dep on libsodium / sodiumoxide).
- Inner `crate::client::Client<()>` / `crate::server::Server` are the protocol state machines; the `_plugin` modules wrap them in Bevy systems and bridge to `Connect`/`Disconnect` / `Connected`/`Disconnected` from `lightyear_connection`.
- `replay.rs` implements the netcode sliding-window replay protection.
- `target_family = "wasm"` swaps in `web-time::Instant` (Cargo.toml line 62).
- Note: wam **does not** use this crate (uses `lightyear_raw_connection` instead). The auth/encrypted path is dormant in this workspace.
