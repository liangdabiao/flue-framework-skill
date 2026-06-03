---
title: Observability
description: Inspect workflow runs, monitor agent activity, and export telemetry from your application.
lastReviewedAt: 2026-05-30
---

Observability helps you understand whether Flue work completed, failed, became slow, or used more model resources than expected. Inspect workflow run history for bounded jobs, and use `observe(...)` to monitor workflows and continuing agents across your application.

## Inspect workflow runs

Each workflow invocation has a `runId`. Its run history records the completed result or error and the observable activity produced while the workflow executes.

Use the workflow context's `log` methods to record application-specific facts that runtime activity alone cannot explain. For example, a summarization workflow can report the size of the accepted document and the usage of the completed operation:

```ts title="src/workflows/summarize.ts"
import { createAgent, type FlueContext } from '@flue/runtime';

const summarizer = createAgent(() => ({
  model: 'anthropic/claude-haiku-4-5',
  instructions: 'Summarize the supplied document clearly and concisely.',
}));

export async function run({ init, log, payload }: FlueContext<{ text: string }>) {
  log.info('Summarization requested', { characters: payload.text.length });

  const harness = await init(summarizer);
  const session = await harness.session();
  const response = await session.prompt(payload.text);

  log.info('Summarization completed', {
    tokens: response.usage.totalTokens,
    cost: response.usage.cost.total,
  });

  return { summary: response.text };
}
```

`log.info(...)`, `log.warn(...)`, and `log.error(...)` accept structured attributes. Use attributes for values that you may later search, aggregate, or forward to a monitoring system.

When a workflow invoked through a running application reports its `runId`, use that identifier to inspect the workflow run from the command line:

```bash
pnpm exec flue logs <runId> --server http://localhost:3583
```

`flue logs` applies only to workflows. A direct prompt to an agent, or input accepted through `dispatch(...)`, is work in a continuing agent session rather than a workflow run.

## Observe application activity

Register `observe(...)` in your application entrypoint when you need telemetry across workflows and continuing agents. The observer receives activity handled by that running application context, including operations triggered by asynchronously dispatched input.

```ts title="src/app.ts"
import { observe } from '@flue/runtime';
import { flue } from '@flue/runtime/routing';
import { Hono } from 'hono';

observe((event) => {
  if (event.type === 'run_end' && event.isError) {
    console.error('Workflow failed', event.runId, event.error);
  }

  if (event.type === 'operation' && event.durationMs > 5_000) {
    console.warn('Slow operation', event.operationKind, event.durationMs);
  }

  if (event.type === 'log' && event.level === 'error') {
    console.error(event.message, event.attributes);
  }
});

const app = new Hono();
app.route('/', flue());

export default app;
```

An operation is the useful finite boundary for agent activity, such as prompting a session, running a skill, or delegating work. Direct and dispatched agent input can therefore be monitored without treating a continuing agent as a series of workflow runs.

When an operation is slow or unexpectedly expensive, its nested activity can provide the explanation. One prompt operation may include multiple model turns or tool calls. Model turns expose latency, token usage, and cost; tool activity shows where the agent spent time or encountered an error.

Callbacks registered with `observe(...)` are invoked while Flue emits activity and receive isolated JSON snapshots. Keep them lightweight: filter events, record metrics, or enqueue exporter work rather than performing blocking work in the callback. Returned promises are observed for rejection but are not awaited. In a distributed deployment, each running application context observes the activity it handles; send telemetry to an external backend if it needs to be aggregated across instances.

## Export telemetry safely

If your application already uses OpenTelemetry, register Flue's observer adapter in `src/app.ts`:

```ts title="src/app.ts"
import { createOpenTelemetryObserver } from '@flue/opentelemetry';
import { observe } from '@flue/runtime';
import { flue } from '@flue/runtime/routing';
import { Hono } from 'hono';

observe(createOpenTelemetryObserver());

const app = new Hono();
app.route('/', flue());

export default app;
```

The adapter turns workflow runs, agent operations, model turns, tools, delegated tasks, compaction, and logs into trace activity. You can also consume `observe(...)` directly to send terminal failures to an error reporter or derive metrics such as operation latency, workflow failures, and model usage or cost.

Start with signals that describe outcomes: failed workflows, explicit application error logs, slow operations, and completed model usage. A model turn or tool call may fail before an agent recovers, so treating every nested error as an incident can create noisy alerts.

Telemetry can include sensitive application and model data, including workflow payloads, log attributes, prompts, output, and tool arguments or results. Prefer exporting timing, error, token, and cost metadata unless content is necessary for your investigation. If you enable content capture in an exporter or write your own observer, redact secrets and personal data before sending events to an external service.

## Next steps

- [Workflows](/docs/guide/workflows/) — create finite operations whose run history can be inspected.
- [Agents](/docs/guide/building-agents/) — create continuing agent instances and deliver direct or dispatched input.
- [Routing](/docs/guide/routing/) — add the application entrypoint where telemetry observers are registered.
- [Develop & Build](/docs/guide/develop-and-build/) — build the application environment that emits your production telemetry.
