---
title: boxd
description: Connect a Flue agent to an application-owned boxd Linux VM.
lastReviewedAt: 2026-05-30
---

The boxd connector adapts an already-initialized boxd `Box` from `@boxd-sh/sdk` into Flue's sandbox interface. Use it when an agent needs a provider-backed Linux virtual machine with filesystem and shell behavior rather than the lightweight default workspace.

## Add the connector

Run the connector installation flow in your Flue project:

```bash
pnpm exec flue add boxd
```

The generated connector expects your application to create and own the boxd VM. It does not decide VM identity, retention, or cleanup for you.

## Requirements

| Requirement         | Value                                                          |
| ------------------- | -------------------------------------------------------------- |
| Provider package    | `@boxd-sh/sdk`                                                 |
| Credential          | `BOXD_API_KEY`, or provider-supported short-lived `BOXD_TOKEN` |
| Execution shape     | Linux VM adapted to `SandboxFactory`                           |
| Lifecycle ownership | Your application owns creation, reuse, and deletion            |

## Use it when

Choose boxd when a task requires real Linux command behavior in an isolated provider VM, particularly where a separate VM per workspace or agent instance is part of your application design.

Before reusing a VM across sessions or tenants, define identity, authorization, egress, secrets, and cleanup policies. Conversation persistence remains controlled separately by Flue session storage.

See [Sandboxes](/docs/guide/sandboxes/) for execution-boundary design and [Sandbox Connector API](/docs/api/sandbox-api/) for the adapter contract.
