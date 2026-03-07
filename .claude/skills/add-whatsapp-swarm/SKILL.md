---
name: add-whatsapp-swarm
description: Add Agent Swarm support to WhatsApp. Multiple agents with different LLM models can collaborate via [AgentName] prefixes. Requires WhatsApp channel to be set up first (use /add-whatsapp). Triggers on "agent swarm whatsapp", "whatsapp swarm", "multi-agent whatsapp".
---

# Add Agent Swarm to WhatsApp

This skill adds Agent Swarm (Teams) support to an existing WhatsApp channel. Multiple agents with different LLM models can collaborate in a WhatsApp group, with each agent's messages prefixed with `[AgentName]` for identification.

**Prerequisite**: WhatsApp must already be set up via the `/add-whatsapp` skill. If `src/channels/whatsapp.ts` does not exist, tell the user to run `/add-whatsapp` first.

## How It Works

Unlike Telegram's bot pool approach (where each agent gets its own bot identity), WhatsApp uses a single phone number. The swarm feature adds `[AgentName]` prefixes to distinguish agents:

```
User: @assistant Analyze this code
[Leader] I'll coordinate the analysis. Let me have the Researcher investigate the patterns.
[Researcher] I found 3 potential issues in the codebase...
[Coder] I've prepared the fixes...
[Leader] Based on the team's findings, here's my synthesis...
```

## Prerequisites

### Third-Party LLM Provider (Optional)

If you want agents to use different LLM models (e.g., Qwen, GLM), configure the third-party LLM provider first. Run `/add-third-party-llm` for setup instructions.

The provider must support Anthropic-compatible API endpoints.

## Implementation

### Step 1: Create Swarm Configuration

Create a `swarm.yaml` file in your group folder (e.g., `groups/main/swarm.yaml`):

```yaml
# groups/{groupFolder}/swarm.yaml
version: 1.0
enabled: true

# Orchestration mode: "parallel" or "sequential"
mode: parallel

# Subagent definitions
agents:
  - name: Leader
    model: default  # Uses default ANTHROPIC_MODEL from .env
    role: orchestrator
    profile: |
      You are the lead agent. Coordinate subagents, synthesize results,
      and provide final responses to the user. Use <internal> tags for
      internal reasoning that shouldn't be shown to users.

  - name: Researcher
    model: glm-5  # Third-party model
    baseUrl: https://coding-intl.dashscope.aliyuncs.com/apps/anthropic
    role: worker
    profile: |
      You are a research specialist. Search for information, analyze data,
      and report findings. Be thorough and cite sources.

  - name: Coder
    model: qwen3.5-plus
    baseUrl: https://coding-intl.dashscope.aliyuncs.com/apps/anthropic
    role: worker
    profile: |
      You are a coding specialist. Write, debug, and explain code.

# Execution settings
settings:
  timeout: 300000        # 5 min per agent
  maxRetries: 3
  parallelTimeout: 60000 # Wait time for all parallel agents
```

### Step 2: Configure Third-Party LLM (if using non-default models)

Add to your `.env` file:

```bash
# Third-party LLM provider
ANTHROPIC_AUTH_TOKEN=sk-xxx
ANTHROPIC_BASE_URL=https://coding-intl.dashscope.aliyuncs.com/apps/anthropic
```

### Step 3: Rebuild and Restart

```bash
npm run build
# macOS:
launchctl kickstart -k gui/$(id -u)/com.nanoclaw
# Linux:
systemctl --user restart nanoclaw
```

### Step 4: Test

Send a message to your WhatsApp group asking for a multi-agent task:

> @assistant Assemble a team of a researcher and a coder to build me a hello world app

You should see:
- Each agent responds with `[AgentName]` prefix
- The orchestrator coordinates and synthesizes results
- Short, scannable messages from each agent

Check logs: `tail -f logs/nanoclaw.log | grep -i swarm`

## Configuration Options

### Mode: parallel

All workers execute simultaneously, then the orchestrator synthesizes:

```
1. Workers start in parallel
2. Each sends [AgentName] responses
3. Orchestrator receives all results
4. Orchestrator sends final synthesis
```

Use for: Independent tasks that can run concurrently.

### Mode: sequential

Agents execute in order, each receiving full history:

```
1. Orchestrator processes first
2. Each worker receives previous outputs
3. Final orchestrator synthesis
```

Use for: Tasks that build on previous results.

### Agent Configuration

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Agent name shown in `[Name]` prefix |
| `model` | No | Model ID or `default` for ANTHROPIC_MODEL |
| `baseUrl` | No | API endpoint for third-party models |
| `authToken` | No | Override token for this agent |
| `profile` | Yes | Agent's system prompt / role description |
| `role` | No | `orchestrator` (leader) or `worker` |

### Settings

| Setting | Default | Description |
|---------|---------|-------------|
| `timeout` | 300000 | Per-agent timeout (5 min) |
| `maxRetries` | 3 | Retry attempts on failure |
| `parallelTimeout` | 60000 | Total wait for all parallel agents |

## Troubleshooting

### Swarm not activating

1. Verify `swarm.yaml` exists in the correct group folder
2. Check `enabled: true` is set
3. Ensure exactly one agent has `role: orchestrator`
4. Check logs for configuration errors

### Third-party models not working

1. Verify `ANTHROPIC_AUTH_TOKEN` and `ANTHROPIC_BASE_URL` are set in `.env`
2. Test the API endpoint directly with curl
3. Check the model ID matches the provider's supported models

### Agents timing out

1. Increase `timeout` in settings
2. Check if the model is responding slowly
3. Reduce task complexity or split into smaller tasks

## Architecture Notes

- Swarm config is loaded per-group from `groups/{folder}/swarm.yaml`
- Each agent runs in its own container with isolated secrets
- Model overrides are passed via stdin (never written to disk)
- Agent prefixes are preserved through the WhatsApp channel
- The orchestrator is identified by `role: orchestrator` (not by name)

## Removal

To remove Agent Swarm support:

1. Delete or disable `swarm.yaml` (set `enabled: false`)
2. Restart NanoClaw: `launchctl kickstart -k gui/$(id -u)/com.nanoclaw` (macOS) or `systemctl --user restart nanoclaw` (Linux)

The core swarm code is built into NanoClaw and doesn't need separate removal.