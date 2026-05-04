# lightyear_avian3d — Claude guidance

## Purpose
Thin Cargo package that publishes the shared `lightyear_avian/src/lib.rs` as a 3D-only crate via `path = "../lightyear_avian/src/lib.rs"` and `required-features = ["3d"]`. No source files of its own — see `lightyear_avian` CLAUDE.md for the actual implementation.

## Key public API
Identical to `lightyear_avian` (same `lib.rs`), restricted to 3D code paths:
- `prelude::LightyearAvianPlugin`, `AvianReplicationMode` (Position / PositionButInterpolateTransform / Transform).
- `prelude::LagCompensationPlugin` + `LagCompensationSpatialQuery` + `LagCompensationHistory` (when `lag_compensation` enabled).
- 3D math types via `lightyear_avian::types_3d` re-exported as `lightyear_avian::types`.

## Feature flags
- `default = ["std", "3d", "avian3d/parry-f32"]`.
- `3d` enables `avian3d/3d`.
- `std`, `deterministic` (`seahash`), `lag_compensation` mirror the parent.

## Notable internals
- `Cargo.toml:16` — `path = "../lightyear_avian/src/lib.rs"`. Same wrapper pattern as the 2D sibling.
- Pulls `lightyear_replication` with the `avian3d` feature.

## Status in wam
**Not in use.** This is the closest off-the-shelf option *if* wam ever swapped its hand-rolled voxel physics for Avian — the 3D variant. Memory says no (`feedback_player_collider_no_aabb.md`, `feedback_fauna_no_avian.md`); the deterministic voxel sampler is intentional and runs identically on server + intermeshed client. Avian as a transitive dep through other crates is acceptable (`feedback_avian_transitive_ok.md`) but `lightyear_avian3d::LightyearAvianPlugin` is never installed.
