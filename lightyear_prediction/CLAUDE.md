# lightyear_prediction — Claude guidance

Lightyear 0.26.4 vendored at `vendor/lightyear/lightyear_prediction/`. Read this first when debugging "predicted entity snaps", rollback storms, missing rollbacks, or VisualCorrection artifacts.

## Purpose
Client-side prediction + rollback. Stores per-component history on the predicted entity, on each network update compares predicted-vs-confirmed at the confirmed tick, and if mismatched re-runs `FixedMain` from that tick forward through the `RollbackSchedule`. Optionally smooths the post-rollback snap via `VisualCorrection`.

## Key public API
- `prelude::PredictionPlugin` (`src/plugin.rs:31`).
- `prelude::Predicted` (re-export from `lightyear_core::prediction::Predicted`, `src/lib.rs:42`) — marker on the client-side predicted entity. Pairs with `Confirmed` (from `lightyear_replication`) which holds the server-authoritative state.
- `prelude::PredictionManager` (`src/manager.rs:101`) — per-link component holding `RollbackPolicy`, `CorrectionPolicy`, the `RwLock<RollbackState>`, and deterministic-despawn bookkeeping. Required-component graph pulls in `InputTimelineConfig`, `PreSpawnedReceiver`, `LastConfirmedInput`, `PredictionSyncBuffer`. On insert it stores its entity in `PredictionResource` so component hooks can locate the link without queries.
- `prelude::RollbackPolicy` (`src/manager.rs:54`) — `state: RollbackMode`, `input: RollbackMode`, `max_rollback_ticks` (default 100). Default = `Check/Check`. State takes precedence over input when both mismatch.
- `prelude::RollbackMode` (`src/manager.rs:30`) — `Always` / `Check` (default) / `Disabled`. `Always` skips the diff and saves on history storage.
- `prelude::PredictionSystems` (`src/plugin.rs:37`) — `Rollback` (PreUpdate), `EntityDespawn` + `UpdateHistory` (FixedPostUpdate), `All`.
- `prelude::RollbackSystems` (`src/rollback.rs:49`) — sub-sets `Check` → `RemoveDisable` → `Prepare` → `Rollback` → `EndRollback` (all PreUpdate, chained, gated on `is_in_rollback` from `Prepare` onward), plus `VisualCorrection` (PostUpdate, after `FrameInterpolationSystems::Interpolate`).
- `prelude::PredictionHistory<C>` (`src/predicted_history.rs:16`) — per-component `HistoryBuffer<C>` on the predicted entity; updated in `FixedPostUpdate` via `update_prediction_history`. Tick-shifted on `SyncEvent<InputTimelineConfig>`.
- `prelude::VisualCorrection<D>` (`src/correction.rs:47`) + `PreviousVisual<C>` — visual smoothing of post-rollback snap. Driven by `Diffable<D>` so the error decays over time without affecting the simulation component.
- `prelude::DeterministicPredicted` / `DisableRollback` / `DisabledDuringRollback` (`src/rollback.rs:175`) — markers to opt entities out of state-rollback while still participating in input-rollback.
- `prelude::LastConfirmedInput` (`src/manager.rs:137`) — atomic tick + flag, used by `RollbackMode::Always` (input) to know how far back to roll.
- `PredictionAppRegistrationExt` / `PredictionRegistrationExt` / `PredictionRegistry` (`src/registry.rs`) — register components for prediction (`add_prediction::<C>(PredictionMode)`).

## Feature flags
`std` (default), `deterministic` (lockstep mode, no state mismatch checks), `server` (enables `lightyear_messages` for server-side prediction utilities), `metrics` (Prometheus-style counters via `metrics` crate).

## Notable internals
- `RollbackSchedule` (`src/rollback.rs:43`) — separate schedule label that re-runs `FixedMain` once per rolled-back tick.
- `check_rollback` is built dynamically via `QueryParamBuilder` (`src/rollback.rs:128`–`:166`) so it queries every registered predicted component's `PredictionHistory<C>` + `Confirmed<C>` in one pass. It excludes `DeterministicPredicted` and `DisableRollback`, but keeps `PredictionDisable` entities (those are predicted-despawned but kept alive to allow rollback restoration).
- Default history is *not* skipped — `RollbackPolicy::no_prediction_history()` returns true only when `state != Disabled && input == Disabled`. Comments admit this is over-conservative; for `RollbackMode::Always (state)` you can in principle skip storage, but the impl doesn't yet.
- `PredictionSyncBuffer` (`src/manager.rs:91`) batches `Confirmed → Predicted` component sync via `lightyear_replication::registry::buffered::BufferedChanges`.
- `Time<Fixed>` is rolled back internally so `FixedMain` systems see consistent time during rollback. **Do not** register `Time<Fixed>` via `add_resource_rollback_systems` — the doc comment in `plugin.rs:82–96` warns it's already handled.
- Pre-spawned entities (`PreSpawned` from `lightyear_replication`) are rolled back to their historical state; entities that didn't exist at the rollback tick get despawned.
- `VisualCorrection` ordering: in `PostUpdate` it runs **after** `FrameInterpolationSystems::Interpolate` so frame-interp doesn't overwrite the correction (`rollback.rs:90–97`).
- `parking_lot::RwLock<RollbackState>` lets multiple PreUpdate systems flag a pending rollback in parallel without exclusive `&mut` on the manager.

## Debugging hooks
- Rollback storms → log `RollbackSystems::Check` decisions; check `PredictionMetrics`.
- "My component never rolls back" → ensure registered via `PredictionAppRegistrationExt::add_prediction::<C>(...)` and that `C: SyncComponent` (Mutable + Clone + PartialEq + Debug).
- "Rollback feels too long" → `RollbackPolicy::max_rollback_ticks` (default 100) caps the depth; receiving a packet older than this is silently ignored.
- Visual jitter post-rollback → tune `CorrectionPolicy` (in `src/correction.rs`); requires `Diffable<D>` impl on the component.

## Used by wam
Only via the umbrella `lightyear` crate. wam currently uses umbrella feature `prediction` but doesn't call into `PredictionPlugin` APIs directly from any wam crate.
