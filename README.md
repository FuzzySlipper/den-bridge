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
