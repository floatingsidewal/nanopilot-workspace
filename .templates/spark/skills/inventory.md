---
summary: User asks about VMs, containers, guest lists, VMIDs, or infrastructure state
---

# VM and Container Inventory

## When to use

The user asks about what VMs or containers are running, wants a list of all guests, asks about a specific VMID, or any time you need to understand the current infrastructure state before making changes.

## Prerequisite

If `proxman --version` fails, run `npm install -g @proxman-io/cli@0.2.1`. Should be pre-baked into the spark persona image.

## Instructions

`proxman` is your inventory primitive. Discover the exact subcommand surface with `proxman --help`; the canonical operations you'll typically want:

- `proxman vm list --json` — VMs (substitute the actual subcommand if `--help` shows otherwise).
- `proxman lxc list --json` — LXC containers.
- `proxman vm config --vmid <id> --json` and `proxman lxc config --vmid <id> --json` — single-guest detail.

Always pipe through `jq` to extract only the fields you need. Never print the raw JSON dump back to the channel.

### Maintaining inventory memory

Use `save_memory` to persist what you learn — but **store summaries, not full dumps** (per rules.md). Counts + names + purposes only.

```text
save_memory(key: "host-inventory-summary",
            content: "Cluster has 3 nodes (ark, atlas, abyss). 12 VMs total, 8 LXC. Notable: VM 100 (gateway), VM 105 (maproom), LXC 200 (proxy). Last updated 2026-04-26.")
```

For each guest you discover:

- If you know its purpose (from name, config, or user context), record it.
- If you DON'T know its purpose, flag it and ask the user: "I found VM 105 (maproom) — what is this used for?"
- Update the inventory memory when you learn new information.

Before any lifecycle operation, `read_memory(key: "host-inventory-summary")` first.

### Presenting inventory

If the user asks "what VMs do we have?" — answer with a count and offer to drill down. Do **not** dump the whole table in one message (rules.md "don't enumerate inventory in one message"). When the user asks for the table, present columns: VMID, Name, Type, Status, Purpose. Flag guests using excessive resources, in unexpected states, or with unknown purpose.
