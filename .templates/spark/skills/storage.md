---
summary: User asks about storage pools, disk usage, ZFS, LVM, backup storage, or capacity planning
---

# Storage Management

## When to use

The user asks about storage pools, disk usage, ZFS, LVM, backup storage, or capacity planning.

## Prerequisite

If `proxman --version` fails, run `npm install -g @proxman-io/cli@0.2.1`.

## Instructions

Discover the exact subcommand surface via `proxman --help`. The shapes you'll typically use:

- `proxman storage list --json` — pools and capacity.
- `proxman storage content --storage <name> --content backup --json` — backup inventory.
- `proxman vm config --vmid <id> --json` — per-guest disk attachments (look for `scsi*` / `virtio*` / `ide*` / `sata*` fields).

**Capacity alerts to surface:**

- Root filesystem >90%: immediate attention needed.
- Storage pool >85%: plan expansion or cleanup.
- ZFS: keep 20% free for COW operations and scrubs.

**Downloading ISOs or templates:** see `iso-management.md` — must use the operator-vetted URL allowlist (rules.md).

**Cleanup:**

- List old backups via `proxman storage content --content backup --json`, identify outdated entries.
- Deletion: confirm with the user first, every time. Then use the appropriate `proxman storage delete` form.

Never dump the full content list to a channel — summarize counts + oldest/largest, then offer to drill down.
