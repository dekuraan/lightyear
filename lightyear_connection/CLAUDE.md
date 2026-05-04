# lightyear_connection — Claude guidance

## Purpose
Connection lifecycle on top of `lightyear_link`/`lightyear_transport`. Defines the client/server role split, connect/disconnect state machines, and `NetworkTarget` for "send to which peers". Pure ECS state types — most logic lives in observer hooks. `no_std`.

## Key public API (file:line)
- `ConnectionPlugin` (`src/lib.rs:87`) — empty top-level plugin; real work is in `client::ConnectionPlugin` and `server::ConnectionPlugin`.
- `ConnectionSystems` (`src/lib.rs:47`) — `Receive` (PreUpdate), `Send` (PostUpdate). `ConnectionSet` is a deprecated alias.
- Client side (`src/client.rs`):
  - `Client { state: ClientState }` (`:39`) — marker on the connection entity.
  - `ClientState` enum (`:26`): `Connected`/`Connecting`/`Disconnecting`/`Disconnected`.
  - State-marker components `Connected`, `Connecting`, `Disconnected { reason }`, `Disconnecting` — each uses `#[component(on_add = ...)]` hooks (`:60..:142`) to mutually evict siblings, sync `Client.state`, and update `PeerMetadata`.
  - `Connect { entity }` / `Disconnect { entity }` (`:45`, `:51`) — `EntityEvent` triggers; `connect` observer (`:156`) auto-fires `LinkStart`. `disconnect_if_link_fails` (`:164`) inserts `Disconnected` when `Unlinked` is added.
  - `PeerMetadata { mapping: HashMap<PeerId, Entity> }` (`:148`) — reverse lookup populated by the `on_add` hooks.
  - `ConnectionError` (`:16`).
- Server side (`src/server.rs`):
  - `Start { entity }` / `Stop { entity }` triggers; `Starting`, `Started`, `Stopping`, `Stopped` marker components mirroring the client state machine. `Started.on_add` registers `PeerId::Server` in `PeerMetadata`.
  - `is_headless_server` helper exported via `prelude::server`.
- `ClientOf` (`src/client_of.rs:10`) — marker on a server-side `LinkOf` entity that has connected; semantically `LinkOf` + `Connected`. `SkipNetcode` (`:21`) lets non-netcode transports (Steam) bypass netcode handshake.
- `NetworkTarget = Target<PeerId>` and `EntityTarget = Target<Entity>` (`src/network_target.rs:18`). `Target` variants: `All`, `AllExceptSingle`, `AllExcept`, `Single`, `Only`, `None`. `apply_targets` walks an iterator of client entities and calls a closure.
- `NetworkDirection` in `src/direction.rs` (`ClientToServer`, `ServerToClient`, `Bidirectional`).
- `host` module — `HostClient` + `HostServer` markers for the "one client lives in the server process" mode.

## Feature flags
- `client`, `server` — gate the side-specific submodule re-exports in `prelude::client` / `prelude::server`. The marker types are always compiled.

## Notable internals
- State transitions are **observer-driven via component hooks**, not systems; adding `Connecting` evicts `Connected`/`Disconnected`/`Disconnecting`. Don't insert two state markers in one frame expecting both to stick.
- `Connected::on_add` panics if the entity has no `RemoteId` — the hook uses it to populate `PeerMetadata`.
- Connecting an entity is a two-step trigger: `Connect` → fires `LinkStart` → link-layer ack → user code inserts `Connected`.
- "Host-server" = a client entity that is also a `ClientOf` of a started server in the same `App`; `host` module wires the no-op connect path.
- The crate is `no_std` (`extern crate alloc`); peer maps come from `bevy_platform::collections::HashMap`.
