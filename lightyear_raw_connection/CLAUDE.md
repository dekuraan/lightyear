# lightyear_raw_connection — Claude guidance

## Purpose
The "no-auth" connection layer. Lightyear normally splits IO (Link) from connection identity (LocalId/RemoteId, handshake). This crate collapses the two: when the underlying Link becomes `Linked`, the entity is also immediately `Connected`. PeerId is just the SocketAddr from the Link. Used when the upper IO already provides authenticity (TLS-over-WebTransport with a pinned cert, trusted LAN UDP, etc.).

**This is the path wam uses.** wam's `Cargo.toml` enables `lightyear`'s `raw_connection` feature; `crates/wam-client/src/net.rs:267,292` spawns `RawClient` and `crates/wam-server/src/lib.rs:2514,2520` spawns `RawServer` for both WebTransport and UDP servers — bypassing netcode entirely.

## Key public API
- `RawClient` marker component (`src/client.rs:23`) — `#[require(Link, lightyear_connection::client::Client)]`, `#[require(Disconnected)]`. Spawn it together with an IO component (e.g. `WebTransportClientIo`, `UdpIo`, or `CrossbeamIo`).
- `RawServer` marker component (`src/server.rs:28`) — `#[require(Server)]`. Same idea on the server side.
- `RawConnectionPlugin` exists in **both** `client.rs` and `server.rs` as separate types (despite identical names; they're in different modules). The full prelude:
  - `prelude::client::RawClient` (gated by `client` feature)
  - `prelude::server::RawServer` (gated by `server` feature)

## How "no auth" works
Client side (`src/client.rs:27-40`):
1. IO plugin (e.g. aeronet) inserts `Linked`.
2. `RawConnectionPlugin::on_linked` observer fires, reads `LocalAddr`, and inserts `(Connected, LocalId(PeerId::Raw(local_addr)), RemoteId(PeerId::Server))`.
3. No challenge, no token, no encryption-handshake — connection identity is just the socket pair. wam relies on WebTransport's TLS or trusted LAN UDP for actual security.

Server side (`src/server.rs:32-61`):
1. `Linked` on the `RawServer` entity → `Started`.
2. `Linked` on a `LinkOf` child (a freshly-accepted client) → inserts `(Connected, LocalId(PeerId::Server), RemoteId(PeerId::Raw(peer_addr)), ClientOf)`.
3. `Stop` → triggers `Unlink`, marks `Stopping`, walks every `LinkOf` and inserts `Disconnecting` so the disconnect propagates through `lightyear_connection` before despawn.

## Feature flags
- `default = []`. Either `client` or `server` (or both) must be enabled — they gate the entire respective module.
- `std` cascades to `lightyear_link/std` + `lightyear_transport/std`.

## Notable
- The plugin chains the system sets `LinkSystems::Receive → TransportSystems::Receive` in PreUpdate and `TransportSystems::Send → LinkSystems::Send` in PostUpdate (`client.rs:64-70`, `server.rs:107-114`). This is the canonical receive/send ordering the rest of Lightyear assumes.
- A `PeerId::Raw(SocketAddr)` from this crate is *not* interchangeable with `PeerId::Netcode(client_id)` — server-side branches that match on `PeerId` (e.g. `server.rs:82`) explicitly reject the wrong variant.
- For wam: if you ever want token-auth, swap `RawClient`/`RawServer` for `NetcodeClient`/`NetcodeServer` from `lightyear_netcode` — they take the same IO components, the connection-layer change is local.
