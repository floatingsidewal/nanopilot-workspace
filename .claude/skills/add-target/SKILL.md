---
name: add-target
description: Deploy the agent to a chosen target — local Docker, remote SSH host, Proxmox LXC, or cloud (ACI). Branches on target type; pushes workspace + .env, runs docker compose, verifies health.
user_invocable: true
---

# /add-target — Deploy the Agent to a Target

Pushes the workspace to the chosen runtime and starts it. Assumes:
- `workspace/personality.md` exists (run `/add-persona` first if not)
- `.env` is populated with at least `NANOPILOT_API_KEY` *or* `COPILOT_PROVIDER_BASE_URL` (run `/add-ollama` first for Ollama; or set `NANOPILOT_API_KEY` for GitHub Models)
- `docker-compose.yml` exists at the workspace root

This skill is normally invoked by `/setup-nanopilot`, but can be run standalone to redeploy or move an existing agent.

## Pre-flight

```bash
test -f workspace/personality.md && echo "persona:ok" || echo "persona:MISSING"
test -f .env && grep -E '^(NANOPILOT_API_KEY|COPILOT_PROVIDER_BASE_URL)=.+' .env > /dev/null && echo "env:ok" || echo "env:MISSING"
test -f docker-compose.yml && echo "compose:ok" || echo "compose:MISSING"
```

If any are missing, STOP and tell the user what to fix before retrying.

## Mode Selection

Ask (or accept from `/setup-nanopilot`):

> Where do you want to deploy?
> 1. **local-docker** — run on this machine (requires Docker)
> 2. **remote-ssh** `<user>@<host>` — push to a Linux host over SSH (requires Docker on the remote)
> 3. **proxmox-lxc** `<host> [vmid]` — create or update an LXC on a Proxmox node
> 4. **cloud-aci** — deploy to Azure Container Instances (requires Azure CLI and `infra/` Bicep)

## Mode A: local-docker

```bash
# Pull the platform image
docker compose pull

# Start
docker compose up -d

# Wait briefly for boot
sleep 3
docker compose ps
```

Health check:

```bash
curl -sf http://localhost:3978/health || (docker compose logs --tail 50 daemon && exit 1)
```

