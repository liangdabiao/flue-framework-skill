---
title: SDK API
description: Reference for consuming deployed Flue agents and workflows with @flue/sdk.
lastReviewedAt: 2026-06-02
---

The client SDK is exported from `@flue/sdk`. Use it from applications that consume deployed Flue agents and workflows.

```ts
import {
  FlueApiError,
  FlueSocketError,
  createFlueClient,
  type AgentInvokeOptions,
  type AgentManifestEntry,
  type AgentSocket,
  type AgentSocketEventContext,
  type AgentSocketEventListener,
  type AgentSocketInvokeResult,
  type AgentSocketPromptOptions,
  type AgentStreamInvokeOptions,
  type AgentSyncInvokeOptions,
  type AgentWebSocketClientMessage,
  type AgentWebSocketServerMessage,
  type AttachedAgentEvent,
  type AttachedAgentStreamError,
  type CreateFlueClientOptions,
  type DirectAgentPayload,
  type FlueClient,
  type FlueEvent,
  type FluePublicError,
  type ListResponse,
  type ListRunsOptions,
  type LlmAssistantMessage,
  type LlmImageContent,
  type LlmMessage,
  type LlmTextContent,
  type LlmThinkingContent,
  type LlmTool,
  type LlmToolCall,
  type LlmToolResultMessage,
  type LlmTurnPurpose,
  type LlmUserMessage,
  type RequestHeaders,
  type RunEventsOptions,
  type RunOwner,
  type RunPointer,
  type RunRecord,
  type RunStatus,
  type RunStreamOptions,
  type SocketEventContext,
  type SocketEventListener,
  type SocketInvokeResult,
  type WebSocketErrorMessage,
  type WebSocketFactory,
  type WebSocketLike,
  type WebSocketServerMessage,
  type WebSocketTarget,
  type WebSocketUrlTransform,
  type WorkflowRunWebSocketErrorMessage,
  type WorkflowSocket,
  type WorkflowSocketEventContext,
  type WorkflowSocketEventListener,
  type WorkflowSocketInvokeResult,
  type WorkflowWebSocketClientMessage,
  type WorkflowWebSocketServerMessage,
} from '@flue/sdk';
```

## `createFlueClient(...)`

```ts
function createFlueClient(options: CreateFlueClientOptions): FlueClient;
```

Creates a client for the public and read-only admin routes of a deployed Flue application.

```ts
const client = createFlueClient({
  baseUrl: 'https://example.com/api',
  token: process.env.FLUE_TOKEN,
});
```

### `CreateFlueClientOptions`

| Field           | Type                    | Default                        | Description                                                                                                |
| --------------- | ----------------------- | ------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| `baseUrl`       | `string`                | —                              | URL where the public `flue()` sub-app is mounted, including any pathname.                                  |
| `fetch`         | `typeof fetch`          | global `fetch`                 | Custom HTTP implementation.                                                                                |
| `headers`       | `RequestHeaders`        | —                              | Headers merged into each HTTP request.                                                                     |
| `token`         | `string`                | —                              | Bearer token added to HTTP requests.                                                                       |
| `adminBasePath` | `string`                | `'/admin'`                     | Origin-relative mount path for read-only admin routes.                                                     |
| `websocket`     | `WebSocketFactory`      | global `WebSocket` constructor | Custom WebSocket implementation.                                                                           |
| `websocketUrl`  | `WebSocketUrlTransform` | —                              | Transforms each WebSocket URL after HTTP protocol conversion, for example to add handshake authentication. |

### `RequestHeaders`

```ts
type RequestHeaders =
  | Record<string, string>
  | (() => Record<string, string> | Promise<Record<string, string>>);
```

Use a function to resolve headers separately for each HTTP request.

## Agents

Direct agent APIs interact with persistent agent instances. They use an agent name, instance id, and optional session name. They do not create workflow runs and do not emit `runId`.

### `client.agents.invoke(...)`

```ts
invoke(name: string, id: string, options: AgentSyncInvokeOptions): Promise<{ result: unknown }>;

invoke(name: string, id: string, options: AgentStreamInvokeOptions): AsyncIterable<AttachedAgentEvent>;
```

