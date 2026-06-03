# Flue SDK Reference

## Installation

```bash
npm install @flue/sdk
```

## createFlueClient()

Create a client for deployed Flue agents:

```typescript
import { createFlueClient } from '@flue/sdk';

const client = createFlueClient({
  baseUrl: 'https://example.com/api',
  token: process.env.FLUE_TOKEN,
});
```

### CreateFlueClientOptions

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `baseUrl` | string | — | API base URL |
| `fetch` | typeof fetch | global fetch | Custom HTTP implementation |
| `headers` | RequestHeaders | — | Default headers |
| `token` | string | — | Bearer token |
| `adminBasePath` | string | '/admin' | Admin routes path |

## Agent APIs

### client.agents.invoke()

Send prompt to agent instance:

```typescript
// Sync mode
const { result } = await client.agents.invoke('my-agent', 'instance-id', {
  mode: 'sync',
  payload: { message: 'Hello!' },
});

// Stream mode
for await (const event of client.agents.invoke('my-agent', 'instance-id', {
  mode: 'stream',
  payload: { message: 'Hello!' },
})) {
  console.log(event);
}
```

### client.agents.connect()

Open WebSocket connection:

```typescript
const socket = client.agents.connect('my-agent', 'instance-id');
await socket.ready;

const { result } = await socket.prompt('Hello!', { session: 'default' });
socket.close();
```

## Workflow APIs

### client.workflows.connect()

Open workflow WebSocket:

```typescript
const workflow = client.workflows.connect('summarize');
await workflow.ready;

const completion = workflow.invoke({ text: 'Summarize this...' });
const runId = await workflow.runId;

console.log(await completion);
```

## Run APIs

### client.runs.get()

Retrieve run record:

```typescript
const record = await client.runs.get('run-id-123');
```

### client.runs.events()

Retrieve run events:

```typescript
const { events } = await client.runs.events('run-id-123', {
  limit: 100,
  types: ['prompt', 'tool_call'],
});
```

### client.runs.stream()

Stream run events:

```typescript
for await (const event of client.runs.stream('run-id-123')) {
  console.log(event);
}
```

## Admin APIs

### client.admin.agents.list()

List available agents:

```typescript
const { items } = await client.admin.agents.list();
```

### client.admin.runs.list()

List workflow runs:

```typescript
const { items } = await client.admin.runs.list({
  status: 'completed',
  limit: 50,
});
```

## Error Handling

### FlueApiError

```typescript
try {
  await client.agents.invoke('agent', 'id', { mode: 'sync', payload: {} });
} catch (error) {
  if (error instanceof FlueApiError) {
    console.error(error.status, error.body);
  }
}
```

### FlueSocketError

```typescript
try {
  const socket = client.agents.connect('agent', 'id');
  await socket.prompt('Hello');
} catch (error) {
  if (error instanceof FlueSocketError) {
    console.error(error.error.message);
  }
}
```
