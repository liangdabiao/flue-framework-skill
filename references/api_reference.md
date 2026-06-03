# Flue Agent API Reference

## Core Exports

```typescript
import {
  createAgent,
  defineAgentProfile,
  defineTool,
  connectMcpServer,
  dispatch,
  type FlueContext,
  type FlueHarness,
  type FlueSession,
} from '@flue/runtime';
```

## createAgent()

Creates an agent initializer:

```typescript
function createAgent<TPayload = unknown, TEnv = Record<string, any>>(
  initialize: (context: AgentCreateContext<TPayload, TEnv>) => AgentRuntimeConfig | Promise<AgentRuntimeConfig>,
): CreatedAgent<TPayload, TEnv>;
```

### AgentCreateContext

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Agent instance id |
| `env` | TEnv | Platform environment bindings |
| `payload` | TPayload \| undefined | Workflow payload |

### AgentRuntimeConfig

| Field | Type | Description |
|-------|------|-------------|
| `model` | string \| false | Model specifier (e.g., 'anthropic/claude-sonnet-4-6') |
| `instructions` | string | Agent instructions |
| `skills` | Skill[] | Registered skills |
| `tools` | ToolDefinition[] | Custom tools |
| `subagents` | AgentProfile[] | Named subagent profiles |
| `cwd` | string | Working directory |
| `sandbox` | false \| SandboxFactory | Sandbox configuration |
| `persist` | SessionStore | Conversation state store |

## defineAgentProfile()

Validates and returns a reusable agent profile:

```typescript
function defineAgentProfile(profile: AgentProfile): AgentProfile;
```

### AgentProfile

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Profile name for task() selection |
| `model` | string \| false | Default model |
| `instructions` | string | Instructions |
| `skills` | Skill[] | Registered skills |
| `tools` | ToolDefinition[] | Custom tools |
| `subagents` | AgentProfile[] | Named subagents |
| `thinkingLevel` | ThinkingLevel | Reasoning effort level |
| `compaction` | false \| CompactionConfig | Conversation compaction |

## defineTool()

Validates a custom model-callable tool:

```typescript
function defineTool<TParams extends ToolParameters>(
  tool: ToolDefinition<TParams>,
): ToolDefinition<TParams>;
```

### ToolDefinition

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Unique tool name |
| `description` | string | Tool usage description |
| `parameters` | ToolParameters | JSON Schema-compatible schema |
| `execute` | Function | Execution handler |

## connectMcpServer()

Connects to a remote MCP server:

```typescript
function connectMcpServer(name: string, options: McpServerOptions): Promise<McpServerConnection>;
```

### McpServerOptions

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `url` | string \| URL | — | MCP server endpoint |
| `transport` | 'streamable-http' \| 'sse' | 'streamable-http' | Transport type |
| `headers` | HeadersInit | — | Request headers |

## FlueContext

The context object passed to agent handlers:

```typescript
interface FlueContext<TPayload = any, TEnv = Record<string, any>> {
  readonly id: string;
  readonly payload: TPayload;
  readonly env: TEnv;
  readonly req: Request | undefined;
  readonly log: FlueLogger;
  init(agent: CreatedAgent<TPayload, TEnv>, options?: AgentHarnessOptions): Promise<FlueHarness>;
}
```

### AgentHarnessOptions (init() options)

| Field | Type | Description |
|-------|------|-------------|
| `model` | string \| false | **Required.** Model specifier (e.g., `'deepseek/deepseek-v4-flash'`) |
| `instructions` | string | Agent system prompt |
| `providers` | ProvidersConfig | Override provider transport settings (baseUrl, apiKey, headers) |
| `id` | string | Agent instance ID override |
| `role` | string | Role name for multi-role agents |
| `sandbox` | false \| SandboxFactory \| 'local' \| 'empty' | Sandbox configuration |
| `persist` | SessionStore | Session persistence store |
| `cwd` | string | Working directory for the agent |
| `skills` | Skill[] | Skills to register |
| `tools` | ToolDefinition[] | Custom tools |
| `commands` | Command[] | Shell commands |

### ProvidersConfig

Override transport settings for providers at runtime. This is the **recommended way** to configure provider API keys and custom endpoints from within agent handlers:

```typescript
interface ProvidersConfig {
  [providerId: string]: {
    baseUrl?: string;
    apiKey?: string;
    headers?: Record<string, string>;
  };
}
```

**Example — Using DeepSeek with built-in provider:**

```typescript
const harness = await init({
  model: 'deepseek/deepseek-v4-flash',
  providers: {
    deepseek: {
      apiKey: env.DEEPSEEK_API_KEY,
    },
  },
});
```

**Example — Using Anthropic-compatible endpoint (e.g., DeepSeek):**

```typescript
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

### ⚠️ Important: Provider Registration vs. Provider Configuration

- **`registerProvider()`** / **`configureProvider()`** must be called in `src/app.ts`, NOT in agent files. The Flue build process only extracts the handler function — top-level side-effect calls in agent files are discarded.
- **`init({ providers: {...} })`** is the correct way to configure provider settings from within an agent handler. These settings override the base provider configuration for that agent instance at runtime.
- `init({ providers })` can only override transport settings (baseUrl, apiKey, headers) for providers that are already built-in or registered. It cannot register new provider IDs.

## FlueHarness

Initialized agent environment:

```typescript
interface FlueHarness {
  readonly name: string;
  session(name?: string): Promise<FlueSession>;
  readonly sessions: FlueSessions;
  shell(command: string, options?: ShellOptions): CallHandle<ShellResult>;
  readonly fs: FlueFs;
}
```

## FlueSession

Named conversation state:

```typescript
interface FlueSession {
  readonly name: string;
  prompt(text: string, options?: PromptOptions): CallHandle<PromptResponse>;
  skill(skill: SkillReference | string, options?: SkillOptions): CallHandle<PromptResponse>;
  task(text: string, options?: TaskOptions): CallHandle<PromptResponse>;
  shell(command: string, options?: ShellOptions): CallHandle<ShellResult>;
  readonly fs: FlueFs;
  compact(): Promise<void>;
  delete(): Promise<void>;
}
```

### PromptOptions

| Field | Type | Description |
|-------|------|-------------|
| `result` | Valibot schema | Structured output validation |
| `tools` | ToolDefinition[] | Additional tools |
| `model` | string | Model override |
| `thinkingLevel` | ThinkingLevel | Reasoning effort |
| `signal` | AbortSignal | Cancellation signal |
| `images` | PromptImage[] | Vision input |

## dispatch()

Asynchronous agent delivery:

```typescript
function dispatch(agent: CreatedAgent, request: AgentDispatchRequest): Promise<DispatchReceipt>;
```

## Error Types

| Type | Description |
|------|-------------|
| `ResultUnavailableError` | When agent cannot produce validated data |
