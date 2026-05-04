# lightyear_aeronet — Claude guidance

## Purpose
Bridge between the `aeronet_io` ECS session model (used by aeronet's WebTransport, WebSocket, native UDP backends) and Lightyear's `Link` component. It does not implement a transport itself — it copies addresses/state/payloads between paired entities each frame.

## Key public API (`src/lib.rs`)
- `AeronetLink` / `AeronetLinkOf` — paired one-to-one `Relationship` (`lib.rs:23-30`). The Lightyear `Link` lives on the `AeronetLink` entity; the aeronet `Session` lives on the matching `AeronetLinkOf` entity.
- `AeronetPlugin` (`lib.rs:32`) — installs observers and the per-frame copy systems:
  - `on_local_addr_added` / `on_peer_addr_added` — propagate aeronet `LocalAddr` / `PeerAddr` onto the Link entity.
  - `on_connecting` (Add `SessionEndpoint`) → insert `Linking`. `on_connected` (Add `Session`) → insert `Linked`. `on_disconnected` → insert `Unlinked` with a reason string.
  - `unlink` observer translates Lightyear `Unlink` into aeronet `Disconnect` (or `Close` if it's a server).
  - `receive` system drains `session.recv` into `link.recv` (using `recv.recv_at` Instant from aeronet); `send` drains `link.send` into `session.send` (`lib.rs:157-187`).
- `ServerAeronetPlugin` (`src/server.rs`) — same shape but for `aeronet_io::server::{ServerEndpoint, Server, Closed}`. `ServerEndpoint` → `Linking`; `Server` → `Linked`; `Closed` → `Unlinked` with `CloseReason::ByUser` / `ByError` formatted into the reason.

## Aeronet primitives bridged
- `aeronet_io::Session` / `SessionEndpoint` ↔ Lightyear `Linked` / `Linking`.
- `aeronet_io::connection::{LocalAddr, PeerAddr, Disconnect, Disconnected, DisconnectReason}` ↔ Lightyear address components + `Unlinked.reason`.
- `aeronet_io::server::{Server, ServerEndpoint, Close, Closed, CloseReason}` ↔ Lightyear server `Linked` / `Linking` / `Unlinked`.
- `aeronet_io::IoSystems::Poll` / `Flush` are sequenced around `LinkSystems::Receive` / `Send` (`lib.rs:203-204`).

## Feature flags
- `default = ["std"]`, `test_utils` (forces `Instant::now()` in receive instead of aeronet's `recv_at`).

## Notable
- `try_insert(Unlinked { ... })` is used in `on_disconnected` because the Link entity may already be despawned by the time the trigger fires (`lib.rs:130`).
