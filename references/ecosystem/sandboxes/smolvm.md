---
title: smolvm
description: Run Flue sandbox work in a local libkrun-backed virtual machine.
lastReviewedAt: 2026-05-30
---

The smolvm connector adapts an initialized `Machine` from `smolvm-embedded` into Flue's sandbox interface. Unlike a hosted sandbox service, smolvm runs locally through a host hypervisor.

## Add the connector

```bash
pnpm exec flue add smolvm
```

## Requirements

| Requirement          | Value                                                    |
| -------------------- | -------------------------------------------------------- |
| Provider package     | `smolvm-embedded`                                        |
| Supported host shape | macOS or Linux host with suitable virtualization support |
| Credential           | None required by smolvm itself                           |
| Unsupported runtime  | Edge/Worker runtimes without local hypervisor execution  |

## Choose this connector when

Use smolvm for trusted desktop, CI, or server environments where local microVM execution is the desired isolation boundary. The host running the Flue application must support the underlying virtualization mechanism; this is not a Cloudflare Worker sandbox option.

The connector recipe treats networking and machine lifetime as explicit choices. Do not assume a local VM has network access or that it will be cleaned up without your application doing so.

See [Deploy on Node.js](/docs/ecosystem/deploy/node/), [Sandboxes](/docs/guide/sandboxes/), and [Sandbox Connector API](/docs/api/sandbox-api/).
