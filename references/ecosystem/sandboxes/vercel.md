---
title: Vercel Sandbox
description: Connect a Flue agent to an application-owned Vercel Sandbox environment.
lastReviewedAt: 2026-05-30
---

The Vercel Sandbox connector adapts an initialized `@vercel/sandbox` `Sandbox` into Flue's sandbox interface. Use it when application code should execute agent work inside a Vercel-managed sandbox rather than on its host filesystem.

## Add the connector

```bash
pnpm exec flue add vercel
```

## Requirements

| Requirement         | Value                                                                                 |
| ------------------- | ------------------------------------------------------------------------------------- |
| Provider package    | `@vercel/sandbox`                                                                     |
| Authentication      | `VERCEL_OIDC_TOKEN` or the authentication flow appropriate to your Vercel environment |
| Integration shape   | Application creates a sandbox, then passes it through the generated connector         |
| Lifecycle ownership | Your application decides retention and cleanup                                        |

## Typical use

```ts
import { Sandbox } from '@vercel/sandbox';
import { createAgent } from '@flue/runtime';
import { vercel } from '../connectors/vercel';

const sandbox = await Sandbox.create({ runtime: 'node24' });
const agent = createAgent(() => ({
  model: 'anthropic/claude-sonnet-4-6',
  sandbox: vercel(sandbox),
}));
```

Keep Vercel authentication values in trusted application configuration and determine whether sandboxes should be fresh per job or reusable for stable agent identities.

See [Sandboxes](/docs/guide/sandboxes/) and [Sandbox Connector API](/docs/api/sandbox-api/).
