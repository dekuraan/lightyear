# lightyear_transport — Claude guidance

## Purpose
Packet/message layer on top of `Link`. Adds channels (with reliability/ordering modes), fragmentation, message IDs/acks, send-priority bandwidth shaping, and a `PacketHeader` carrying tick + acks. `Link` moves bytes; `Transport` moves typed messages.

## Key public API
- `Transport` component (`channel/builder.rs:59`) — `#[require(Link)]`. Holds per-channel `senders` and `receivers` (`HashMap<ChannelId, _>`). One per linked entity.
- `Channel` trait (`channel/mod.rs:17`) — blanket-impl for any `Send + Sync + 'static` type, so user channels are usually empty marker structs.
- `ChannelSettings { mode, send_frequency, priority }` + `ChannelMode` enum (`channel/builder.rs:344`): `UnorderedUnreliable`, `UnorderedUnreliableWithAcks`, `SequencedUnreliable`, `UnorderedReliable(ReliableSettings)`, `SequencedReliable(_)`, `OrderedReliable(_)`. `ChannelMode::is_reliable()` / `is_with_ack()` for branching.
- `AppChannelExt` (`channel/registry.rs`) — `app.add_channel::<C>(settings)`. Registers channels into `ChannelRegistry`.
- `PriorityManager` / `PriorityConfig` (`packet/priority_manager.rs`) — token-bucket bandwidth limiter using `governor`.
- `TransportPlugin` + `TransportSystems::{Receive, Send}` (`plugin.rs:36`). Receive runs in PreUpdate after `LinkSystems::Receive`; send runs in PostUpdate before `LinkSystems::Send` (chain configured by the connection plugin, e.g. `RawConnectionPlugin`).
- `PacketReceived { entity, remote_tick }` event triggered per inbound packet (`plugin.rs:48`).

## Feature flags
- `default = ["std"]`. Crate is `#![no_std]`.
- `client` / `server` — gate `client.rs` / `server.rs` and pull in matching `lightyear_connection` features.
- `metrics` — `dep:metrics` + `TimerGauge` instrumentation (`transport/recv` etc).
- `trace` — extra trace logging hooks.

## Notable internals
- `buffer_receive` (`plugin.rs:59`) excludes `HostClient` (in-process loopback) from packet processing.
- Uses `crossbeam-channel` internally between channels and `enum_dispatch` to monomorphise sender/receiver enums (no dyn-trait fan-out).
- `RecvPayload` / `SendPayload` aliases come from `lightyear_link`.
