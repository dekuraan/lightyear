# lightyear_sync — Claude guidance

Lightyear 0.26.4 vendored at `vendor/lightyear/lightyear_sync/`. Read this first when debugging desync, time-skew, RTT estimation, "client tick jumps backwards/forwards", or input-delay glitches.

## Purpose
Time/tick synchronization between peers. Maintains the client's estimate of remote time, drives `Time<Virtual>` relative-speed nudges to keep client and server tick-aligned, and computes how far ahead of the server the client should simulate so inputs land on time. Server-side it provides the reciprocal Pong + per-client view.

## Key public API
- `prelude::TimelineSyncPlugin` (`src/plugin.rs:14`) — base plugin; pulls in `PingPlugin`. Configures `SyncSystems::Sync` set in `PostUpdate`.
- `prelude::SyncSystems::Sync` (`src/plugin.rs:9`) — single PostUpdate system set; everything that adjusts a synced timeline runs here.
- `client::ClientPlugin` (`src/client.rs:19`) — registers `InputTimeline` as the **driving** timeline + `RemoteTimeline` as the sync target. Adds observers that recompute input delay on `SyncEvent` / `InputTimelineConfig` change.
- `prelude::PingManager` / `PingConfig` (`src/ping/manager.rs:36,18`) — RTT/jitter estimator. Default `ping_interval = 100ms`. Component required on every synced entity. Exposes `rtt_estimator_ewma`, `pings_sent`, `pongs_recv`.
- `prelude::SyncConfig` (`src/timeline/sync.rs:89`) — controls speed-nudge behavior. Defaults: `jitter_multiple=4`, `error_margin=1.0` tick, `max_error_margin=10.0` ticks (snap-resync threshold), `consecutive_errors_threshold=3`, `speedup_factor=1.05`.
- `prelude::IsSynced<T>` (`src/timeline/sync.rs:20`) — marker component inserted once a timeline reaches sync. Removed on `Disconnected`. Always present immediately on `HostClient`.
- `prelude::DrivingTimeline<T>` (`src/timeline/mod.rs:11`) — marker for the timeline that controls `Time<Virtual>` relative speed. On the client this is `InputTimeline`.
- `prelude::client::InputTimeline` + `InputTimelineConfig` (`src/timeline/input.rs:258,20`) — the predicted/ahead-of-server clock. `InputDelayConfig::balanced` / `no_input_delay` / `no_prediction` / `fixed_input_delay` (`:168`–`:201`).
- `prelude::client::RemoteTimeline` + `RemoteEstimate` (`src/timeline/remote.rs:67,27`) — local estimate of server time using EMA-smoothed offset (`min_ema_alpha=0.02`, `max_ema_alpha=0.10`, `handshake_pings=3`).

## Traits
- `SyncedTimeline` (`src/timeline/sync.rs:33`) — implemented by `InputTimeline` and `InterpolationTimeline` (in `lightyear_interpolation`). Methods: `sync_objective`, `resync`, `sync`, `is_synced`, `relative_speed`, `set_relative_speed`, `reset`.
- `SyncTargetTimeline` (`:74`) — implemented by `RemoteTimeline`. Provides `current_estimate()` + `received_packet()` gate.

## Feature flags
`std` (default), `client`, `server`. The `client` module compiles only with `client`, ditto `server`.

## Notable internals
- `SyncedTimelinePlugin<Synced, Remote, DRIVING>` (`src/timeline/sync.rs:202`) is the per-timeline plugin. Both `InputTimeline` (driving) and `InterpolationTimeline` (non-driving) use it. The `DRIVING=true` variant adds: `sync_from_local_timeline` (PostUpdate), `update_virtual_time` (Last), and a `SyncEvent` observer that applies tick-deltas to `LocalTimeline`.
- `SyncContext::speed_adjustment` (`:163`) is the throttle: small offset → do nothing; sustained same-sign offsets above `error_margin` → `SpeedAdjust(ratio)` where ratio scales linearly with offset/`max_error_margin`; offset above `max_error_margin` → `Resync` (snap, emits `SyncEvent`).
- Input-delay math: `InputDelayConfig::input_delay_ticks` (`src/timeline/input.rs:209`). `effective_rtt = rtt + jitter_margin`; clamps to `[minimum_input_delay_ticks, maximum_input_delay_before_prediction]`, anything beyond gets covered by prediction up to `maximum_predicted_ticks`. Recomputed on every `SyncEvent<InputTimelineConfig>` and on config insert.
- `ClientPlugin` always sets `InputTimeline` as `DrivingTimeline` and the prediction crate hangs the rollback off of `SyncEvent<InputTimelineConfig>`.

## Debugging hooks
- Desync? Watch for repeated `Resync Interpolation timeline!` / `apply delta to LocalTimeline` traces — means `max_error_margin` is being exceeded.
- "Inputs arriving late on server" — bump `jitter_multiple`, lower `maximum_input_delay_before_prediction`, or check `PingManager.rtt_estimator_ewma`.
- `IsSynced<InputTimeline>` missing → handshake hasn't completed (`handshake_pings`, default 3).

## Used by wam
Only via the umbrella `lightyear` crate (`crates/wam-client/Cargo.toml`, `crates/wam-protocol/Cargo.toml`). No direct dep.
