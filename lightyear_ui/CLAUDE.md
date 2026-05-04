# lightyear_ui — Claude guidance

## Purpose
Runtime debug overlay for Lightyear's metrics. Reads from `lightyear_metrics`'s `MetricsRegistry` (which sits on top of the `metrics` crate) and renders a compact panel using **native Bevy UI** (`bevy_ui` nodes + `bevy_text`). Single feature: a counters overlay similar to `bevy_diagnostic`'s FPS overlay but specialised for Lightyear send/recv channel stats, ping, etc.

## Key public API
- `prelude::DebugUIPlugin` (`src/debug.rs:498`) — the only plugin. Registers `MetricsPlugin` if absent, inserts `MetricsPanelSettings`, `MetricsPanelLayout`, `VisibilityFilter`, `CollapseState`, `MetricHistory`, spawns the panel in `Startup`, runs `update_visibility / handle_button_interactions / update_collapsible_displays / sample_metrics_history / update_metrics` chained in `Last` before `ClearBucketsSystem`.
- `MetricsPanelSettings` (`src/debug.rs:31`) — `enabled: bool`, `window_len: usize` (rolling-average length, default 50), `alpha: f32` (background alpha, default 0.2).
- `MetricDirection::{Send, Receive, Neutral}` (`src/debug.rs:59`) — drives left/right grouping.
- `MetricSpec` (`src/debug.rs:73`) — `label: &'static str`, `key: CompositeKey` (fuzzy-matched against the metrics registry), `per_second: bool`, direction.

## Feature flags
`std` (default), `test_utils`. No bevy-feature gating (always uses `bevy_ui` + `bevy_text`).

## Notable internals
- All Bevy UI, no egui. Built from `Node` / `Text` entities; uses `bevy_color`, `bevy_text`, `bevy_ui` directly.
- Pre-canned metric specs cover replication channels, ping/pong, sync, transport — see `src/debug.rs:480` for examples like `Key::from_parts("channel/recv_messages", &[("channel", "lightyear_sync::ping::PingChannel")])`.
- Rolling `MetricHistory` keeps `window_len` samples per metric so a smoothed value can be drawn.
- Collapsible sections via `SectionHeader` / `SubsectionHeader` components and `CollapseState` resource.

## Status in wam
**Not in use, but fits.** wam UI policy is "Bevy UI + feathers, egui only for dev tools" (`feedback_ui_bevy_not_egui.md`), so `DebugUIPlugin` is a clean fit — no overlap with `bevy_egui` and won't fight the production HUD (`project_hud_decision.md`). wam currently has its own dev panels (`wam-client/src/dev_panels/` behind a `dev-panels` feature flag) and uses `chill_bevy_console` for the backtick console (`crates/wam-client/CLAUDE.md`). Adding `DebugUIPlugin` would give a turn-key Lightyear-metrics overlay alongside those, no conflict expected. Right now it isn't pulled in because wam's `lightyear` feature list doesn't include `metrics`/`ui`.
