# lightyear_avian2d — Claude guidance

## Purpose
Thin Cargo package that publishes the shared `lightyear_avian/src/lib.rs` as a 2D-only crate via `path = "../lightyear_avian/src/lib.rs"` and `required-features = ["2d"]`. There are no source files here — every public item lives in `lightyear_avian` and is documented in that crate's CLAUDE.md.

## Key public API
Identical to `lightyear_avian` (same `lib.rs`), restricted to the 2D code paths:
- `prelude::LightyearAvianPlugin`, `AvianReplicationMode`.
- `prelude::LagCompensationPlugin`, `LagCompensationConfig`, `LagCompensationHistory`, `LagCompensationSpatialQuery`, `AabbEnvelopeHolder` (when `lag_compensation` is on).
- 2D types via `lightyear_avian::types_2d` re-exported as `lightyear_avian::types`.

## Feature flags
- `default = ["std", "2d", "avian2d/parry-f32"]`.
- `2d` enables `avian2d/2d`.
- `std`, `deterministic` (`seahash`), `lag_compensation` mirror the parent.

## Notable internals
- `Cargo.toml:16` — `path = "../lightyear_avian/src/lib.rs"`. Editing this crate means editing `lightyear_avian/src/`. The wrapper exists only to give downstream users a clean `lightyear_avian2d` import name and to lock in `avian2d` as the physics dep.
- Pulls `lightyear_replication` with the `avian2d` feature.

## Status in wam
**Not in use.** wam is a 3D voxel game with hand-rolled physics. Mentioned only because it's part of the vendored set; pretend it's a 2D-only redirect to `lightyear_avian` and look there for the real code.
