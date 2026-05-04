# lightyear_messages — Claude guidance

## Purpose
Typed message channels on top of `lightyear_transport`. Anything `Send + Sync + 'static` is a `Message`; the crate handles serialise → push to transport channel → receive → deserialise → buffer in a per-type component. Also handles "remote events" (remote `Trigger`s). `no_std` core.

## Key public API
- `Message` trait blanket-impls for `Send + Sync + 'static` (`src/lib.rs:61`).
- `MessageManager` component (`src/lib.rs:77`) — required on any sending/receiving entity. Auto-required on `Client` (client feature) and `ClientOf` (server feature) via `register_required_components`. Holds `entity_mapper: RemoteEntityMap`.
- `MessagePlugin` (`src/plugin.rs:31`) — adds `TransportPlugin` if missing, wires `recv`/`send`/`clear`/`send_local` systems built dynamically in `Plugin::finish` from the `MessageRegistry` (`:64`). Clears `entity_mapper` on `Disconnected`.
- `MessageSystems` (`src/plugin.rs:16`): `Receive` (PreUpdate, after `TransportSystems::Receive`), `Send` (PostUpdate, before `TransportSystems::Send`).
- `MessageSender<M>` (`src/send.rs:52`): `send::<C: Channel>(M)`, `send_with_priority::<C>(M, Priority)` (`:90`, `:100`).
- `MessageReceiver<M>` (`src/receive.rs:53`): `receive() -> impl Iterator<Item = M>` (`:83`), `receive_with_tick()` returning `ReceivedMessage<M>` (`:88`), `has_messages`, `num_messages`.
- `ReceivedMessage<M> { data, remote_tick, channel_kind, message_id }` (`:59`).
- `MessageKind(TypeId)` (`src/registry.rs:55`), `MessageRegistry` (`:157`). Trait `AppMessageExt` (`:341`) provides `register_message::<M>()` returning `MessageRegistration` with `add_map_entities()`, `add_direction(NetworkDirection)`, `register_message_to_bytes()`.
- Triggers (`src/trigger.rs`): `AppTriggerExt::register_trigger::<M: Event>()`, `EventSender<M>` (`src/send_trigger.rs`), `RemoteEvent<M>` (`src/receive_event.rs`) for receiver side. Triggers are messages that fire as ECS triggers when received.
- Server (`src/server.rs:31`): `ServerMultiMessageSender<'w, 's, F>` — `send::<M, C>(message, &Server, NetworkTarget)` (`:38`), `send_with_priority`, `send_to_entities` (`:106`), `send_to_entities_with_priority` (`:117`). Single-call fan-out to many `ClientOf` peers.
- `multi.rs` — multi-client send helpers shared between client and server.

## Feature flags
- `std` (default, propagates to `lightyear_transport/std`)
- `client` (re-exports client-side helpers, pulls `lightyear_connection/client`)
- `server` (pulls `lightyear_link` and `lightyear_connection/server`)
- `metrics`, `test_utils`

## Notable internals
- Receive/send systems are built with `ParamBuilder + QueryParamBuilder` in `Plugin::finish`, dynamically requesting mutable access to every registered `MessageSender<M>` / `MessageReceiver<M>` / `EventSender<M>` component id. This means **all messages must be registered before `app.finish()`**.
- `MessageManager::send_messages`, `send_triggers`, `receive_messages` are populated by component `on_add` hooks on `MessageSender`/`MessageReceiver`/`EventSender` (e.g. `MessageSender::on_add_hook` `src/send.rs:169`).
- `send_local` system (registered in `Last`) handles the "loopback" path used by host-server: messages sent locally bypass the wire and land in receivers in the same App. Runs after `clear` so they survive the frame.
- Message NetIds (`MessageNetId = lightyear_core::network::NetId`) are assigned by `MessageRegistry` and serialised on the wire — registration order must match across peers, which `register_message` enforces by hashing.
- `RemoteEntityMap` on `MessageManager` is the canonical local↔remote `Entity` mapping; used by `add_map_entities()` to rewrite entity fields on send/receive.
