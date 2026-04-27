---
name: home-lan-ca-setup
description: >
  Set up a private Certificate Authority using step-ca on a Proxmox LXC container
  for issuing trusted TLS certificates across a home network. Use this skill
  when the user mentions setting up internal certificates, step-ca, private CA,
  home lab TLS, Proxmox certificates, ACME for internal services, or wants to
  eliminate browser certificate warnings on a local network. Also trigger for
  requests involving self-signed cert replacement, internal PKI, or LXC-based
  certificate infrastructure.
---

# Home LAN CA Setup

Deploy a lightweight private Certificate Authority (step-ca by Smallstep) in a
Proxmox LXC container. Once running, any ACME-capable service on the LAN
(Proxmox, Traefik, Nginx Proxy Manager, etc.) can request and auto-renew
trusted TLS certificates — no internet dependency, no public DNS, no CT log
exposure.

## Tooling note

This skill straddles two surfaces:

- **Cluster API operations** (next-VMID lookup, container create, storage
  queries, ACME account registration on the Proxmox side) — use `proxman`.
  Discover the exact subcommand surface via `proxman --help`.
- **In-LXC commands** (apt install, file edits inside the step-ca container,
  systemd unit creation) — use `ssh root@<node>` from your agent-managed SSH
  key (see the `ssh-keys` skill) and then `pct exec <vmid> -- <cmd>` from
  there. `proxman` does not exec commands inside guests; that's a host-level
  primitive.

If `proxman --version` fails, run `npm install -g @proxman-io/cli@0.2.1`.

## Architecture

```
┌─────────────────────────────────────────────────┐
│  Proxmox Cluster                                │
│                                                 │
│  ┌──────────────┐    ACME     ┌──────────────┐  │
│  │ step-ca LXC  │◄───────────│ Proxmox Node │  │
│  │ :443         │            │ pveproxy      │  │
│  └──────┬───────┘            └──────────────┘  │
│         │                                       │
│         │  ACME    ┌──────────────────────┐     │
│         ├─────────►│ Any other service    │     │
│         │          │ (Traefik, nginx, etc)│     │
│         │          └──────────────────────┘     │
│         ▼                                       │
│  Root CA cert trusted on all LAN devices        │
└─────────────────────────────────────────────────┘
```

## Strategy

This skill follows a **discover-then-build** approach. Every cluster and home
network is different — storage backends, CT ID ranges, network topology, HA
configuration. Rather than assuming defaults, Phase 0 discovers what exists and
Phase 1 confirms the plan with the user before any changes are made.

## Phase 0: Discover cluster state

Before creating anything, gather the following from the target Proxmox host.
Run all discovery commands and present a summary to the user.

### 0.1 Cluster topology

```bash
pvecm status          # Cluster name, node count, quorum status
pvecm nodes           # Node names and IDs
```

Key things to note:
- **2-node clusters** cannot do reliable HA without a QDevice (losing one node
  = losing quorum). Check for QDevice: `corosync-quorumtool -s | grep -i qdevice`.
  If absent, recommend `onboot: 1` instead of HA, and note the QDevice gap.
- **Single-node setups** have no HA concern — just use `onboot: 1`.

### 0.2 Storage

```bash
pvesm status                    # Available storage backends
cat /etc/pve/storage.cfg        # Full storage config with content types
zpool list 2>/dev/null          # ZFS pool health (if any)
```

Pick the best storage for the LXC rootfs:
1. **Prefer ZFS** (`zfspool` type with `rootdir` in content) — gives snapshots,
   replication, and compression for free.
2. **Fall back to LVM-thin** (`lvmthin` type with `rootdir` in content).
3. **Last resort: directory** (`dir` type) — no snapshots.

Record the storage name (e.g., `zfs-storage`, `local-lvm`). Do NOT hardcode
`local-lvm` — it may not exist.

### 0.3 Existing containers and VMs

```bash
pvesh get /cluster/resources --type vm --output-format json
```

Or if `jq` is not available on the host, pipe to a local parser. This reveals:
- Which CT IDs are taken (avoid collisions)
- Which node each CT/VM runs on
- Naming patterns in use

Pick the next available CT ID in the user's range (default: 200+).

### 0.4 Network

```bash
cat /etc/network/interfaces     # Bridge names, node IPs, gateway, DNS
ip -4 addr show vmbr0           # Confirm bridge IP
```

Also inspect an existing container config for conventions:

```bash
pct config <existing_ct_id>     # See nameserver, searchdomain, bridge patterns
```