Sends one prompt to a persistent agent instance. Use `mode: 'sync'` for the terminal result or `mode: 'stream'` to consume attached-agent events. `AgentInvokeOptions` is the union of `AgentSyncInvokeOptions` and `AgentStreamInvokeOptions` for wrappers that forward either mode.

| Field     | Type                 | Default | Description                        |
| --------- | -------------------- | ------- | ---------------------------------- |
| `mode`    | `'sync' \| 'stream'` | —       | Select the response mode.          |
| `payload` | `DirectAgentPayload` | —       | Prompt payload.                    |
| `signal`  | `AbortSignal`        | —       | Cancel the in-flight HTTP request. |

#### `DirectAgentPayload`

| Field     | Type     | Default     | Description                        |
| --------- | -------- | ----------- | ---------------------------------- |
| `message` | `string` | —           | Prompt sent to the agent instance. |
| `session` | `string` | `'default'` | Session name.                      |

### `client.agents.connect(...)`

```ts
connect(name: string, id: string): AgentSocket;
```

Opens a reusable WebSocket connection to an agent instance.

#### `AgentSocket`

```ts
interface AgentSocket {
  readonly ready: Promise<void>;
  prompt(message: string, options?: AgentSocketPromptOptions): Promise<AgentSocketInvokeResult>;
  ping(): Promise<void>;
  onEvent(listener: AgentSocketEventListener): () => void;
  close(code?: number, reason?: string): void;
}
```

`ready` resolves after the server accepts the connection. Sequential `prompt()` calls may reuse the socket. `onEvent()` subscribes to prompt events and returns an unsubscribe function. `close()` rejects pending work.

#### `AgentSocketPromptOptions`

| Field     | Type     | Default     | Description   |
| --------- | -------- | ----------- | ------------- |
| `session` | `string` | `'default'` | Session name. |

#### `AgentSocketInvokeResult`

```ts
interface AgentSocketInvokeResult {
  result: unknown;
}
```

## Workflows

### `client.workflows.connect(...)`

```ts
connect(name: string): WorkflowSocket;
```

Opens a WebSocket connection for one workflow invocation.

#### `WorkflowSocket`

```ts
interface WorkflowSocket {
  readonly ready: Promise<void>;
  readonly runId: Promise<string>;
  invoke(payload?: unknown): Promise<WorkflowSocketInvokeResult>;
  onEvent(listener: WorkflowSocketEventListener): () => void;
  close(code?: number, reason?: string): void;
}
```

`ready` resolves after the server accepts the connection. A workflow socket accepts only one `invoke()` call. `runId` resolves after that invocation is admitted, before the terminal result arrives. Start the invocation before awaiting `runId`:

```ts
const workflow = client.workflows.connect('summarize');
await workflow.ready;

const completion = workflow.invoke({ text: 'Summarize me' });
const runId = await workflow.runId;

console.log('admitted run', runId);
console.log(await completion);
```

`runId` rejects if the socket closes or the invocation fails before admission. Once resolved, it remains available if the initiating socket later disconnects. `onEvent()` subscribes to workflow-run events and returns an unsubscribe function. `close()` rejects pending work.

#### `WorkflowSocketInvokeResult`

```ts
interface WorkflowSocketInvokeResult {
  result: unknown;
  runId: string;
}
```

## Workflow runs

Run APIs inspect workflow runs only. Direct agent prompts and dispatched agent inputs are not runs.

### `client.runs.get(...)`

```ts
get(runId: string): Promise<RunRecord>;
```

Retrieves one workflow-run record.

### `client.runs.events(...)`

```ts
events(runId: string, options?: RunEventsOptions): Promise<{ events: FlueEvent[] }>;
```

Retrieves recorded workflow-run events. `after` returns events strictly after one event index. `limit` defaults to `100` and accepts `1..1000`. Use `types` to select event types.

### `client.runs.stream(...)`

```ts
stream(runId: string, options?: RunStreamOptions): AsyncIterable<FlueEvent>;
```

