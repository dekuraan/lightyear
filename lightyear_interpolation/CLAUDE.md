# lightyear_interpolation — Claude guidance

Lightyear 0.26.4 vendored at `vendor/lightyear/lightyear_interpolation/`. Read this first when debugging "remote players teleport / stutter / look like they're in the past", buffering glitches at low tickrate, or lag-compensation on the server.

**This is the *network* interpolation crate** — it interpolates between *server snapshots* on entities you don't predict. For sub-tick *visual* smoothing on the client's own entities see `lightyear_frame_interpolation`. They are independent and can stack (a remote entity is network-interp'd; its visual position is then frame-interp'd between fixed ticks).

## Purpose
Replicated remote entities are buffered into a `ConfirmedHistory<C>` and the local `Interpolated` view is computed by interpolating between two consecutive confirmed snapshots, while the `InterpolationTimeline` runs deliberately *behind* `RemoteTimeline` so there's always a future snapshot to lerp toward.

## Key public API
- `prelude::InterpolationPlugin` (`src/plugin.rs:74`).
- `prelude::Interpolated` (re-export from `lightyear_core::interpolation::Interpolated`, `src/lib.rs:30`) — marker on the client-side interpolated entity (analogous to `Predicted`).
- `prelude::InterpolationDelay` (`src/plugin.rs:28`) — `Serialize`/`Deserialize`; replicated to the **server** as the client's current interp lag, used for server-side lag compensation. `tick_and_overstep(now)` returns the (tick, overstep) the client is currently displaying.
- `prelude::InterpolationTimeline` + `timeline::InterpolationConfig` (`src/timeline.rs:84,29`) — `min_delay = 5ms`, `send_interval_ratio = 1.7`, `sync = SyncConfig::default()`. Sync objective = `RemoteTimeline.current_estimate() - delay - jitter_margin`. Wraps `Timeline<InterpolationConfig>` with its own `SyncContext`.
- `prelude::ConfirmedHistory<C>` (`src/interpolation_history.rs:15`) — per-component `HistoryBuffer<C>` on the interpolated entity. `start()` (oldest) + `end()` (second-oldest) feed `interpolate()`.
- `prelude::interpolation_fraction(start, end, current, overstep)` (`src/interpolate.rs:23`) — convenience for custom interpolators.
- `prelude::InterpolationSystems` (`src/plugin.rs:80`) — `Sync` (PreUpdate, after `ReplicationSystems::Receive`), `Prepare` + `Interpolate` (Update, chained, after `SyncSystems::Sync`), `All`.
- `prelude::InterpolationRegistry` + `InterpolationRegistrationExt` (`src/registry.rs`) — register `interpolate_fn` per component. Required for `lightyear_frame_interpolation` and `lightyear_prediction::correction` too.
- `add_interpolation_systems::<C>` / `add_prepare_interpolation_systems::<C>` (`src/plugin.rs:115,101`) — escape hatches for custom interpolation logic per-component.

## Feature flags
`std` (default), `metrics`.

## Notable internals
- `update_confirmed_history` (`src/interpolate.rs:28`) prunes the buffer: keeps the snapshot just before the current interp tick + the next one ahead, so `start/end` straddle "now". Constant `SEND_INTERVAL_TICK_FACTOR = 1.3` (`:20`) gates when a stale buffer is reset to avoid interpolating from very-old data after packet loss.
- The component is **not inserted on the entity until 2 history entries exist** (`src/interpolate.rs:56–69`); until then there's nothing visible. Worth knowing if "interpolated entity has no Transform on first frame".
- `InterpolationTimeline` is a `SyncedTimeline` (in `src/timeline.rs`) but **non-driving** (`SyncedTimelinePlugin::<InterpolationTimeline, RemoteTimeline, false>`) — it never touches `Time<Virtual>`. Its sync uses the same `SyncContext`/error-margin machinery as `lightyear_sync` but with the interpolation-config sync params.
- `remote_send_interval` is bootstrapped from a `RemoteEvent<SenderMetadata>` trigger sent by the server on connection (`src/timeline.rs` mentions in the doc-comment at top); affects `InterpolationConfig::to_duration`.
- Host-Client (`HostClient`) gets `InterpolationDelay::default()` auto-inserted (zero delay) since there's no actual remote.

## Debugging hooks
- "Remote entity teleports" → buffer was reset (gap > `SEND_INTERVAL_TICK_FACTOR * remote_send_interval`); check packet loss / `min_delay`.
- "Remote entity stutters at low tickrate" → bump `send_interval_ratio` (default 1.7) so interp-delay grows with `remote_send_interval`.
- "Server's lag compensation is wrong" → server reads the client's replicated `InterpolationDelay` to roll-back hits; check the value matches `InterpolationTimeline.now()` lag from `RemoteTimeline`.

## Used by wam
Only via the umbrella `lightyear` crate. wam currently uses umbrella feature `interpolation` but no wam crate directly imports `lightyear_interpolation`.