If health fails, inspect logs and route to `/debug` (don't try to fix it inline — `/debug` knows the patterns).

## Mode B: remote-ssh

Get the target. Test connectivity FIRST:

```bash
ssh -o ConnectTimeout=5 -o BatchMode=yes <target> "hostname && uname -a"
```

If SSH fails, ask the user to fix it before retrying. Common causes: wrong key, host not reachable, key not accepted.

Detect remote prerequisites:

```bash
ssh <target> "command -v docker > /dev/null && echo docker:ok || echo docker:MISSING"
ssh <target> "command -v docker compose > /dev/null && echo compose:ok || echo compose:MISSING"
```

If Docker is missing, ask consent then install (Debian/Ubuntu pattern):

```bash
ssh <target> "
  apt-get update -qq &&
  apt-get install -y -qq ca-certificates curl gnupg &&
  install -m 0755 -d /etc/apt/keyrings &&
  curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg &&
  chmod a+r /etc/apt/keyrings/docker.gpg &&
  echo \"deb [arch=\$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \$(. /etc/os-release && echo \$VERSION_CODENAME) stable\" > /etc/apt/sources.list.d/docker.list &&
  apt-get update -qq &&
  apt-get install -y -qq docker-ce docker-ce-cli containerd.io docker-compose-plugin &&
  systemctl enable --now docker
"
```

Push the workspace:

```bash
INSTALL_DIR="/opt/<agent-name>"
TARBALL="/tmp/agent-deploy-$$.tar.gz"

# Tarball the workspace (exclude secrets and junk)
tar czf "$TARBALL" \
  --exclude=.DS_Store --exclude=.git --exclude=.envrc \
  -C . .env workspace docker-compose.yml Dockerfile.persona

scp -q "$TARBALL" "<target>:/tmp/agent-deploy.tar.gz"
ssh <target> "
  mkdir -p $INSTALL_DIR &&
  tar xzf /tmp/agent-deploy.tar.gz -C $INSTALL_DIR &&
  chmod 600 $INSTALL_DIR/.env &&
  rm /tmp/agent-deploy.tar.gz
"
rm "$TARBALL"
```

Start:

```bash
ssh <target> "cd $INSTALL_DIR && docker compose pull && docker compose up -d"
```

Health check (allow time for image pull on first run):

```bash
sleep 5
ssh <target> "curl -sf http://localhost:3978/health" || ssh <target> "cd $INSTALL_DIR && docker compose logs --tail 50 daemon"
```

## Mode C: proxmox-lxc

Get the host (`p1.home.lan` etc.) and optional VMID.

Test SSH to the Proxmox node:

```bash
ssh -o ConnectTimeout=5 root@<host> "pveversion"
```

If `pveversion` fails, the host isn't a Proxmox node — flag and stop.

### Auto-pick or use existing VMID

```bash
if [ -z "$VMID" ]; then
  VMID=$(ssh root@<host> "pvesh get /cluster/nextid" | tr -d '"')
  echo "Auto-picked VMID: $VMID"
fi

if ssh root@<host> "pct status $VMID" > /dev/null 2>&1; then
  echo "LXC $VMID exists — will update"
  LXC_EXISTS=true
else
  echo "LXC $VMID does not exist — will create"
  LXC_EXISTS=false
fi
```

### Create LXC if needed

```bash
if [ "$LXC_EXISTS" = "false" ]; then
  TEMPLATE=$(ssh root@<host> "pveam available --section system | grep debian-12-standard | awk '{print \$2}' | sort -V | tail -1")
  ssh root@<host> "ls /var/lib/vz/template/cache/$TEMPLATE > /dev/null 2>&1 || pveam download local $TEMPLATE"

  ssh root@<host> "pct create $VMID local:vztmpl/$TEMPLATE \
    --hostname <agent-name> \
    --memory 1024 --cores 2 \
    --rootfs local-lvm:8 \
    --net0 name=eth0,bridge=vmbr0,ip=dhcp \
    --unprivileged 1 --features nesting=1 \
    --start 1"

  # Wait for network
  for i in $(seq 1 30); do
    CT_IP=$(ssh root@<host> "pct exec $VMID -- hostname -I 2>/dev/null" | awk '{print $1}')
    [ -n "$CT_IP" ] && break
    sleep 2
  done
  echo "Container IP: ${CT_IP:-unknown}"
else
  ssh root@<host> "pct status $VMID" | grep -q running || ssh root@<host> "pct start $VMID"
fi
```

### Install Docker inside the LXC

```bash
ssh root@<host> "pct exec $VMID -- bash -c '
  command -v docker > /dev/null && exit 0
  apt-get update -qq &&
  apt-get install -y -qq ca-certificates curl gnupg &&
  install -m 0755 -d /etc/apt/keyrings &&
  curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg &&
  chmod a+r /etc/apt/keyrings/docker.gpg &&
  echo \"deb [arch=\$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \$(. /etc/os-release && echo \$VERSION_CODENAME) stable\" > /etc/apt/sources.list.d/docker.list &&
  apt-get update -qq &&
  apt-get install -y -qq docker-ce docker-ce-cli containerd.io docker-compose-plugin &&
  systemctl enable --now docker
'"
```

### Push the workspace

```bash
INSTALL_DIR="/opt/<agent-name>"
TARBALL="/tmp/agent-deploy-$$.tar.gz"

tar czf "$TARBALL" \
  --exclude=.DS_Store --exclude=.git --exclude=.envrc \
  -C . .env workspace docker-compose.yml Dockerfile.persona

scp -q "$TARBALL" "root@<host>:/tmp/agent-deploy.tar.gz"
ssh root@<host> "pct push $VMID /tmp/agent-deploy.tar.gz /tmp/agent-deploy.tar.gz"

ssh root@<host> "pct exec $VMID -- bash -c '
  mkdir -p $INSTALL_DIR &&
  tar xzf /tmp/agent-deploy.tar.gz -C $INSTALL_DIR &&
  chmod 600 $INSTALL_DIR/.env &&
  rm /tmp/agent-deploy.tar.gz
'"

ssh root@<host> "rm /tmp/agent-deploy.tar.gz"
rm "$TARBALL"
```

### Start

```bash
ssh root@<host> "pct exec $VMID -- bash -c '
  cd $INSTALL_DIR &&
  docker compose pull &&
  docker compose up -d
'"
```

### Health check

```bash
sleep 5
ssh root@<host> "pct exec $VMID -- curl -sf http://localhost:3978/health" \
  || ssh root@<host> "pct exec $VMID -- bash -c 'cd $INSTALL_DIR && docker compose logs --tail 50 daemon'"
```

## Mode D: cloud-aci

Check `infra/` directory in the workspace for Bicep templates:

```bash
ls infra/*.bicep 2>/dev/null
```

If no Bicep is bundled, tell the user this target requires `infra/main.bicep` (or similar) and stop. Don't fabricate one.

If Bicep is present, ask for:
- Azure subscription ID
- Resource group name (create if missing)
- Region

Then:

```bash
az login                    # if not already
az group create --name <rg> --location <region>
az deployment group create \
  --resource-group <rg> \
  --template-file infra/main.bicep \
  --parameters .env-as-bicep-params  # whatever the template expects
```

Health check post-deploy: query the ACI public IP and `curl /health`.

## After start: log-scan + `/ready` verification (all modes)

Per-mode `curl /health` is a liveness probe — "the process is up." It is
**not enough** to declare deploy success. Two cross-mode checks every mode
must run before returning success:

### 1. Log-scan for known failure patterns

After `docker compose up -d`, the daemon needs ~5 seconds to boot. For the
next ~30 seconds, watch logs for deterministic failure patterns. If any
match, FAIL the deploy and surface the offending log line.

| Pattern | What it means |
|---|---|
| `Persona container exited.*exitCode":(125\|137)` | persona image pull failed or runtime error |
| `Model ".*" is not available` | BYOK misconfigured (#36) or model not pulled on Ollama host |
| `Timed out waiting for host.docker.internal` | extra_hosts scaffold drift (see #34) |
| `pull access denied` | persona image tag missing on registry |

```bash
# Adapt the log command per mode (docker compose directly, or ssh + pct exec)
BAD='Persona container exited.*exitCode":(125|137)|Model ".*" is not available|Timed out waiting for host.docker.internal|pull access denied'
for _ in $(seq 1 6); do
  LOGS=$(<log command for this mode>)
  if echo "$LOGS" | grep -qE "$BAD"; then
    echo "FAIL — known-bad log pattern:"
    echo "$LOGS" | grep -E "$BAD" | tail -5
    exit 1
  fi
  sleep 5
done
echo "ok — no known-bad patterns in first 30s"
```

### 2. `/ready` pipeline probe (new in platform 0.2.x)

```bash
curl -s http://<target>:3978/ready
# HTTP 200 + {"status":"ready"}                  → deploy OK
# HTTP 503 + {"status":"degraded","components":{...}} → STOP, name failing component
```

`/ready` tests database, each channel's `isConnected()`, persona-image
presence, and LLM backend reachability — all pre-message signals. Every
failure mode from #38 surfaces here where `/health` passed cleanly.

**Do not declare success from `/add-target` until both checks pass.**

## Persist deployment-target marker (all modes)

Once both checks pass, write a `NANOPILOT_DEPLOY_TARGET` line to the **local
workspace's** `.env` (the one in the operator's working directory, NOT the
deployed agent's `.env` on the target). `/upgrade-nanopilot` reads this on
its next run to skip the recall-the-target interview.

Format: `<kind>:<host>[:<vmid>]:<install_dir>` — fields after `<kind>` are
colon-delimited; empty positional fields are kept (`::`) so `vmid` is always
in the same slot.

```bash
# Examples — pick whichever shape matches the mode that just succeeded:
TARGET_LINE_LOCAL="local-docker:localhost::$(pwd)"
TARGET_LINE_SSH="remote-ssh:$TARGET_HOST::$INSTALL_DIR"
TARGET_LINE_LXC="proxmox-lxc:$TARGET_HOST:$VMID:$INSTALL_DIR"

# Update or append. Use `sed -i.bak ... && rm .bak` so we don't leave a
# .bak lying around the workspace.
if grep -q '^NANOPILOT_DEPLOY_TARGET=' .env 2>/dev/null; then
  sed -i.bak "s|^NANOPILOT_DEPLOY_TARGET=.*$|NANOPILOT_DEPLOY_TARGET=$TARGET_LINE|" .env
  rm -f .env.bak
else
  printf "\n# Deployment target marker — written by /add-target on first deploy.\n# /upgrade-nanopilot reads this to skip the recall-the-target interview.\n# Format: <kind>:<host>[:<vmid>]:<install_dir>\nNANOPILOT_DEPLOY_TARGET=%s\n" "$TARGET_LINE" >> .env
fi
```

Skip the marker only if the operator explicitly opts out (multi-target
workspaces where one canonical target makes no sense). In that case, surface
"target not persisted; future `/upgrade-nanopilot` calls will need to ask
again" so they're not surprised later.

## Troubleshooting

- **`docker compose pull` 401**: image is private; user needs to `docker login ghcr.io` with a PAT that has `read:packages`.
- **Health endpoint 404**: container started but server isn't binding port 3978. Check `NANOPILOT_PORT` in `.env`.
- **Health endpoint connection refused**: container exited. `docker compose logs daemon` will show why — usually missing `NANOPILOT_API_KEY` or malformed `.env`.
- **LXC create fails — "no such storage"**: the host uses non-default storage names. Run `pvesm status` to see what's available, override `--rootfs <storage>:<gb>`.
- **LXC network never gets IP**: bridge is wrong or DHCP isn't running on that VLAN. Inspect `cat /etc/network/interfaces` on the host or assign a static IP via `--net0 name=eth0,bridge=<br>,ip=<cidr>,gw=<gw>`.

For deeper troubleshooting, route to `/debug`.

## Outputs (for the orchestrator)

When invoked by `/setup-nanopilot`, return:
- `target_kind`: `local-docker | remote-ssh | proxmox-lxc | cloud-aci`
- `target_address`: the host/IP the user should think of as the agent's home
- `health_url`: the curl-able health endpoint
- `manage_command`: a one-liner the user can run later to inspect/restart (e.g., `ssh root@p1.home.lan "pct exec 110 -- docker compose -C /opt/spark logs -f"`)
