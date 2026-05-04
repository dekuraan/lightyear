# lightyear_inputs_leafwing — Claude guidance

## Purpose
Adapter that lets `leafwing-input-manager` `Actionlike` enums flow over Lightyear's input pipeline: pressed/released states + axis/dual-axis values are tick-buffered and replicated using leafwing's existing `ActionState<A>` component (no wam-side wrapper needed).

## Key public API
- `prelude::InputPlugin<A>` (`src/plugin.rs:18`) — `A: LeafwingUserAction` (i.e. a leafwing `Actionlike` enum). Adds `leafwing_input_manager::InputManagerPlugin::<A>` if the bevy `InputPlugin` is present (client side), else server-only. Inserts `lightyear_inputs::client::ClientInputPlugin::<LeafwingSequence<A>>` / matching server plugin.
- `prelude::LeafwingBuffer`, `prelude::LeafwingSnapshot` (`src/input_message.rs`) — the snapshot type and per-tick buffer; serialised with leafwing's `ActionDiff` style deltas (`src/action_diff.rs`).
- `action_state::LeafwingUserAction` (`src/action_state.rs:9`) — supertrait blanket-impl'd for any `Actionlike + Serialize + DeserializeOwned + Clone + PartialEq + Copy + Debug + GetTypeRegistration + 'static`.
- `action_state::ActionStateWrapper<A>` (`src/action_state.rs:43`) — `QueryData` newtype around `&mut leafwing_input_manager::ActionState<A>` to bypass the orphan rule when implementing `ActionStateQueryData`.

## Feature flags
`std` (default), `client`, `server`.

## Notable internals
- Leafwing version pin is **`0.20`** (`Cargo.toml:21`, resolved against workspace `lightyear/Cargo.toml`). wam pins the same `leafwing-input-manager = "0.20"` (workspace `Cargo.toml`), so versions match exactly.
- Configures `InputSystems::RestoreInputs` to run **before** `InputManagerSystem::Tick` in `FixedPreUpdate` (see PR 820 reference in `plugin.rs:54`) so rollback-restored states aren't immediately overwritten by leafwing's per-tick refresh.
- Action serialization uses diffs (`action_diff.rs`) rather than full state, reducing bandwidth for sparse press/release events.
- Server-only mode adds `InputManagerPlugin::<A>::server()` in `finish` if the standard plugin wasn't already added.

## Status in wam
**Not wired.** wam uses leafwing 0.20 + `PlayerAction` (`crates/wam-protocol/src/lib.rs:1609`) for *local* input only; `lightyear_inputs_leafwing` is not pulled in (Lightyear feature `input_leafwing` is not enabled in workspace `Cargo.toml`). If wam ever wants replicated leafwing input, the `PlayerAction` enum already satisfies `Actionlike` and would just need to gain `Serialize + Deserialize + Reflect + MapEntities` for `LeafwingUserAction`.
