# lightyear_serde — Claude guidance

## Purpose
Wire-format primitives. Defines the `ToBytes` trait, the `Reader`/`Writer` byte streams, varint encoding, an `EntityMap` for entity remapping across peers, and a type-erased serialization registry. `no_std` (uses `no_std_io2`); the `std` feature adds standard I/O paths in `bincode`/`bytes`/`no_std_io2`.

## Key public API
- **`ToBytes` trait** (`src/lib.rs:75`) — `bytes_len()`, `to_bytes(&impl WriteInteger)`, `from_bytes(&mut Reader)`. Impls in `lib.rs` for `u8`/`u16` (varint), `Bytes`, `Option<M>`, `Vec<M>`, `HashMap`, tuples up to arity 8 (`variadics_please::all_tuples!`).
- **`SerializationError`** (`src/lib.rs:51`) — `#[non_exhaustive]` enum: `Io`, `InvalidPacketType`, `InvalidValue`, `SubtractionOverflow`, `BincodeEncode`/`BincodeDecode`.
- **`reader::Reader`** (`src/reader.rs`) — wraps `Bytes`, supplies `ReadInteger`, `ReadVarInt`, `split_len`, `Seek`. The deserialization input throughout Lightyear.
- **`writer::Writer`** + `WriteInteger` (`src/writer.rs`) — buffer that grows via `BytesMut`; `write_varint`, `write_u8/16/32/64`, etc.
- **`varint`** (`src/varint.rs`) — `varint_len(u64)`, encoding helpers. Used for length-prefixing and `u16`/`u32` packing.
- **`entity_map`** (`src/entity_map.rs`) —
  - `SendEntityMap` / `RemoteEntityMap` / `ReceiveEntityMap` — implement `EntityMapper`. Map local `Entity` ↔ remote `Entity` during (de)serialization.
  - `EntityMap` (`:27`) — softer variant used by prediction/interpolation: missing mappings leave the entity unchanged instead of substituting `Entity::PLACEHOLDER`.
  - Top bit (`MARKED = 1 << 63`) flags entities that have been remapped to a target world's index, to avoid double-mapping.
- **`registry::ErasedSerializeFns`** (`src/registry.rs:14`) — type-erased function pointers (`erased_serialize`, `serialize`, `context_serialize`, `deserialize`, `context_deserialize`, `erased_clone`, `map_entities`). Backbone of `MessageRegistry` / `ComponentRegistry` in `lightyear_messages` / `lightyear_replication`.
- **`prelude`** (`src/lib.rs:38`) — `SerializationError`, the three entity maps, and `Seek`/`SeekFrom` from `no_std_io2`.

## Feature flags
- `default = ["std"]`.
- `std` — turns on `std` for `bincode`, `bytes`, `no_std_io2`. The crate itself is always `#![no_std]`; the feature just unlocks std-backed implementations of its dependencies.

## Notable internals
- `RawData = Vec<u8>` (`src/lib.rs:46`) — alias used through the codebase for owned packet bodies.
- `Bytes::from_bytes` does zero-copy slicing via `Reader::split_len` — no allocation on the receive path.
- The `ToBytes` impl for `Vec` and `HashMap` length-prefixes with a varint, then defers to element impls; this is the canonical pattern when adding new container impls.
