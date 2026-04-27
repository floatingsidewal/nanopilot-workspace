---
name: setup-nanopilot
description: One-stop bootstrap for a NanoPilot agent — discovers context from invocation/.env, builds a plan, asks the user to confirm, then dispatches to focused sub-skills (add-persona, add-ollama, add-telegram, add-teams, add-target, add-user, configure-access).
user_invocable: true
---

# /setup-nanopilot — Bootstrap a NanoPilot Agent

Single canonical entry point for getting a NanoPilot agent running. Works from a clean clone of `nanopilot-workspace`, or in-place inside the platform repo's `agents/` directory.

This skill is an **orchestrator**. It does the discovery, the plan, the confirmation, and the verification. The actual work — installing Ollama, registering a Telegram bot, building a Proxmox LXC, etc. — is delegated to focused sub-skills.

## Operating Principle

**Discover before you ask.** If the user passed instructions or has populated `.env` / `.envrc`, infer everything you can. Present a plan. Ask only for the gaps. Confirm. Execute.

Only fall back to a full step-by-step interview when discovery yields nothing actionable.

## Stage 0 — Discover

Build a context bundle BEFORE you ask the user anything. Read in this order:

### 0.1 Parse the invocation

Whatever text followed `/setup-nanopilot` is user intent. Examples to recognize:

| User said | Inferred |
|---|---|
| "deploy spark to p1.home.lan with ollama on inferno using gemma2:27b" | persona=spark, target=remote-ssh `p1.home.lan`, inference=ollama-remote `inferno.home.lan`, model=`gemma2:27b` |
| "set up a basic agent locally" | persona=blank/interview, target=local-docker, inference=interview |
| "deploy the spark template to my proxmox cluster" | persona=template `spark` (the Proxmox operator persona; bakes `@proxman-io/cli` into its persona image), target=proxmox-lxc (need host), inference=interview |
| (nothing) | full interview |

### 0.2 Scan the workspace

Run these in parallel — none of them ask the user anything:

```bash
ls -la                          # what's in the workspace root?
ls .templates/ 2>/dev/null      # any prebuilt persona templates available?
ls workspace/ 2>/dev/null       # is there an existing workspace?
cat .env 2>/dev/null            # already-populated environment
cat .envrc 2>/dev/null          # direnv / Keychain layer
cat .env.example 2>/dev/null    # what variables does this workspace expect?
cat workspace/personality.md 2>/dev/null   # is the persona already authored?
```

Build a map of:
- **Persona state**: blank / partially authored / complete / template-available
- **Inference state**: `NANOPILOT_API_KEY` set? `COPILOT_PROVIDER_BASE_URL` set? Pointing where?
- **Channel state**: `TELEGRAM_BOT_TOKEN`? `MICROSOFT_APP_ID`? Allowlists?
- **Target state**: any deployment hints (e.g., `docker-compose.yml` already pointing at a remote registry)

### 0.3 Detect the host environment (orchestrator machine)

```bash
command -v docker && docker info > /dev/null 2>&1 && echo "docker:ok"
command -v gh                                      # for repo creation later
command -v direnv                                  # for keychain-resolved secrets
uname -s                                           # darwin/linux affects ollama install path
```

Note what's missing — you'll either install it (with consent) or route around it.

### 0.4 Fingerprint the target host

**Target mode must come from evidence, not from words.** The invocation is a
hint ("deploy to ark.home.lan"); the host is the source of truth. Probe the
host BEFORE planning so we don't present a `remote-ssh` plan only to discover
it's a Proxmox node.

```bash
HOST="<target host from invocation or .env>"

# Try common SSH users in order — first non-empty unique value that works
for user in "$INVOCATION_USER" "$USER" root; do
  [ -z "$user" ] && continue
  if ssh -o BatchMode=yes -o ConnectTimeout=5 -o StrictHostKeyChecking=accept-new \
       "$user@$HOST" "true" 2>/dev/null; then
    WORKING_USER="$user"; break
  fi
done

# Fingerprint the host role
ssh "$WORKING_USER@$HOST" "
  pveversion 2>/dev/null
  docker info --format '{{.ServerVersion}}' 2>/dev/null
  uname -a
"
```

