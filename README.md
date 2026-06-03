---
name: flue-framework
description: Flue Agent Framework v0.9.1+ — TypeScript harness-driven agent framework. Covers createAgent, init, defineTool, defineAgentProfile, routing, SSE streaming, production deployment, and all known pitfalls.
---

# Flue Framework Skill

Flue is a TypeScript framework for building AI agents using the harness-driven architecture. This skill provides comprehensive guidance for creating, developing, and deploying Flue agents (tested with v0.9.1).

## Quick Start

### 1. Initialize a Flue Project

```bash
npm install @flue/runtime valibot@^1.0.0
npm install --save-dev @flue/cli typescript
npx flue init --target node
```

### 2. Create Your First Agent

Create `src/agents/hello-world.ts`:

```typescript
import { createAgent } from '@flue/runtime';

export default createAgent(() => ({
  model: 'anthropic/claude-sonnet-4-6',
  instructions: 'Tell a funny hello world engineering joke.',
}));
```

### 3. Run the Agent

```bash
npx flue connect hello-world --target node --env .env
```

## Core Concepts

Agent equals Model plus Harness. The model generates responses while the harness provides sessions for conversation context, tools for custom functions, skills for reusable knowledge, and a sandbox for isolated execution.

## Two Agent Patterns (Critical)

Flue has **two distinct patterns** for creating agents. Using the wrong one causes errors.

### Pattern 1: Interactive Agent (for web chat, SSE, WebSocket)

Uses `createAgent()` directly. The agent is long-lived with sessions and routing.

```typescript
// src/agents/social-search.ts
import { createAgent, defineTool, Type, local } from '@flue/runtime';
import type { AgentRouteHandler, AgentWebSocketHandler } from '@flue/runtime';

const myTool = defineTool({
  name: 'my_tool',
  description: 'Does something useful',
  parameters: Type.Object({
    query: Type.String({ description: 'Search query' }),
  }),
  execute: async ({ query }) => {
    return JSON.stringify({ result: query });
  },
});

const agent = createAgent(() => ({
  model: 'deepseek/deepseek-v4-flash',
  instructions: 'You are a helpful assistant.',
  sandbox: local(),
  tools: [myTool],
}));

// These exports are REQUIRED for HTTP/WS access
export const route: AgentRouteHandler = agent.route;
export const websocket: AgentWebSocketHandler = agent.websocket;
export default agent;
```

### Pattern 2: Workflow Agent (for CLI `flue run`)

Uses a handler function with `FlueContext`. Must call `createAgent()` then `init(agent)`.

```typescript
// src/workflows/keyword-search.ts
import { createAgent, type FlueContext } from '@flue/runtime';
import * as v from 'valibot';

async function handler({ init, payload, env }: FlueContext) {
  const agent = createAgent(() => ({
    model: env.MODEL || 'deepseek/deepseek-v4-flash',
    instructions: 'Search for keywords and return structured results.',
  }));

  const harness = await init(agent);
  const session = await harness.session();

  const result = await session.prompt(
    `Search for: ${payload.keyword}`,
    { result: v.object({ items: v.array(v.string()) }) },
  );

  return result;
}

export default handler;
```

### ⚠️ CRITICAL: `init()` API

```typescript
// ❌ WRONG — init() does NOT accept a plain config object
const harness = await init({
  model: 'deepseek/deepseek-v4-flash',
  instructions: '...',
  providers: { deepseek: { apiKey: '...' } },
});

// ✅ CORRECT — init() requires a createAgent() result
const agent = createAgent(() => ({
  model: 'deepseek/deepseek-v4-flash',
  instructions: '...',
}));
const harness = await init(agent);

// ✅ init() second parameter is AgentHarnessOptions { name, tools, skills, subagents }
const harness = await init(agent, {
  tools: [myTool],
  subagents: [researcherProfile],
});
```

The `init()` function does NOT accept `providers`, `instructions`, `model`, or `sandbox` in its second parameter. Those belong in `createAgent()`.

## Provider Configuration

### Built-in Providers

| Provider ID | Environment Variable | Example Model Specifier |
|-------------|---------------------|------------------------|
| `anthropic` | `ANTHROPIC_API_KEY` | `anthropic/claude-sonnet-4-6` |
| `openai` | `OPENAI_API_KEY` | `openai/gpt-4.1` |
| `deepseek` | `DEEPSEEK_API_KEY` | `deepseek/deepseek-v4-flash` |
| `openrouter` | `OPENROUTER_API_KEY` | `openrouter/anthropic/claude-sonnet-4-6` |

### Using DeepSeek

DeepSeek is a **built-in provider**. Set the env var and use the model specifier — no provider config needed:

