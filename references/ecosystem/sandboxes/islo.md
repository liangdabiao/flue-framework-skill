---
title: islo
description: Connect a Node-target Flue application to a named islo sandbox through its CLI.
lastReviewedAt: 2026-05-30
---

The islo connector adapts a named islo sandbox into Flue's sandbox interface by invoking the local `islo` CLI. It is designed for a Node.js server, container, or CI runner where the binary is installed and can launch remote commands.

## Add the connector

```bash
pnpm exec flue add islo
```

## Requirements

| Requirement        | Value                                                                            |
| ------------------ | -------------------------------------------------------------------------------- |
| Target             | Node.js or another host with Node child-process capability                       |
| Runtime dependency | The `islo` binary installed on `PATH`                                            |
| Credential         | Existing CLI authentication or `ISLO_API_KEY` for server/CI operation            |
| Sandbox identity   | A named islo sandbox created and managed by your application or deployment setup |

## Choose this connector when

Use islo when an application can rely on a host-installed CLI and wants to connect to named sandboxes from a Node execution environment. Do not use it in Cloudflare Workers or other runtimes that cannot execute native child processes.

The connector runs remote shell/file work through the CLI; ensure its host process, credentials, and agent inputs match your intended trust boundary.

See [Deploy on Node.js](/docs/ecosystem/deploy/node/), [Sandboxes](/docs/guide/sandboxes/), and [Sandbox Connector API](/docs/api/sandbox-api/).
