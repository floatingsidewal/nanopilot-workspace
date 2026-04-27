---
name: add-telegram
description: Add Telegram channel to NanoPilot — create bot, configure token, allowlist, test connection
user_invocable: true
---

# /add-telegram — Add Telegram Channel

## Pre-flight
1. Check if `TELEGRAM_BOT_TOKEN` is already in `.env`
2. If yes, ask if they want to reconfigure

## Steps

### 1. Create a Telegram Bot
Guide the user through @BotFather:
1. Open Telegram and search for @BotFather
2. Send `/newbot`
3. Choose a display name (e.g., "ProxMan" or "NanoPilot")
4. Choose a username — must end in "bot" (e.g., "my_proxman_bot")
5. Copy the bot token @BotFather provides (format: `123456789:ABCdefGHIjklMNOpqrsTUVwxyz`)

### 2. Configure Token
- Ask the user to paste their bot token
- Write `TELEGRAM_BOT_TOKEN=<token>` to `.env`

### 3. Group Privacy (Optional)
If the user wants the bot in group chats:
1. Send `/mybots` to @BotFather
2. Select the bot → Bot Settings → Group Privacy → Turn Off
This allows the bot to see all messages in groups, not just /commands and @mentions.

### 4. Build and Restart
```bash
npm run build
npx tsx setup/index.ts --step restart
```

If service isn't installed yet, suggest running `/setup` first.

### 5. Get User IDs for the Allowlist

The allowlist uses **numeric Telegram user IDs** (not usernames — usernames are mutable and unreliable for access control). There are several ways to get a user ID:

**Method A — Use the bot's `/chatid` command (recommended)**
1. Open a **private DM** with the bot (use the `https://t.me/<bot_username>` link)
2. Tap **Start** or send `/start`
3. Send `/chatid` — the bot replies with both the chat ID and your user ID
4. The `/chatid` command works for anyone — it does NOT require being on the allowlist

**Method B — Use @userinfobot**
1. Search for `@userinfobot` in Telegram
2. Tap Start — it immediately replies with your numeric user ID

**Method C — Use @RawDataBot**
1. Forward any message to `@RawDataBot`
2. It replies with detailed JSON including `"from": {"id": 123456789}`

For adding other users: have them use any of these methods and send you their numeric ID.

### 6. Configure Allowlist (Recommended)
Restrict who can interact with the bot:
1. Add to `.env`: `NANOPILOT_TELEGRAM_ALLOWLIST=<user_id>` (comma-separated for multiple users)
2. Restart the service: `npx tsx setup/index.ts --step restart`
3. Verify the startup log shows "Telegram allowlist active"

If unset, the bot is open to anyone who discovers the username. This is a security concern if the bot has tool execution enabled.

### 7. Test Connection
1. Send a message to the bot in a private DM
2. Verify the bot responds (if on the allowlist) or silently ignores (if not)
3. If the allowlist is configured, also verify that an unauthorized user gets no response

## Adding More Users Later

1. Have the new user send `/chatid` to the bot in a private DM — it works without being on the allowlist
2. They share their user ID with the operator
3. Add the ID to `NANOPILOT_TELEGRAM_ALLOWLIST` (comma-separated)
4. Restart the service

Or use the `/add-user` skill for a guided flow on deployed agents.

## Troubleshooting
- **Bot doesn't respond**: Check `npx tsx setup/index.ts --step status` for errors. Check logs: `journalctl -u nanopilot --no-pager -n 30`
- **"Not authorized" in DM / silent in group**: User's numeric ID is not in `NANOPILOT_TELEGRAM_ALLOWLIST`. Have them send `/chatid` to get their ID.
- **"Conflict: terminated by other getUpdates request"**: Another instance is running with the same bot token. Stop it first.
- **Markdown parse errors**: The bot sends responses as Markdown. Some special characters may need escaping.
- **Bot works in DM but not in groups**: Check that group privacy is disabled (Step 3) and `NANOPILOT_TELEGRAM_GROUPS_ENABLED=true` is set.
