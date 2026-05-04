# lightyear_websocket — Claude guidance

## Purpose
WebSocket (`ws://` / `wss://`) transport for Lightyear. Thin wrapper over `aeronet_websocket`'s plugins; this crate just defines marker components and `LinkStart` observers that delegate to aeronet.

## Key public API
- `client::WebSocketClientPlugin` (`src/client.rs:10`).
- `client::WebSocketClientIo { config: ClientConfig, target: WebSocketTarget }` (`src/client.rs:43`) — `#[require(Link)]`.
- `client::WebSocketTarget::{Url(String), Addr(WebSocketScheme)}` (`src/client.rs:49`) — either a full URL or scheme + sibling `PeerAddr`.
- `client::WebSocketScheme::{Plain, Secure}` (`src/client.rs:13`); `Secure` (wss) is the default.
- `server::WebSocketServerPlugin`, `server::WebSocketServerIo { config: ServerConfig }` (`src/server.rs`) — needs sibling `LocalAddr`.
- `WebSocketError::{Certificate, PeerAddrMissing, LocalAddrMissing}` (`src/lib.rs:13`).
- `prelude::{client::*, server::*}` re-exports plus pass-through `aeronet_websocket::*`.

## Feature flags
- `default = ["self-signed"]`
- `client` → `aeronet_websocket/client`
- `server` → `aeronet_websocket/server` + `bevy_reflect/std`
- `self-signed` → enables `wtransport/self-signed` (note the doc string mentions `wtransport`, but this is a WebSocket crate — feature passes through to aeronet for cert generation).

## Platform gates
- `pub mod server` is `#[cfg(all(feature = "server", not(target_family = "wasm")))]` — no WS server on WASM.
- Client compiles on both native and WASM (browsers can connect to WS servers natively).

## Notable internals
- `LinkStart` observers spawn a child entity with `AeronetLinkOf(parent)` and call `WebSocketClient::connect` / `WebSocketServer::open`, which apply themselves as commands on the spawned entity.
- All actual IO/cert/handshake logic lives in `aeronet_websocket`, not here.

## Used by wam
Not used. wam's workspace `lightyear` features list does not include any websocket flag; native uses UDP, WASM uses WebTransport.
