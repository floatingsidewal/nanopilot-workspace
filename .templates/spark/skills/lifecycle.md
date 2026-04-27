---
summary: User wants to create, clone, snapshot, resize, migrate, start, stop, or delete VMs or containers
---

# Guest Lifecycle Management

## When to use

The user wants to create, clone, snapshot, resize, migrate, start, stop, or delete VMs or containers.

## Prerequisite

If `proxman --version` fails, run `npm install -g @proxman-io/cli@0.2.1`.

## Instructions

`proxman` covers the full guest lifecycle through typed JSON commands. **Always discover exact argv with `proxman --help` and `proxman <subcommand> --help`** rather than guessing. The shapes you'll typically use (substitute the real `--help`-confirmed forms):

- `proxman vm create ... --json` / `proxman lxc create ... --json`
- `proxman vm start --vmid <id> --json` / `proxman lxc start --vmid <id> --json`
- `proxman vm shutdown --vmid <id> --json` (graceful) / `proxman vm stop --vmid <id> --json` (hard)
- `proxman vm clone --source <id> --target <newid> --name <name> --full --json`
- `proxman vm snapshot --vmid <id> --name <snapname> --json`
- `proxman vm resize --vmid <id> --disk scsi0 --size +10G --json`

**Always ask the user to confirm before destructive operations** (stop, delete, rollback, resize-down).

### Creating a new VM or container — guard rails

Before creating any guest:

1. Run `proxman --help` to find the next-id subcommand (e.g. `proxman cluster nextid --json`) — **NEVER** pick a VMID yourself.
2. Run `proxman vm list --json` and `proxman lxc list --json` to confirm the chosen ID is free.
3. If a VMID is taken, use the next available — **NEVER** suggest deleting an existing guest to free the ID.

### Safety checklist before ANY destructive operation

1. Run inventory first — know what the guest is and what it's used for (read your `host-inventory-summary` memory).
2. Is the guest stopped? If running, explain the impact of stopping it.
3. Is there a recent backup or snapshot?
4. Are there dependent services that will be affected?
5. Has the user **explicitly** confirmed with full understanding of what will happen?

### NEVER do these without explicit confirmation

- `proxman vm delete` / `proxman lxc delete` — permanent.
- Snapshot rollback — loses all changes since the snapshot.
- Hard stop — prefer graceful shutdown.
- Overwriting an existing guest's config.

After any destructive operation completes, update the `host-inventory-summary` memory.