```env
# .env
DEEPSEEK_API_KEY=sk-your-key
MODEL=deepseek/deepseek-v4-flash
```

```typescript
// In createAgent — just use the model specifier
const agent = createAgent(() => ({
  model: env.MODEL || 'deepseek/deepseek-v4-flash',
  instructions: '...',
}));
```

### Overriding Provider Settings (baseUrl, apiKey)

Use `init({ providers: {...} })` ONLY when you need to override transport settings (custom baseUrl, different apiKey). The `providers` option goes in the **first argument** (`createAgent` config), NOT in `init()`:

```typescript
// ✅ Provider overrides go in createAgent
const agent = createAgent(() => ({
  model: 'anthropic/claude-haiku-4-5',
  instructions: '...',
  // providers config is NOT available here in current version
}));

// For DeepSeek via Anthropic-compatible endpoint, set env vars:
// ANTHROPIC_API_KEY=sk-your-deepseek-key
// ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic
```

### Third-Party OpenAI-Compatible Providers

Register in `src/app.ts` (NOT in agent files — build discards top-level side effects):

```typescript
import { registerProvider } from '@flue/runtime';

registerProvider('my-provider', {
  api: 'openai-completions',
  baseUrl: 'https://my-provider.example.com/v1',
  apiKey: process.env.MY_PROVIDER_API_KEY,
  models: { 'my-model': { maxTokens: 8192 } },
});
```

## Custom Tools with defineTool()

### ⚠️ Tool Parameters Use TypeBox, NOT Valibot

```typescript
import { defineTool, Type } from '@flue/runtime';

const myTool = defineTool({
  name: 'http_request',
  description: 'Make an HTTP request',
  parameters: Type.Object({
    method: Type.Union([Type.Literal('GET'), Type.Literal('POST')]),
    url: Type.String({ description: 'Target URL' }),
    body: Type.Optional(Type.String({ description: 'POST body as JSON string' })),
  }),
  execute: async ({ method, url, body }) => {
    // execute() must return Promise<string>
    const result = await fetch(url, { method, body });
    return JSON.stringify(await result.json());
  },
});
```

Key points:
- Import `Type` from `@flue/runtime` (TypeBox), not from valibot
- `execute()` must return `Promise<string>` (return JSON.stringify for objects)
- `execute()` runs in Node.js — you can use any Node.js API (https, child_process, fs, etc.)
- No need for `harness.env.exec()` inside `execute()` — use Node.js APIs directly

### Tools Can Use Node.js APIs Directly

```typescript
// ✅ Direct Node.js HTTP in defineTool — no sandbox needed
import https from 'node:https';

execute: async ({ path }) => {
  return new Promise((resolve) => {
    const req = https.request({ hostname: 'api.example.com', path }, (res) => {
      let data = '';
      res.on('data', (chunk) => (data += chunk));
      res.on('end', () => resolve(data));
    });
    req.end();
  });
}
```

## Sub-Agents with defineAgentProfile()

```typescript
import { defineAgentProfile } from '@flue/runtime';

export const researcher = defineAgentProfile({
  name: 'researcher',
  description: 'Deep data collection agent with pagination support',
  instructions: 'You are a data collection specialist...',
});
```

Use in agent:

```typescript
import { researcher } from '../shared/profiles/researcher';

const agent = createAgent(() => ({
  model: 'deepseek/deepseek-v4-flash',
  instructions: '...',
  subagents: [researcher],
}));
```

## Sandbox

### local() — Host Machine Access

```typescript
import { local } from '@flue/runtime/node';

const agent = createAgent(() => ({
  model: 'deepseek/deepseek-v4-flash',
  instructions: '...',
  sandbox: local(), // Access host filesystem, Python, curl, etc.
}));
```

Use `local()` when the agent needs to run shell commands, access host Python, or interact with the real filesystem. Import from `@flue/runtime/node`.

## Project Layout

Flue discovers agents and workflows from a source directory (priority: `.flue/` > `src/` > project root):

```
my-project/
├─ src/
│  ├─ agents/
│  │  └─ social-search.ts     ← agent "social-search"
│  ├─ workflows/
│  │  └─ keyword-search.ts    ← workflow "keyword-search"
│  ├─ shared/
│  │  ├─ tools/               ← defineTool modules
│  │  ├─ profiles/            ← defineAgentProfile modules
│  │  └─ constants.ts
│  └─ app.ts                  ← Entry point (Hono + flue routes)
├─ public/
│  └─ index.html              ← Web chat frontend
├─ .env
├─ package.json
└─ flue.config.ts
```

