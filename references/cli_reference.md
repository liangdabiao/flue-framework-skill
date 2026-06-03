# Flue CLI Reference

## Installation

```bash
npm install --save-dev @flue/cli
```

## Commands

### flue init

Create initial configuration:

```bash
flue init --target node    # Node.js target
flue init --target cloudflare  # Cloudflare target
```

### flue dev

Start development server (watch mode):

```bash
flue dev --target node --port 3583
```

### flue run

Execute workflow locally:

```bash
flue run workflows/my-workflow.ts --payload '{"input": "test"}'
```

### flue connect

Open interactive agent session:

```bash
flue connect hello-world local
```

### flue build

Build deployable artifacts:

```bash
flue build --target node --output ./dist
```

### flue logs

View workflow logs:

```bash
flue logs --follow
flue logs --run-id abc123
```

### flue add

Install connectors:

```bash
flue add e2b
flue add daytona
```

## Common Options

| Option | Description |
|--------|-------------|
| `--target <node | cloudflare>` | Build target |
| `--root <path>` | Project root directory |
| `--output <path>` | Build output directory |
| `--config <path>` | Configuration file path |
| `--env <path>` | Environment file path |

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Command failure |
| 130 | SIGINT interrupt |
| 143 | SIGTERM interrupt |

## Configuration File

`flue.config.ts`:

```typescript
import { defineConfig } from '@flue/cli';

export default defineConfig({
  target: 'node',
  root: '.',
  output: './dist',
  env: '.env',
});
```

## Environment Variables

| Variable | Description |
|----------|-------------|
| `ANTHROPIC_API_KEY` | Anthropic API key |
| `OPENAI_API_KEY` | OpenAI API key |
| `DEEPSEEK_API_KEY` | DeepSeek API key (built-in provider) |
| `OPENROUTER_API_KEY` | OpenRouter API key |
| `CLOUDFLARE_ACCOUNT_ID` | Cloudflare account ID |
| `FLUE_LOG_LEVEL` | Logging level (debug, info, warn, error) |
| `MODEL` | Default model specifier (e.g., `deepseek/deepseek-v4-flash`) |
| `ANTHROPIC_BASE_URL` | Override Anthropic endpoint (e.g., for DeepSeek Anthropic-compatible API) |

### Using DeepSeek API

DeepSeek provides two compatible API endpoints:

1. **OpenAI-compatible** (default for `deepseek/` provider): `https://api.deepseek.com`
2. **Anthropic-compatible**: `https://api.deepseek.com/anthropic`

**.env for DeepSeek native provider:**

```env
DEEPSEEK_API_KEY=sk-your-key
MODEL=deepseek/deepseek-v4-flash
```

**.env for DeepSeek via Anthropic-compatible endpoint:**

```env
ANTHROPIC_API_KEY=sk-your-key
ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic
MODEL=anthropic/claude-haiku-4-5
```

### Running with Custom Environment

```bash
flue run hello-world --target node --id test-001 --payload '{"message": "Hello"}' --env ".env"
```
