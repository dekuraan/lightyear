# lightyear_utils — Claude guidance

## Purpose
Grab-bag of pure-data utilities. No Bevy plugin, no I/O. `no_std` by default.

## Key public API (one module per file, all listed in `src/lib.rs`)
- **`wrapping_id`** (`src/wrapping_id.rs`) — `wrapping_id!` macro that produces a wrapping `u16` newtype (used by `lightyear_core::Tick`, `lightyear_messages::MessageId`, `lightyear_transport::PacketId`, etc.). Generates `ToBytes`, `Add`/`Sub`/`AddAssign`, `Ord` via `wrapping_diff`. The trait `WrappedId::rem(&self, total)` lets these IDs index into fixed-size circular buffers.
- **`sequence_buffer::SequenceBuffer<K, T, N>`** (`src/sequence_buffer.rs:14`) — fixed-size `[Option<T>; N]` indexed by `K::rem(N)`. O(1) insert / get / remove. Used wherever Lightyear stores per-tick or per-packet state (input buffers, ack tracking).
- **`ready_buffer::ReadyBuffer<K, T>`** (`src/ready_buffer.rs:13`) — `BinaryHeap` min-heap keyed by `K`. Pop yields items whose key is `<= current_key`. Used for tick-deferred work.
- **`free_list::FreeList<T, N>`** (`src/free_list.rs:1`) — fixed `[Option<T>; N]` with a `len` counter; pre-allocated stack-friendly slot pool.
- **`registry::TypeMapper<K: TypeKind>`** (`src/registry.rs:14`) — bi-directional `TypeId ↔ NetId (u16)` map. The shared primitive behind every Lightyear `*Registry` (messages, components, channels). `NetId` is `pub(crate)` here but the consumer crates re-export their own.
- **`collections`** (`src/collections.rs`) — type aliases `HashMap<K, V> = hashbrown::HashMap<K, V, FixedHasher>` and `HashSet<K>`, plus `EntityHashMap`/`EntityHashSet` re-exports. Use these instead of `std::HashMap` to keep bevy/lightyear hashers consistent.
- **`ecs`** (`src/ecs.rs`) — small ECS helpers (system param utilities, etc.).
- **`captures`** (`src/captures.rs`, 6 lines) — lifetime-capturing trait for `impl Trait` returns.
- **`metrics`** (`src/metrics.rs`, behind `metrics` feature) — bridges to the `metrics` crate.

## Feature flags
- `default = []`.
- `std` — flips `std` on bevy_ecs / bevy_platform / bevy_reflect.
- `metrics` — pulls `metrics` crate (forces `std`).

## Notable internals
- The `wrapping_id!` macro defines its own private module `<name>_module` and re-exports the type to avoid trait-impl collisions when multiple ID types share the same crate.
- `wrapping_diff(a, b) -> i32` returns the signed minimal distance accounting for wrap; this is what makes "tick 65535 < tick 1" work correctly across the codebase.
