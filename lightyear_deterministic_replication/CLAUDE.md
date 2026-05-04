# lightyear_deterministic_replication — Claude guidance

## Purpose
Lockstep / deterministic replication variant: instead of sending component state, all peers run the same simulation off the same inputs and only exchange checksums to detect desync. Sits on top of `lightyear_prediction` (rollback) + `lightyear_inputs` (input replication).

## Key public API
- `Deterministic` marker component (`src/lib.rs:32`). `DeterministicReplicationPlugin` (`src/plugin.rs:5`) registers it as a required component of `lightyear_prediction::rollback::DeterministicPredicted` — i.e. anything flagged for deterministic prediction is automatically deterministic.
- `ChecksumSendPlugin` (`src/checksum.rs:43`) — client-side plugin. Iterates entities with `Deterministic` + hashable components inside a `ChecksumWorld` system param (`src/archetypes.rs`), XORs per-entity hashes (order-independent — see `lib.rs:32` doc / `checksum.rs:6`), packages the result as a `ChecksumMessage` and sends to server.
- `ChecksumReceivePlugin` (server side) — receives `ChecksumMessage`, stores per-tick into `ChecksumHistory` (`src/checksum.rs:35`), compares against the server's locally-computed checksum to detect desync.
- `ChecksumHistory { history: BTreeMap<Tick, u64> }` (`:35`).
- `ChecksumMessage` — registered as a regular `lightyear_messages::Message`, sent on `InputChannel` (clients piggyback on input traffic).

## Feature flags
- `std` (default) — propagates `std` to inputs/messages/prediction.

## Notable internals
- Order-independent checksum is XOR-of-hashes. Cheap, but it can hide swap-bugs (A↔B); fine for desync detection, useless for ordering.
- Plugin requires `lightyear_inputs/client`, `lightyear_prediction/deterministic`, `lightyear_replication/deterministic`. Checksum send only runs on `Single<.., (With<Client>, With<IsSynced<InputTimeline>>)>` — clients that have synced their input timeline.
- Frequency is tied to `LastConfirmedInput` — checksums are emitted as the input timeline advances, not every frame.
- `messages.rs` is currently a stub (commented-out `ConnectionEvent`). Don't add anything here without checking; the real protocol additions go through `register_message_to_bytes` in `ChecksumSendPlugin::build`.
- This is the *replication* half. Determinism also requires: deterministic worldgen, deterministic FixedUpdate ordering, and avoiding non-deterministic floats (transcendentals across CPUs). Out of scope for this crate.
