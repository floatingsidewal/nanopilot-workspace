---
name: add-ollama
description: Install Ollama and configure NanoPilot to use a local LLM (no GitHub PAT required)
user_invocable: true
---

# /add-ollama — Local LLM via Ollama

Get a NanoPilot agent running against a local Ollama model instead of GitHub Models. Ideal for Proxmox LXC / homelab / air-gapped deployments where you don't want (or can't have) a GitHub PAT.

## When to use

- User is deploying on Proxmox LXC or a home Linux box.
- User says "use a local model" / "use Ollama" / "no GitHub PAT" / "no cloud LLM" / "set up local inference".
- User hits GitHub Models rate limits and wants an offline fallback.

## When NOT to use

- User is on resource-constrained hardware (< 8GB RAM, < 10GB free disk) — a 7B/8B model won't run well.
- User's agent uses tool-heavy workflows — local models often handle tool calling worse than `gpt-4o` and below. Flag this tradeoff before proceeding.

## Pre-flight

Check what's already there:

1. `command -v ollama` — is Ollama installed?
2. `curl -sf http://localhost:11434/api/version` — is the daemon running?
3. `ollama list` — are any models pulled?
4. Inspect the agent's `.env` — is `COPILOT_PROVIDER_BASE_URL` already set?

Skip any step that's already done. Only do what's needed.

## Step 1: Install Ollama

Detect platform, install accordingly. Explicitly ask the user's consent before `curl | sh`.

```bash
# Linux (covers Debian/Ubuntu LXC, RHEL variants, etc.)
curl -fsSL https://ollama.com/install.sh | sh

# macOS
brew install ollama
# or: download Ollama.app from https://ollama.com/download

# Proxmox LXC:
#   - If the LXC is unprivileged, create it with nesting=1 features
#   - CPU inference works out of the box; GPU requires GPU passthrough
#   - 4GB RAM minimum for 7B models; 8GB+ for comfort
```

Start the daemon:

```bash
# systemd (most LXCs)
systemctl enable --now ollama

# macOS / manual
ollama serve &
```

Verify:
```bash
curl -sf http://localhost:11434/api/version
```

## Step 2: Pull a model

Ask the user what they want to run. Sensible defaults by resource budget:

| Budget | Model | Pull command | Notes |
|---|---|---|---|
| 4 GB RAM | `llama3.2:3b` | `ollama pull llama3.2:3b` | Lightest viable; simpler agents only. |
| 8 GB RAM | `llama3.1:8b` | `ollama pull llama3.1:8b` | **Recommended default.** Solid general-purpose. |
| 8 GB RAM | `qwen2.5:7b` | `ollama pull qwen2.5:7b` | Strong tool-use / structured output. |
| 16+ GB RAM | `llama3.1:70b` | `ollama pull llama3.1:70b` | Quality bump; requires GPU or patience. |

After the pull finishes:

```bash
ollama list
```

## Step 3: Point the agent at Ollama

Edit the agent's `.env` (at `agents/<name>/.env` for local dev, or inside the deployment target for containerized agents).

NanoPilot runs the Copilot CLI inside a **persona container**. The daemon forwards `COPILOT_PROVIDER_*` vars directly into that container, so those are the vars to set:

**Four vars to set:**

```bash
COPILOT_PROVIDER_BASE_URL=http://host.docker.internal:11434/v1
COPILOT_PROVIDER_TYPE=openai
COPILOT_PROVIDER_API_KEY=ollama    # any non-empty string; Ollama ignores it
COPILOT_MODEL=llama3.1:8b          # or whatever you pulled
```

**Choose the right host for `COPILOT_PROVIDER_BASE_URL`:**

| Scenario | Host part |
|---|---|
| CLI dev (`npm run dev`) on same machine | `localhost` |
| Docker Desktop (macOS / Windows) container | `host.docker.internal` |
| Linux bridge network container | `172.17.0.1` (default Docker bridge gateway) or the LXC host's LAN IP |
| LXC container with host networking | `localhost` |
| Ollama on a different machine on the LAN | the machine's LAN IP (e.g. `192.168.x.x`) |

Note: the URL ends at `/v1` — no `/chat/completions` suffix. The Copilot CLI appends the path itself.

Comment out any GitHub PAT (`NANOPILOT_API_KEY=ghp_…`) if it's still set — it isn't needed when a BYOK provider is configured.

## Step 4: Restart and verify

```bash
# Containerized deployment
docker compose restart

# CLI dev
npm run dev
# Watch for log line "Forwarding BYOK custom-provider env vars to persona container"
```

Verification prompt — send this through the agent's channel (Telegram, CLI, etc.):

> Say "hello" in exactly five words.

Expected: a five-word response. If the agent says "I'm the mock backend" or similar, the BYOK vars are missing — go back to Step 3.

If the agent hangs or errors with "connection refused" / "ECONNREFUSED" / "no such host":
- Ollama daemon isn't running (Step 1 verification)
- Wrong host in `COPILOT_PROVIDER_BASE_URL` (Step 3 table)
- LXC firewall blocking 11434 — `ss -tlnp | grep 11434` from the container to confirm routing

If the agent errors with "Model … is not available" or "BYOK providers require an explicit model":
- `COPILOT_MODEL` is missing or doesn't match any `ollama list` entry
- Try the exact tag you see in `ollama list`, including the size suffix (`:8b`, `:7b`, etc.)

## Step 5: Document what you did

If the user has a `docs/` dir in their agent repo, drop a line in `docs/DEPLOYMENT.md` or similar: `LLM: Ollama (llama3.1:8b) at <host>:11434`. This is the kind of thing future-user forgets.

## Caveats to flag

- **Tool-calling quality on local models is uneven.** If the agent has tools (SSH, file ops, memory), plan for more structured-output errors than with GitHub Models. `qwen2.5:7b` handles tool calls better than `llama3.1:8b` in our testing; consider bumping.
- **No guarantees on inference speed.** Cold starts take 3–10s; subsequent turns are fast. A busy Ollama instance queues requests — consider dedicating the host.
- **Bulkhead scanning is unaffected.** `@bulkhead-ai/core` runs in the daemon, independent of the LLM. Local LLM + Bulkhead is fully supported.

## Assumption flagged

This skill assumes the NanoPilot backend speaks OpenAI-compatible endpoints at `/v1/chat/completions`. If you hit integration errors specific to Ollama's wire format (not 404 / connection issues), file an issue against the platform — it may require a dedicated `OllamaBackend` in `src/agent.ts`.
