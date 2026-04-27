---
name: upgrade-nanopilot
description: Upgrade a deployed NanoPilot agent to a newer platform image. Pulls the platform daemon image and rebuilds the agent's persona image (the persona container IS the agent runtime — see docs/persona-as-agent-runtime.md), diffs env vars against the new platform, recreates the daemon container, verifies with /ready and a log-scan, rolls back on failure. Reads NANOPILOT_DEPLOY_TARGET to skip the recall-the-target interview, and detects/auto-migrates legacy workspace shapes (pre-PR-#63 with the retired tools container). Use when the operator says "upgrade", "pull latest", "update the agent", or after a platform release.
user_invocable: true
---

# /upgrade-nanopilot — Upgrade a Deployed Agent

Take a running deployment from the current platform image to a newer one (or to a fresh pull of the same tag).

NanoPilot deployments today have **two images**: the platform daemon (`ghcr.io/floatingsidewal/nanopilot`) and a per-agent persona image that extends `ghcr.io/floatingsidewal/nanopilot-persona`. The persona container is the full agent runtime — bash, npm, ssh, agent-managed credentials — documented in `docs/persona-as-agent-runtime.md` in the platform repo. The former third image (`nanopilot-tools`) was retired 2026-04-26.

**`docker compose pull && up -d` alone is not safe.** It:

- Only upgrades the daemon image — the agent's persona image stays cached stale, so any pinned CLI versions (e.g. `@proxman-io/cli`) inside it don't pick up the new platform's persona-base updates.
- Silently accepts new required env vars until a user message surfaces them.
- Only checks `/health` (liveness), not `/ready` (pipeline).
- Has no rollback if the new image is broken.

This skill fixes all of that.

## Stage 0 — Discover where the agent is deployed

The fastest path is reading the deployment marker that `/add-target` writes to the workspace's `.env` on first deploy:

```bash
grep -E '^NANOPILOT_DEPLOY_TARGET=' .env
# Format: <kind>:<host>[:<vmid>][:<install_dir>]
# Examples:
#   local-docker:localhost::/Users/brad/git/spark-workspace
#   remote-ssh:p1.home.lan::/opt/spark
#   proxmox-lxc:ark.home.lan:110:/opt/spark
```

If present, parse it into `TARGET_KIND`, `TARGET_HOST`, `TARGET_VMID`, `INSTALL_DIR` and skip the interview. If absent (older deployments, or the operator deployed manually), fall back to the interview:

| Target kind | Transport prefix for remote commands |
|---|---|
| `local-docker` | run commands directly |
| `remote-ssh <target>` | `ssh <target> "<cmd>"` |
| `proxmox-lxc <host> <vmid>` | `ssh root@<host> "pct exec <vmid> -- <cmd>"` |

Install dir is typically `/opt/<agent-name>` for ssh/lxc, or the workspace root for local-docker. Below, **TX:** means "apply the appropriate transport prefix for this target."

After a successful interview, **persist the resolved target back to `.env`** so future runs skip the interview:

```bash
grep -q '^NANOPILOT_DEPLOY_TARGET=' .env \
  || printf "\n# Deployment target marker — written by /upgrade-nanopilot or /add-target\nNANOPILOT_DEPLOY_TARGET=%s:%s:%s:%s\n" \
       "$TARGET_KIND" "$TARGET_HOST" "${TARGET_VMID:-}" "$INSTALL_DIR" >> .env
```

## Stage 1 — Detect workspace shape (legacy → current)

PR #63 (2026-04-26) retired the `nanopilot-tools` container and changed the agent template's compose / Dockerfile shape. Workspace mirrors created before PR #63 still carry the old shape (a `tools` build service, a top-level `Dockerfile` extending `nanopilot-tools`, host-mounted `./credentials` and `./ssh-keys` directories). Stage 3 will silently rebuild the dead-tools image against the legacy base unless we migrate first.

### 1.1 Detect

