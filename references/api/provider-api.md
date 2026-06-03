---
title: Provider API
description: Configure built-in model providers and register custom provider integrations.
lastReviewedAt: 2026-05-31
---

The provider API configures model connection paths at runtime. Import ordinary provider APIs from `@flue/runtime`. For model selection, authentication setup, and Workers AI examples, see [Models & Providers](/docs/guide/models/).

## Imports

```ts
import {
  configureProvider,
  registerApiProvider,
  registerProvider,
  type HttpProviderRegistration,
  type ProviderConfiguration,
  type ProviderRegistration,
} from '@flue/runtime';
```

## `configureProvider()`

```ts
function configureProvider(providerId: string, settings: ProviderConfiguration): void;
```

Configures transport-level settings for an existing built-in or registered provider while preserving its resolved model metadata. The provider ID is the prefix used in model specifiers, such as `anthropic` in `anthropic/claude-sonnet-4-6`.

Repeated calls for the same provider ID replace the previous settings object.

### `ProviderConfiguration`

```ts
interface ProviderConfiguration {
  baseUrl?: string;
  headers?: Record<string, string>;
  apiKey?: string;
  storeResponses?: boolean;
}
```

| Property         | Purpose                                                                                                                           |
| ---------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `baseUrl`        | Override the provider endpoint.                                                                                                   |
| `headers`        | Merge headers into the resolved model's provider-level headers.                                                                   |
| `apiKey`         | Override the API key returned to the underlying model runtime.                                                                    |
| `storeResponses` | Send `store: true` for OpenAI Responses API providers. Enable only when your application accepts the provider's retention policy. |

### ⚠️ Where to call configureProvider()

**Must be called in `src/app.ts`**, NOT in agent files under `.flue/agents/`. The Flue build process only extracts the handler function from agent files — top-level side-effect calls are discarded from the build output.

For configuring providers from within agent handlers, use `init({ providers: {...} })` instead (see below).

### Runtime provider configuration with init()

The `init()` function in `FlueContext` accepts a `providers` option that overrides transport settings for any built-in or registered provider at runtime. This is the **recommended approach** for configuring API keys and custom endpoints from within agent handlers:

```ts
async function handler({ init, payload, env }: FlueContext) {
  const harness = await init({
    model: 'deepseek/deepseek-v4-flash',
    providers: {
      deepseek: {
        apiKey: env.DEEPSEEK_API_KEY,
      },
    },
  });
  // ...
}
```

The `providers` option in `init()` uses the same `ProviderConfiguration` shape per provider ID, with one key difference: it is scoped to the agent instance and does not affect global provider state.

**Merging behavior:** Settings from `init({ providers })` are merged with the base provider configuration (from `configureProvider()` or built-in defaults). `baseUrl` and `apiKey` override provider defaults; `headers` are merged with existing headers.

### Built-in DeepSeek provider

DeepSeek is a built-in provider in pi-ai with models `deepseek-v4-flash` and `deepseek-v4-pro`. It uses the `openai-completions` API protocol and connects to `https://api.deepseek.com` by default.

```ts
// In agent handler — pass API key via init()
const harness = await init({
  model: 'deepseek/deepseek-v4-flash',
  providers: {
    deepseek: {
      apiKey: env.DEEPSEEK_API_KEY,
    },
  },
});
```

DeepSeek also provides an Anthropic-compatible endpoint. Use the `anthropic` provider with a custom `baseUrl`:

```ts
const harness = await init({
  model: 'anthropic/claude-haiku-4-5',
  providers: {
    anthropic: {
      baseUrl: 'https://api.deepseek.com/anthropic',
      apiKey: env.ANTHROPIC_API_KEY,
    },
  },
});
```

## `registerProvider()`

```ts
function registerProvider(providerId: string, registration: ProviderRegistration): void;
```

Registers a model provider keyed by the provider ID used in model specifiers. Re-registering the same provider ID replaces its previous registration.

For example, registering `ollama` makes model specifiers such as `ollama/llama3.1:8b` available to agents and operations.

```ts
registerProvider('ollama', {
  api: 'openai-completions',
  baseUrl: 'http://localhost:11434/v1',
});
```

### `ProviderRegistration`

```ts
type ProviderRegistration = HttpProviderRegistration | CloudflareAIBindingRegistration;
```

Use an HTTP registration for ordinary URL-backed providers. Workers AI binding registrations are Cloudflare-specific and described below.

### `HttpProviderRegistration`

```ts
interface HttpProviderRegistration {
  api: Api;
  baseUrl: string;
  apiKey?: string;
  headers?: Record<string, string>;
  contextWindow?: number;
  maxTokens?: number;
  models?: Record<
    string,
    {
      contextWindow?: number;
      maxTokens?: number;
    }
  >;
}
```

| Property        | Purpose                                                                                                                   |
| --------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `api`           | Wire protocol used for requests. Use a Pi-provided API slug or register one with `registerApiProvider()`.                 |
| `baseUrl`       | Endpoint root, such as `https://api.anthropic.com/v1`.                                                                    |
| `apiKey`        | Optional API key. When omitted, the underlying provider integration may use its normal environment-variable lookup.       |
| `headers`       | Default headers for outgoing requests.                                                                                    |
| `contextWindow` | Default context-window size for models resolved through this registration. Defaults to `0`, meaning unknown.              |
| `maxTokens`     | Default output-token limit for models resolved through this registration. Defaults to `0`.                                |
| `models`        | Per-model `contextWindow` and `maxTokens` overrides keyed by model ID. Per-model values override provider-level defaults. |

## `registerApiProvider()`

```ts
const registerApiProvider: typeof import('@earendil-works/pi-ai').registerApiProvider;
```

Registers a wire-protocol handler for an API slug not shipped by Pi. Register the protocol first, then pass its `api` slug to `registerProvider()`.

Pi's API-provider registry is module-scoped and last-write-wins. Registering the same API slug again replaces the previous handler.

## Cloudflare binding registrations

Import Workers AI binding registration types from `@flue/runtime/cloudflare`:

```ts
import {
  type CloudflareAIBinding,
  type CloudflareAIBindingRegistration,
  type CloudflareGatewayOptions,
} from '@flue/runtime/cloudflare';
```

`CloudflareAIBindingRegistration` registers a provider backed by an `env.AI` Workers AI binding instead of an HTTP endpoint. Its optional `gateway` setting forwards AI Gateway options to each `env.AI.run(...)` call; set `gateway: false` to omit the gateway option.

Cloudflare builds register the `cloudflare` provider ID automatically unless `app.ts` registers it first. Register that provider ID in `app.ts` when you intentionally want an authored binding registration to take precedence over the generated default. See [Cloudflare Workers AI](/docs/guide/models/#cloudflare-workers-ai-cloudflare-only) for setup and gateway examples.