- `flue run <name>` → discovers from `agents/` only
- Workflows → invoked via HTTP `POST /workflows/<name>`
- `.flue/` takes priority over `src/` — if both exist, `src/` is ignored

## Application Entry Point (app.ts)

Custom routes can be added before mounting Flue:

```typescript
import { flue } from '@flue/runtime/routing';
import { Hono } from 'hono';
import { readFileSync } from 'node:fs';
import { resolve } from 'node:path';

const app = new Hono();

// Custom routes BEFORE flue()
app.get('/', (c) => {
  const html = readFileSync(resolve(process.cwd(), 'public', 'index.html'), 'utf-8');
  return c.html(html);
});

// Mount Flue agent routes
app.route('/', flue());

export default app;
```

## HTTP API & SSE Streaming

### Agent Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/agents/<name>/<session-id>` | POST | Send message (`{ "message": "..." }`) |
| `/agents/<name>/<session-id>` | GET | WebSocket upgrade |
| `/openapi.json` | GET | OpenAPI schema |

### SSE Streaming (Real-time Intermediate Events)

Add `Accept: text/event-stream` header to get streaming events:

```bash
curl -N -X POST http://localhost:3583/agents/social-search/my-session \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -d '{"message":"微博热搜Top3"}'
```

SSE events:

| Event | Description |
|-------|-------------|
| `thinking_start` / `thinking_delta` / `thinking_end` | Agent reasoning process |
| `tool_execution_start` | Tool starts (contains `toolName`, `tool.input`) |
| `tool_execution_end` | Tool finishes (contains result/output) |
| `tool_call` | Tool call summary (contains `durationMs`) |
| `text_delta` | Streaming text fragment of final response |
| `agent_end` / `operation` | Agent finishes (may contain `result.text`) |

SSE format: each line is `data: <JSON>\n\n`.

### Response Format (non-SSE)

```json
{
  "result": { "text": "## 微博热搜 Top 5\n..." },
  "usage": { "input": 9598, "output": 717, "totalTokens": 23627 },
  "model": { "provider": "deepseek", "id": "deepseek-v4-flash" }
}
```

## Session API

### session.prompt()

```typescript
// Without result → returns { text: string }
const response = await session.prompt('Hello');
console.log(response.text);

// With result → returns parsed valibot data DIRECTLY (not wrapped)
const result = await session.prompt('2+2?', {
  result: v.object({ answer: v.number() }),
});
console.log(result.answer); // Direct access, NOT result.data.answer
```

### DeepSeek Structured Output Hint

DeepSeek needs explicit JSON instructions in the prompt:

```typescript
// ✅ Include JSON format hint for DeepSeek
await session.prompt(
  'What is 2 + 2? Respond as JSON: {"answer": 4}.',
  { result: v.object({ answer: v.number() }) },
);
```

Add to agent instructions: `"When asked for structured data, always respond with valid JSON matching the requested schema."`

### session.task()

```typescript
// Delegate to sub-agent — also returns data directly with result schema
const data = await session.task('Research this topic', {
  agent: 'researcher',
  result: v.object({ findings: v.array(v.string()) }),
});
```

## Harness API

### ⚠️ harness.fs Does NOT Exist — Use harness.env

```typescript
// ❌ WRONG
await harness.fs.writeFile('/file.md', content);

// ✅ CORRECT
await harness.env.writeFile('/file.md', content);
await harness.env.readFile('/file.md');
await harness.env.exists('/file.md');
await harness.env.mkdir('/dir', { recursive: true });
await harness.env.readdir('/dir');
await harness.env.rm('/file');
await harness.env.cwd;
```

## Production Deployment

### Build & Start

```bash
npm run build   # Build to dist/
npm start       # Start production server
```

### Production Start Script

The production server needs env vars loaded and port set explicitly (defaults to 3000):

```json
{
  "scripts": {
    "start": "bash -c 'set -a && source .env && set +a && PORT=3583 node dist/server.mjs'"
  }
}
```

### Dev vs Production

| Mode | Command | Behavior |
|------|---------|----------|
| Dev | `flue dev --target node --env .env` | Watches ALL root files, hot reloads, **kills active connections** on any file change |
| Production | `npm start` | No hot reload, stable connections |

**⚠️ Dev server watches ALL files in project root** — changes to `.md`, `.txt`, `scripts/`, etc. trigger rebuilds and restart the server, dropping active SSE connections. Use production mode for stable serving.

## Dependency Requirements

| Dependency | Version | Why |
|-----------|---------|-----|
| Node.js | >= 22.18.0 | Engine requirement |
| valibot | ^1.0.0 | v0.x causes `TypeError: Cannot read properties of undefined (reading 'flatMap')` |

