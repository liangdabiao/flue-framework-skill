---
title: Sandbox Connector API
description: Adapt a provider sandbox SDK into Flue's public sandbox contract.
---

A sandbox connector adapts a third-party sandbox provider's SDK into Flue's `SandboxFactory` interface so that agents can run shell commands and read or write files inside that sandbox.

If you are a coding agent building a connector for a user, follow this document literally and produce one TypeScript file that exports a factory function such as `daytona(...)` returning a `SandboxFactory`.

## High-level shape

A connector is one TypeScript file. It exports a factory function that takes an already-initialized provider sandbox plus options and returns a `SandboxFactory`. Flue calls `factory.createSessionEnv({ id })` once per initialized harness and uses the returned `SessionEnv` for all shell and file operations.

```ts
// <source-dir>/connectors/<provider>.ts
import { createSandboxSessionEnv } from '@flue/runtime';
import type { SandboxApi, SandboxFactory, SessionEnv, FileStat } from '@flue/runtime';
import type { Sandbox as ProviderSandbox } from '<provider-sdk>';

class ProviderSandboxApi implements SandboxApi {
  constructor(private sandbox: ProviderSandbox) {}
  // Implement every method on SandboxApi.
}

export function provider(sandbox: ProviderSandbox): SandboxFactory {
  return {
    async createSessionEnv(): Promise<SessionEnv> {
      const sandboxCwd = '/workspace';
      const api = new ProviderSandboxApi(sandbox);
      return createSandboxSessionEnv(api, sandboxCwd);
    },
  };
}
```

Connectors are pure adapters. They map a provider sandbox to a `SessionEnv` rooted at the provider-owned base cwd and stop there. They must not apply a created agent's `cwd`: Flue resolves that value once against the connector's base cwd during `init()`. Connectors do not manage the sandbox's lifetime. The user owns what they create.

## Imports

Import these from `@flue/runtime`:

- `createSandboxSessionEnv(api, cwd)` wraps your `SandboxApi` into a `SessionEnv` that Flue can drive. Pass the provider-owned base cwd, not a created agent's cwd.
- `SandboxApi` is the interface you implement.
- `SandboxFactory` is what your factory returns.
- `SessionToolFactory` is the optional model-facing tool factory type for a custom sandbox.
- `SessionEnv` is what `createSandboxSessionEnv` returns. Do not construct one yourself.
- `FileStat` is the return type for `stat()`.

Do not import internal runtime paths. `@flue/runtime` is the public surface for connector authors.

## TypeScript contracts

Always typecheck against the real types from `@flue/runtime`. If this page drifts from the runtime package, the runtime package wins.

### `SandboxApi`

```ts
export interface SandboxApi {
  readFile(path: string): Promise<string>;
  readFileBuffer(path: string): Promise<Uint8Array>;
  writeFile(path: string, content: string | Uint8Array): Promise<void>;
  stat(path: string): Promise<FileStat>;
  readdir(path: string): Promise<string[]>;
  exists(path: string): Promise<boolean>;
  mkdir(path: string, options?: { recursive?: boolean }): Promise<void>;
  rm(path: string, options?: { recursive?: boolean; force?: boolean }): Promise<void>;
  exec(
    command: string,
    options?: {
      cwd?: string;
      env?: Record<string, string>;
      timeout?: number;
      signal?: AbortSignal;
    },
  ): Promise<{ stdout: string; stderr: string; exitCode: number }>;
}
```

`timeout` is the primary cancellation contract. Every connector should honor it by forwarding to the provider SDK's native timeout option. `signal` is optional: connectors whose provider SDK supports mid-flight cancellation should forward it; others may ignore it.

### `SandboxFactory`

```ts
export interface SandboxFactory {
  createSessionEnv(options: { id: string }): Promise<SessionEnv>;
  tools?: SessionToolFactory;
}
```

`tools` replaces the framework's default model-facing tool list for this sandbox. Omit it for the standard filesystem and shell tools.

### `SessionToolFactory`

```ts
export type SessionToolFactory = (
  env: SessionEnv,
  options: { subagents: Record<string, AgentProfile> },
) => AgentTool<any>[];
```

Use this optional factory when the sandbox exposes provider-specific model-facing tools. Flue appends the `task` tool separately.

