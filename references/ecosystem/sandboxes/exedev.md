---
title: exe.dev
description: Connect a Node-target Flue application to an exe.dev VM over SSH.
lastReviewedAt: 2026-05-30
---

The exe.dev connector adapts an existing exe.dev VM into Flue's sandbox interface using SSH for commands and SFTP for files. Because it depends on Node.js APIs and `ssh2`, use it with the Node target rather than a Cloudflare Worker target.

## Add the connector

```bash
pnpm exec flue add exedev
```

## Requirements

| Requirement     | Value                                                                                           |
| --------------- | ----------------------------------------------------------------------------------------------- |
| Target          | Node.js                                                                                         |
| Package         | `ssh2`                                                                                          |
| Remote resource | An existing exe.dev VM reachable by SSH                                                         |
| Authentication  | SSH configuration; optional provider API credentials when your application manages VM lifecycle |

## Choose this connector when

Use exe.dev when a Node-hosted Flue application should operate inside a VM you reach through SSH/SFTP. The connector recipe includes optional lifecycle helpers, but the sandbox adapter itself is designed around a VM your application owns.

Treat SSH keys and provider tokens as server-side secrets. Decide whether agent instances share or allocate VMs, and clean up application-owned VMs according to your retention policy.

See [Deploy on Node.js](/docs/ecosystem/deploy/node/), [Sandboxes](/docs/guide/sandboxes/), and [Sandbox Connector API](/docs/api/sandbox-api/).