```bash
TX: cd <INSTALL_DIR> && {
  echo "--- workspace shape report ---"
  grep -qE '^\s*tools:\s*$' docker-compose.yml      && echo "LEGACY: tools build service present"
  grep -qE '^\s*persona:\s*$' docker-compose.yml    && echo "current: persona build service present"
  test -f Dockerfile && ! test -f Dockerfile.persona && echo "LEGACY: Dockerfile (not Dockerfile.persona)"
  test -f Dockerfile.persona                         && echo "current: Dockerfile.persona"
  grep -qE 'nanopilot-tools' Dockerfile* 2>/dev/null && echo "LEGACY: FROM nanopilot-tools"
  grep -qE './credentials:' docker-compose.yml       && echo "LEGACY: ./credentials host mount"
  grep -qE './ssh-keys:' docker-compose.yml          && echo "LEGACY: ./ssh-keys host mount"
  grep -qE 'NANOPILOT_PERSONA_CONTAINER_IMAGE.*\$\{NANOPILOT_VERSION' docker-compose.yml \
                                                     && echo "current: persona image pinned to NANOPILOT_VERSION"
}
```

If any line starts with `LEGACY:`, the workspace is on the pre-PR-#63 shape. Surface the report to the operator and ask whether to auto-migrate (Stage 2) or proceed with the legacy shape (Stage 3 will produce a warning but still works because the daemon doesn't actually spawn the dead-tools container).

### 1.2 Decide

Three operator options:

- **Migrate now** → Stage 2 below, then continue.
- **Skip migration, proceed with legacy** → Stage 3 onwards. Surface "wasted build cycles for the dead `tools` service" as a known cost; offer to migrate next time.
- **Cancel** → exit without changes; the operator may want to migrate manually using the migration-notes block in `docs/CHANGELOG.md`.

## Stage 2 — Auto-migrate workspace shape (opt-in)

Only run on operator confirm. Default to a **dry run** that prints the diff first.

### 2.1 Generate the migration patch (dry run)

```bash
TX: cd <INSTALL_DIR>

# Rename Dockerfile → Dockerfile.persona; update FROM line.
[ -f Dockerfile ] && [ ! -f Dockerfile.persona ] && {
  cp Dockerfile Dockerfile.persona.tmp
  sed -i.bak \
    -e 's|ghcr\.io/floatingsidewal/nanopilot-tools|ghcr.io/floatingsidewal/nanopilot-persona|g' \
    Dockerfile.persona.tmp
  diff -u Dockerfile Dockerfile.persona.tmp || true
}

# Patch docker-compose.yml: replace `tools` build service with `persona`,
# drop ./credentials and ./ssh-keys host mounts, add persona image pin.
# Use a temp file + diff so the operator sees what will change.
cp docker-compose.yml docker-compose.yml.tmp
sed -i.bak \
  -e 's|^\(\s*\)tools:|\1persona:|' \
  -e 's|<AGENT>-tools:latest|<AGENT>-persona:latest|g' \
  -e 's|dockerfile: Dockerfile$|dockerfile: Dockerfile.persona|' \
  -e '/- \.\/credentials:/d' \
  -e '/- \.\/ssh-keys:/d' \
  docker-compose.yml.tmp

diff -u docker-compose.yml docker-compose.yml.tmp || true
```

Show the operator the diff (both files) and ask: "Apply this migration?"

### 2.2 Apply (only on operator confirm)

```bash
TX: cd <INSTALL_DIR>
[ -f Dockerfile.persona.tmp ] && mv Dockerfile.persona.tmp Dockerfile.persona && rm -f Dockerfile.bak Dockerfile
mv docker-compose.yml.tmp docker-compose.yml
rm -f docker-compose.yml.bak

# If the persona image pin isn't already in docker-compose.yml, add it.
# (The template now includes this; older mirrors won't have it.)
grep -q 'NANOPILOT_PERSONA_CONTAINER_IMAGE' docker-compose.yml \
  || cat <<'EOF' >> docker-compose.yml
# (Migration note: if the daemon service above doesn't yet set
# NANOPILOT_PERSONA_CONTAINER_IMAGE explicitly, add it under environment:
#   - NANOPILOT_PERSONA_CONTAINER_IMAGE=ghcr.io/floatingsidewal/nanopilot-persona:${NANOPILOT_VERSION:-latest}
# so the persona tag follows NANOPILOT_VERSION.)
EOF

# The bind-mounted ./credentials and ./ssh-keys directories on the host can
# be retired, but DO NOT delete them automatically — they may contain the
# operator's only copy of an SSH key the agent was using. Tell the operator:
# "Your <INSTALL_DIR>/credentials and <INSTALL_DIR>/ssh-keys directories
# are no longer mounted. Verify the agent has its own ~/.ssh/id_ed25519 in
# the nanopilot-ssh-keys-* named volume (run the ssh-keys skill against it
# to regenerate if not), then archive the host directories at your leisure."
```

