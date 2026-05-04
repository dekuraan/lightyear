# lightyear_inputs — Claude guidance

## Purpose
Base crate for Lightyear's input replication. Defines the generic traits, channel, message format, tick-buffer, and client/server plugins that the per-backend adapters (`*_native`, `*_leafwing`, `*_bei`) plug into. Adapters depend on this crate; you usually do not depend on it directly unless writing a new adapter.

## Key public API
- `InputChannel` (`src/lib.rs:31`) — sequenced unreliable channel marker for client→server input messages.
- `plugin::InputPlugin<S: ActionStateSequence>` (`src/plugin.rs:13`) — registers the channel + `InputMessage<S>` (bidirectional, with entity mapping). Generic over an `ActionStateSequence`; per-backend crates wrap this.
- `client::ClientInputPlugin` and `server::ServerInputPlugin` (`src/client.rs`, `src/server.rs`) — the schedules that buffer/send inputs on the client and rebroadcast on the server.
- `client::InputSystems` (`src/client.rs:87`): `WriteClientInputs`, `BufferClientInputs`, `RestoreInputs`, `ReceiveInputMessages`, … — public ordering hooks (FixedPreUpdate / RunFixedMainLoop).
- `server::ServerInputConfig` and `server::InputRebroadcaster` (re-exported via `prelude::server`).
- `config::InputConfig<A>` (`src/config.rs:9`) — `packet_redundancy: u16` (default 5), `send_interval`, `ignore_rollbacks`, `rebroadcast_inputs`, `lag_compensation` (interpolation feature).
- `input_buffer::InputBuffer<S, A>` — per-tick ring with `Compressed::{Absent, SameAsPrecedent, Input(_)}` deduping.
- `input_message::ActionStateSequence` (`src/input_message.rs:113`) — **the central trait every input backend implements**. Associated types: `Action`, `Snapshot: InputSnapshot`, `State: ActionStateQueryData`, `Marker: Component`. Defines `update_buffer`, `register_required_components`.
- `input_message::InputSnapshot` (`src/input_message.rs:50`), `ActionStateQueryData` (`src/input_message.rs:62`), `InputTarget::{Entity, PreSpawned}`, `PerTargetData`, `InputMessage<S>`.

## Feature flags
`std` (default), `client`, `server`, `metrics`, `prediction`, `interpolation`. `client`/`server` toggle the matching submodule + Lightyear-side deps.

## Notable internals
- History depth is fixed at `HISTORY_DEPTH = 20` ticks (`src/lib.rs:23`); each `InputMessage` includes the last N ticks for redundancy against packet loss.
- The channel is `UnorderedUnreliable` with `priority: f32::INFINITY` and `send_frequency: Duration::default()` — inputs ride every outbound packet.
- Bidirectional channel direction supports the rebroadcast path (server → other clients), giving spectators/predictors access to remote inputs.

## Status in wam
**Not wired.** wam pulls `lightyear` with `input_native` (workspace `Cargo.toml`), so this crate ships in the binary, but no `InputPlugin::<A>` is added and no `InputMessage` flows. Gameplay input runs on raw `KeyCode` + leafwing 0.20 `PlayerAction` (`crates/wam-protocol/src/lib.rs:1609`); no replication. If/when wam adopts Lightyear input replication, the entry points are `client::ClientInputPlugin` (via a backend adapter like `lightyear_inputs_native::InputPlugin<MyInput>`) and `server::ServerInputPlugin`.
