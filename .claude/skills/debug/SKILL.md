---
name: debug
description: Troubleshoot NanoPilot — check logs, service status, database, channel connectivity
user_invocable: true
---

# /debug — NanoPilot Troubleshooting

## Quick Diagnostics

### 1. Service Status
```bash
npx tsx setup/index.ts --step status
```

### 2. Health Check
```bash
curl -s http://localhost:3978/health | jq .
```

Expected: `{"status":"ok","channels":["teams","telegram"]}`

### 3. Verbose Logs
```bash
NANOPILOT_LOG_LEVEL=debug npm run server
```

### 4. Database Inspection
```bash
sqlite3 data/nanopilot.db
```

Useful queries:
```sql
-- Recent messages
SELECT id, group_id, role, substr(content, 1, 80), processed, created_at
FROM messages ORDER BY created_at DESC LIMIT 20;

-- Unprocessed messages (stuck in queue)
SELECT * FROM messages WHERE processed = 0;

-- Channel contexts
SELECT cc.*, m.group_id FROM channel_contexts cc
JOIN messages m ON cc.message_id = m.id
ORDER BY cc.created_at DESC LIMIT 10;

-- Groups
SELECT * FROM groups;

-- Scheduled tasks
SELECT * FROM scheduled_tasks;
```

## Common Issues

### Backend / LLM

| Symptom | Cause | Fix |
|---------|-------|-----|
| "Mock backend" in logs | No NANOPILOT_API_KEY or API endpoint | Set NANOPILOT_API_KEY in .env |
| "unknown_model" errors | Model not available | Check COPILOT_MODEL, try different model |
| Timeout on backend query | API endpoint unreachable | Check COPILOT_PROVIDER_BASE_URL, test with curl |

### Teams Channel

| Symptom | Cause | Fix |
|---------|-------|-----|
| 401 on /api/messages | JWT validation failed | Verify MICROSOFT_APP_ID, PASSWORD, TENANT_ID match Entra app |
| Bot doesn't respond | Messaging endpoint wrong | Update bot endpoint to current Dev Tunnel URL |
| "Teams channel not configured" | Missing credentials | Set MICROSOFT_APP_* env vars |

### Telegram Channel

| Symptom | Cause | Fix |
|---------|-------|-----|
| "Conflict: terminated by other getUpdates" | Multiple instances | Stop other instances using the same bot token |
| Bot receives but doesn't reply | Orchestrator not running | Check server mode is running, not just CLI |
| Markdown parse errors | Invalid Markdown in response | Check LLM response formatting |

### Service Management

| Symptom | Cause | Fix |
|---------|-------|-----|
| Service not starting | Build not compiled | Run `npm run build` first |
| Service crashes on start | Missing .env | Create .env with required config |
| Port already in use | Another process on 3978 | `lsof -i :3978` to find and kill it |

## Log Locations

- **macOS (launchd)**: `~/Library/Logs/nanopilot/stdout.log`, `stderr.log`
- **Linux (systemd)**: `journalctl --user -u nanopilot -f`
- **Foreground**: stdout in terminal

## Reset

If things are broken beyond repair:
```bash
# Stop service
npx tsx setup/index.ts --step uninstall

# Clear database
rm -f data/nanopilot.db

# Rebuild
npm run build

# Reinstall service
npx tsx setup/index.ts --step service

# Verify
npx tsx setup/index.ts --step verify
```
