---
title: Mirage
description: Connect Flue agents to Mirage workspaces and mounted resources.
lastReviewedAt: 2026-05-30
---

The Mirage connector adapts an application-owned Mirage `Workspace` into Flue's sandbox interface. Mirage offers runtime packages for Node and browser-class runtimes, allowing the connector pattern to be used on Node or Cloudflare when you choose compatible resources.

## Add the connector

```bash
pnpm exec flue add mirage
```

## Requirements

| Target     | Runtime package             | Notes                                            |
| ---------- | --------------------------- | ------------------------------------------------ |
| Node.js    | `@struktoai/mirage-node`    | Can use Node-compatible Mirage resources.        |
| Cloudflare | `@struktoai/mirage-browser` | Use browser-compatible Workspace resources only. |

The generated connector uses Mirage's shared workspace contract. Some Mirage resources, such as SSH- or database-oriented Node resources, require the Node runtime and must not be imported into a Cloudflare build.

## Choose this connector when

Use Mirage when your application wants to assemble a workspace from explicit mounted resources and present that workspace to an agent through a single sandbox boundary. Your application owns resource mounting, credentials, writable boundaries, and workspace lifetime.

See [Sandboxes](/docs/guide/sandboxes/), [Deploy on Node.js](/docs/ecosystem/deploy/node/), [Deploy on Cloudflare](/docs/ecosystem/deploy/cloudflare/), and [Sandbox Connector API](/docs/api/sandbox-api/).