Record: bridge name, gateway, nameserver(s), searchdomain, subnet mask.

### 0.5 Debian template

```bash
pveam available --section system | grep debian-12
pveam list local | grep debian-12
```

The template version changes over time. Never hardcode it. If not cached,
download it:

```bash
pveam update
pveam download local <exact_template_filename>
```

## Phase 1: Confirm the plan

Present the user with the discovered values and proposed configuration:

| Variable            | Value                | Source                              |
|---------------------|----------------------|-------------------------------------|
| `CT_ID`             | (next available)     | Phase 0.3 — avoid collisions        |
| `CT_IP`             | (user provides)      | Must be a free static IP            |
| `CT_GATEWAY`        | (discovered)         | Phase 0.4 — from network config     |
| `CT_NAMESERVER`     | (discovered)         | Phase 0.4 — from existing CTs       |
| `CT_BRIDGE`         | (discovered)         | Phase 0.4 — usually `vmbr0`         |
| `CT_STORAGE`        | (discovered)         | Phase 0.2 — prefer ZFS              |
| `SEARCHDOMAIN`      | (discovered/user)    | e.g., `home.lan`                    |
| `CA_NAME`           | `Home LAN CA`        | Default, user can override          |
| `CA_DNS`            | `step-ca.<domain>`   | Derived from searchdomain           |
| `CA_PORT`           | `443`                | Default                             |
| `CA_ADMIN_EMAIL`    | `admin@<domain>`     | Derived from searchdomain           |
| `TEMPLATE`          | (discovered)         | Phase 0.5 — exact filename          |
| `DISK_SIZE`         | `2`                  | GB — sufficient for CA data         |
| `MEMORY`            | `256`                | MB — step-ca is lightweight         |
| `HA_MODE`           | `onboot` or `ha`     | Phase 0.1 — based on cluster size   |

Wait for user confirmation or adjustments before proceeding.

## Phase 2: Create the LXC container

```bash
pct create ${CT_ID} local:vztmpl/${TEMPLATE} \
  --hostname step-ca \
  --memory ${MEMORY} \
  --cores 1 \
  --rootfs ${CT_STORAGE}:${DISK_SIZE} \
  --net0 name=eth0,bridge=${CT_BRIDGE},ip=${CT_IP}/24,gw=${CT_GATEWAY} \
  --nameserver ${CT_NAMESERVER} \
  --searchdomain ${SEARCHDOMAIN} \
  --unprivileged 1 \
  --features nesting=1 \
  --onboot 1 \
  --start 1
```

Verify:

```bash
pct status ${CT_ID}
pct exec ${CT_ID} -- ping -c 2 ${CT_GATEWAY}
```

## Phase 3: Install step-ca

All commands run inside the container via `pct exec ${CT_ID} -- bash -c '...'`.

### 3.1 Update and install dependencies

```bash
apt update && apt upgrade -y
apt install -y curl jq
```

### 3.2 Install step-cli and step-ca via .deb packages

The `curl | bash` installer from Smallstep is unreliable in minimal LXC
containers (may produce no output and install nothing). Use the `.deb` packages
directly:

```bash
curl -sL https://dl.smallstep.com/cli/docs-cli-install/latest/step-cli_amd64.deb -o /tmp/step-cli.deb
curl -sL https://dl.smallstep.com/certificates/docs-ca-install/latest/step-ca_amd64.deb -o /tmp/step-ca.deb
dpkg -i /tmp/step-cli.deb /tmp/step-ca.deb
rm /tmp/step-cli.deb /tmp/step-ca.deb
```

### 3.3 Verify

```bash
step-cli version
step-ca version
```

## Phase 4: Initialize the Certificate Authority

### 4.1 Generate a password and initialize

The CA password is high-value; never `echo`, `cat`, or `set -x` it. Generate
into a 0600 file directly and pass via `--password-file`. The `set +x; ...; set
+x` discipline matters even if you don't *think* shell tracing is on — some
LXC environments enable `BASH_XTRACEFD` for systemd journaling.

```bash
set +x
umask 077
openssl rand -base64 32 > /root/ca-password.txt
chmod 600 /root/ca-password.txt

step ca init \
  --name "${CA_NAME}" \
  --provisioner admin \
  --dns ${CA_DNS} \
  --dns ${CT_IP} \
  --address :${CA_PORT} \
  --acme \
  --password-file /root/ca-password.txt
unset CA_PASS  # if any earlier step set it
```

Do NOT save `CA_PASS` to a memory entry, do NOT echo it back to the channel,
and do NOT `cat /root/ca-password.txt` for verification. The only legitimate
audit signal is whether `step ca init` succeeded.