Streams workflow-run events over server-sent events until `run_end`, cancellation, or an unrecoverable error. Interrupted streams resume after the latest received event index. A stream-infrastructure `event: error` frame carries `{ error: FluePublicError }`; the SDK rejects iteration with `error.message` rather than yielding the envelope as a workflow event.

#### `RunStreamOptions`

| Option           | Type          | Default | Description                                                |
| ---------------- | ------------- | ------- | ---------------------------------------------------------- |
| `lastEventId`    | `number`      | —       | Resume after this event index.                             |
| `signal`         | `AbortSignal` | —       | Stop consuming events when aborted.                        |
| `maxRetries`     | `number`      | `3`     | Maximum reconnection attempts after an interrupted stream. |
| `initialRetryMs` | `number`      | `250`   | Initial reconnection delay in milliseconds.                |

## Admin

Admin APIs use the origin-relative read-only mount path configured with `adminBasePath`, independently from the public `baseUrl` pathname. This option only tells the SDK where an already-mounted `admin()` sub-app lives. The application must [mount `admin()` explicitly and protect that mount with application-owned authorization](/docs/api/routing-api/#admin).

### `client.admin.agents.list()`

```ts
list(): Promise<{ items: AgentManifestEntry[] }>;
```

Lists all built agents and their transport metadata. This response is intentionally unpaginated.

### `client.admin.runs.list(...)`

```ts
list(options?: ListRunsOptions): Promise<ListResponse<RunPointer>>;
```

Lists workflow-run summaries. Direct agent interactions and dispatches are not included.

#### `ListRunsOptions`

| Field          | Type                                   | Default | Description                                |
| -------------- | -------------------------------------- | ------- | ------------------------------------------ |
| `cursor`       | `string`                               | —       | Resume after this pagination cursor.       |
| `limit`        | `number`                               | —       | Maximum runs to return; accepts `1..1000`. |
| `status`       | `'active' \| 'completed' \| 'errored'` | —       | Select workflow-run statuses.              |
| `workflowName` | `string`                               | —       | Select one workflow name.                  |

### `client.admin.runs.get(...)`

```ts
get(runId: string): Promise<RunRecord>;
```

Retrieves one workflow-run record from the admin mount path.

## WebSocket configuration

Agent and workflow socket URLs inherit the public mount pathname from `baseUrl`. HTTP URLs are converted to `ws:` or `wss:` URLs before applying `websocketUrl`.

`token` and `headers` apply only to HTTP requests. Browser socket authentication should use cookies or an application-designed URL transformation through `websocketUrl`. Node consumers that need implementation-specific handshake headers can supply a custom `websocket` factory.

### `WebSocketLike`

```ts
interface WebSocketLike {
  addEventListener(type: 'message' | 'close' | 'error', listener: (event: unknown) => void): void;
  send(data: string): void;
  close(code?: number, reason?: string): void;
}
```

Minimal socket interface required by the client SDK.

### WebSocket configuration types

| Type                    | Description                                                                  |
| ----------------------- | ---------------------------------------------------------------------------- |
| `WebSocketFactory`      | Creates a socket for a fully resolved WebSocket URL.                         |
| `WebSocketTarget`       | Identifies the agent or workflow route that a WebSocket URL will connect to. |
| `WebSocketUrlTransform` | Transforms a WebSocket URL before connection.                                |

### WebSocket listener types

| Type                          | Description                                                       |
| ----------------------------- | ----------------------------------------------------------------- |
| `AgentSocketEventListener`    | Receives direct-agent events and prompt correlation metadata.     |
| `WorkflowSocketEventListener` | Receives workflow-run events and invocation correlation metadata. |
| `SocketEventListener`         | Union of the agent and workflow listener types.                   |
| `AgentSocketEventContext`     | Contains the prompt `requestId`.                                  |
| `WorkflowSocketEventContext`  | Contains the invocation `requestId` and workflow `runId`.         |
| `SocketEventContext`          | Union of the agent and workflow event-context types.              |
| `SocketInvokeResult`          | Union of the agent and workflow invocation-result types.          |

## Events and records

### `FlueEvent`

`FlueEvent` is the observable workflow-run event union. It includes run lifecycle, agent lifecycle, model turn, message, tool, task, compaction, operation, log, and idle events. Workflow-run events may include correlation fields such as `runId`, `instanceId`, `session`, `harness`, `operationId`, `turnId`, and `eventIndex`.

### `AttachedAgentEvent`

`AttachedAgentEvent` is emitted by direct interactions with persistent agent instances. It excludes workflow-run lifecycle events, requires `instanceId`, and does not include `runId`.

### Run and discovery types

| Type                 | Description                                                                                     |
| -------------------- | ----------------------------------------------------------------------------------------------- |
| `RunOwner`           | Workflow identity recorded for a run.                                                           |
| `RunRecord`          | Persisted workflow-run record, including status, timestamps, payload, result, and error fields. |
| `RunPointer`         | Workflow-run summary returned by admin listing routes.                                          |
| `RunStatus`          | Workflow-run status: `'active'`, `'completed'`, or `'errored'`.                                 |
| `AgentManifestEntry` | Agent discovery metadata returned by the read-only admin route.                                 |
| `ListResponse<T>`    | Cursor-paginated response with `items` and optional `nextCursor`.                               |

### Normalized model-turn types

`turn_request` and `turn` events expose normalized model data through these exported types:

| Type                   | Description                                                              |
| ---------------------- | ------------------------------------------------------------------------ |
| `LlmMessage`           | Union of normalized user, assistant, and tool-result messages.           |
| `LlmUserMessage`       | Normalized user message.                                                 |
| `LlmAssistantMessage`  | Normalized assistant message.                                            |
| `LlmToolResultMessage` | Normalized tool-result message.                                          |
| `LlmTextContent`       | Text content.                                                            |
| `LlmThinkingContent`   | Reasoning content.                                                       |
| `LlmImageContent`      | Image content.                                                           |
| `LlmToolCall`          | Tool call content.                                                       |
| `LlmTool`              | Tool definition.                                                         |
| `LlmTurnPurpose`       | Model-turn purpose: `'agent'`, `'compaction'`, or `'compaction_prefix'`. |

## Errors

See [Errors Reference](/docs/api/errors-reference/) for shared transport envelopes and stable public error categories.

### `FlueApiError`

```ts
class FlueApiError extends Error {
  readonly status: number;
  readonly body: unknown;
}
```

Failed SDK HTTP JSON request. `status` is the HTTP response status. `body` is the parsed response body when available, or the response text otherwise. Framework-owned routes normally return `{ error: FluePublicError }`; application-owned middleware may return arbitrary bodies.

### `FlueSocketError`

```ts
class FlueSocketError extends Error {
  readonly error: FluePublicError;
  readonly requestId: string | undefined;
  readonly runId: string | undefined;
}
```

Structured server error received over a WebSocket connection. An operation-scoped error rejects the matching request. An unscoped socket error closes the connection and rejects pending work.

### `FluePublicError`

```ts
interface FluePublicError {
  type: string;
  message: string;
  details: string;
  dev?: string;
  meta?: Record<string, unknown>;
}
```

Structured server error data used by socket errors and protocol messages.

### `AttachedAgentStreamError`

```ts
interface AttachedAgentStreamError {
  type: 'error';
  instanceId: string;
  error: FluePublicError;
}
```

Structured error envelope received while streaming a direct agent interaction. The stream throws `error.message` rather than yielding this envelope.

## Low-level WebSocket protocol

Most consumers should use `AgentSocket` and `WorkflowSocket`. Low-level protocol consumers can use the exported message types:

| Type                               | Description                                                 |
| ---------------------------------- | ----------------------------------------------------------- |
| `AgentWebSocketClientMessage`      | Messages sent over an agent WebSocket.                      |
| `AgentWebSocketServerMessage`      | Messages received from an agent WebSocket.                  |
| `WorkflowWebSocketClientMessage`   | Message sent over a workflow WebSocket.                     |
| `WorkflowWebSocketServerMessage`   | Messages received from a workflow WebSocket.                |
| `WebSocketServerMessage`           | Union of agent and workflow server messages.                |
| `WebSocketErrorMessage`            | Connection- or request-scoped socket error message.         |
| `WorkflowRunWebSocketErrorMessage` | Workflow-run-scoped socket failure after run id allocation. |