### 2.3 Verify migration

```bash
TX: cd <INSTALL_DIR> && {
  test -f Dockerfile.persona && ! test -f Dockerfile && echo "OK: Dockerfile.persona only"
  grep -qE '^\s*persona:\s*$' docker-compose.yml && echo "OK: persona service present"
  ! grep -qE '^\s*tools:\s*$' docker-compose.yml && echo "OK: tools service gone"
}
```

If anything fails, STOP — restore `Dockerfile` and `docker-compose.yml` from `.bak` and surface to the operator.

## Stage 3 — Record rollback anchors

Before touching the running deployment, record the currently-deployed image IDs so we can restore them if the new image is broken.

```bash
TX: docker image inspect --format '{{.RepoTags}} {{.Id}}' \
      ghcr.io/floatingsidewal/nanopilot:latest \
      ghcr.io/floatingsidewal/nanopilot-persona:latest \
      <agent-name>-persona:latest \
      > /tmp/nanopilot-upgrade-rollback.txt 2>&1 || true

TX: cp <INSTALL_DIR>/.env <INSTALL_DIR>/.env.pre-upgrade
```

Per-persona named volumes (`nanopilot-session-state-*`, `nanopilot-ssh-keys-*`) are managed by `PersonaContainerManager` and survive `--force-recreate` automatically — no explicit credential backup needed.

## Stage 4 — Compat check (new env vars)

Reuse the logic from `/setup-nanopilot` Stage 0.5: pull the NEW image, read its `config.js`, diff referenced env var names against the live `.env`. If any new required var is missing, STOP and prompt the operator.

```bash
NEW_VERSION=$(TX: grep -E '^NANOPILOT_VERSION=' <INSTALL_DIR>/.env | cut -d= -f2 || echo latest)
TX: docker pull ghcr.io/floatingsidewal/nanopilot:$NEW_VERSION

EXPECTED=$(TX: docker run --rm --entrypoint cat \
  ghcr.io/floatingsidewal/nanopilot:$NEW_VERSION /app/dist/config.js \
  | grep -oE 'NANOPILOT_[A-Z_]+|COPILOT_[A-Z_]+' | sort -u)

PRESENT=$(TX: grep -oE '^[A-Z_]+=' <INSTALL_DIR>/.env | tr -d '=' | sort -u)
```

Report additions and deprecations to the operator; get explicit consent to proceed if anything changed.

## Stage 5 — Pull both images

The daemon spawns the persona container via the mounted Docker socket. `docker compose pull` handles only the daemon. The agent's persona image extends `nanopilot-persona`, so the base persona image needs an explicit `docker pull` and the agent persona needs a rebuild with `--pull` to pick up the new base.

```bash
# Daemon — via compose (respects NANOPILOT_VERSION)
TX: cd <INSTALL_DIR> && docker compose pull

# Persona base — referenced by the agent's Dockerfile.persona FROM line
TX: docker pull ghcr.io/floatingsidewal/nanopilot-persona:$NEW_VERSION

# Agent persona — rebuild on top of the freshly-pulled base
TX: cd <INSTALL_DIR> && docker compose --profile build build --pull persona
```

## Stage 6 — Recreate

```bash
TX: cd <INSTALL_DIR> && docker compose up -d --force-recreate
```

Per-persona named volumes survive recreate. Nothing else to do here.

## Stage 7 — Verify

### 7.1 Liveness

```bash
TX: curl -sf http://localhost:3978/health
# expect {"status":"ok", ...}
```

### 7.2 Readiness (pipeline)

```bash
TX: curl -s http://localhost:3978/ready
# HTTP 200 + {"status":"ready"}                 → continue
# HTTP 503 + {"status":"degraded","components":...} → GO TO Stage 8
```

