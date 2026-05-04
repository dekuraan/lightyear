# lightyear_steam — Claude guidance

## Purpose
Steam Networking Sockets transport for Lightyear via `aeronet_steam` (which wraps `steamworks` / `steamworks::networking_sockets`). Supports both dedicated-server-by-IP and Steam P2P (by `SteamId`). Requires a running Steam client and a valid app id.

## Key public API
- `SteamAppExt::add_steam_resources(app_id: u32)` (`src/lib.rs:50`) — initializes `steamworks::Client`, calls `init_relay_network_access()`, inserts as `SteamworksClient` resource, registers `run_callbacks` in `PreUpdate`. Must be called before lightyear plugins.
- `client::SteamClientPlugin` (`src/client.rs:18`).
- `client::SteamClientIo { target: ConnectTarget, config: SessionConfig }` (`src/client.rs:39`) — `#[require(Link)]`. `on_add` hook auto-inserts `LocalId(PeerId::Steam(local_steam_id))`.
- `server::SteamServerPlugin`, `server::SteamServerIo { target: ListenTarget, config: SessionConfig }` (`src/server.rs:53`).
- `prelude` re-exports `steamworks`, `SteamId`, `SteamworksClient`, `SessionConfig`, plus `client::ConnectTarget` and `server::ListenTarget`.
- `SteamError` is empty (`src/lib.rs:26`).

## Feature flags
- `default = ["std"]`
- `std`
- `client` — pulls `aeronet_io`, `aeronet_steam/client`, `lightyear_aeronet`, `lightyear_core`, `lightyear_link`, `lightyear_connection/client`.
- `server` — same set with `aeronet_steam/server` + `lightyear_connection/server`.

## Platform gates
- `pub mod server` is `#[cfg(all(feature = "server", not(target_family = "wasm")))]` — no Steam server on WASM.
- Crate is `#![no_std]` with `extern crate alloc` and conditional `extern crate std`.

## Notable internals
- Steam acts as both Link and Connection (`Connected` is added on `Add<Linked>`, `Disconnect` triggers `Unlink`) — see `client.rs:98` and `:113`. This is unusual vs. UDP/WT, where netcode owns connection state separately.
- `ConnectTarget::Addr` sets `RemoteId(PeerId::Server)`; `ConnectTarget::Peer { steam_id }` sets `RemoteId(PeerId::Steam(_))`. Comment at `client.rs:74` notes `RemoteId` for `Addr` connections is a placeholder ("we need a RemoteId here. Maybe SteamP2PAddr?").
- Server uses `SkipNetcode` from `lightyear_connection::client_of` — Steam handshake replaces netcode entirely.

## Used by wam
Not used. wam's workspace feature list does not include any steam flag; the project ships UDP + WT only.