```bash
node --version    # Must be >= 22.18.0
npm ls valibot    # Must show 1.x.x
```

## Troubleshooting

### `init() requires an agent created with createAgent(...)`

**Cause:** Passing a plain config object to `init()` instead of a `createAgent()` result.

**Fix:** Always use `const agent = createAgent(() => ({...})); init(agent)`.

### `Invalid model "xxx". Use "provider/model-id" format`

**Fix:** Use `provider/model-id` format: `deepseek/deepseek-v4-flash` not `deepseek-v4-flash`.

### `403 Forbidden` with Anthropic provider + DeepSeek API

**Cause:** DeepSeek default endpoint is OpenAI-compatible, not Anthropic-compatible.

**Fix:** Use built-in `deepseek` provider: `model: 'deepseek/deepseek-v4-flash'`, or use `baseUrl: 'https://api.deepseek.com/anthropic'`.

### `registerProvider()` not working in agent files

**Cause:** Flue build extracts only the handler function from agents. Top-level side effects are discarded.

**Fix:** Place `registerProvider()` in `src/app.ts`.

### `Agent "xxx" not found` with `flue run`

**Cause:** File is in `workflows/`. `flue run` only discovers from `agents/`.

**Fix:** Move to `agents/` or use HTTP routes for workflows.

### Dev server keeps dropping connections

**Cause:** `flue dev` watches all root files and restarts on any change.

**Fix:** Use production mode (`npm start`) or remove non-essential files from project root.

## CLI Commands

| Command | Description |
|---------|-------------|
| `flue init` | Initialize project configuration |
| `flue dev` | Start dev server (hot reload, unstable for active connections) |
| `flue build` | Build to dist/ |
| `flue run <agent>` | Execute an agent (discovers from `agents/` only) |
| `flue connect <agent>` | Interactive agent session |

### flue run

```bash
npx flue run <agent-name> --target node --payload '<json>' --env ".env"
```

Note: `--id` flag is NOT supported in current version.

## Complete Example: Web Chat Agent

### Agent (src/agents/social-search.ts)

```typescript
import { createAgent, defineTool, Type, local } from '@flue/runtime';
import type { AgentRouteHandler, AgentWebSocketHandler } from '@flue/runtime';

const apiCall = defineTool({
  name: 'api_call',
  description: 'Call an external API',
  parameters: Type.Object({
    method: Type.Union([Type.Literal('GET'), Type.Literal('POST')]),
    path: Type.String({ description: 'API path' }),
    params: Type.Optional(Type.Record(Type.String(), Type.String())),
  }),
  execute: async ({ method, path, params }) => {
    // Direct Node.js https request — no Python/shell needed
    return JSON.stringify({ method, path, params });
  },
});

const agent = createAgent(() => ({
  model: 'deepseek/deepseek-v4-flash',
  instructions: `You are a social media search assistant.
Analyze user intent, call appropriate APIs, and return results.

When asked for structured data, always respond with valid JSON.`,
  sandbox: local(),
  tools: [apiCall],
}));

export const route: AgentRouteHandler = agent.route;
export const websocket: AgentWebSocketHandler = agent.websocket;
export default agent;
```

### Frontend SSE Consumer (excerpt)

```javascript
const res = await fetch(`/agents/social-search/${sessionId}`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Accept': 'text/event-stream',
  },
  body: JSON.stringify({ message: userInput }),
});

const reader = res.body.getReader();
const decoder = new TextDecoder();
let buf = '';

while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  buf += decoder.decode(value, { stream: true });
  const lines = buf.split('\n');
  buf = lines.pop();
  for (const line of lines) {
    if (line.startsWith('data: ')) {
      const evt = JSON.parse(line.slice(6));
      // evt.type: thinking_start, tool_execution_start, text_delta, etc.
      handleEvent(evt);
    }
  }
}
```

## Documentation Reference

### Core Docs
- [Quickstart](/docs/getting-started/quickstart/) | [Local Development](/docs/getting-started/local-development/)
- [Agents](/docs/guide/building-agents/) | [Workflows](/docs/guide/workflows/) | [Tools](/docs/guide/tools/)
- [Subagents](/docs/guide/subagents/) | [Routing](/docs/guide/routing/) | [Models](/docs/guide/models/)
- [Agent API](/docs/api/agent-api/) | [Routing API](/docs/api/routing-api/)

### Deployment
- [Node.js](/docs/ecosystem/deploy/node/) | [Cloudflare](/docs/ecosystem/deploy/cloudflare/) | [Render](/docs/ecosystem/deploy/render/)

## 特别感谢：
https://linux.do 佬友支持， https://liang.348349.xyz/ 更多agent项目
