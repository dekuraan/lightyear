# lightyear_crossbeam — Claude guidance

## Purpose
In-process loopback transport using `crossbeam_channel`. Used for tests and host-server / single-process integration where two Bevy worlds (or a fake "remote") share memory. Compiles `no_std`.

## Key public API (`src/lib.rs`)
- `CrossbeamIo` component (`lib.rs:38`) — `#[require(Link::new(None))]`, `#[require(LocalAddr(LOCALHOST))]`, `#[require(PeerAddr(LOCALHOST))]`. Wraps a `Sender<Bytes>` + `Receiver<Bytes>`.
- `CrossbeamIo::new(sender, receiver)` and `CrossbeamIo::new_pair()` — `new_pair()` creates two crossed unbounded channels and returns the matched `(client_io, server_io)` pair (`lib.rs:49`).
- `CrossbeamPlugin` (`lib.rs:61`):
  - On `LinkStart` for any entity with `CrossbeamIo`, immediately inserts `Linked` (no handshake).
  - `receive` (PreUpdate, in `LinkReceiveSystems::BufferToLink`) drains the receiver via `try_recv` and pushes into `link.recv` with `Instant::now()`.
  - `send` (PostUpdate, in `LinkSystems::Send`) drains `link.send` into the crossbeam sender via `try_send`. With `test_utils`, `TestHelper::block_send` short-circuits sends.

## Constants
- `MTU = 1472` (matches typical UDP MTU; not enforced by the channel itself, just a hint).
- `LOCALHOST = 127.0.0.1:0` — the auto-injected `LocalAddr`/`PeerAddr` so address-based plugins (e.g. `RawConnectionPlugin`) work uniformly.

## Feature flags
- `default = []` (note: no `std` feature; crate is `no_std` and stays that way).
- `test_utils` — pulls in `lightyear_core::test::TestHelper` for blocked-send simulation.

## Notable
- No `aeronet` involvement despite `aeronet_io` being a dep — only the `LocalAddr`/`PeerAddr` types are reused for cross-plugin compatibility.
- `try_send` on unbounded channels never errors in practice; the `Result` propagates a panic on a closed channel.
