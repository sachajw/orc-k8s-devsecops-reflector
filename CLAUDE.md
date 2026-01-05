# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Kubernetes Reflector is a Kubernetes addon that monitors changes to resources (secrets and configmaps) and reflects those changes to mirror resources in the same or other namespaces. It enables automatic synchronization of secrets and configmaps across namespaces using annotation-based configuration.

## Build and Development Commands

### Build
```bash
dotnet restore
dotnet build --configuration Debug
dotnet build --configuration Release
```

### Testing
```bash
# Run all tests
dotnet test --configuration Debug --verbosity normal

# Run tests with no build (if already built)
dotnet test --no-build --configuration Debug --verbosity normal

# Test results are written to .artifacts/TestResults/*.trx
```

### Docker
The application is containerized. The Dockerfile is located at `src/ES.Kubernetes.Reflector/Dockerfile`.

Build for multiple platforms:
```bash
docker buildx build --platform linux/amd64,linux/arm/v7,linux/arm64 -f src/ES.Kubernetes.Reflector/Dockerfile .
```

## Architecture Overview

### Core Components

**Watchers** (`src/ES.Kubernetes.Reflector/Watchers/`):
- `WatcherBackgroundService<TResource, TResourceList>` - Base class for Kubernetes resource watchers that continuously monitors resources using streaming watch APIs
- `SecretWatcher` - Watches Secret resources across all namespaces (filters out helm.sh secrets to reduce traffic)
- `ConfigMapWatcher` - Watches ConfigMap resources across all namespaces
- `NamespaceWatcher` - Watches Namespace resources to trigger auto-reflections to new namespaces

Watchers use a bounded channel (capacity 256) to process watch events asynchronously. They have configurable timeouts (default 3600 seconds) and automatically restart sessions on faults or cancellation.

**Mirroring** (`src/ES.Kubernetes.Reflector/Mirroring/`):
- `ResourceMirror<TResource>` - Abstract base class implementing the core reflection logic, cache management, and event handling
- `SecretMirror` - Concrete implementation for Secret resources
- `ConfigMapMirror` - Concrete implementation for ConfigMap resources

The mirroring system maintains several concurrent dictionaries for tracking:
- `_autoReflectionCache` - Auto-created mirrors indexed by source
- `_directReflectionCache` - Manually configured mirrors indexed by source
- `_propertiesCache` - Cached mirroring properties for all resources
- `_notFoundCache` - Resources that were not found (avoid repeated lookups)

**Event System** (`src/ES.Kubernetes.Reflector/Watchers/Core/Events/`):
- `IWatcherEventHandler` - Handles resource change events (Added, Modified, Deleted)
- `IWatcherClosedHandler` - Handles watcher session closure and cache clearing
- Mirror classes implement both interfaces to respond to Kubernetes events

### Reflection Logic

**Source Resource Annotations** (on the original secret/configmap):
- `reflector.v1.k8s.emberstack.com/reflection-allowed` - Must be "true" to allow reflection
- `reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces` - Comma-separated list of allowed namespaces or regex patterns
- `reflector.v1.k8s.emberstack.com/reflection-auto-enabled` - When "true", automatically creates mirrors in allowed namespaces
- `reflector.v1.k8s.emberstack.com/reflection-auto-namespaces` - Restricts which namespaces get auto-mirrors

**Mirror Resource Annotations** (on the reflected secret/configmap):
- `reflector.v1.k8s.emberstack.com/reflects` - Points to source in "namespace/name" format
- `reflector.v1.k8s.emberstack.com/reflected-version` - Tracks source resourceVersion to detect changes
- `reflector.v1.k8s.emberstack.com/reflected-at` - Timestamp of last reflection
- `reflector.v1.k8s.emberstack.com/auto-reflects` - "true" if this is an auto-created mirror

See `Annotations.cs:5-19` for the annotation constant definitions.

**Reflection Types**:
1. **Direct Reflection**: Mirror resource explicitly annotates the source it reflects from
2. **Auto Reflection**: Source resource automatically creates mirrors in permitted namespaces