### `FileStat`

```ts
export interface FileStat {
  isFile: boolean;
  isDirectory: boolean;
  isSymbolicLink: boolean;
  size: number;
  mtime: Date;
}
```

### `SessionEnv`

Return a `SessionEnv` from `createSessionEnv`, but get it from `createSandboxSessionEnv(api, cwd)`. Do not write `SessionEnv` methods by hand in a connector.

## Required `SandboxApi` methods

Implement every method below. If your provider SDK does not have a direct analogue for an operation, fall back to shell commands through `exec()`. The Daytona connector does this for `mkdir -p`, for example.

### `readFile(path)`

UTF-8 decode the file at `path` and return its contents.

### `readFileBuffer(path)`

Return raw bytes as a `Uint8Array`. If the SDK gives you a Node `Buffer`, wrap it with `new Uint8Array(buffer)`.

### `writeFile(path, content)`

Write `content` to `path`. Accept both `string` and `Uint8Array`. Convert strings to UTF-8 bytes before sending them to providers that only accept buffers.

### `stat(path)`

Return a `FileStat`. If the provider SDK does not expose modification time or size, use sensible defaults such as `new Date()` and `0`. Use `isSymbolicLink: false` if the SDK does not expose symlink information.

### `readdir(path)`

Return names, not full paths, for entries in the directory.

### `exists(path)`

Return `true` when the path exists. Most providers throw for missing paths, so catch that error and return `false`.

### `mkdir(path, options?)`

Create a directory. If `options.recursive` is set, create parents as needed. If the provider SDK only supports a single-level operation, fall back to `exec('mkdir -p ...')` for the recursive case.

### `rm(path, options?)`

Delete a file or directory. Honor `options.recursive` and `options.force`.

### `exec(command, options?)`

Run a shell command. Honor `options.cwd`, `options.env`, and `options.timeout`. The `timeout` hint is measured in seconds. If the provider SDK does not expose a native timeout option, translate it into `AbortSignal.timeout(options.timeout * 1000)` and pass that signal to an SDK that accepts one, or as a last resort race the call against a timer and reject. Make a best-effort attempt to honor `timeout`: it is how the model-facing bash tool stops a command and retries. Returning an exit-code-124 result with timeout details in `stderr` matches the convention used by other connectors and `timeout(1)`.

If the provider SDK also supports an `AbortSignal`, forward `options.signal` for true mid-flight cancellation. If it cannot observe a signal, ignore that option. The `createSandboxSessionEnv` wrapper performs pre- and post-operation `signal.aborted` checks. Do not fake mid-flight signal cancellation with `Promise.race`: the underlying remote process would keep running.

If the provider does not separately expose `stderr`, return `''`. Default `exitCode` to `0` only when the call clearly succeeded.

## Sandbox lifetime

Flue does not manage sandbox lifetime. The user creates the sandbox and decides when or whether to delete it. Connectors must not call `sandbox.delete()`, `sandbox.terminate()`, `sandbox.kill()`, or any equivalent on the user's behalf.

Connector factories therefore take no `cleanup` option, and `createSandboxSessionEnv` takes no cleanup callback. If the connector opens a real socket such as SSH, it may manage that socket internally, but it must not assume Flue will trigger teardown.

## Reference implementation

See the deployed [Daytona connector](https://flueframework.com/cli/connectors/daytona.md) for a complete implementation. It demonstrates shell fallback for recursive mkdir, `exists()` error handling, and buffer or string conversion in `writeFile()`.

## Connector file location

The user's project root does not change. The selected source directory inside it may vary. Flue selects the first existing directory in this order:

1. `<root>/.flue/`
2. `<root>/src/`
3. `<root>/`

Write the connector to `<source-dir>/connectors/<name>.ts`. If the selected source directory is unclear, ask the user before writing.

## Verify a generated connector

Before finishing:

1. Typecheck the file with `npx tsc --noEmit` or the project's existing typecheck command.
2. Confirm that the connector imports from `@flue/runtime`.
3. If the project does not depend on the provider SDK, tell the user to install it.
4. Tell the user which environment variables they need to set.
5. Show a minimal snippet wiring the connector into an agent.
