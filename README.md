# Den.Bridge

Reusable bridge/sidecar infrastructure for Electron and web-based desktop applications.

## Layout

```
src/Den.Bridge/          — Generic .NET bridge abstractions, protocol, registry, schema, and WebSocket transport
tests/Den.Bridge.Tests/  — Bridge unit and boundary tests
packages/den-bridge/     — (planned) Generic TypeScript/web bridge contract and client helpers
packages/den-bridge-electron/ — (planned) Electron-specific bridge transport and preload wiring
```

## .NET Projects

- **Den.Bridge** — Generic bridge runtime with no product-specific dependencies.
  - `Abstractions` — handler and transport contracts
  - `Protocol` — JSON frame types and serialization
  - `Registry` — command/event registry and builder
  - `Schema` — schema bundle generation
  - `Transport/WebSockets` — WebSocket server/client transport
  - `Hosting` — bridge host integration and command invoker
  - `InMemory` — in-memory test harness

- **Den.Bridge.Tests** — Boundary and functional tests ensuring Den.Bridge does not reference product assemblies.

## Build

```bash
dotnet build den-bridge.slnx
dotnet test den-bridge.slnx
```

## Submodule Consumer Setup

Add this repository as a git submodule:

```bash
git submodule add git@github.com:FuzzySlipper/den-bridge.git external/den-bridge
```

Then reference the projects from your solution:

```xml
<Project Path="external/den-bridge/src/Den.Bridge/Den.Bridge.csproj" />
```

## Boundary Policy

- `Den.Bridge` must not reference any `DenMcp.*` assemblies.
- `Den.Bridge` must not reference Electron, Tauri, WebView, ASP.NET Core, or Terminal.Gui packages.
- Electron- or product-specific handlers, DTOs, and services belong in the consumer project, not here.

## Error Propagation

All catch-all error paths in the command invoker and WebSocket transport include
structured diagnostic `details` in `BridgeError`. This ensures TypeScript clients
receive actionable error context (exception type, message, command, request ID)
instead of opaque "failed" messages.

- `BridgeCommandInvoker` includes `command`, `request_id`, `exception_type`, and
  `exception_message` in `HandlerFailed` and `RequestCancelled` error details.
- `WebSocketBridgeServer` transport-level dispatch failures include the same
  structured details with server-side context.
- `BridgeHandlerException` (explicit handler errors) already carried optional
  `details` and `caused_by` chains; this enhancement ensures generic exceptions
  are not silently swallowed.

## Request / Response Correlation

Request/response correlation is already part of the core frame contract via
`BridgeRequestFrame.Correlation` and `BridgeResponseFrame.Correlation`. This
capability is intentionally not listed as deferred work below; future hardening
should preserve correlation on both success and error responses.

## Deferred Upstream Work

These reusable bridge hardening areas were identified during VoxelForge
Electron integration but are **not implemented upstream yet**. Each has
an explicit rationale and follow-up tracking.

### Binary Payload Support

- **Gap:** The bridge protocol and WebSocket transport support only text JSON
  frames. Large mesh payloads (vertex buffers, index buffers) must be serialized
  as JSON arrays, which is wasteful for float/int arrays.
- **Upstream scope:** `WebSocketBridgeFrameIO` should support binary WebSocket
  frames with a lightweight framing header (payload ID, byte offset, total size).
  `IBridgeFrame` needs a binary variant or side-channel.
- **Consumer workaround:** VoxelForge uses JSON-only mesh payloads with a
  documented size threshold (~10K vertices) before binary is warranted. Binary
  support is deferred until the upstream den-bridge transport layer implements it.
- **Follow-up:** Design + implement binary frame type + WebSocket binary frame
  support in `Den.Bridge.Transport.WebSockets`.

### Large Message Framing / Backpressure

- **Gap:** The WebSocket server sends events to all connected clients with
  `Task.WhenAll`, which provides no backpressure. A slow client can accumulate
  pending sends in the OS socket buffer. The `MaxFrameBytes` option bounds
  individual frames but not aggregate pressure.
- **Upstream scope:** Configurable send buffer per-connection, optional
  backpressure signal (frame acknowledgment), and client-side flow control hint.
- **Consumer workaround:** VoxelForge bounds event frequency and payload size
  at the application level; mesh updates are batched per-frame.
- **Follow-up:** Add per-connection send queue with bounded capacity and
  backpressure event for consumer PubSub wiring.

### Request Deadline / Timeout Enforcement

- **Gap:** `BridgeRequestFrame` has a `deadline_ms` field, but no code reads
  it. The cancellation token is threaded through but timeout enforcement is
  purely advisory — consumers must build their own `CancellationTokenSource`
  with the deadline.
- **Upstream scope:** `BridgeCommandInvoker` (and `WebSocketBridgeServer`)
  should optionally derive a `CancellationTokenSource` from `deadline_ms` and
  link it to the handler's cancellation token.
- **Consumer workaround:** VoxelForge sets per-request timeouts at the bridge
  client level (Electron TS `setTimeout` wrapping `send()`).
- **Follow-up:** Add optional deadline enforcement in the invoker, configurable
  via `BridgeCommandInvokerOptions`.

### Process Lifecycle / Restart Behavior

- **Gap:** No built-in health check loop, process supervision, or auto-restart.
  `BridgeHealthFrame` exists as a data type but nothing emits it periodically.
- **Upstream scope:** `BridgeHostIntegration` could expose a periodic health
  publisher. A restart supervisor is a separate concern for the host process.
- **Consumer workaround:** VoxelForge manages the C# sidecar process lifecycle
  via Electron's child_process with manual restart logic.
- **Follow-up:** Add optional periodic health frame emission to
  `BridgeCapabilitiesProvider` or a dedicated `BridgeHealthService`.

### Structured Logging / Diagnostics

- **Gap:** Logging exists (`ILogger<BridgeCommandInvoker>`, `ILogger<WebSocketBridgeServer>`)
  but uses `NullLogger` by default. There is no structured diagnostic event
  channel that TS clients can subscribe to.
- **Upstream scope:** A `IBridgeDiagnosticChannel` interface that routes
  structured diagnostic frames to connected clients via a den-bridge event.
- **Consumer workaround:** VoxelForge built `voxelforge.diagnostics.event` as
  a VoxelForge-specific bridge event.
- **Follow-up:** Consider adding `IBridgeDiagnosticChannel` to the abstractions
  package once a concrete consumer pattern emerges across repositories.

### Schema / Version Negotiation

- **Gap:** The bridge protocol has `protocol_version` and `schema_version` on
  every frame but no built-in negotiation handshake. Consumers must build their
  own (as VoxelForge did with `voxelforge.handshake`).
- **Upstream scope:** A `BridgeHandshakeHandler` or capability exchange that
  runs automatically after transport connect, exposing `IBridgeCapabilitiesProvider`
  results and negotiating schema version compatibility.
- **Consumer workaround:** VoxelForge implements multi-step handshake in
  `VersionHandshakeHandler` + `VoxelForgeSchemaHandshakeHandler`, which is
  explicitly documented as VoxelForge-specific adapter code.
- **Follow-up:** Evaluate whether a generic handshake flow should be added to
  `Den.Bridge.Hosting` or kept entirely consumer-specific.

### Error Propagation (Completed in this round)

- **Status:** Improved. Catch-all error paths in `BridgeCommandInvoker` and
  `WebSocketBridgeServer` now include structured `details` with exception type,
  message, command, and request ID instead of opaque "failed" messages.
- See [Error Propagation](#error-propagation) above for details.