When a source is deleted, all auto-mirrors are automatically deleted. Direct mirrors remain but stop receiving updates.

### ES.FX Framework

This project uses the ES.FX.Ignite framework for bootstrapping and common infrastructure:
- `ES.FX.Ignite.Hosting` - Application bootstrap and hosting
- `ES.FX.Ignite.Serilog` - Structured logging with Serilog
- `ES.FX.Ignite.KubernetesClient` - Kubernetes client configuration
- `ES.FX.Ignite.OpenTelemetry.Exporter.Seq` - OpenTelemetry integration for Seq
- `ES.FX.Additions.KubernetesClient` - Extension methods for k8s models (e.g., `NamespacedName()`, `ObjectReference()`)

The `Program.cs:14-46` uses `ProgramEntry.CreateBuilder()` for ES.FX initialization and `.Ignite()` extension methods for framework setup.

## Key Implementation Details

### Caching Strategy
The ResourceMirror maintains in-memory caches that are cleared when:
1. A watcher session closes (normal or faulted)
2. A namespace watcher closes

This ensures cache consistency across watcher restarts while providing performance during normal operation.

### Event Processing Flow
1. Watcher detects Kubernetes resource change via streaming watch API
2. Event written to bounded channel (blocks if channel is full)
3. Separate task reads from channel and invokes all registered `IWatcherEventHandler` implementations
4. Mirror classes process events and update reflection state
5. Kubernetes API calls made to patch/create/delete mirror resources

### Concurrency
- Watchers use channels for decoupling watch events from processing
- Mirrors use `ConcurrentDictionary` for thread-safe cache access
- Each watcher runs in its own `BackgroundService` (hosted service)
- Session cancellation uses linked cancellation tokens (stopping token + absolute timeout)

### JsonPatch for Updates
Updates to mirrors use JSON Patch (RFC 6902) rather than full resource replacement:
- Only `data`/`binaryData` fields and annotations are patched
- Uses `JsonPatchDocument<TResource>` from `Microsoft.AspNetCore.JsonPatch`
- Employs `JsonPropertyNameContractResolver` for correct serialization

## Configuration

Configuration is bound from:
1. `appsettings.json` and `appsettings.Development.json`
2. Environment variables prefixed with `ES_` (see `Program.cs:18`)

Configuration structure is defined in `ReflectorOptions` and `WatcherOptions` classes.

## Testing

Tests use:
- xUnit v3 as the test framework
- Testcontainers.K3s for integration tests (spins up a real K3s cluster)
- Moq for mocking
- `Microsoft.AspNetCore.Mvc.Testing` for in-process testing

Integration tests are in `tests/ES.Kubernetes.Reflector.Tests/Integration/` and use a shared fixture pattern with K3s testcontainers.

## Central Package Management

This repository uses Central Package Management:
- `Directory.Build.props` - Common MSBuild properties for all projects
- `Directory.Packages.props` - Centralized package versions
- Projects reference packages without version numbers

## CI/CD

The `.github/workflows/pipeline.yaml` handles:
1. **Discovery** - Path filtering, GitVersion, build evaluation
2. **Build** - Restore, build, test, Docker multi-arch build, Helm packaging
3. **Release** - Tagging, pushing to registries (Docker Hub, GHCR), GitHub releases

Builds target:
- .NET 9.0 (`net9.0`)
- Platforms: linux/amd64, linux/arm/v7, linux/arm64
- Configuration: Release for main branch, Debug for other branches

## Important Notes

- **Helm secrets are filtered**: `SecretWatcher` explicitly ignores secrets with type starting with "helm.sh" to reduce unnecessary traffic (see `SecretWatcher.cs:24-29`)
- **Namespace-scoped operations**: All watchers use `*ForAllNamespaces` APIs but mirrors operate on specific namespaces
- **Resource version tracking**: Critical for avoiding redundant updates; checked before every reflection operation
- **Regex support**: Allowed namespaces can use regex patterns for flexible namespace matching
