# lightyear_inputs_bei — Claude guidance

## Purpose
Adapter for `bevy_enhanced_input` (BEI) — a context-driven input system that splits each player's actions across many entities (`Actions<C>` context root + child `ActionOf<C>` entities). The crate makes BEI's per-context, per-action `ActionState` component replicate over Lightyear, including auto-spawning marker components on action entities and special-casing the headless-server case.

## Key public API
- `prelude::InputPlugin<C>` (`src/plugin.rs:31`) — generic over the *context* component `C` (a `Component<Mutability: GetWriteFns<C>> + PartialEq + Clone + Debug + Serialize + DeserializeOwned + TypePath`). Wires:
  - `bevy_enhanced_input::EnhancedInputPlugin` (if missing).
  - `add_input_context_to::<FixedPreUpdate, C>()` so BEI contexts run on the fixed timeline.
  - `app.register_component::<C>()` and `register_component::<ActionOf<C>>().add_component_map_entities()` for replication.
  - Client: observers `propagate_input_marker`, `add_input_marker_from_parent`, `add_input_marker_from_binding` + `ClientInputPlugin::<BEIStateSequence<C>>`.
  - Server: `ServerInputPlugin::<BEIStateSequence<C>>`. If headless (no `client` feature), disables `EnhancedInputSystems::Update`/`Prepare`. If both features, gates Update on `not(is_headless_server)`.
- `prelude::InputMarker<C>` (`src/marker.rs:12`) — marks an entity as actively producing physical inputs for context `C`. Auto-propagated from the context entity to action entities.
- `prelude::BEIBuffer` (`src/input_message.rs`), `prelude::InputRegistryExt` (`src/setup.rs`) — extension trait that registers BEI contexts on the App.
- `BEIStateSequence<C>` — internal `ActionStateSequence` impl that serialises BEI's per-tick action component changes.

## Feature flags
`std` (default), `client`, `server`.

## Notable internals
- BEI represents action state across multiple components (one per `ActionOf<C>` action entity), so `ActionStateQueryData` here is more involved than the leafwing/native cases.
- `register_required_components::<ActionOf<C>, ActionState>` ensures the buffer is automatically inserted when an action entity is spawned client-side.
- Rebroadcast path uses observers `add_action_of_host_server_rebroadcast` and `on_rebroadcast_action_received` to synthesise action entities on receiving clients.
- `EnhancedInputSystems::Apply` is sequenced *after* `BufferClientInputs` so rollback-replayed state can re-trigger BEI events.

## Status in wam
**Not wired.** wam does not depend on `bevy_enhanced_input` and Lightyear's `input_bei` feature is not enabled. Vendored only because it lives in the same workspace. Skip unless wam adopts BEI — not on the roadmap.
