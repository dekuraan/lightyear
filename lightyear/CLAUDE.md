# lightyear — Claude guidance

## Purpose
Umbrella crate for the Lightyear networking library (v0.26.4). Re-exports every sub-crate and assembles two `PluginGroup`s — `ClientPlugins` and `ServerPlugins` — gated by Cargo features. This is the only Lightyear crate the wam workspace depends on directly; sub-crates are reached through `lightyear::*` re-exports.

## Key public API
- `lightyear::prelude` (`src/lib.rs:333`) — flat re-export of common types from `lightyear_connection`, `lightyear_core`, `lightyear_link`, `lightyear_messages`, `lightyear_replication`, `lightyear_serde`, `lightyear_sync`, `lightyear_transport`. Conditional re-exports for udp / webtransport / websocket / steam / netcode / prediction / interpolation.
- `lightyear::prelude::client` (`src/lib.rs:396`) — `ClientPlugins` + client variants.
- `lightyear::prelude::server` (`src/lib.rs:420`) — `ServerPlugins` + server variants.
- `ClientPlugins` (`src/client.rs:13`) — `tick_duration: Duration` (default 1/60 s). Adds `lightyear_sync::client::ClientPlugin`, `SharedPlugins`, then feature-gated: `PredictionPlugin`, `WebTransportClientPlugin`, `WebSocketClientPlugin`, `SteamClientPlugin`, `NetcodeClientPlugin`, `RawConnectionPlugin`. WASM also adds `lightyear_web::WebKeepalivePlugin` (`wake_delay: 100ms`).
- `ServerPlugins` (`src/server.rs:33`) — same shape: `lightyear_sync::server::ServerPlugin` + `ServerLinkPlugin` + `SharedPlugins`, then `HostPlugin`, `HostServerPlugin`, plus feature-gated UDP / WebTransport / WebSocket / Steam / Netcode / RawConnection servers.
- `SharedPlugins` (`src/shared.rs:7`) — installed by both groups. Adds `CorePlugins`, `TransportPlugin`, `MessagePlugin`, `ConnectionPlugin`, plus `Replication*` plugins when `replication` is on. Marked `is_unique = false` to allow ClientPlugins + ServerPlugins co-installation in host-server mode.
- `protocol` module (`src/protocol.rs`) — `ProtocolCheckPlugin`, registers a hash check across peers when replication is enabled.

## Feature flags (what each gates)
Defaults: `std`, `client`, `server`, `replication`, `prediction`, `interpolation`.

- **`std`** — adds `std` feature on every workspace crate that has one. Required for `metrics`, `udp`, `webtransport`, `websocket`, `steam`.
- **`client`** — pulls client-side bits from `lightyear_connection`, `lightyear_messages`, `lightyear_sync`, `lightyear_transport`, plus client paths in optional crates. Disabling = no `ClientPlugins`.
- **`server`** — symmetric; enables `ServerPlugins`. wam-launcher gates the dedicated-server binary on this.
- **`replication`** — pulls `lightyear_replication`. Enables `Replicate`, `ReplicationSender`, `ReplicationReceiver`, network visibility, hierarchy + authority plugins.
- **`prediction`** — pulls `lightyear_prediction` and adds `PredictionPlugin`; cascades `prediction` to `lightyear_inputs`. Also pulls `lightyear_frame_interpolation`.
- **`interpolation`** — pulls `lightyear_interpolation`; enables remote-entity smoothing.
- **`frame_interpolation`** — `lightyear_frame_interpolation` only (subset of `prediction`).
- **`deterministic`** — `lightyear_deterministic_replication` + checksum plugins on both sides.
- **IO layers** (each forwards `client` / `server` to the matching crate):
  - `udp` (native only, requires `std`) — `lightyear_udp` + `UdpPlugin`/`ServerUdpPlugin`.
  - `crossbeam` — local in-process IO; no `std` requirement.
  - `webtransport` (requires `std`) — `lightyear_webtransport`. `webtransport_self_signed` enables self-signed-cert support; `webtransport_dangerous_configuration` allows unencrypted (test only).
  - `websocket` — `lightyear_websocket` + `websocket_self_signed`.
- **Connection layers**:
  - `netcode` — `lightyear_netcode` (netcode.io standard, layered over UDP).
  - `steam` (requires `std`) — Steam networking, IO + connection both.
  - `raw_connection` — uses the IO link directly as the connection layer (used by wam for WT/UDP without netcode).
- **Inputs**:
  - `input_native` — user-defined input structs.
  - `leafwing` — `lightyear_inputs_leafwing` for `leafwing-input-manager` integration.
  - `input_bei` — `bevy_enhanced_input` integration.
- **Avian**: `avian2d` / `avian3d` (must enable upstream `f32`/`f64` features yourself).
- **Diagnostics**: `metrics` (forces `std`, pulls `lightyear_metrics`); `debug` (= `metrics` + `lightyear_ui`); `trace` (extra tracing on netcode/replication/transport).

## wam usage
wam workspace pulls `lightyear` with: `client`, `server`, `replication`, `prediction`, `interpolation`, `input_native`, `udp`, `webtransport`, `webtransport_self_signed`, `raw_connection`. Direct consumers: `wam-protocol`, `wam-client`, `wam-server`, `wam-launcher` (Cargo.toml entries). No wam crate references the sub-crates directly — everything goes through `lightyear::prelude`.
