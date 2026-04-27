---
name: add-user
description: Onboard a new Telegram user to a deployed NanoPilot agent — allowlist, group_config, verification
user_invocable: true
---

# /add-user — Onboard a Telegram User

## Pre-flight
1. Confirm the target agent is deployed and running (check memory for known deployments, e.g., ProxMan on kraken VMID 106)
2. Confirm the Telegram channel is already configured (`TELEGRAM_BOT_TOKEN` exists in the agent's `.env`)

## Steps

### 1. Get the User's Telegram ID
Ask the operator for the new user's **numeric Telegram user ID**.

- Telegram usernames are mutable and not used for access control — only numeric IDs are reliable
- If the user doesn't know their ID, they can send `/chatid` to the bot — this command is unrestricted and works even for users not on the allowlist
- The ID is a number like `123456789`

### 2. Get Deployment Details
Ask which agent, host, and VMID to target. Check memory for known deployments first.

Example known deployments:
- **ProxMan**: LXC 106 on kraken (192.168.1.243)

Needed values:
- `HOST` — Proxmox host (e.g., `kraken` or its IP)
- `VMID` — LXC container ID (e.g., `106`)
- `APP_DIR` — App directory inside the LXC (typically `/opt/nanopilot`)

SSH pattern: `ssh root@<HOST>` then `pct exec <VMID> -- <command>`, or directly into the LXC if accessible.

### 3. Add to Allowlist
SSH to the host and update `NANOPILOT_TELEGRAM_ALLOWLIST` in the agent's `.env` file.

Read the current value:
```bash
ssh root@<HOST> "pct exec <VMID> -- cat <APP_DIR>/.env" | grep TELEGRAM_ALLOWLIST
```

If `NANOPILOT_TELEGRAM_ALLOWLIST` exists, append the new user ID (comma-separated). If it doesn't exist, create the entry.

Example — updating an existing allowlist:
```bash
ssh root@<HOST> "pct exec <VMID> -- sed -i 's/^NANOPILOT_TELEGRAM_ALLOWLIST=.*/&,<NEW_USER_ID>/' <APP_DIR>/.env"
```

Example — creating a new allowlist entry:
```bash
ssh root@<HOST> "pct exec <VMID> -- bash -c 'echo NANOPILOT_TELEGRAM_ALLOWLIST=<NEW_USER_ID> >> <APP_DIR>/.env'"
```

Verify the change:
```bash
ssh root@<HOST> "pct exec <VMID> -- grep TELEGRAM_ALLOWLIST <APP_DIR>/.env"
```

### 4. Add group_config
The `group_config` table controls what commands and hosts a user can access via tool execution. The group ID for a Telegram DM is the same as the user's Telegram ID.

Ask the operator what access level the new user should have:

**Option A — Clone from existing user**: Copy config from a known user's group_config row.
```bash
ssh root@<HOST> "pct exec <VMID> -- node -e \"
const Database = require('better-sqlite3');
const db = new Database('<APP_DIR>/data/nanopilot.db');
const source = db.prepare('SELECT allowed_hosts, allowed_commands FROM group_config WHERE group_id = ?').get('<SOURCE_GROUP_ID>');
if (!source) { console.log('Source group not found'); process.exit(1); }
db.prepare('INSERT OR REPLACE INTO group_config (group_id, allowed_hosts, allowed_commands) VALUES (?, ?, ?)').run('<NEW_USER_ID>', source.allowed_hosts, source.allowed_commands);
console.log('Cloned config:', source);
db.close();
\""
```

**Option B — Restricted access**: Insert a minimal config.
```bash
ssh root@<HOST> "pct exec <VMID> -- node -e \"
const Database = require('better-sqlite3');
const db = new Database('<APP_DIR>/data/nanopilot.db');
db.prepare('INSERT OR REPLACE INTO group_config (group_id, allowed_hosts, allowed_commands) VALUES (?, ?, ?)').run('<NEW_USER_ID>', '', '');
console.log('Created restricted config for <NEW_USER_ID>');
db.close();
\""
```

Verify the row exists:
```bash
ssh root@<HOST> "pct exec <VMID> -- node -e \"
const Database = require('better-sqlite3');
const db = new Database('<APP_DIR>/data/nanopilot.db');
const row = db.prepare('SELECT * FROM group_config WHERE group_id = ?').get('<NEW_USER_ID>');
console.log(row);
db.close();
\""
```

### 5. Restart the Service
Restart so the updated allowlist takes effect:
```bash
ssh root@<HOST> "pct exec <VMID> -- bash -c 'cd <APP_DIR> && npx tsx setup/index.ts --step restart'"
```

### 6. Verify
Check the startup logs to confirm the allowlist was loaded with the correct count:
```bash
ssh root@<HOST> "pct exec <VMID> -- journalctl -u nanopilot --no-pager -n 20" | grep -i allowlist
```

Expected output should show something like: `Telegram allowlist active (N users)`

Then ask the new user to:
1. Open a chat with the bot in Telegram
2. Tap **Start** or send `/start`
3. Send a test message and confirm the bot responds

## Updating an Existing User
To modify an existing user's access:
- **Change group_config**: Re-run step 4 with `INSERT OR REPLACE` (same commands work for updates)
- **Remove from allowlist**: Edit `.env` to remove their ID from the comma-separated list, then restart

## Troubleshooting
- **User sends messages but bot doesn't respond**: Check the allowlist — their numeric ID must be in `NANOPILOT_TELEGRAM_ALLOWLIST`. Verify with `/chatid`.
- **"Not authorized" or silently ignored**: The allowlist is active but their ID is missing or mistyped. Double-check the ID is numeric, not a username.
- **group_config missing**: Tool execution will fail even if the user is on the allowlist. Ensure step 4 was completed.
- **Restart didn't pick up changes**: Verify the `.env` file was actually modified (`grep ALLOWLIST`). Check that the service restarted cleanly (`journalctl -u nanopilot --no-pager -n 30`).
- **User can chat but tools fail**: The user is on the allowlist (good) but their `group_config` row is missing or has empty `allowed_hosts`/`allowed_commands`. Update the group_config.
- **Wrong user ID**: If you added the wrong ID, remove it from `.env`, delete the `group_config` row (`DELETE FROM group_config WHERE group_id = '<WRONG_ID>'`), and restart.

## Security Reminders
- Never commit user IDs, tokens, or `.env` contents to the repository
- The allowlist is a security boundary — if tool execution is enabled, allowlisted users can run commands on configured hosts
- Prefer restricted access (Option B) for new users until trust is established
