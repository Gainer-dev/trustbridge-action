# TrustBridge Plugin Architecture

Design document for the extensible check system.

Related docs: [README](../README.md) · [Usage](USAGE.md) · [Architecture](ARCHITECTURE.md) · [Contributing](../CONTRIBUTING.md)

---

## Why plugins?

The original `runAccountChecks` in `src/checks.ts` hard-codes three checks in a single function. The plugin system makes each check a self-contained unit that composes into the same `ValidationResult` structure.

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

---

## Core plugins

`src/corePlugins.ts` exports these built-in plugins. `corePlugins` lists them in this order, which is the order they appear in the comment table.

<!-- core-plugins:start -->
| Export | Plugin id | Label |
| ------ | --------- | ----- |
| `accountFundedPlugin` | `trustbridge/account-funded` | Account funded |
| `trustlinePlugin` | `trustbridge/trustline` | Trustline |
| `xlmReservePlugin` | `trustbridge/xlm-reserve` | XLM reserve |
| `homeDomainPlugin` | `trustbridge/home-domain` | SEP-0001 home domain |
<!-- core-plugins:end -->

`__tests__/plugin.test.ts` parses this table and fails when it drifts from `corePlugins`. When adding a core plugin, export it from `src/corePlugins.ts`, append it to `corePlugins`, and add a row here in the same change.

---

## Loading external plugins

`loadPluginsFromAllowlist()` loads each allowlisted path with `loadPlugin()`:

- A missing file, a failed import, or an export without `id`, `label` and `run()` throws `PluginLoadError` (`not_found`, `load_failed`, `invalid_export`). The allowlist loader logs a warning and skips that plugin (fail-open).
- Two allowlisted plugins that export the same `id` throw `PluginLoadError` with reason `duplicate_id`, naming both paths. The action then warns and continues with core plugins only, rather than silently dropping one of them.

---

## Security

Plugins must not execute arbitrary code sourced from issue bodies.

### 1. Typed context only
`run()` receives typed action inputs and Horizon data only.

### 2. No dynamic imports or eval
Plugins are reviewed TypeScript source files. The runner does not evaluate strings.

### 3. Output escaping
The runner automatically escapes Markdown metacharacters in plugin `label`, `detail`, and `remediation` strings for all external plugins. Plugins should return plain text and must not attempt to include Markdown formatting (like `**bold**` or links), as it will be escaped and rendered literally. Core plugins (`trustbridge/*`) are trusted and may use Markdown formatting.

### 4. No runtime npm loading
Arbitrary npm packages are out of scope for v1.

### 5. Frozen workspace layout for optional plugins
Optional plugins are loaded from the workspace only, never from `node_modules` or remote URLs.

- Workspace root: `GITHUB_WORKSPACE`
- Example plugin path: `plugins/kyc.ts`
- Enable via action input: `trustbridge_plugins_path: plugins/kyc.ts`
- Example secret source: `process.env.KYC_API_KEY`

The loader rejects absolute paths and any path that escapes the workspace root.

---

## File map

```text
src/
  plugin.ts         - CheckPlugin, CheckPluginContext, CheckPluginResult, PluginRegistry
  pluginRunner.ts   - runPlugins(ctx, registry?) -> ValidationResult
  pluginLoader.ts   - workspace-only plugin loader with allowlist + path guards
  corePlugins.ts    - accountFundedPlugin, trustlinePlugin, xlmReservePlugin, homeDomainPlugin
__tests__/
  plugin.test.ts    - registry, runner, core plugins, security contract
  plugin-loader.test.ts - loader path guards, allowlist, load failures, duplicate ids
docs/
  PLUGIN_ARCHITECTURE.md - this document
```

---

[← Back to Architecture](ARCHITECTURE.md) · [← Back to README](../README.md)
