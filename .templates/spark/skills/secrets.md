---
name: secrets
description: >
  How to locate, request, and self-provision API keys, tokens, passwords, and
  SSH access needed to interact with external services. Use this skill whenever
  a task requires authentication. Spark resolves credentials in order; if
  nothing is found, it asks the user for the value or generates one (SSH) and
  guides the user through installing it on the target — then stores the result
  so future invocations succeed automatically.
---

# Secret Resolution and Self-Provisioning

Spark needs credentials for almost everything: Proxmox API tokens, Technitium
DNS tokens, SSH access to nodes, etc. This skill is the single source of truth
for **how Spark obtains them**.

The flow is a chain. Walk it in order — stop at the first method that yields a
working credential. If none do, **provision** the credential (Method 4) instead
of giving up.

---

## Method 1 — macOS Keychain (host / dev)

Generic-password entries on the user's Mac. Service name pattern:
`<service>-<purpose>` in lowercase with hyphens.

```bash
security find-generic-password -s "<service-name>" -w
```

Examples:
- `technitium-api-token`
- `proxmox-api-token`
- `ollama-api-key`

Returns the secret on stdout, exits non-zero if missing.

## Method 2 — Environment variable

Set in `.env` files (loaded by direnv on host, loaded as container env in
production). Variable name = the Keychain service in `UPPER_SNAKE_CASE`:

```bash
echo "${TECHNITIUM_API_TOKEN}"
echo "${PROXMOX_API_TOKEN}"
```

Empty/unset → fall through.

## Method 3 — Shell session

The user may have exported the variable directly:

```bash
printenv | grep -i <service_name>
```

---

## Method 4 — Self-provision (NEW)

If no existing secret is found by Methods 1–3, **don't fail and don't silently
guess**. Provision the credential and store it so this never has to happen
again for the same target.

There are two sub-flows depending on what's missing.

### 4A — Missing API key / token

When a service needs an API token Spark doesn't have:

1. **Tell the user what you need and why**, in one short message:

   > I need a Technitium DNS API token to add the record you asked for. I
   > don't have one stored. You can generate one at
   > `http://dns1.home.lan:5380/#admin/sessions` (Admin → API Tokens →
   > Create Token). Paste it here when ready.

2. **Wait for the user's value.** Don't proceed without it.

3. **Validate before storing.** Make a minimal API call using an
   `Authorization` header — never a `?token=` URL parameter (URL
   parameters land in HTTP access logs and proxy logs, where the token
   leaks). Example:

   ```bash
   curl -sf -H "Authorization: Bearer $value" "<endpoint>/version"
   ```

   If it fails with 401/403, tell the user the token didn't work and
   stop — don't store a bad credential.

4. **Store it where you'll find it next time.** Pick the right store for
   the runtime environment:

   | Runtime | Store | Command |
   |---|---|---|
   | macOS host (dev) | Keychain | `security add-generic-password -a "$USER" -s "<service-name>" -w "<value>" -U` |
   | Linux host / container | Agent env file | append `<UPPER_SNAKE_CASE>=<value>` to the runtime env (`.env` or `/data/secrets.env`, whichever the platform reads). Set `chmod 600`. |

   The `-U` flag on `security` updates an existing entry instead of failing
   on duplicates.

5. **Confirm to the user**, without echoing the value:

   > Stored as `technitium-api-token` in your Keychain. Future Spark
   > sessions will resolve this automatically. Continuing with your
   > original request…

6. **Continue with the task** that needed the credential.

### 4B — Missing SSH access to a remote host

The full SSH key handshake (generate, share-pubkey-via-channel, operator
administers, you verify) lives in the dedicated **`ssh-keys`** skill — read
that and follow it. The high-level flow:

1. Diagnose unreachable vs unauthorized (`ssh-keys` skill, "Step 0").
2. Check for existing `~/.ssh/id_ed25519` (mounted on a persistent volume).
3. Generate one if missing (`ssh-keys` skill, Step 2).
4. Output the public half to the channel and ask the operator to install it
   on the target's `authorized_keys`.
5. Verify with a `ssh ... 'hostname && uname -a'` test.
6. (Optional) `save_memory(key: "ssh-host:<host>-summary", content: "user=<user>, verified=<date>")`.

Do NOT fabricate or copy-paste public keys from elsewhere; they must come
from your own `~/.ssh/id_ed25519.pub`.

---

## Safety rules

- **NEVER log, echo, or print secret values** in user-visible output. Use
  them in commands; show only "stored as `<name>`" confirmations.
- **NEVER write secrets to files inside the workspace** (`workspace/` is
  read-only in production anyway, but be deliberate elsewhere too — no
  hardcoded values in skill files, configs, or anything tracked by git).
- **NEVER read `.env` files directly** (e.g. `cat .env`). The runtime
  injects the values you need as environment variables; reach for them
  via `printenv` / `${VAR}`. Reading `.env` exposes the entire bundle
  including secrets you don't have a legitimate need for in the
  current task.
- **NEVER pass tokens in URL query strings** (`?token=<value>`,
  `?api_key=<value>`). Tokens land in access logs that way. Always use
  HTTP `Authorization` headers (`-H "Authorization: Bearer <value>"`).
- **Validate API credentials before storing them.** Don't persist a bad
  token; the user will get cryptic failures the next session.
- **Public SSH keys are not secrets.** Showing the public key in chat is
  fine and necessary. The `.pub` file content can also go in memory if
  useful.
- **Private SSH keys never leave the agent.** Don't show them, don't copy
  them, don't put them in memory.
- **Ask before destructive operations** that use the credentials —
  deleting DNS records, revoking certificates, changing firewall rules,
  anything that could cause an outage. State the change, the impact, and
  what you recommend instead if applicable.

## Why self-provision instead of asking the user every time

Spark is meant to be a long-running operator. If it has to interrupt the
user every session to re-collect the same Proxmox token, it's noise. The
goal is: ask once → store correctly → never ask again unless the credential
is rotated or revoked.

The user's mental model should be: "I gave Spark access to a thing once,
and now Spark just operates that thing."
