# 04 — Telegram Channel Setup

> **Source notes:** Based on workspace research notes dated 2026-03-30.
> Live docs: [docs.openclaw.ai/channels/telegram](https://docs.openclaw.ai/channels/telegram) [unverified — not fetched this session]

<!-- IMAGE: BotFather conversation creating a new bot in Telegram -->

---

## Why Telegram First?

Telegram is the recommended first channel for OpenClaw because:

- Bot creation takes under 5 minutes via @BotFather
- No approval process or business verification required
- Numeric user IDs are stable and easy to find
- `dmPolicy: "allowlist"` provides strong access control
- Free with no rate limits for personal bots [estimated]

---

## Step 1 — Create a Bot via BotFather

1. Open Telegram and search for **@BotFather** (verify the blue verification checkmark)
2. Send the command: `/newbot`
3. Follow the prompts:
   - Enter a display name (e.g., `My Research Agent`)
   - Enter a username ending in `bot` (e.g., `myresearch_bot`)
4. BotFather replies with your **bot token**:

```
Done! Congratulations on your new bot. You will find it at t.me/myresearch_bot.
Use this token to access the HTTP API:
123456789:ABCdefGHIjklMNOpqrSTUvwxYZ
```

5. **Copy and save this token securely** — treat it like a password. Anyone with it can control your bot.

---

## Step 2 — Find Your Numeric Telegram ID

The `allowFrom` field in `openclaw.json` requires your **numeric Telegram user ID** (not your username). This is a permanent integer like `987654321`.

### Method A — Via OpenClaw logs (recommended)

1. Start a conversation with your new bot in Telegram
2. Send any message (e.g., "hello")
3. In your terminal, run:

```bash
openclaw logs --follow
```

4. Look for a log line containing `from.id` — that integer is your numeric ID:

```json
{ "from": { "id": 987654321, "username": "yourname" }, "text": "hello" }
```

### Method B — Via @userinfobot

1. Search Telegram for `@userinfobot`
2. Send `/start`
3. It replies with your numeric ID immediately

### Method C — Via Telegram API directly [estimated]

```bash
curl "https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates"
```

After sending a message to your bot, the JSON response contains `message.from.id`.

---

## Step 3 — Configure Telegram in openclaw.json

Edit `~/.openclaw/openclaw.json`:

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123456789:ABCdefGHIjklMNOpqrSTUvwxYZ",
      dmPolicy: "allowlist",
      allowFrom: ["987654321"]    // your numeric Telegram ID
    }
  }
}
```

### dmPolicy Options

| Policy | Behavior | Recommended? |
|--------|----------|--------------|
| `"allowlist"` | Only users in `allowFrom` array can message the bot | **Yes — use this** |
| `"pairing"` | New users can pair via a one-time code displayed in logs | For sharing with others |
| `"open"` | Anyone who finds your bot can message it | Never — security risk |

**Always use `"allowlist"` for personal setups.** It survives gateway restarts and prevents unauthorized access. `"pairing"` mode resets on each restart.

---

## Step 4 — Add Channel via CLI (Alternative to Manual Config)

If you prefer not to edit JSON directly:

```bash
openclaw channel add telegram
```

The interactive prompt will ask for your bot token and numeric ID.

---

## Step 5 — Reload and Test

After editing `openclaw.json`, reload the daemon:

```bash
openclaw gateway stop
openclaw gateway start
```

Then send a message to your bot in Telegram. You should receive a response from OpenClaw within a few seconds.

To watch the exchange in real time:

```bash
openclaw logs --follow
```

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Bot doesn't respond | Daemon not running | `openclaw gateway start` |
| Bot responds "not authorized" | Your ID not in `allowFrom` | Add your numeric ID to config, restart |
| Bot token invalid | Token copied incorrectly | Regenerate via `/token` in @BotFather |
| `from.id` not appearing in logs | No message sent yet | Send any message to the bot first |
| Config changes not taking effect | Daemon not restarted | Stop then start daemon |

---

## BotFather — Useful Commands Reference

| Command | Purpose |
|---------|---------|
| `/newbot` | Create a new bot |
| `/mybots` | List all your bots |
| `/token` | Regenerate bot token |
| `/setdescription` | Set bot description |
| `/setcommands` | Define command menu |
| `/deletebot` | Remove a bot |

---

## Security Notes for Telegram Channel

- The bot token grants **full control** over your bot — never share it or commit it to Git
- Store the token as an environment variable in `openclaw.json`:

```json5
{
  env: {
    TELEGRAM_BOT_TOKEN: "123456789:ABCdef..."
  },
  channels: {
    telegram: {
      botToken: "${TELEGRAM_BOT_TOKEN}"
    }
  }
}
```

- Your bot is publicly discoverable by username — always use `dmPolicy: "allowlist"`
- Telegram bots cannot initiate conversations — they can only respond to users who message them first

---

## Sources

- Workspace notes: `research/setup-guide-2026-03-30.md`
- Workspace notes: `research/getting-started-openclaw-2026-03-30.md`
- [docs.openclaw.ai/channels/telegram](https://docs.openclaw.ai/channels/telegram) [link unverified this session]
