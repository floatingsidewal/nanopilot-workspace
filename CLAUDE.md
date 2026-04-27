# CLAUDE.md — {{DISPLAY_NAME}} Agent

This file gives coding agents (Claude Code etc.) the context they need when working
inside a {{DISPLAY_NAME}} deployment repo. This is **not** the NanoPilot platform
repo — it is a single agent built on NanoPilot.

## What this repo is

A NanoPilot agent deployment. Layout:

```
agent.json              # Agent metadata, platform version constraint
workspace/
  personality.md        # System prompt / persona
  rules.md              # Reflection rules
  skills/               # Skill files (*.md) — also included in the system prompt
docker-compose.yml      # Mounts workspace into the ghcr.io/floatingsidewal/nanopilot image
Dockerfile.persona      # Agent-specific persona image (extends nanopilot-persona) — pre-bake CLIs the agent will need
.env                    # Secrets + runtime config (gitignored)
INSTALL.md              # Full deploy/configuration reference
```

The agent runs as a container from `ghcr.io/floatingsidewal/nanopilot:${NANOPILOT_VERSION:-latest}`.
The platform source lives at https://github.com/floatingsidewal/nanopilot — **do not
clone it to answer a question about this agent**. Everything you need to change an
agent's behavior is in `workspace/` (prompt/skills) or `.env` (runtime config).

## Bulkhead (data protection) — how to apply a policy

