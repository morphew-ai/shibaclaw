---
name: add-third-party-llm
description: Configure NanoClaw to use third-party LLM providers (Qwen, OpenRouter, AWS Bedrock, etc.) via Anthropic-compatible API endpoints.
---

# Add Third-Party LLM Support

This skill configures NanoClaw to use third-party LLM providers that offer Anthropic-compatible APIs. This includes providers like:

- **Alibaba Qwen** (via DashScope)
- **OpenRouter**
- **AWS Bedrock**
- **Google Vertex AI**
- **Azure OpenAI** (with Anthropic models)
- Any other provider with Anthropic-compatible endpoints

## Phase 1: Pre-flight

### Check if already applied

Read `.nanoclaw/state.yaml`. If `third-party-llm` is in `applied_skills`, skip to Phase 3 (Configure). The code changes are already in place.

## Phase 2: Apply Code Changes

Run the skills engine to apply this skill's code package.

### Initialize skills system (if needed)

If `.nanoclaw/` directory doesn't exist yet:

```bash
npx tsx scripts/apply-skill.ts --init
```

### Apply the skill

```bash
npx tsx scripts/apply-skill.ts .claude/skills/add-third-party-llm
```

This deterministically:
- Adds `ANTHROPIC_MODEL` to the secrets list in `src/container-runner.ts`
- Records the application in `.nanoclaw/state.yaml`

### Validate code changes

```bash
npm run build
./container/build.sh
```

Build must be clean before proceeding.

## Phase 3: Configure

### Add environment variables to .env

Add the following variables to your `.env` file:

```bash
# Third-party LLM configuration
# Use ANTHROPIC_AUTH_TOKEN (not ANTHROPIC_API_KEY) for third-party providers
ANTHROPIC_AUTH_TOKEN=your-api-key-here
ANTHROPIC_BASE_URL=https://your-provider-endpoint.com/anthropic
ANTHROPIC_MODEL=your-model-name

# Optional: Keep ANTHROPIC_API_KEY empty or remove it
# ANTHROPIC_API_KEY=
```

### Example configurations

#### Alibaba Qwen (DashScope)

```bash
ANTHROPIC_AUTH_TOKEN=sk-xxxxxxxx
ANTHROPIC_BASE_URL=https://coding-intl.dashscope.aliyuncs.com/apps/anthropic
ANTHROPIC_MODEL=qwen3.5-plus
```

#### OpenRouter

```bash
ANTHROPIC_AUTH_TOKEN=sk-or-xxxxxxxx
ANTHROPIC_BASE_URL=https://openrouter.ai/api/v1
ANTHROPIC_MODEL=anthropic/claude-3.5-sonnet
```

### Restart the service

```bash
launchctl kickstart -k gui/$(id -u)/com.nanoclaw  # macOS
# Linux: systemctl --user restart nanoclaw
```

## Phase 4: Verify

### Test via your messaging channel

Send a message to test the integration:

> "Hello, can you confirm which model you're running on?"

### Check logs if needed

```bash
tail -f logs/nanoclaw.log
```

Look for:
- Successful container spawns
- Any API authentication errors
- Model-related errors

## Troubleshooting

### "Invalid API key" or authentication errors

1. Verify `ANTHROPIC_AUTH_TOKEN` is set correctly (not `ANTHROPIC_API_KEY`)
2. Check if your provider uses a different auth header
3. Ensure the token has proper permissions

### "Model not found" errors

1. Verify `ANTHROPIC_MODEL` matches your provider's model names exactly
2. Check your provider's documentation for available models
3. Some providers require model IDs like `anthropic/claude-3.5-sonnet`

### "Connection refused" or network errors

1. Verify `ANTHROPIC_BASE_URL` is correct
2. Check if the URL needs a trailing path (some providers need `/v1` or `/api/v1`)
3. Test the endpoint manually with curl

### Container still using Anthropic directly

1. Ensure `ANTHROPIC_API_KEY` is not set (or is empty)
2. Verify the skill was applied: check `.nanoclaw/state.yaml` for `third-party-llm`
3. Rebuild the container: `./container/build.sh`

## Technical Details

The skill modifies `readSecrets()` in `src/container-runner.ts` to include:

- `ANTHROPIC_AUTH_TOKEN` - API token for third-party provider
- `ANTHROPIC_BASE_URL` - Custom API endpoint URL
- `ANTHROPIC_MODEL` - Model identifier for the third-party provider
- `CLAUDE_CODE_OAUTH_TOKEN` - (unchanged) Claude Code OAuth token
- `ANTHROPIC_API_KEY` - (unchanged) Native Anthropic API key

These secrets are passed to the container via stdin (never written to disk) and merged into the SDK's environment.