### 7.3 Log-scan for known failure patterns

Run the same 30-second log-scan as `/add-target`:

| Pattern | What it means |
|---|---|
| `Persona container exited.*exitCode":(125\|137)` | persona image pull failed or runtime error |
| `Model ".*" is not available` | BYOK misconfigured |
| `Timed out waiting for host.docker.internal` | extra_hosts scaffold drift |
| `pull access denied` | persona image tag missing on registry |
| `image not found` | local agent persona was never built (run `docker compose --profile build build persona`) |

Any match → GO TO Stage 8.

## Stage 8 — Rollback on failure

Restore the previously-deployed images by retagging the recorded IDs:

```bash
# For each service, retag the OLD image ID as the expected tag
TX: docker tag <OLD_DAEMON_ID>          ghcr.io/floatingsidewal/nanopilot:$NEW_VERSION
TX: docker tag <OLD_PERSONA_BASE_ID>    ghcr.io/floatingsidewal/nanopilot-persona:$NEW_VERSION
TX: docker tag <OLD_AGENT_PERSONA_ID>   <agent-name>-persona:latest

TX: mv <INSTALL_DIR>/.env.pre-upgrade <INSTALL_DIR>/.env
TX: cd <INSTALL_DIR> && docker compose up -d --force-recreate

# Confirm rollback itself is healthy before declaring done
TX: curl -sf http://localhost:3978/ready
```

STOP and surface the failure. Don't retry the upgrade; the reason needs to be understood before the next attempt — usually a compat-check miss, a new required env var, or a runtime regression the test suite didn't catch.

## Stage 9 — Report

Success output (short, ~10 lines):

```
NanoPilot upgraded.

Agent:     <name>
Target:    <where>           (from NANOPILOT_DEPLOY_TARGET)
From:      daemon  <old daemon ID, short>   →  persona <old agent persona ID, short>
To:        daemon  <new daemon ID, short>   →  persona <new agent persona ID, short>
Migrated:  workspace shape  pre-PR-#63 → current   (if Stage 2 ran)
Verified:  /health ok | /ready 200 (db+telegram+personaImage+backend all green) | log-scan clean (30s)
Rollback:  <INSTALL_DIR>/.env.pre-upgrade  +  /tmp/nanopilot-upgrade-rollback.txt
           — remove after a day of successful operation
```

Offer (don't force): `docker image prune -f` to reclaim space from the previous-version layers + the orphaned `nanopilot-tools:*` images on legacy-shape deployments. Only after the operator has confirmed the upgrade is stable enough to discard the rollback point.

## Scope / out of scope

- **Does not pin a version.** `NANOPILOT_VERSION=latest` is the scaffold default; operators who want pinning set `NANOPILOT_VERSION=0.2.1` in `.env` themselves before invoking this. That's an operator decision.
- **Does not migrate data or handle schema changes.** If a release introduces a breaking schema change, release notes are the right place to announce it — this skill handles upgrade mechanics only.
- **Does not change channel configuration.** Telegram/Teams tokens, allowlists, `group_config` rows are untouched.
- **Does not delete legacy `./credentials/` or `./ssh-keys/` host directories** even after Stage 2 migrates the workspace shape. The operator's manual review/archive step.

## Failure modes

- **Stage 0 deployment-target marker is wrong (host moved, VMID reassigned)** → fall back to interview, persist the corrected marker after the upgrade succeeds.
- **Stage 2 migration patch fails to apply cleanly** → the workspace likely has hand-edited compose entries the regex doesn't anticipate. Restore from `.bak`, surface to operator, fall through to legacy-shape proceed.
- **Compat check flags required new var, operator declines** → abort, surface the var with a pointer to `docs/ENV.md` (or the PR that introduced it), leave the running deployment untouched.
- **Stage 5 pull fails (registry 401 / 404)** → abort, leave running deployment untouched. Usually the new tag doesn't exist yet or the user's Docker login expired.
- **Stage 7 verification fails** → roll back per Stage 8, surface the failing component / log line.
- **Rollback itself fails** → red alert. The previous image IDs are in `/tmp/nanopilot-upgrade-rollback.txt`; route the operator to `/debug` with that file and the new daemon's logs.
