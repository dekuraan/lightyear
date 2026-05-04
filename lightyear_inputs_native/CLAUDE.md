# lightyear_inputs_native — Claude guidance

## Purpose
The "native" / hand-rolled input adapter — the simplest implementation of `lightyear_inputs::ActionStateSequence`. The user defines an arbitrary input type `A` (any `Serialize + Deserialize + Clone + PartialEq + Reflect + MapEntities + Default`) and stores it in an `ActionState<A>` component; the plugin handles tick-buffering, replication, redundancy, rollback restore. This is the adapter wam currently enables but does not actually use.

## Key public API
- `prelude::InputPlugin<A>` (`src/plugin.rs:15`) — wraps `ClientInputPlugin::<NativeStateSequence<A>>` and `ServerInputPlugin::<NativeStateSequence<A>>` based on enabled features. Constructed with `InputConfig<A>`.
- `prelude::ActionState<A>` (`src/action_state.rs:21`) — the tuple-struct `Component` holding the active input snapshot. `Default` represents "no input pressed" — distinct from "input not received".
- `prelude::InputMarker<A>` (`src/action_state.rs:65`) — marker `Component` that flags which `ActionState<A>` entity the local player is *actively* updating (vs replicated copies of remote players' actions).
- `prelude::NativeBuffer<A>` (`src/input_message.rs:14`) — alias for `InputBuffer<ActionState<A>, A>`; the per-tick ring backing the snapshot history.
- `NativeStateSequence<A>` (`src/input_message.rs:17`) — the `ActionStateSequence` impl: `Vec<Compressed<A>>` over the redundancy window. Trivial `decay_tick` (no decay).

## Feature flags
`std` (default), `client`, `server` — pass-through to `lightyear_inputs`.

## Notable internals
- The `MapEntities` requirement on `A` lets the input itself reference replicated entities (e.g. `target_entity: Entity` actions) and have them remapped on the receiving side.
- Add input writing in `FixedPreUpdate` inside `InputSystems::WriteClientInputs` (see lib.rs doc-comment); read in `FixedUpdate` from `ActionState<A>`.
- `ActionState<A>` impls `Deref/DerefMut` to `A` for ergonomic access.

## Status in wam
**Vendored + feature-enabled, not used at runtime.** wam's workspace `Cargo.toml` enables Lightyear's `input_native` feature which compiles this crate in, but no `InputPlugin::<...>` is registered and no `ActionState`/`InputMarker` is spawned. Player input is local-only via `leafwing` `PlayerAction` (`crates/wam-protocol/src/lib.rs:1609`) attached to `PlayerBody`; mode toggles + toolbar 1-10 stay on raw `KeyCode`. If wam wires networked input later, the simplest path is to declare a `wam_protocol::NetworkedInput` enum implementing the trait list above, then `app.add_plugins(InputPlugin::<NetworkedInput>::default())` on both client and server.
