# lightyear_metrics — Claude guidance

## Purpose
Optional integration with the `metrics` crate for runtime counters/gauges/histograms. Off by default in `lightyear`; opt in via `metrics` or `debug` feature on the umbrella crate. `no_std` by default but most useful paths require `std`.

## Key public API
- **`MetricsPlugin`** (`src/plugin.rs:13`) — Bevy plugin. `MetricsPlugin::default()` creates a fresh `MetricsRegistry`; `MetricsPlugin::with_registry(reg)` reuses one (and silences `set_global_recorder` errors so the user can install their own recorder upstream). On `build`, calls `metrics::set_global_recorder` and inserts the registry as a `Resource`. Schedules `MetricsRegistry::clear_atomic_buckets_system` in `Last`, in `SystemSet` `ClearBucketsSystem`.
- **`ClearBucketsSystem`** (`src/plugin.rs:22`) — empty marker `SystemSet`. Histogram consumers (e.g. plot widgets) must run in `Last` *before* this set or they will miss samples.
- **`MetricsRegistry`** (`src/registry.rs:30`) — `Clone + Resource`. Wraps `metrics_util::registry::Registry<Key, AtomicStorage>` in an `Arc<Inner>`. Implements `metrics::Recorder`. `fetch_metric_value(&CompositeKey) -> Option<f64>` dispatches to counter/gauge/histogram-mean lookups (`registry.rs:35`). Has both per-instance and global lookup paths.
- **`GLOBAL_RECORDER`** (`src/registry.rs:18`, `std` only) — `LazyLock<MetricsRegistry>` that auto-installs itself via `set_global_recorder`. Use for ad-hoc reads without a Bevy `World` handle.
- **`SearchResult`** (`registry.rs`) — name-prefix search over registered metrics (used by the debug UI plugin).
- **Re-exports**: `pub use metrics; pub use metrics_util;` (`src/lib.rs:11`) — sub-crates pull these through `lightyear_metrics` instead of declaring their own dep.
- **`prelude`** (`src/lib.rs:14`) — `MetricsPlugin`, `ClearBucketsSystem`, `MetricsRegistry`, `SearchResult`, `GLOBAL_RECORDER` (when `std`).

## Feature flags
- `default = ["std"]`.
- `std` — required for the global recorder + `set_global_recorder`. The `metrics` crate is not no_std-compatible currently.
- `test_utils` — placeholder, no extra deps.

## Notable internals
- All metrics use `AtomicStorage` (atomic counters, gauges, atomic-bucket histograms). The `ClearBucketsSystem` system drains the buckets each frame — a histogram observer that runs *after* this set will read empty data.
- The umbrella crate's `metrics` feature force-enables `std` and cascades the per-crate `metrics` feature on `lightyear_inputs`, `_interpolation`, `_messages`, `_prediction`, `_replication`, `_transport`, `_utils`, `_udp`. Without it those crates compile out their counter/gauge calls entirely.
- The crate-level doc comment (`//! # Lightyear UI`) is a stale paste; this is the metrics crate, not the UI crate.
