# Intent: Add Third-Party LLM Support

## What Changed

Added `ANTHROPIC_MODEL` to the list of secrets read from `.env` and passed to the container.

## Modified Function

### `readSecrets()`

**Before:**
```typescript
function readSecrets(): Record<string, string> {
  return readEnvFile([
    'CLAUDE_CODE_OAUTH_TOKEN',
    'ANTHROPIC_API_KEY',
    'ANTHROPIC_BASE_URL',
    'ANTHROPIC_AUTH_TOKEN',
  ]);
}
```

**After:**
```typescript
function readSecrets(): Record<string, string> {
  return readEnvFile([
    'CLAUDE_CODE_OAUTH_TOKEN',
    'ANTHROPIC_API_KEY',
    'ANTHROPIC_BASE_URL',
    'ANTHROPIC_AUTH_TOKEN',
    'ANTHROPIC_MODEL',
  ]);
}
```

## Invariants

1. **Secrets are never written to disk** - They are passed via stdin to the container
2. **Backwards compatible** - If `ANTHROPIC_MODEL` is not set, the SDK uses its default model
3. **No code changes to agent-runner** - The SDK reads environment variables directly from `sdkEnv`

## How It Works

1. `readSecrets()` reads `ANTHROPIC_MODEL` from `.env`
2. The secret is passed to the container via stdin as part of `ContainerInput`
3. The agent-runner merges secrets into `sdkEnv` which is passed to the SDK's `query()` function
4. The Claude Agent SDK reads `ANTHROPIC_MODEL` from the environment and uses it for API calls

## Supported Third-Party Providers

- Alibaba Qwen (DashScope)
- OpenRouter
- AWS Bedrock
- Google Vertex AI
- Any provider with Anthropic-compatible endpoints