Map the fingerprint to target_kind:
| Probe result | Target |
|---|---|
| `pveversion` succeeds | **proxmox-lxc** — pick VMID via `pvesh get /cluster/nextid` |
| Only `docker info` works | **remote-ssh** |
| Neither works (Linux, no Docker) | **remote-ssh** with Docker install required — ask consent |
| Host unreachable after all probe attempts | **stop, ask the user** — don't guess |

**SSH-user fallback warning.** When falling back to `root@`, print:

> SSH as `<user>` failed. Falling back to `root@<host>` — required for
> Proxmox (pct/pvesh need root). For regular Linux hosts, consider a
> dedicated service user in a follow-up. Continue? [Y/n]

Don't encourage creating a new service user as part of setup. Proxmox forces
root; that debate is moot for the most common LXC case, and SSH + Docker
group membership for a dedicated user is a system-admin task that depends
on the operator's posture, not our deployment's problem.

### 0.5 Platform-scaffold compatibility check

Read the actual platform image to see which env vars it consumes, and diff
against what the local `.env` / `.env.example` sets. Catches drift between
the scaffold and the platform image.

```bash
VERSION=$(grep -E '^NANOPILOT_VERSION=' .env 2>/dev/null | cut -d= -f2 \
          || echo latest)

# Expected: every env var name the platform references in its compiled config
EXPECTED=$(docker run --rm --entrypoint cat \
  "ghcr.io/floatingsidewal/nanopilot:$VERSION" /app/dist/config.js \
  2>/dev/null | grep -oE 'NANOPILOT_[A-Z_]+|COPILOT_[A-Z_]+' | sort -u)

# Present: what the .env actually declares
PRESENT=$(grep -oE '^[A-Z_]+=' .env 2>/dev/null | tr -d '=' | sort -u)
```

Flag drift cases:
- Var in `EXPECTED` but missing from `PRESENT` AND required (no sensible
  default) → "platform `$VERSION` expects `$VAR` but `.env` doesn't set it"
- Var in `PRESENT` but not `EXPECTED` AND known to be deprecated → "`.env`
  sets `$VAR`; this platform version ignores it (stale from older scaffold)"

Lightweight check. Once the platform API stabilizes this is effectively a
no-op, but the cost of leaving it in is near-zero and the return on catching
one drift bug pre-deploy is large.

## Stage 1 — Build the Plan

From the discovery bundle, fill in this plan template. Mark any field you couldn't infer with `?`.

```
== NanoPilot Setup Plan ==

Persona:        <name> (source: <invocation | template | existing | INTERVIEW>)
Target:         <local-docker | remote-ssh <host> | proxmox-lxc <host> [vmid] | cloud-aci>
Inference:      <github-models | ollama (host:model) | openai | other>
Channels:       <telegram | teams | cli-only>  [+ allowlist status]
Tool execution: <on|off>  (governance: <hosts list>)

Persona image:  <agents/<name>/Dockerfile.persona>
                Extends ghcr.io/floatingsidewal/nanopilot-persona.
                Pre-baked CLIs:  <e.g. @proxman-io/cli@0.2.1>
                Pre-baked apt packages: <e.g. openssh-client (already in base)>
                Built once via: docker compose --profile build build persona
                (See docs/persona-as-agent-runtime.md — the persona container
                IS the agent runtime; bake CLIs the agent will need so it
                doesn't have to install them on first use.)

Will run sub-skills:
  1. /add-persona      (if persona == INTERVIEW or template)
  2. /add-ollama       (if inference == ollama)
  3. /add-target       (always; branches on target)
  4. /add-telegram     (if telegram chosen)
  5. /add-teams        (if teams chosen)
  6. /configure-access (if tool execution == on)
  7. /add-user         (if telegram chosen and allowlist needed)

Gaps I need from you:
  - <every ? above>
  - <plus any secrets that must come from a human, e.g. Telegram bot token>
```