The `--acme` flag enables the ACME provisioner, which allows Proxmox and other
services to request certificates automatically.

Record the root fingerprint from the output.

### 4.2 Move configuration to a system path

```bash
useradd --system --home /etc/step-ca --shell /bin/false step
mv /root/.step /etc/step-ca
mv /root/ca-password.txt /etc/step-ca/password.txt
chown -R step:step /etc/step-ca
chmod 600 /etc/step-ca/password.txt
```

### 4.3 Fix paths in config files

After moving from `/root/.step` to `/etc/step-ca`, all absolute paths in the
config files must be updated:

```bash
sed -i 's|/root/.step|/etc/step-ca|g' /etc/step-ca/config/ca.json
sed -i 's|/root/.step|/etc/step-ca|g' /etc/step-ca/config/defaults.json
```

### 4.4 Set certificate durations for homelab use

Step-ca defaults to 24-hour certificate lifetimes. For homelab use, set
longer durations during initial setup (not as a post-task):

```bash
jq '.authority.claims = {
  "minTLSCertDuration": "24h",
  "maxTLSCertDuration": "2160h",
  "defaultTLSCertDuration": "720h"
}' /etc/step-ca/config/ca.json > /tmp/ca.json.tmp \
  && mv /tmp/ca.json.tmp /etc/step-ca/config/ca.json
chown step:step /etc/step-ca/config/ca.json
```

This gives a 30-day default with a 90-day max.

## Phase 5: Configure as a systemd service

### 5.1 Fix unprivileged port binding

Unprivileged LXC containers cannot bind to ports below 1024 by default. This
is the most common failure mode — step-ca will appear to start but won't
actually be listening.

```bash
echo "net.ipv4.ip_unprivileged_port_start=${CA_PORT}" > /etc/sysctl.d/step-ca.conf
sysctl -p /etc/sysctl.d/step-ca.conf
```

### 5.2 Create the systemd unit

```bash
cat > /etc/systemd/system/step-ca.service << 'EOF'
[Unit]
Description=Smallstep Certificate Authority
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=step
Environment=STEPPATH=/etc/step-ca
ExecStart=/usr/bin/step-ca /etc/step-ca/config/ca.json --password-file=/etc/step-ca/password.txt
Restart=on-failure
RestartSec=10
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
EOF
```

### 5.3 Enable and start

```bash
systemctl daemon-reload
systemctl enable --now step-ca
```

### 5.4 Verify the service is listening

```bash
sleep 2
ss -tlnp | grep ${CA_PORT}
systemctl status step-ca --no-pager
```

If `ss` shows no listener, check `journalctl -u step-ca -e` for errors. Common
causes: wrong paths in `ca.json`, permission denied on key files, port binding
failure (Phase 5.1 missed).

### 5.5 Verify the ACME endpoint

From the Proxmox host (not inside the container):

```bash
curl -sk https://${CT_IP}/acme/acme/directory
```

Should return JSON with `newNonce`, `newAccount`, `newOrder`, etc.

## Phase 6: Distribute the root CA certificate

The root cert is at `/etc/step-ca/certs/root_ca.crt` inside the container.

### 6.1 Pull the cert to the Proxmox host

```bash
pct pull ${CT_ID} /etc/step-ca/certs/root_ca.crt /tmp/home-lan-ca.crt
```

### 6.2 Install on all Proxmox nodes

On the local node:

```bash
cp /tmp/home-lan-ca.crt /usr/local/share/ca-certificates/home-lan-ca.crt
update-ca-certificates
```

On remote cluster nodes, use `StrictHostKeyChecking=accept-new` to handle
nodes that haven't exchanged SSH host keys yet:

```bash
ssh -o StrictHostKeyChecking=accept-new root@<remote_node_ip> \
  'cat > /usr/local/share/ca-certificates/home-lan-ca.crt && update-ca-certificates' \
  < /tmp/home-lan-ca.crt
```

### 6.3 Verify trust

From the Proxmox host, confirm the CA is trusted (no `-k` flag):

```bash
curl -s https://${CT_IP}/acme/acme/directory
```

If this fails with a TLS error, `update-ca-certificates` didn't pick up the
cert. Check that the file has a `.crt` extension in `/usr/local/share/ca-certificates/`.

### 6.4 Client trust

Distribute the root CA cert to client machines so browsers and CLI tools trust
certificates issued by the CA. Copy the cert file off the Proxmox host first
(e.g., `scp root@<proxmox_ip>:/tmp/home-lan-ca.crt .`).

