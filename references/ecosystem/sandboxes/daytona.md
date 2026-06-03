---
title: Daytona
description: Connect a Flue agent to an application-owned Daytona sandbox.
lastReviewedAt: 2026-06-01
---

The Daytona connector adapts an already-initialized Daytona sandbox from `@daytona/sdk` into Flue's sandbox interface. Use it when a Node-hosted application needs a provider-managed Linux environment with filesystem and shell operations.

## Add the connector

```bash
pnpm exec flue add daytona
```

The generated connector expects your application to create and own the Daytona sandbox. It does not decide sandbox identity, retention, or cleanup for you.

## Requirements

| Requirement         | Value                                                                          |
| ------------------- | ------------------------------------------------------------------------------ |
| Provider package    | `@daytona/sdk`                                                                 |
| Credential          | `DAYTONA_API_KEY`                                                              |
| Optional settings   | `DAYTONA_API_URL`, `DAYTONA_TARGET`                                            |
| Integration shape   | Your code creates a Daytona sandbox, then passes it through `daytona(sandbox)` |
| Lifecycle ownership | Your application owns creation, retention, and deletion                        |

## Typical use

```ts
import { Daytona } from '@daytona/sdk';
import { createAgent } from '@flue/runtime';
import { daytona } from '../connectors/daytona';

const client = new Daytona({ apiKey: env.DAYTONA_API_KEY });
const sandbox = await client.create();
const agent = createAgent(() => ({
  model: 'anthropic/claude-sonnet-4-6',
  sandbox: daytona(sandbox),
}));
```

Configure images, snapshots, regions, environment variables, and volumes through the Daytona SDK before passing the sandbox to `daytona(...)`. For a narrower working directory, configure `cwd` on the created agent; Flue resolves it once against the connector's provider-owned base directory during `init()`.

See [Sandboxes](/docs/guide/sandboxes/#remote-sandboxes), [Sandbox Connector API](/docs/api/sandbox-api/), and [Daytona's TypeScript SDK reference](https://www.daytona.io/docs/en/typescript-sdk/daytona/).