> **Plan-output wording matters.** Do NOT paraphrase "Persona image" as
> "tools image" in the rendered plan — the `nanopilot-tools` image was
> retired 2026-04-26 and the term is misleading. The two-image model
> (daemon + persona) is documented in
> `docs/persona-as-agent-runtime.md`. If the user's pre-existing
> `Dockerfile` is named `Dockerfile` (legacy template) rather than
> `Dockerfile.persona`, flag the rename as a migration item; don't
> just re-use the legacy name.

## Stage 2 — Confirm

Show the plan to the user. Two acceptable responses:

- **"go" / "yes" / "do it" / "looks right"** → proceed to Stage 3.
- **Any correction** → adjust the plan, show the diff, re-ask.

Do NOT proceed until you get explicit confirmation. The plan is a contract.

If there are gaps, ask for them in a single message — not one at a time. Example:

> Before I start, I need:
> 1. Your Proxmox host (one of `p1.home.lan` / `p2.home.lan` / IP)
> 2. Your Telegram bot token (from @BotFather, format `123456789:ABC…`)
> 3. Your numeric Telegram user ID (for the allowlist — send `/chatid` to your bot once it's up, or use @userinfobot)

## Stage 3 — Execute

Run the sub-skills in the order from the plan. Pass the discovered/confirmed context to each so they don't re-ask.

### Branch: Persona

| Source | Action |
|---|---|
| `INTERVIEW` | Invoke `/add-persona` (interview mode — name, role, tone, scope) |
| `template <name>` | Copy `.templates/<name>/personality.md` and `.templates/<name>/skills/*` into `workspace/`; show diff to user |
| `existing` | Skip — read it once for awareness, don't modify |

### Branch: Inference

| Choice | Action |
|---|---|
| `github-models` | Confirm `NANOPILOT_API_KEY` (a GitHub PAT with `read:models`) is in `.env`. If missing, point user at the README's "Getting a GitHub PAT" section. |
| `ollama (local)` | Invoke `/add-ollama` — installs Ollama on this machine, pulls the model, sets `COPILOT_PROVIDER_BASE_URL=http://localhost:11434/v1` and related BYOK vars. |
| `ollama (remote)` | **Do not run `/add-ollama` against a remote host.** Instead: SSH to the host, verify Ollama is running (`curl http://<host>:11434/api/tags`) and the model is pulled. Then write `COPILOT_PROVIDER_BASE_URL=http://<host>:11434/v1`, `COPILOT_PROVIDER_TYPE=openai`, `COPILOT_PROVIDER_API_KEY=ollama`, and `COPILOT_MODEL=<model>` to the agent's `.env`. If Ollama isn't running on the remote host, ask the user whether to abort or to SSH in and run `/add-ollama` from there. |
| `openai / other` | Set `COPILOT_PROVIDER_BASE_URL` and `COPILOT_PROVIDER_API_KEY` from user-provided values. |

### Branch: Target

Invoke `/add-target` with the chosen target. That sub-skill handles:
- `local-docker` → `docker compose up -d` in the workspace dir
- `remote-ssh <host>` → tarball + scp + extract + `docker compose up -d`
- `proxmox-lxc <host> [vmid]` → create LXC if needed, install Docker, push workspace, start
- `cloud-aci` → run the Bicep template under `infra/`

### Branch: Channels

For each channel chosen, invoke its sub-skill:
- `/add-telegram` — register bot with @BotFather (or accept token if user already has one), write `TELEGRAM_BOT_TOKEN`, restart
- `/add-teams` — Entra app + Azure Bot resource + manifest

### Branch: Allowlist (Telegram)

If Telegram is on and an allowlist hasn't been provided:
- Invoke `/add-user` to walk the user through getting their numeric ID and writing it to `NANOPILOT_TELEGRAM_ALLOWLIST`.

### Branch: Tool Execution

If tool execution is on, invoke `/configure-access` to populate `group_config` (allowed hosts, commands, SSH credentials).

## Stage 4 — Verify

Run these in order. Report results inline; don't wait for user acknowledgment
between checks unless one fails. **Do not declare success until every
applicable check passes.** The retro issue (#38) exists because an earlier
version of this skill declared success on `/health` alone and sent the
operator home with a broken pipeline.

### 4.1 Process is running

```bash
docker compose ps                                              # local/remote-ssh
ssh root@<host> "pct exec <vmid> -- docker compose -C /opt/<agent> ps"   # proxmox-lxc
```

### 4.2 Liveness

```bash
curl -sf http://<target>:3978/health    # expect {"status":"ok",...}
```

### 4.3 Readiness (pipeline, not just process)

```bash
curl -s http://<target>:3978/ready
# expect HTTP 200 + {"status":"ready","components":{...}}
# HTTP 503 + {"status":"degraded", ...} → STOP, surface the failing component
```

`/ready` tests `database`, each channel's `isConnected()`, persona-image
presence in local Docker, and LLM backend reachability. This is the single
strongest pipeline-level signal available pre-message; a green `/ready`
catches all four scaffold-era failure modes from #38 (persona pull denied,
host.docker.internal ENOTFOUND, BYOK missing, backend unreachable).

### 4.4 Log scan for known failure patterns

The `/add-target` sub-skill already runs a 30-second log scan after startup
(see `/add-target`'s error-surface section) and fails loudly on known-bad
regex patterns. If `/add-target` reported success, this check has already
passed — but re-verify at the end of Stage 4 as a final gate.

### 4.5 Functional check (if a channel is set up)

Send via the configured channel and confirm a real response (not
`[Mock] Echo:` and not an API error). Expect a five-word reply:
> Say hello in exactly five words.

### 4.6 Tool execution (if enabled)

Send `What's your hostname?` and confirm the agent invokes `execute_command`
against the allowed host and returns its real hostname.

If any check fails, diagnose using:
- `docker compose logs --tail 50 daemon`
- The `/debug` skill (covers common failure modes)

## Stage 5 — Hand-off

Report a short summary, no more than ~10 lines:

```
NanoPilot deployed.

Agent:     <name>
Target:    <where>
Inference: <backend + model>
Channel:   <telegram bot @<username>> (allowlist: <count> users)
Health:    http://<target>:3978/health  ✓
Logs:      <how to tail>
Update:    <how to redeploy>
```

If anything is partially configured (e.g., Telegram works but tool execution is off because the user wanted to add hosts later), call it out as a "next step" — don't claim full completion.

## Failure Modes

- **Discovery yields conflicting signals** (e.g., `.env` has `TELEGRAM_BOT_TOKEN` but invocation says "no telegram") → trust the invocation, but ask the user to confirm the conflict before clearing the existing token.
- **A sub-skill fails partway through** → stop, report what completed, ask the user how to proceed (retry / skip / abort). Do not silently continue.
- **No `.templates/` dir present and user picked a template** → list what IS available; if nothing, fall back to interview persona.
- **User refuses to install a missing tool** (e.g., declines `curl | sh` for Ollama) → don't argue. Document the gap in the plan output and skip the affected branch.

## Examples

```
/setup-nanopilot
```
→ Full discovery + interview. Asks for everything not in `.env`.

```
/setup-nanopilot deploy spark to p1.home.lan with ollama on inferno using gemma2:27b
```
→ Plan inferred from invocation. Likely only asks for: Telegram token (if no `.env`), allowlist user ID. Confirms, then executes.

```
/setup-nanopilot use the spark template, deploy locally, github models is fine
```
→ Persona from `.templates/spark/`, target=local-docker, inference=github-models. Asks for Telegram preferences if not in `.env`.
