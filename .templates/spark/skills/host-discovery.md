---
summary: User needs to run commands but API connectivity has not been verified yet
always: true
---

# API Connectivity and Node Discovery

## When to use

The user asks a question that requires running Proxmox commands, AND you have not yet verified API connectivity. Check this by calling `read_memory(key: "managed-hosts-summary")` first — if it returns data, skip this skill and use the node name(s) from memory.

## Prerequisite

If `proxman --version` fails, run `npm install -g @proxman-io/cli@0.2.1`.

## Step 1: Check memory for known nodes

```
read_memory(key: "managed-hosts-summary")
```

If this returns valid node data, use the node name(s) listed there. You're done with discovery.

## Step 2: Verify API connectivity

If no memory exists, verify that `proxman` can reach the Proxmox API:

```bash
proxman version --json
```

If this succeeds, the API connection is working. If it fails with **authentication / permission denied**, you don't have a working credential yet — invoke the `secrets` skill to bootstrap a Proxmox API token. Do NOT ask the user to "fix their setup"; walk them through the token-handshake, store it, then retry this step.

If it fails with a **network error** (host unreachable, DNS), check the operator's deployment notes — `proxman` is configured via env var or config file, see `proxman config --help`.

## Step 3: Discover cluster nodes

```bash
proxman cluster status --json
```

Substitute the actual `--help`-confirmed subcommand if different. Note the node name(s) — needed for all subsequent calls that take `--node <name>`.

## Step 4: Persist a *summary* to memory

Save **counts + names + version + last-checked date**, not a full per-node dump (rules.md):

```
save_memory(key: "managed-hosts-summary",
            content: "Cluster: 3 nodes — ark, atlas, abyss. PVE version: 8.x. Last verified 2026-04-26.")
```

## Step 5: Confirm to user

Tell the user what you found: "Connected to Proxmox VE <version> via proxman. Found node(s): <node list>. I'll remember this for future conversations."
