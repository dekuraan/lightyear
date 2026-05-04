# lightyear_replication — Claude guidance

## Purpose
Entity + component replication: tracking changes, serialising actions/updates, applying them on the receiver, plus authority transfer, hierarchy replication, prespawned entities, delta compression, and visibility (rooms / immediate). Built on `lightyear_messages` + `lightyear_transport`. `no_std` core, `std` is default.

## Key public API
Re-exports concentrated in `src/lib.rs:50` (prelude). Plugins:
- `ReplicationSendPlugin` (`src/send/plugin.rs:38`) — buffers component changes, runs `handle_acks`, sends `ActionsMessage`/`UpdatesMessage`. Companion `ReplicationBufferSystems`.
- `ReplicationReceivePlugin` (`src/receive.rs:49`) — applies received messages, manages `ReplicationReceiver` (`:244`).
- `SharedPlugin` (`src/plugin.rs:30`, internal) — registers the three reserved channels: `MetadataChannel` (reliable, prio 10), `UpdatesChannel` (`UnorderedUnreliableWithAcks`, prio 1), `ActionsChannel` (reliable, prio 10) — all `Bidirectional`.
- `AuthorityPlugin`, `RoomPlugin` (`src/visibility/room.rs:77`), `NetworkVisibilityPlugin` (`src/visibility/immediate.rs:79`), `HierarchySendPlugin`.

`ReplicationSystems` (`src/plugin.rs:20`): `Receive` (PreUpdate, after `MessageSystems::Receive`), `Send` (PostUpdate, before `MessageSystems::Send`).

Core components/resources:
- `Replicate` (`src/send/components.rs:764`) — opt into replication. Constructors: `Replicate::to_server()` (client feature), `Replicate::to_clients(NetworkTarget)` (server feature), `Replicate::manual(Vec<Entity>)`. Requires `Replicating`, `ReplicationGroup`, `ReplicationState`. `on_insert` hook seeds `AuthorityBroker` with `PeerId::Server`.
- `ReplicationGroup` (`:182`) — guarantees grouped entities ship in the same packet/tick. `DEFAULT_GROUP` (id 0) and `PREDICTION_GROUP` (id 1) (`:41-43`).
- `ReplicationMode` (`:573`) — `SingleClient` / `SingleServer(NetworkTarget)` / `Manual(Vec<Entity>)` / `Disabled`.
- `ReplicationTarget<T>` (`:369`) — generic target wrapper.
- `Replicated { receiver }` and `InitialReplicated { receiver }` (`src/components.rs:41,53`) — markers on receiver-side spawned entities. `Persistent` (`:67`) opts the entity out of disconnect-cleanup despawning.
- `ConfirmedTick { tick }` (`:72`) — last tick a remote update was applied; auto-inserted on `Predicted`/`Interpolated` add (`:81`).
- `ReplicationSender` (`src/send/sender.rs:57`), `ReplicationReceiver` (`src/receive.rs:244`) — per-link components doing the actual buffering. `SendUpdatesMode` controls full-state vs change-detection sends.
- `ComponentRegistry` (`src/registry/registry.rs:124`) + `AppComponentExt` (`:401`): `app.register_component::<C>()`, `register_component_custom_serde`, `non_networked_component`. Returned `ComponentRegistration` has `with_replication_config`, `with_delta_compression`, `add_prediction`, `add_interpolation`, `add_correction_fn`, `add_map_entities`, `add_linear_interpolation_fn`, etc.
- `ComponentReplicationConfig` (`src/send/components.rs:46`): `replicate_once`, `disable`, `delta_compression`. Per-entity overrides via `ComponentReplicationOverrides<C>` (`src/components.rs:17`) supporting `global_override` and `override_for_sender`.

Authority (`src/authority.rs`):
- `HasAuthority` marker — present on the peer that simulates the entity. Adding `Replicate` claims authority unless the entity arrived via replication.
- Triggers `GiveAuthority { entity, peer }`, `RequestAuthority { entity }`, `AuthorityTransfer` (controls whether a peer is allowed to give it away). `AuthorityBroker` resource lives on the server entity and tracks owners.

Delta compression (`src/delta.rs`):
- `Diffable<Delta=Self>` trait (`:43`): `base_value`, `diff(&self, new) -> Delta`, `apply_diff(&mut self, &Delta)`. Implement on the component (or with a separate `Delta` type), then call `.add_delta_compression()` in registration.
- `DeltaMessage<M> { delta_type, delta }` (`:30`); `DeltaType::FromBase` for first message, `Normal { previous_tick }` for incremental.
- `DeltaComponentHistory<C>` (`:59`) — `BTreeMap<Tick, C>` per-tick history kept on receiver to reconstruct state at arbitrary acked ticks.
- `DeltaManager` resource on sender (server or single-client). The plugin (`src/send/plugin.rs:55-75`) looks up `DeltaManager` either directly on the sender or via `LinkOf::server`.

Visibility:
- Immediate model (`visibility/immediate.rs`): `NetworkVisibility` marker + per-(entity,client) `VisibilityState`. Manual gain/lose visibility events.
- Room model (`visibility/room.rs`): `Room` (`:62`) is an entity with members; `RoomEvent` adds/removes clients or entities; `RoomTarget` (`:266`) and `RoomSystems` (`:251`) drive scheduling.

Other modules:
- `hierarchy.rs` — `ReplicateLike`, `ReplicateLikeChildren`, `DisableReplicateHierarchy`.
- `prespawn.rs` — `PreSpawned` marker for client-predicted spawns that the server later "adopts".
- `control.rs` — `Controlled`, `ControlledBy`, `ControlledByRemote`, `Lifetime`.
- `host.rs` — `SpawnedOnHostServer` etc. for host-server mode.
- `message.rs` — `ActionsMessage`, `UpdatesMessage`, `SenderMetadata`, `SpawnAction`, channel marker types.

## Feature flags
`std` (default), `client`, `server` (pulls `lightyear_connection/server`), `prediction`, `interpolation`, `deterministic`, `metrics`, `trace`, `test_utils`, `avian2d`, `avian3d`. The `prediction`/`interpolation` flags only gate the `add_prediction`/`add_interpolation` registration extensions; the prediction systems live in `lightyear_prediction`.

## Notable internals
- Three reserved channel marker types (`MetadataChannel`, `UpdatesChannel`, `ActionsChannel`) — don't shadow them in user code.
- `Replicate::on_insert` claims authority *only* if no other peer already owns the entity in `AuthorityBroker.owners`.
- The receive path stores the source entity in `Replicated.receiver` (and `InitialReplicated.receiver` once, never changed) — that's how the receiver finds the right `RemoteEntityMap`.
- "Updates" use unreliable-with-acks; "Actions" (spawns/inserts/removes) use reliable. Acks drive the delta-compression baseline tick.
- `DeltaManager` can live either on a single-client sender entity or on the server root; the send plugin checks both spots (`plugin.rs:62`).
- The big `recv` and `send` systems are built with `ParamBuilder + QueryParamBuilder` after `ComponentRegistry::finish()`, mirroring `lightyear_messages::plugin` — registrations must complete before `app.finish()`.
- Channels declared `Bidirectional` — both client→server and server→client replication share the same wire format.