**macOS:**

This requires `sudo` and may prompt for a password interactively. If running
via an agent, ask the user to run this command themselves.

```bash
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain root_ca.crt
```

Verify:

```bash
security find-certificate -a -c "<CA_NAME>" /Library/Keychains/System.keychain
```

**Linux:**

```bash
sudo cp root_ca.crt /usr/local/share/ca-certificates/home-lan-ca.crt
sudo update-ca-certificates
```

**Windows (PowerShell as admin):**

```powershell
Import-Certificate -FilePath root_ca.crt -CertStoreLocation Cert:\LocalMachine\Root
```

**Firefox (all platforms):**

Firefox uses its own certificate store and ignores the OS trust store. Import
via: Settings > Privacy & Security > Certificates > View Certificates >
Authorities > Import.

**Verification (all platforms):**

Open `https://<hostname>.<domain>:8006` in a browser. The connection should
show as trusted with a certificate issued by your CA — no warnings, no
exceptions needed.

## Phase 7: Register Proxmox nodes with the CA

### 7.1 Register ACME account

In a Proxmox cluster, the ACME account configuration is shared across all
nodes via pmxcfs. Register **once** from any node:

```bash
pvenode acme account register default ${CA_ADMIN_EMAIL} \
  --directory https://${CT_IP}/acme/acme/directory
```

Do NOT register again on other nodes — it will fail with "already exists".

### 7.2 Order certificates per node

On each node, set the domain and order:

```bash
pvenode config set --acme domains=<hostname>.${SEARCHDOMAIN}
pvenode acme cert order
```

For remote nodes:

```bash
ssh -o StrictHostKeyChecking=accept-new root@<node_ip> \
  "pvenode config set --acme domains=<hostname>.${SEARCHDOMAIN} && pvenode acme cert order"
```

### 7.3 Set up automatic certificate renewal

Proxmox does **not** create an ACME renewal timer automatically. Without one,
certificates will expire silently (default 30 days with the durations set in
Phase 4.4). Create a systemd timer on **every Proxmox node**.

On each node:

```bash
cat > /etc/systemd/system/pve-acme-renew.service << 'EOF'
[Unit]
Description=Renew Proxmox ACME certificate
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/bin/pvenode acme cert renew
EOF

cat > /etc/systemd/system/pve-acme-renew.timer << 'EOF'
[Unit]
Description=Daily ACME certificate renewal check

[Timer]
OnCalendar=daily
RandomizedDelaySec=3600
Persistent=true

[Install]
WantedBy=timers.target
EOF

systemctl daemon-reload
systemctl enable --now pve-acme-renew.timer
```

For remote nodes:

```bash
ssh -o StrictHostKeyChecking=accept-new root@<node_ip> 'cat > /etc/systemd/system/pve-acme-renew.service << "EOF"
[Unit]
Description=Renew Proxmox ACME certificate
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/bin/pvenode acme cert renew
EOF

cat > /etc/systemd/system/pve-acme-renew.timer << "EOF"
[Unit]
Description=Daily ACME certificate renewal check

[Timer]
OnCalendar=daily
RandomizedDelaySec=3600
Persistent=true

[Install]
WantedBy=timers.target
EOF

systemctl daemon-reload
systemctl enable --now pve-acme-renew.timer'
```

Verify the timer is active:

```bash
systemctl list-timers | grep acme
```

### 7.4 Test renewal

Run a manual renewal to confirm the full cycle works:

```bash
pvenode acme cert renew
```

This should place a new order, validate, download a fresh certificate, restart
pveproxy, and revoke the old certificate.

### 7.5 Verify

Visit `https://<hostname>.<domain>:8006` — the certificate should be trusted
and issued by your CA.

To verify from the command line:

```bash
openssl s_client -connect <node_ip>:8006 -servername <hostname>.<domain> </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

Should show:
- `subject=CN=<hostname>.<domain>`
- `issuer=O=<CA_NAME>, CN=<CA_NAME> Intermediate CA`
- `notAfter` ~30 days from now

## Phase 8: DNS records

Add these records to the local DNS server:

| Record                      | Type | Value       |
|-----------------------------|------|-------------|
| `step-ca.<domain>`          | A    | ${CT_IP}    |
| `<node-hostname>.<domain>`  | A    | <node IP>   |
| (additional services)       | A    | <service IP>|

## Phase 9: Backup

The CA private keys are the crown jewels. **Never write them to a persistent
filesystem path on the Proxmox host** — once they hit `/root/ca-backup/`, an
operator forgetting to delete them leaves long-lived plaintext keys on a
production node.

Use a tmpfs staging directory (memory-backed, vanishes on reboot), copy the
files for the operator to grab over a single short-lived window, then `shred`
and unmount. The operator copies the bundle to offline storage during this
window; nothing persists on the Proxmox host afterward.

```bash
# Mount a tmpfs scratch — files exist only in RAM
STAGE="/dev/shm/ca-backup-$$"
mkdir -p "$STAGE"
chmod 700 "$STAGE"
# (chmod 700 in case /dev/shm has odd perms; this is the operator-only window)

