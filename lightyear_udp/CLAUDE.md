# lightyear_udp — Claude guidance

## Purpose
Native UDP transport for Lightyear. Wraps `std::net::UdpSocket` (non-blocking) and bridges it to the `Link` component's send/recv buffers. Real-time-game friendly, no in-order/reliability guarantees (those live higher in the stack).

## Key public API
- `UdpIo` (`src/lib.rs:54`) — client-side socket component, `#[require(Link)]`. Needs a sibling `LocalAddr` to bind on `LinkStart`.
- `UdpPlugin` (`src/lib.rs:84`) — observers for `LinkStart` / `Unlink`, parallel `send`/`receive` systems in `PreUpdate` under `LinkSystems::Send` and `LinkReceiveSystems::BufferToLink`.
- `server::ServerUdpIo` (`src/server.rs:33`) — server-side socket; tracks `connected_addresses: HashMap<SocketAddr, LinkOfStatus>` to spawn one `LinkOf` child entity per remote.
- `server::ServerUdpPlugin`, `server::UdpLinkOfIO`.
- `prelude` re-exports under `prelude::` and `prelude::server::`.
- `UdpError::LocalAddrMissing` (`src/lib.rs:69`).
- `MTU = 1472` (`src/lib.rs:43`, also redefined in `server.rs:26`).

## Feature flags
- `default = []`
- `server` — enables `src/server.rs`, pulls `bevy_platform` for its `HashMap`.
- `metrics` — emits `udp/send` gauge per send.

## Platform gates
Implicit: this crate uses `std::net::UdpSocket` and is unusable on `wasm32-unknown-unknown`. There are no `cfg(target_family = "wasm")` guards in source — the workspace just doesn't compile it for WASM.

## Notable internals
- `receive` uses `unsafe` to write directly into `BytesMut` uninitialised tail (one MTU per call), loops until `WouldBlock`. Comment notes COW/realloc risk if frame backlog exceeds buffer; not yet addressed.
- `Send` runs `par_iter_mut` over `(Link, UdpIo, PeerAddr)`; one `send_to` per drained payload, errors logged and swallowed.
- Server uses a two-phase `LinkOfStatus::{Spawning, Spawned}` to avoid race conditions when multiple connection packets arrive in the same frame from a new address.
- No async runtime, no `tokio`. Pure blocking-socket-with-`set_nonblocking(true)`.

## Used by wam
- `wam-server/src/lib.rs:2520` spawns `ServerUdpIo` on `WT port + 1` (auto-offset companion to the WebTransport listener).
- `wam-client/src/net.rs:268` spawns `UdpIo::default()` for native client connections; native client always uses UDP, never WebTransport (despite the launcher CLI taking a WT addr — see `net.rs:264`).
- Pulled in via the umbrella `lightyear` crate `udp` feature in workspace `Cargo.toml`.
