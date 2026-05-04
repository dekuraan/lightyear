# lightyear_avian — Claude guidance

## Purpose
Shared implementation for `lightyear_avian2d` and `lightyear_avian3d` — both sibling crates re-export this single `src/` directory under different feature flags. Provides the Bevy plugin that integrates Avian (parry/wgpu physics) with Lightyear's prediction/rollback/interpolation pipeline, plus optional server-side lag compensation (history of collider AABBs so client-predicted bullets can hit interpolated targets at their past positions).

## Key public API
- `prelude::LightyearAvianPlugin` (`src/plugin.rs:103`) — main plugin. Fields:
  - `replication_mode: AvianReplicationMode` — `Position` (default), `PositionButInterpolateTransform`, `Transform`. Determines whether rollback history / frame-interp operates on `Position`/`Rotation` or `Transform`. See `src/plugin.rs:78`.
  - `update_syncs_manually: bool` — disables the auto-installed `transform_to_position` / `position_to_transform` systems for advanced users.
  - `rollback_resources: bool` — also rolls back `ContactGraph`, `ConstraintGraph`, `CollidingEntities`. Required for deterministic replication.
  - `rollback_islands: bool` — rolls back `PhysicsIslands`, `BodyIslandNode`, `Sleeping`. Only enable if Avian's island plugin is on.
- **Lag compensation** (gated by `lag_compensation` feature, `src/lag_compensation/`):
  - `LagCompensationPlugin` (`history.rs:23`), `LagCompensationConfig::max_collider_history_ticks` (default 35 ≈ 500ms @ 64Hz).
  - `LagCompensationHistory = HistoryBuffer<(Position, Rotation, ColliderAabb)>` — per-entity rolling buffer.
  - `LagCompensationSpatialQuery` (`query.rs:28`) — `SystemParam` for "shoot a ray at where this entity *was* `delay_ms` ago"; reads the client's `InterpolationDelay` to pick the right history slot.
  - `AabbEnvelopeHolder` — child collider whose `ColliderAabb` is the broad-phase envelope spanning the last N ticks.
- `correction_2d`/`correction_3d` (private) — visual correction systems for the `PositionButInterpolateTransform` mode.
- `types_2d`/`types_3d` — math aliases.

## Feature flags
`2d`, `3d` (mutually exclusive at compile time inside this directory; the wrapper crates set exactly one), `std`, `lag_compensation`, `deterministic` (pulls `seahash` for state hashing).

## Notable internals
- Heavy doc-comment at `plugin.rs:1-31` describes Avian replication footguns: don't add `RigidBody` to interpolated entities (forces `Transform::default()` until first update), disable `PhysicsTransformPlugin` and `PhysicsInterpolationPlugin` on the Avian side, etc.
- `add_transform` (`plugin.rs:495`) computes the correct local Transform for newly-replicated entities that received `Position`/`Rotation` but no `Transform` yet, accounting for parent global transforms.
- `update_child_collider_position` (`plugin.rs:589`) — Avian only does this in `PhysicsSystems::First`; Lightyear re-runs it after physics so replication sees up-to-date child positions.
- The plugin is **not** auto-added by `ClientPlugins`/`ServerPlugins` — users opt in.

## Status in wam
**Not in use.** wam mobs and players use hand-rolled voxel physics that sample `ClientVoxelStore` directly (`crates/wam-fauna/CLAUDE.md`, `crates/wam-client/CLAUDE.md`). Avian is fine as a transitive dep but no `RigidBody`/`Position` is replicated. Memo: lag compensation here is the closest off-the-shelf reference if wam ever needs hit-registration-with-rewind for ranged combat (`project_combat_and_classes_vision.md`), but the architecture (history-of-AABB) would have to be re-implemented on top of voxel sampling because nothing here is voxel-aware.
