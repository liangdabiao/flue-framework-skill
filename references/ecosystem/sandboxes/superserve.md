---
title: Superserve
description: Track the Superserve sandbox connector recipe and its current compatibility status.
lastReviewedAt: 2026-05-30
---

Superserve has a sandbox connector entry in the Flue connector catalog, intended to adapt an initialized Superserve sandbox into Flue's sandbox interface.

## Current compatibility status

Do not use the generated Superserve recipe with the current runtime without reviewing and updating it first. The catalog recipe currently imports an older `@flue/sdk/sandbox` surface and assumes a cleanup callback accepted by `createSandboxSessionEnv(...)`; the current public connector API is exported from `@flue/runtime` and leaves provider resource cleanup to application code.

| Intended requirement | Value                                                                                                                                      |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Provider package     | `@superserve/sdk`                                                                                                                          |
| Credential           | `SUPERSERVE_API_KEY`                                                                                                                       |
| Intended environment | Provider-managed sandbox                                                                                                                   |
| Required before use  | Reconcile the generated connector with [Sandbox Connector API](/docs/api/sandbox-api/) |

Until that recipe is updated and type-checked against the current runtime, choose another available sandbox connector or implement a project-owned adapter against the public `SandboxApi` contract.

See [Sandboxes](/docs/guide/sandboxes/) and [Sandbox Connector API](/docs/api/sandbox-api/).
