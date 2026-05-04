# lightyear_frame_interpolation — Claude guidance

Lightyear 0.26.4 vendored at `vendor/lightyear/lightyear_frame_interpolation/`. Read this first when debugging "my own predicted character looks jittery between fixed ticks" or "Transform doesn't propagate after a rollback".

**Not to be confused with `lightyear_interpolation`** — that crate interpolates between *server snapshots* of remote entities. This crate interpolates between consecutive *FixedUpdate ticks* of any entity (typically your own predicted character) to smooth the gap between simulation rate and frame rate. They can stack.

## Purpose
The simulation runs in `FixedUpdate` at a fixed cadence; rendering runs in `Update` per-frame. Without smoothing the camera "ticks" at the simulation rate. This crate stores `previous_value` + `current_value` of a component per fixed tick and in `PostUpdate` lerps the live component using `Time<Fixed>::overstep_fraction()`. Restored to the true value before the next `FixedMainLoop`.

## Key public API
- `prelude::FrameInterpolationPlugin<C>` (`src/lib.rs:96`) — generic per-component plugin. Add once per component you want frame-interpolated. Requires `C: Component<Mutability=Mutable> + Clone + Debug` and a registered interpolation function in `lightyear_interpolation::InterpolationRegistry`.
- `prelude::FrameInterpolate<C>` (`src/lib.rs:156`) — opt-in marker component. Holds `previous_value`, `current_value`, and a `trigger_change_detection` bool. **Add this manually to entities you want smoothed.** Default `trigger_change_detection = true`, which is required when frame-interping `Transform` (so `TransformPropagate` runs) or `Position` (so the `Position → Transform` sync runs).
- `prelude::FrameInterpolationSystems` (`src/lib.rs:76`) — three sets:
  - `Restore` — runs in `RunFixedMainLoop::BeforeFixedMainLoop`, writes `current_value` back to the live component (bypassing change detection) so simulation sees the true tick-aligned state.
  - `Update` — runs in `FixedLast` when `not(is_in_rollback)`. Shifts `current → previous` and snapshots the new `current` from the live component.
  - `Interpolate` — runs in `PostUpdate`, before `bevy_transform::TransformSystems::Propagate` and after `ReplicationBufferSystems::Buffer` (so the replicated value is the real tick value, not the smoothed one).
- `SkipFrameInterpolation` (`src/lib.rs:188`) — replicable marker component to disable frame-interp on a per-entity basis (e.g. teleports). When present and `current_value` is set, the live component snaps to `current_value` instead of lerping.

## Feature flags
`std` (default). No metrics / server / client gating — single, always-built plugin.

## Notable internals
- `Update` does **not** run during rollback (`run_if(not(is_in_rollback))`, `src/lib.rs:120`). After rollback ends, `lightyear_prediction::correction::update_frame_interpolation_post_rollback` (in the prediction crate) manually repopulates `previous_value`/`current_value` from `PredictionHistory` so frame-interp resumes seamlessly.
- Ordering relative to `lightyear_prediction`: `RollbackSystems::VisualCorrection` runs `after(FrameInterpolationSystems::Interpolate)` so `VisualCorrection` is layered on top of frame-interp instead of getting overwritten.
- `visual_interpolation` (`src/lib.rs:191`) reads `Time<Fixed>::overstep_fraction()` — note the TODO comment about `LocalTimeline` having no overstep; the doc warns this overstep may not match what you'd compute from `InputTimeline`.
- `restore_from_visual_interpolation` uses `bypass_change_detection()` to avoid spuriously triggering systems on the restore step. The interpolation step itself respects `trigger_change_detection` per entity.
- The plugin doesn't auto-add `FrameInterpolate<C>` to predicted entities — you opt in manually. There's a TODO at `src/lib.rs:146` to make this automatic for predicted components.

## Typical wiring
```rust
app.add_plugins(FrameInterpolationPlugin::<Transform>::default());
// then on each predicted entity:
commands.spawn((Predicted, Transform::default(), FrameInterpolate::<Transform>::default()));
```

## Debugging hooks
- "Transform doesn't propagate" → ensure `trigger_change_detection = true` (the default) on `FrameInterpolate<Transform>`.
- "Replicated value is the smoothed value" → can't happen by ordering; if it does, something is running `Interpolate` before `ReplicationBufferSystems::Buffer`.
- "Rollback causes a visual snap" → that's `VisualCorrection`'s job; verify the prediction-correction system ran (it sets `current/previous` on `FrameInterpolate` from `PredictionHistory`).
- Teleport without smoothing → insert `SkipFrameInterpolation` for one frame.

## Used by wam
Only via the umbrella `lightyear` crate. wam currently uses umbrella feature `interpolation` but `FrameInterpolationPlugin` and `FrameInterpolate<C>` opt-in markers must be added explicitly per component — not seen in any wam crate today.
