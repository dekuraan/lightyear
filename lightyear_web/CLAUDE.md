# lightyear_web — Claude guidance

## Purpose
WASM-only browser glue: keeps Bevy's `Main` schedule running when the browser tab is hidden. Without this, `requestAnimationFrame` is throttled and the netcode loop stops sending keep-alives; the server then drops the client.

## Key public API
- `WebKeepalivePlugin { wake_delay: f64 }` (`src/background_worker.rs:25`) — `wake_delay` is the worker tick period in ms; default `1000.0/60.0` (~60 Hz).
- `KeepaliveSettings { wake_delay: f64, worker: Option<Worker> }` (`src/background_worker.rs:57`) — runtime-mutable resource. Currently can't switch to `setInterval` from `setTimeout` after init (its docs lie slightly: implementation uses `setInterval` from the start).
- Re-exported at crate root: `pub use background_worker::{KeepaliveSettings, WebKeepalivePlugin}` (`src/lib.rs:6`).

## Feature flags
None.

## Platform gates
**WASM-only by design.** No `cfg` guards in source — the crate unconditionally pulls in `wasm-bindgen` + `web-sys` and uses `web_sys::Worker` / `Blob` / `Url`. Will not build for non-wasm targets if depended on. `#![no_std]` with `extern crate alloc`.

## Notable internals
- Spawns a `web_sys::Worker` from an inline JS Blob containing a `setInterval` loop that posts a message to the main thread.
- The main thread `onmessage` handler checks `document.hidden`; if hidden, sends a `WinitUserEvent::WakeUp` through `EventLoopProxyWrapper`, which forces Bevy to run another frame.
- `unsafe impl Send for KeepaliveSettings` / `Sync` — justified by single-threaded WASM environment; only present to satisfy Bevy's `Resource` trait bounds.
- `Drop` calls `worker.terminate()`.
- Stores `Rc<*mut World>` and `Closure::forget()`s the message handler — typical wasm-bindgen lifecycle, intentionally leaks the closure.
- Depends on `bevy_winit` for `EventLoopProxyWrapper`/`WinitUserEvent`. Won't help apps using `ScheduleRunnerPlugin` (e.g. headless-via-non-winit).

## Used by wam
Not directly imported in `crates/`. wam's launcher uses `bevy_winit` on WASM, so installing this plugin would be straightforward, but no current call site.
