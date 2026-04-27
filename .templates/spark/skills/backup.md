---
summary: User asks about backups, snapshots, restore operations, or disaster recovery
---

# Backup and Restore

## When to use

The user asks about backups, snapshots, restore operations, or disaster recovery.

## Prerequisite

If `proxman --version` fails, run `npm install -g @proxman-io/cli@0.2.1`.

## Instructions

Discover argv via `proxman --help`. Typical shapes:

- `proxman backup run --vmid <id> --storage <name> --mode snapshot --compress zstd --json`
- `proxman backup list --storage <name> --json`
- `proxman backup restore --archive <volid> --target-vmid <newid> --target-storage <name> --json`
- `proxman backup delete --volid <volid> --json` (confirmation required)
- `proxman backup schedule list --json` / `proxman backup schedule show --id <jobid> --json`

Modes when running a backup:

- `snapshot` — no downtime, recommended for running guests.
- `suspend` — brief pause; safer than snapshot for VMs without good guest agents.
- `stop` — full stop; only for offline windows.

### Snapshot vs backup

| Feature | Snapshot | Backup |
|---------|----------|--------|
| Speed | Instant | Minutes-hours |
| Storage | Same pool | Separate storage |
| Portability | No | Yes (file) |
| Purpose | Quick rollback point | Disaster recovery |

### Long-running operations

Backups and restores can take minutes to hours. `proxman` returns a task ID; poll status:

- `proxman task status --upid <upid> --json`
- `proxman task list --node <name> --json`

### Safety

- Always verify a backup by checking its size + listing contents before relying on it.
- Test restores periodically. Untested backups are not backups.
- Snapshot mode is preferred for running guests.
- Restoring overwrites the target VMID if it exists — always use a fresh VMID.
- Deletion of a backup requires explicit user confirmation, every time.
