---
name: configure-access
description: Configure tool execution access for a NanoPilot agent group — allowed hosts, commands, and the agent's SSH access via the agent-managed key handshake
user_invocable: true
---

# /configure-access — Configure Group Tool Execution Access

Set up `group_config` (allowed hosts, commands) for a NanoPilot agent group, and walk the operator through the **agent-managed SSH key handshake** when the agent needs SSH access to a remote host.

> **What changed (2026-04-26):** the credential-proxy / `credentials.json` model was retired in favor of agent-managed keys. The agent generates its own SSH keypair on first need (inside its persona container, persisted to a per-persona named volume `nanopilot-ssh-keys-*`) and shares the public half with the operator via the active channel. The operator administers the public key on the target host. There is no `credentials/` or `ssh-keys/` host directory to populate. See `docs/persona-as-agent-runtime.md` for the architectural rationale and the `ssh-keys` workspace skill for the agent-side handshake.

## Pre-flight

1. Read the `.env` file to get `NANOPILOT_DATA_DIR` (default: `./data`).
2. Confirm the database exists at `<DATA_DIR>/nanopilot.db`.
3. **No filesystem credential setup required.** Skip any references in older docs to populating `credentials.json`, `./credentials/`, or `./ssh-keys/`.

## Steps

### 1. Identify the Group

Ask the operator which group to configure. Common patterns:

- **Telegram DM**: `tg:<user_telegram_id>` (e.g., `tg:123456789`)
- **Telegram group**: `tg:<group_chat_id>` (e.g., `tg:-1001234567890`)
- **CLI**: `cli`
- **Teams**: `teams:<conversation_id>`

If unsure, check the database for existing groups:

```bash
sqlite3 <DATA_DIR>/nanopilot.db "SELECT id, name FROM groups;"
```

### 2. Determine the Agent Type

Different agents need different access profiles:

**Spark** (Proxmox management via `proxman`):

- Hosts: Proxmox nodes (e.g., `ark.home.lan`) — only needed for the
  out-of-band shell paths in `home-lan-ca-setup.md` and
  `gpu-passthrough.md`. Most Proxmox operations now happen via
  `proxman` running locally inside the persona container.
- Commands: `proxman`, `jq`, `ssh` (and small utilities like `cat`,
  `ls`, `grep` if your skills shell out to remote hosts).
- Note: `pvesh`, `qm`, `pct` are **not** allowlisted in modern spark —
  the migration to `proxman` (PR #63) removed them.

**Custom agent**: Ask the operator what hosts and command patterns the agent's skills require. Default to the smallest allowlist that lets the skills work.

### 3. Configure the agent's SSH access (when applicable)

If the agent's skills need SSH (e.g. `gpu-passthrough.md` for host-level prereqs, `home-lan-ca-setup.md` for in-LXC ops, anything `ssh user@host`-like), the **agent generates its own keypair** the first time it tries to use SSH. The flow:

1. Tell the operator: "When the agent first needs SSH to `<host>`, it will generate an Ed25519 keypair inside its persona container and post the public key to this channel. You'll install that public key on the target's `~/.ssh/authorized_keys`."
2. The agent's `ssh-keys` skill walks the agent through generate → cat-pubkey → wait-for-operator-confirm → verify. Don't pre-populate keys for the agent.
3. Once the operator has installed the public key on the target, the agent can `ssh user@host '<cmd>'` directly via its built-in Bash tool. Governance gating happens via the SDK permission handler against `allowed_hosts` / `allowed_commands` (step 4 below).

> **Don't try to give the agent your own SSH key.** Per-deployment unique keys are intentional — compromising one deployment doesn't compromise others. The handshake is one-way: agent shares its public key with the operator, never the other direction.

### 4. Set Up group_config

The `group_config` table stores JSON arrays for hosts and commands. Governance enforcement happens in `src/backends/sdk-tools.ts` (`buildPermissionHandler`) when the SDK's built-in Bash tool fires a `kind: 'shell'` PermissionRequest.

```bash
sqlite3 <DATA_DIR>/nanopilot.db "
INSERT INTO groups (id, name) VALUES ('<GROUP_ID>', '<GROUP_NAME>')
  ON CONFLICT(id) DO NOTHING;
INSERT INTO group_config (group_id, allowed_hosts, allowed_commands)
  VALUES ('<GROUP_ID>', '<HOSTS_JSON>', '<COMMANDS_JSON>')
  ON CONFLICT(group_id) DO UPDATE SET
    allowed_hosts = excluded.allowed_hosts,
    allowed_commands = excluded.allowed_commands,
    updated_at = datetime('now');
"
```

Example for spark:

```bash
sqlite3 <DATA_DIR>/nanopilot.db "
INSERT INTO groups (id, name) VALUES ('tg:123456789', 'Spark DM')
  ON CONFLICT(id) DO NOTHING;
INSERT INTO group_config (group_id, allowed_hosts, allowed_commands)
  VALUES ('tg:123456789',
    '[\"ark.home.lan\"]',
    '[\"proxman\",\"jq\",\"ssh\",\"cat\",\"ls\",\"grep\",\"hostname\",\"uname\"]')
  ON CONFLICT(group_id) DO UPDATE SET
    allowed_hosts = excluded.allowed_hosts,
    allowed_commands = excluded.allowed_commands,
    updated_at = datetime('now');
"
```

### 5. Verify

Confirm the config was written:

```bash
sqlite3 <DATA_DIR>/nanopilot.db "SELECT group_id, allowed_hosts, allowed_commands FROM group_config WHERE group_id = '<GROUP_ID>';"
```

### 6. Test

Start NanoPilot with tool execution enabled and trigger a command. Send a message that exercises the agent's skills (e.g. "What's the cluster status?" for spark) and verify:

- The agent's relevant skills fire.
- The SDK Bash tool runs `proxman` (or whatever the skill calls) inside the persona container.
- `buildPermissionHandler` approves the call against the allowlist; check daemon logs for `Bulkhead tool-result scan complete` and the absence of `denied-interactively-by-user` entries.
- Response contains real system data.

If the agent needs SSH for the first time, follow the `ssh-keys` skill flow on the agent's side and confirm with the operator.

## Access Profile Templates

### Spark (full Proxmox via proxman + occasional ssh)

```json
{
  "allowed_hosts": ["ark.home.lan"],
  "allowed_commands": ["proxman","jq","ssh","cat","ls","df","free","uptime","grep","hostname","uname"]
}
```

### Read-only observer (proxman queries only, no shell-out)

```json
{
  "allowed_hosts": [],
  "allowed_commands": ["proxman list","proxman vm list","proxman lxc list","proxman storage list","cat","ls","df","free","uptime"]
}
```

### No tool access (chat only)

```json
{
  "allowed_hosts": [],
  "allowed_commands": []
}
```

## Troubleshooting

- **"No hosts configured" / `denied-interactively-by-user`**: `allowed_hosts` is empty or the `group_config` row doesn't exist for this group.
- **"Host not in allowed hosts"**: The host the agent tried to SSH to doesn't match any entry — check for IP vs hostname mismatch.
- **"Command not permitted"**: The command prefix doesn't match any `allowed_commands` pattern. Check daemon logs for the rejected command and update the allowlist.
- **SSH permission denied**: The agent's public key isn't installed on the target's `authorized_keys`. The agent should run its `ssh-keys` skill again (it will detect the existing key, re-share the public half) and the operator administers it.
- **`group_config` table missing**: Start the daemon once to auto-create the schema.
