---
name: ssh-keys
summary: Generate, persist, and share an SSH keypair so the operator can grant the agent access to remote hosts.
---

# SSH Key Handshake

You manage your own SSH credentials. The platform does not give you keys
— you generate them, persist them, and share the public half with the
operator (the human running this agent) through the active channel. The
operator administers the public key on the target resource. You then
use ordinary `ssh user@host` commands.

This is a deliberate design: the operator stays in control of access,
and your private key is unique to this deployment.

## Step 1 — Check whether you already have a key

Before generating anything, see whether a previous session already set
this up. The `~/.ssh/` directory is mounted on a persistent volume, so
keys survive container restarts.

```bash
ls -la ~/.ssh/
test -f ~/.ssh/id_ed25519 && echo "private key exists" || echo "no key yet"
test -f ~/.ssh/id_ed25519.pub && cat ~/.ssh/id_ed25519.pub
```

If you see a `.pub` file, you already have a key — skip to Step 3 and
share its public half (or, if it was already administered on the
target you need now, skip to Step 4 and just connect).

## Step 2 — Generate a keypair

Use Ed25519 (smaller, faster, modern default):

```bash
ssh-keygen -t ed25519 -N "" -C "{{AGENT_NAME}}@$(hostname)" -f ~/.ssh/id_ed25519
```

- `-N ""` — no passphrase. The private key is already protected by
  living inside the persona container's persistent volume; an
  additional passphrase would just block automation.
- `-C "..."` — comment for human-readable identification on the
  remote `authorized_keys` file. The hostname is the persona container
  ID, which makes it obvious which deployment a given key belongs to.
- `-f ~/.ssh/id_ed25519` — write to the standard path so subsequent
  `ssh` calls find it without flags.

Verify:

```bash
ls -la ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub
```

## Step 3 — Share the public key with the operator

Send the contents of `~/.ssh/id_ed25519.pub` back to the operator
through the active channel (Telegram, Teams, whatever they're using to
talk to you right now). Frame it clearly so they know what to do with
it.

```bash
echo "---PUBLIC KEY START---"
cat ~/.ssh/id_ed25519.pub
echo "---PUBLIC KEY END---"
```

Then say something like:

> I've generated my SSH identity. Please add the public key above to
> `~/.ssh/authorized_keys` on `<target-host>` for the user
> `<target-user>`, then let me know when it's installed.

**Never** share the contents of `~/.ssh/id_ed25519` (no `.pub`
suffix). That is the private half. If you ever accidentally output it,
treat the key as compromised: regenerate immediately (`rm
~/.ssh/id_ed25519*` then redo Step 2) and report the slip-up.

## Step 4 — Use the key

Once the operator confirms they've installed your public key, connect
normally:

```bash
ssh -o StrictHostKeyChecking=accept-new <target-user>@<target-host> '<command>'
```

`StrictHostKeyChecking=accept-new` lets you accept a new host
fingerprint on first connection while still rejecting changes (which
would indicate a man-in-the-middle). Pin the fingerprint after first
contact:

```bash
ssh-keyscan -t ed25519 <target-host> >> ~/.ssh/known_hosts
```

For repeated calls to the same host, prefer adding a `Host` block to
`~/.ssh/config` so you don't have to keep re-typing flags.

## Step 5 — Rotate when warranted

Rotate your key when:

- The operator says to (compromise, role change, deployment retirement).
- You detect that the private key was ever logged or sent over the
  channel (you should never do this — but if it happens, rotate
  immediately).
- The agent is being moved between operators (new deployment = new key).

Rotation is destructive — old `authorized_keys` entries become
invalid. Coordinate with the operator before rotating.

```bash
# Only after the operator has confirmed the rotation plan:
rm ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub
# Then redo Step 2; share the new pub key per Step 3.
```

## What you do NOT do

- Do not store keys outside `~/.ssh/`. Other paths are not on the
  persistent volume and will vanish on restart.
- Do not relax the `0700` mode on `~/.ssh/` or the `0600` mode on the
  private key. `ssh` and `ssh-keygen` will refuse to use them otherwise.
- Do not generate per-call ephemeral keys. One identity per
  deployment, persisted across restarts.
- Do not paste the operator's keys (e.g. their personal id_ed25519)
  into your authorized_keys or config. The handshake is one-way: you
  share your public key with them, not the other way around.
