---
title: Modal
description: Connect a Flue agent to an application-owned Modal Sandbox.
lastReviewedAt: 2026-05-30
---

The Modal connector adapts an already-initialized Modal Sandbox from the `modal` JavaScript SDK into Flue's sandbox interface. Use it for provider-backed command execution and files when your application provisions Modal sandbox resources.

## Add the connector

```bash
pnpm exec flue add modal
```

## Requirements

| Requirement      | Value                                                                     |
| ---------------- | ------------------------------------------------------------------------- |
| Provider package | `modal`                                                                   |
| Runtime          | Node.js 22 or later for the connector recipe's Modal SDK use              |
| Credentials      | `MODAL_TOKEN_ID` and `MODAL_TOKEN_SECRET`                                 |
| Image behavior   | Choose an image with the shell/system utilities needed by your agent work |

## Choose this connector when

Use Modal when your application already manages Modal applications, images, or sandbox lifetimes and needs to expose that compute boundary to Flue operations. The connector adapts the created sandbox; creation, shutdown, secret handling, networking, and image content remain your responsibility.

See [Sandboxes](/docs/guide/sandboxes/) and [Sandbox Connector API](/docs/api/sandbox-api/).
