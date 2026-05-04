# lightyear_tests — Claude guidance

## Purpose
Integration test harness. Builds an N-client + 1-server topology in two `App`s wired via crossbeam channels and steps them frame-by-frame so tests have deterministic control over send/receive interleaving. Also home to the cross-feature integration tests (replication, delta, authority, visibility, prediction, inputs, hierarchy, avian).

## Key public API (worth reusing)
- `ClientServerStepper` (`src/stepper.rs:36`) — owns `client_apps: Vec<App>`, `server_app: App`, plus the connected `Entity` ids on both sides (`client_entities`, `server_entity`, `client_of_entities`, `host_client_entity`). `frame_duration` and `tick_duration` drive the per-step time advancement; `current_time` is a fake clock backed by `mock_instant`.
- `StepperConfig` (`:68`): `clients: Vec<ClientType>`, `server: ServerType`, `init`, optional metrics registries, `avian_mode`.
- `ClientType` (`:51`): `Host` / `Raw` / `Netcode` / `Steam` (feature-gated).
- `ServerType` (`:60`): `Raw` / `Netcode` / `Steam`.
- Constants: `SERVER_PORT = 56789`, `SERVER_ADDR` localhost (`:22-24`), `STEAM_APP_ID = 480` (Spacewar), `TICK_DURATION = 10ms` (`:28`).
- `ProtocolPlugin` (`src/protocol.rs`) — registers shared message / channel / component types used across tests:
  - Messages: `StringMessage`, `EntityMessage` (entity-mapped).
  - Triggers: `StringTrigger`, `EntityTrigger` (entity-mapped).
  - Channels: `Channel1`, `Channel2`.
  - Plus inputs (leafwing + bei + native), avian2d sync, frame interpolation.

## Feature flags
- `std` (default), `test_utils` (default; enables test helpers in `core`/`transport`/`replication`/`crossbeam`/`aeronet`), `steam` (default; enables Steam transport tests).
- The umbrella `lightyear` dep here pulls **almost everything**: `client`, `server`, `crossbeam`, `avian2d`, `frame_interpolation`, `metrics`, `netcode`, `input_native`, `input_bei`, `leafwing`, `interpolation`, `prediction`, `raw_connection`, `replication`. So tests can exercise any combination.

## Layout
- `src/client_server/` — main test suite: `base`, `connection`, `replication`, `replication_advanced`, `delta`, `authority`, `visibility`, `hierarchy`, `messages`, plus `avian/`, `input/`, `prediction/` subdirs.
- `src/host_server/` — host-server mode (one client lives in the server App).
- `src/multi_server/` — multiple-server tests (incl. Steam).
- `src/timeline/sync_tests.rs` — clock-sync edge cases.
- `src/bin/replicate.rs` — manual repro/demo binary.

## Use as a template
When writing new lightyear integration tests for wam (or upstream), prefer constructing a `ClientServerStepper` from `StepperConfig` over hand-wiring two Apps — it already handles netcode handshake, crossbeam plumbing, mock-instant time advancement, and host-server / steam variants. `mock_instant::MockClock` + `TimeUpdateStrategy` is how the harness gets deterministic timing; `test-log` is set up so `RUST_LOG` works under `cargo test`.

## Caveats
- `test_utils` features must be on for the harness to compile — the `default` set already includes them.
- Tests rely on `bevy/reflect_auto_register`; if you add new registered types, make sure they implement `Reflect`.
- This crate is not part of any release artefact; treat it as a sandbox.
