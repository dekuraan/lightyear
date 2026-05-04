# lightyear_webtransport — Claude guidance

## Purpose
WebTransport (HTTP/3 over QUIC) transport for Lightyear. Works natively (`wtransport`/`quinn` backend) and on WASM (browser `WebTransport` API via `xwt-web`). This is the canonical WASM-friendly transport.

## Key public API
- `client::WebTransportClientPlugin` (`src/client.rs:12`).
- `client::WebTransportClientIo { certificate_digest: String }` (`src/client.rs:32`) — `#[require(Link)]`. Hex SHA-256 digest of the server cert (no colons). Empty string means "no validation" (native: requires `dangerous-configuration` feature; WASM: empty hash list).
- `server::WebTransportServerPlugin` (`src/server.rs:19`).
- `server::WebTransportServerIo { certificate: Identity }` (`src/server.rs:48`) — `#[require(Server)]`. Needs sibling `LocalAddr`.
- `WebTransportError::{Certificate, PeerAddrMissing, LocalAddrMissing}` (`src/lib.rs:13`).
- `prelude` re-exports `wtransport::Identity` (native only) plus `client::WebTransportClientIo` and `server::WebTransportServerIo`.

## Feature flags
- `default = ["self-signed"]`
- `client` → `aeronet_webtransport/client`
- `server` → `aeronet_webtransport/server` + `bevy_reflect/std`
- `self-signed` → `aeronet_webtransport/self-signed` → `wtransport/self-signed` (server cert generation helper).
- `dangerous-configuration` → allows clients to skip cert validation entirely. Native-only branch in `client_config` (`src/client.rs:107`); not for production.

## Platform gates
- `pub mod server` is `#[cfg(all(feature = "server", not(target_family = "wasm")))]` — no WT server in browser.
- Client: dual-target. WASM branch uses `xwt_web::CertificateHash` + `HashAlgorithm::Sha256`; native branch uses `wtransport::tls::Sha256Digest`.
- `prelude::Identity` re-export is gated `#[cfg(not(target_family = "wasm"))]`.

## Notable internals
- Native client config (`src/client.rs:97`) hard-codes `IpBindConfig::InAddrAnyV4` due to a Linux IPv6 bind issue, with `keep_alive_interval=1s`, `max_idle_timeout=5s`. Same timeouts on the server side (`src/server.rs:67`).
- `from_hex` / `from_hex_digit` (`src/client.rs:132`) parse the cert digest manually, no `hex` crate dep.
- Server `on_session_request` blanket-accepts every connection (`src/server.rs:82`) — wam reuses this as-is. To gate connections, override with another observer.
- Server `on_connection` (`src/server.rs:88`) spawns the per-client `LinkOf` entity wired to the parent `AeronetLinkOf`.

## Used by wam
- `wam-server/src/lib.rs:2440` imports `lightyear::webtransport::prelude::Identity`; `:2514` spawns `WebTransportServerIo { certificate }` on the WT port (default `:5000`).
- Workspace `Cargo.toml` enables `webtransport` + `webtransport_self_signed` umbrella features.
- WASM client path uses this; native client falls back to UDP at `WT port + 1` (memory note "auto-offsets WT:5000 → UDP:5001").
- Self-signed cert + SHA-256 digest is the dev-cert mechanism; passed via launcher CLI / WASM `?addr=` query.
