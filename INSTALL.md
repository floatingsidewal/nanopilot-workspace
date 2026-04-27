# {{DISPLAY_NAME}} — Installation Guide

This guide is written for both humans and coding agents (e.g., Claude Code with the `/add-nanopilot` skill).

## Prerequisites

- Docker and Docker Compose installed on the target host
- A GitHub personal access token with Models API access (or a GitHub Copilot subscription)

## Quick Start

```bash
# 1. Configure secrets
cp .env.example .env
# Edit .env — at minimum set GITHUB_TOKEN

# 2. Start the agent
docker compose up -d

# 3. Verify
curl http://localhost:3978/health
docker compose logs -f
```

The agent is now running and listening for messages on configured channels.

## Interactive Setup (optional)

If you have the NanoPilot CLI available:

```bash
# Interactive .env builder — prompts for each value with validation
npx nanopilot init-env

# Readiness check — verifies workspace, credentials, connectivity
npx nanopilot check
```

## Channel Setup

### Telegram

1. Message [@BotFather](https://t.me/BotFather) on Telegram and create a new bot
2. Copy the bot token to `TELEGRAM_BOT_TOKEN` in `.env`
3. Send `/start` to your bot, then use `/chatid` to get your user ID
4. Add your user ID to `NANOPILOT_TELEGRAM_ALLOWLIST` in `.env` (comma-separated for multiple users)
5. Restart: `docker compose restart`

### Microsoft Teams

1. Create an Azure Bot resource and Entra app registration
2. Set `MICROSOFT_APP_ID`, `MICROSOFT_APP_PASSWORD`, and `MICROSOFT_APP_TENANT_ID` in `.env`
3. Build and upload the Teams manifest from `teams-app/`
4. Restart: `docker compose restart`

## Data Protection (Bulkhead)

NanoPilot uses [@bulkhead-ai/core](https://github.com/floatingsidewal/bulkhead) for data protection — 154 secret patterns, 45+ PII entity types, prompt injection detection, and system prompt leakage detection. Enabled by default.

```bash
# In .env — scan point bitmask (default: 3 = outbound + tool-results)
#   1 = outbound (before channel delivery)
#   2 = tool-results (before feeding back to LLM)
#   4 = inbound (reserved, not yet implemented)
# Also accepts named values: "outbound,tool", "all", "false"
NANOPILOT_BULKHEAD_ENABLED=3
NANOPILOT_BULKHEAD_POLICY=moderate    # "strict" or "moderate"
NANOPILOT_BULKHEAD_MODE=local         # "local" (in-process) or "remote" (HTTP sidecar)
# NANOPILOT_BULKHEAD_SERVER_URL=http://bulkhead:3000  # Required when MODE=remote
```

## Tool Execution

To allow the agent to run commands on remote hosts via SSH:

1. Set `NANOPILOT_TOOL_EXECUTION=true` in `.env`
2. Configure SSH access: `NANOPILOT_SSH_HOST`, `NANOPILOT_SSH_USER`, `NANOPILOT_SSH_KEY_PATH`
3. Mount the SSH key into the container (add to `docker-compose.yml` volumes)
4. Configure per-group access control via the `group_config` database table

## Diagnostics

### Container won't start
```bash
docker compose logs -f
```
Common causes: missing `GITHUB_TOKEN`, invalid `.env` syntax, port conflict.

### "platformVersion mismatch" error
The `agent.json` requires a specific NanoPilot platform version. Either:
- Update `NANOPILOT_VERSION` in `.env` to a compatible image tag
- Or adjust `platformVersion` in `agent.json` to match your image

### No workspace found
- Verify `docker-compose.yml` mounts `./workspace:/workspace`
- Check that `workspace/personality.md` exists

### Agent not responding in Telegram
- Verify `TELEGRAM_BOT_TOKEN` is correct
- Check that your user ID is in `NANOPILOT_TELEGRAM_ALLOWLIST`
- Look for auth errors in logs: `docker compose logs | grep -i "unauthorized"`

### Bulkhead blocking content unexpectedly
- Check policy: `NANOPILOT_BULKHEAD_POLICY=moderate` (redacts instead of blocking)
- To disable specific scan points: `NANOPILOT_BULKHEAD_ENABLED=1` (outbound only)
- To disable entirely: `NANOPILOT_BULKHEAD_ENABLED=0`

## Updating

```bash
# Pull latest platform image
docker compose pull

# Restart with new image
docker compose up -d
```

To pin a specific version, set `NANOPILOT_VERSION=1.0.0` in `.env`.
