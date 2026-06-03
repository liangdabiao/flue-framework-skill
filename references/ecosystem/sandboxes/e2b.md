---
title: E2B
description: Connect a Flue agent to an E2B Linux sandbox.
lastReviewedAt: 2026-05-30
---

The E2B connector adapts an initialized E2B sandbox from the `e2b` package into Flue's sandbox interface. Use it for provider-managed Linux execution when an agent needs shell commands and workspace files outside the application host.

## Add the connector

```bash
pnpm exec flue add e2b
```

## Requirements

| Requirement         | Value                                                      |
| ------------------- | ---------------------------------------------------------- |
| Provider package    | `e2b`                                                      |
| Credential          | `E2B_API_KEY`                                              |
| Environment         | Provider-managed Linux sandbox                             |
| Lifecycle ownership | Your application creates and closes or retains the sandbox |

## Integration shape

```ts
import { Sandbox } from 'e2b';
import { createAgent } from '@flue/runtime';
import { e2b } from '../connectors/e2b';

const sandbox = await Sandbox.create();
const agent = createAgent(() => ({
  model: 'anthropic/claude-sonnet-4-6',
  sandbox: e2b(sandbox),
}));
```

Select templates, timeouts, network access, secret exposure, and resource reuse through your application and provider policy. Flue adapts the active environment; it does not choose provider retention for you.

See [Sandboxes](/docs/guide/sandboxes/) and [Sandbox Connector API](/docs/api/sandbox-api/).
