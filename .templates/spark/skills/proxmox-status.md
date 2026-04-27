---
summary: User asks about node health, cluster status, resource usage, or server health
---

# Cluster and Node Status

## When to use
The user asks about node health, cluster status, resource usage, or general "how's the server doing" questions.

## Prerequisite
If `proxman --version` fails inside this persona container, install:

```bash
npm install -g @proxman-io/cli@0.2.1
```

Normally pre-baked into the spark persona image — falling through to runtime install means a missed rebuild; tell the operator after recovery.

## Instructions

Use your built-in Bash tool. `proxman` returns typed JSON with `--json`; pipe to `jq` for the fields you actually want, never dump the whole payload.

- **Cluster summary:** start with `proxman cluster status --json` (or run `proxman cluster --help` to confirm the exact subcommand). Capture node count, healthy count, quorum state — never the full per-node payload.
- **Single-node deep-dive:** `proxman node status --node <name> --json` (substitute `proxman --help`'s actual subcommand if different).
- **Version / API surface:** `proxman version --json`.

## Key metrics to surface

- CPU usage (%) and load average
- Memory usage vs total (watch for >85%)
- Root filesystem usage (watch for >90%)
- Pending updates / kernel age (alert if last reboot > 90 days while patches pending)

> **Note:** Node names should come from your `managed-hosts` memory if it exists; otherwise run `proxman cluster status --json` once and save the count + names (NOT the full inventory) to memory.

Always present status data in a compact table format. Flag anything concerning. If you have to choose between completeness and concision, pick concision — the rules.md "don't enumerate inventory in one message" rule applies.
