# TrustBridge Plugin Architecture

Design document for the extensible check system.

Related docs: [README](../README.md) · [Usage](USAGE.md) · [Architecture](ARCHITECTURE.md) · [Contributing](../CONTRIBUTING.md)

---

## Why plugins?

The original `runAccountChecks` in `src/checks.ts` hard-coded checks in a single function. The plugin system makes each check a self-contained unit that composes into the same `ValidationResult` structure.

---

## Core concepts

### `CheckPlugin`

```ts
interface CheckPlugin {
  readonly id: string;
  readonly label: string;
  run(ctx: CheckPluginContext): CheckPluginResult;
}
```

### `CheckPluginContext`

```ts
interface CheckPluginContext {
  readonly account: HorizonAccount | null;
  readonly config: Readonly<CheckConfig>;
  readonly stellarAddress: string;
}
```

### `CheckPluginResult`

```ts
interface CheckPluginResult {
  readonly passed: boolean;
  readonly detail: string;
  readonly remediation?: string;
}
```

### `PluginRegistry`

A deduplication-safe registry storing `CheckPlugin` instances.
- **First-wins semantics**: Registering a plugin with an already existing `id` is a no-op.
- Maintains insertion order for running plugins.

### `ValidationResult` Composition

`runPlugins(ctx, registry)` executes plugins sequentially and builds a composite `ValidationResult`:
- `valid`: true only when *all* plugins pass.
- Top-level flags (`accountFunded`, `trustlineExists`, `xlmReserveMet`) are derived dynamically from well-known plugin ids (e.g., `'account-funded'`, `'trustline'`, `'xlm-reserve'`).
- `checks`: One `CheckResultItem` array item per plugin, preserving execution order.
- `remediation`: Aggregates all non-empty `remediation` strings from failed plugins.

---

## Lifecycle & Loader (`pluginLoader`)

### Asynchronous Loading

Plugins are dynamically imported at runtime via `file://` URLs. The `pluginLoader` resolves plugins relative to the workspace root (`GITHUB_WORKSPACE`).

- `loadPlugin`: Asynchronously loads and validates a single plugin. It looks for a `default` export, a named `plugin` export, or the first exported object matching the `CheckPlugin` interface.
- `loadPluginsFromAllowlist`: Loads multiple plugins defined in an allowlist. Missing or invalid plugins log warnings but **fail-open** so the core action is not blocked.

### Core Plugins (`corePlugins`)

The core checks are shipped as built-in plugins:
- `accountFundedPlugin` (`trustbridge/account-funded`)
- `trustlinePlugin` (`trustbridge/trustline`)
- `xlmReservePlugin` (`trustbridge/xlm-reserve`)
- `homeDomainPlugin` (`trustbridge/home-domain`)

These are pre-registered into the `defaultRegistry` at action startup via `registerCorePlugins()`.

---

## Security

Plugins must not execute arbitrary code sourced from issue bodies.

### 1. Typed context only
`run()` receives typed action inputs and Horizon data only.

### 2. No dynamic imports or eval
Plugins are reviewed TypeScript source files. The runner does not evaluate strings.

### 3. Output escaping responsibility
Plugin strings must escape external values before returning them.

### 4. No runtime npm loading
Arbitrary npm packages are out of scope for v1.

### 5. Workspace-only path constraints
Optional plugins are loaded from the workspace only, never from `node_modules` or remote URLs.

- Workspace root: `GITHUB_WORKSPACE`
- Example plugin path: `plugins/kyc.ts`
- Enable via action input: `trustbridge_plugins_path: plugins/kyc.ts`

The loader actively rejects:
- Absolute paths.
- Path traversal sequences (`../`) attempting to escape the workspace root.
- Non-file targets (e.g., directories or symlinks).

---

## File map

```text
src/
  plugin.ts         - CheckPlugin, CheckPluginContext, CheckPluginResult, PluginRegistry
  pluginRunner.ts   - runPlugins(ctx, registry?) -> ValidationResult
  pluginLoader.ts   - Workspace-only async plugin loader with allowlist + path guards
  corePlugins.ts    - accountFundedPlugin, trustlinePlugin, xlmReservePlugin, homeDomainPlugin
__tests__/
  plugin.test.ts    - Registry, runner, core plugins, security contract
  plugin-loader.test.ts - Loader path guards and allowlist behavior
docs/
  PLUGIN_ARCHITECTURE.md - This document
```

---

[← Back to Architecture](ARCHITECTURE.md) · [← Back to README](../README.md)
