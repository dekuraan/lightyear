# lightyear_link — Claude guidance

## Purpose
Transport-agnostic link abstraction. A `Link` is just two `VecDeque<Bytes>` buffers (recv + send) plus a state-machine and stats. The actual byte movement is done by an IO plugin (aeronet UDP/WT, crossbeam, etc.) reading/writing those buffers.

## Key public API (`src/lib.rs`)
- `Link` component (`lib.rs:65`) — `recv: LinkReceiver`, `send: LinkSender`, `state: LinkState`, `stats: LinkStats`. IO plugins push into `link.recv` and drain `link.send`.
- `LinkState` enum + state-marker components `Linked` / `Linking` / `Unlinked` (`lib.rs:236-288`). All three use `on_insert` hooks that mutate `Link::state` and remove the other two markers — so adding `Linked` automatically clears `Linking`/`Unlinked`.
- Events: `LinkStart { entity }` (`lib.rs:220`) starts the IO connect; `Unlink { entity, reason }` (`lib.rs:229`) requests teardown.
- `LinkSystems::Receive` (PreUpdate) / `LinkSystems::Send` (PostUpdate) — system sets IO plugins hook into. Sub-sets `LinkReceiveSystems::BufferToLink` then `ApplyConditioner` (`lib.rs:206`).
- `RecvLinkConditioner` / `LinkConditioner` (`src/conditioner.rs`) — latency/jitter/loss simulation; `good_condition` / `average_condition` / `poor_condition` presets.
- `LinkPlugin` (`lib.rs:298`) — registers the system sets and the `unlink` observer.
- Server side (`src/server.rs`): `Server` component + `LinkOf { server: Entity }` relationship. Server also auto-inserts `Unlinked` if no state marker is present at spawn (`server.rs:26`). Re-exported as `prelude::server::{LinkOf, Server}`.

## Feature flags
- `default = ["std"]` — `std` toggles `extern crate std`. Crate is `#![no_std]` otherwise.
- `test_utils` — exposes `LinkSender::iter` / `LinkReceiver::iter` for assertions.

## Notable internals
- `Link` derives `Default` and `new()` only takes a recv conditioner; the send side never simulates network conditions (only the receive path does, by design).
- `LinkSender` is `pub struct LinkSender(VecDeque<SendPayload>)` — push/pop/drain/len, no other methods.
- The state markers' on-insert hooks are why you should always insert `Linked`/`Linking`/`Unlinked` rather than mutating `Link::state` directly.