NanoPilot has **built-in** data protection powered by
[`@bulkhead-ai/core`](https://github.com/floatingsidewal/bulkhead): 154 secret
patterns, 45+ PII entity types, prompt-injection detection, and system-prompt
leakage detection. It is **enabled by default** and runs in-process inside the
agent container. No MCP, no sidecar, no extra service required.

> ⚠️ **Do not confuse this with `bulkhead-mcp`.** That is a separate project — a
> Cedar-policy MCP proxy for MCP-based clients (Claude Desktop, Claude Code's MCP
> integration). It is **not** used by NanoPilot agents. NanoPilot scans in-process
> via `@bulkhead-ai/core`. If someone asks how to apply a bulkhead policy to this
> agent, the answer is the `.env` configuration below — nothing else.

### Configure via `.env`

```bash
# Scan-point bitmask. Default: 3 (outbound + tool-results).
#   1 = outbound        (scan LLM output before delivering to channels / CLI)
#   2 = tool-results    (scan tool output before feeding back to the LLM)
#   4 = inbound         (reserved, not yet wired)
# Also accepts named values: "outbound,tool", "all", "false"
# `true` is a backwards-compat alias — it maps to the default value 3.
NANOPILOT_BULKHEAD_ENABLED=3

# Policy — governs thresholds and block-vs-redact behavior.
#   moderate = redact detections (recommended default)
#   strict   = block on detection, return a generic message
NANOPILOT_BULKHEAD_POLICY=moderate

# Mode — where the scan runs.
#   local  = in-process @bulkhead-ai/core (default; no extra infra)
#   remote = HTTP client to @bulkhead-ai/server sidecar (stub; not wired yet)
NANOPILOT_BULKHEAD_MODE=local
# NANOPILOT_BULKHEAD_SERVER_URL=http://bulkhead:3000   # only when MODE=remote
```

### Apply changes

```bash
# Edit .env, then:
docker compose up -d            # recreates the container, picking up new .env values
```

### Tuning guide

| Goal                                         | Setting                                       |
| -------------------------------------------- | --------------------------------------------- |
| Default protection                           | `NANOPILOT_BULKHEAD_ENABLED=3` + `moderate`   |
| High-assurance (block on detection)          | `NANOPILOT_BULKHEAD_POLICY=strict`            |
| Outbound only (no tool-result scanning)      | `NANOPILOT_BULKHEAD_ENABLED=1`                |
| Tool-results only (no outbound scanning)     | `NANOPILOT_BULKHEAD_ENABLED=2`                |
| Disable entirely (dev only)                  | `NANOPILOT_BULKHEAD_ENABLED=0`                |
| Bulkhead blocking legitimate content         | Switch `strict` → `moderate`, or drop bit 2   |

Policies (`strict`, `moderate`) are defined by `@bulkhead-ai/core`. To author a
**custom** policy, that is a platform-level change — file an issue on the
`nanopilot` repo rather than editing this agent workspace.

### Turning bulkhead off (or scoping it down)

Bulkhead is **on by default for a reason** — it is the layer that keeps secrets
out of LLM output and sensitive tool results from being echoed back. Only turn
it off deliberately, and only for these cases:

| Case                                    | What to do                                | Notes                                                                 |
| --------------------------------------- | ----------------------------------------- | --------------------------------------------------------------------- |
| Local dev / debugging a false positive  | `NANOPILOT_BULKHEAD_ENABLED=0` in `.env`  | Revert before redeploying. Do not ship `0` to a production agent.     |
| Unit / scenario tests (platform repo)  | `NANOPILOT_BULKHEAD_ENABLED=0` or construct `LocalProvider('moderate')` directly | `src/bulkhead/bulkhead.test.ts` lives in the **NanoPilot platform repo** — it is not present in this agent repo.  |
| Bulkhead is redacting content the agent legitimately needs | Drop bit 2: `NANOPILOT_BULKHEAD_ENABLED=1` | Outbound stays protected; only tool-result scanning is disabled.      |
| Need to see raw LLM output in dev logs  | Drop bit 1: `NANOPILOT_BULKHEAD_ENABLED=2` | Tool-result scanning stays on; outbound redaction is off.             |
| Strict policy is over-blocking          | `NANOPILOT_BULKHEAD_POLICY=moderate`       | Moderate redacts instead of blocking — usually the right first step.  |

Apply any of the above with `docker compose up -d`. (`docker compose restart` does
**not** re-read `.env` — it keeps the env baked in at container creation time.)

**Do not** turn bulkhead off because a scan produced a warning in the logs —
inspect the detection first. A log line like `"Bulkhead: content redacted"`
means bulkhead did its job, not that it's broken.

## Redeploy after any change

```bash
# Workspace changes (personality, rules, skills) — no rebuild needed,
# just restart so the container re-reads the mounted volume:
docker compose restart daemon

# .env changes:
docker compose up -d

# Persona-image changes (Dockerfile.persona — pre-baked CLIs the agent uses):
docker compose --profile build build persona
docker compose up -d --force-recreate

# Platform upgrade (preferred: use the /upgrade-nanopilot skill — it pulls
# both daemon + persona base, rebuilds the agent persona, and verifies):
docker compose pull && docker compose up -d
```

## Where to look first

| Question                                    | File                                 |
| ------------------------------------------- | ------------------------------------ |
| What does this agent do / how it behaves    | `workspace/personality.md`           |
| What capabilities it has                    | `workspace/skills/*.md`              |
| How to install / full deploy reference      | `INSTALL.md`                         |
| All available `.env` settings               | `.env.example` + `INSTALL.md`        |
| How to wire Telegram / Teams                | `INSTALL.md` → "Channel Setup"       |
| Tool execution / SSH allowlist              | `INSTALL.md` → "Tool Execution"      |

## Things NOT to do

- **Do not** suggest cloning or modifying the `nanopilot` platform repo to change
  this agent's bulkhead policy, personality, or skills — everything lives here.
- **Do not** confuse `@bulkhead-ai/core` (what this agent uses) with `bulkhead-mcp`
  (an unrelated MCP proxy).
- **Do not** assume NanoPilot is MCP-based. It talks to the LLM via the Copilot
  SDK / GitHub Models API and runs tools through its own `NANOPILOT_TOOL_EXECUTION`
  pipeline, not JSON-RPC MCP.
