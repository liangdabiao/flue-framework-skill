---
title: Targets and output
description: Choose Node or Cloudflare output and control where build artifacts are written.
---

Flue builds self-contained server artifacts for the selected target. Use configuration defaults for routine work and CLI flags for one-off builds.

## Select a target

```bash
pnpm exec flue build --target node
pnpm exec flue build --target cloudflare
```

Node builds run with your configured provider credentials. Cloudflare builds can access Workers platform bindings such as Workers AI where configured.