pct pull ${CT_ID} /etc/step-ca/secrets/intermediate_ca_key "$STAGE/intermediate_ca_key"
pct pull ${CT_ID} /etc/step-ca/secrets/root_ca_key        "$STAGE/root_ca_key"
pct pull ${CT_ID} /etc/step-ca/certs/root_ca.crt          "$STAGE/root_ca.crt"
pct pull ${CT_ID} /etc/step-ca/certs/intermediate_ca.crt  "$STAGE/intermediate_ca.crt"
pct pull ${CT_ID} /etc/step-ca/config/ca.json             "$STAGE/ca.json"
pct pull ${CT_ID} /etc/step-ca/password.txt               "$STAGE/password.txt"

# Tell the operator: "I've staged the CA bundle in $STAGE on $NODE for
# offline copy. SCP it now; I will shred and unmount in 5 minutes."

# After the operator confirms they've copied:
shred -u "$STAGE"/*
rmdir "$STAGE"
```

Store the offline bundle on encrypted media (1Password vault, encrypted USB
drive, hardware token). **Not** on another networked machine. **Not** on the
NAS. **Not** in iCloud. The blast radius of a compromised CA private key is
the entire LAN's trust graph.

## Issuing certs for other services

Any ACME-compatible service can use the CA. Point its ACME directory to:

```
https://step-ca.<domain>/acme/acme/directory
```

For services that don't support ACME, issue certs manually:

```bash
step ca certificate "service.<domain>" service.crt service.key \
  --ca-url https://step-ca.<domain> \
  --root /path/to/root_ca.crt
```

## Troubleshooting

| Problem | Check |
|---------|-------|
| step-ca appears to start but isn't listening | Unprivileged port binding. Verify `sysctl net.ipv4.ip_unprivileged_port_start` is <= CA_PORT inside the container. |
| `pvenode acme account register` fails with TLS error | Root CA not trusted on the node. Re-run `update-ca-certificates`. |
| `step-ca` won't start | Check `journalctl -u step-ca -e`. Common: wrong paths in `ca.json` after move to `/etc/step-ca/`. |
| ACME account "already exists" on second node | Expected in a cluster. ACME config is shared via pmxcfs. Only register once. |
| ACME challenge fails | Ensure Proxmox can reach step-ca on port ${CA_PORT}. Check firewall rules. |
| Certificate not trusted in browser | Root CA not imported into the browser/OS trust store. Firefox needs separate import. |
| "authority not found" from step-ca | The ACME provisioner may not be enabled. Check `step ca provisioner list --ca-url https://localhost`. |
| Certs expire too quickly | Adjust `defaultTLSCertDuration` in `ca.json` and restart step-ca. |
| `curl \| bash` installer produces no output | Known issue in minimal LXC containers. Use the `.deb` package method instead. |
| Cross-node SSH fails with host key error | Use `ssh -o StrictHostKeyChecking=accept-new` for first-time connections between cluster nodes. |
| Certificates expire unexpectedly | Proxmox does NOT auto-renew. Verify `systemctl list-timers \| grep acme` shows an active timer on every node. If missing, see Phase 7.3. |
| `pvenode acme cert renew` fails | Check that step-ca is running (`curl -s https://<CT_IP>/acme/acme/directory`), root CA is still trusted, and the node can reach the CA on port 443. |
| macOS `sudo security add-trusted-cert` fails in automation | Requires interactive terminal for password. Ask the user to run it manually. |

## Security notes

- The CA private keys are the crown jewels. If compromised, an attacker can
  issue trusted certs for any domain on your network.
- Keep the LXC container minimal — no unnecessary packages or services.
- Consider restricting network access to the CA container via Proxmox firewall
  rules (only allow ACME traffic from known service IPs).
- The password file (`password.txt`) should only be readable by the `step` user.
- Back up CA keys to offline/encrypted storage, not just another networked machine.
