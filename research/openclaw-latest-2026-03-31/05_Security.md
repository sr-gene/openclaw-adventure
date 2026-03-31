# 05 — Security

> **Source notes:** Based on workspace research notes dated 2026-03-30.
> Live docs: [docs.openclaw.ai/gateway/security](https://docs.openclaw.ai/gateway/security) [unverified — not fetched this session]

<!-- IMAGE: Security hardening checklist diagram for OpenClaw -->

---

## Security Posture Overview

OpenClaw runs as a persistent daemon with access to your filesystem, shell, and LLM API keys. The `~/.openclaw/` directory is a **high-privilege directory** — treat it like a password vault.

Threats to mitigate:

| Threat | Risk | Mitigation |
|--------|------|------------|
| Unauthorized gateway access | Anyone on local network controls your agent | Bind to loopback only |
| Stolen API keys | Financial loss from unauthorized API usage | File permissions + env vars |
| Bot hijacking via Telegram | Strangers control your agent via Telegram | `dmPolicy: "allowlist"` |
| Malicious ClawHub skills | Credential-harvesting, data exfiltration | Vet all skills before install |
| Config committed to Git | API keys exposed publicly | `.gitignore` rules |

---

## Immediate Post-Install Security Steps

Run these three commands immediately after `openclaw onboard`:

```bash
# 1. Restrict directory access to owner only
chmod 700 ~/.openclaw

# 2. Restrict config file to owner read/write only
chmod 600 ~/.openclaw/openclaw.json

# 3. Run OpenClaw's built-in security audit
openclaw security audit --fix
```

---

## File Permissions

### Required Permissions

| File/Directory | Required Permission | Command |
|---------------|---------------------|---------|
| `~/.openclaw/` | `700` (owner rwx only) | `chmod 700 ~/.openclaw` |
| `~/.openclaw/openclaw.json` | `600` (owner rw only) | `chmod 600 ~/.openclaw/openclaw.json` |
| `~/.openclaw/auth-profiles.json` | `600` | `chmod 600 ~/.openclaw/auth-profiles.json` |
| `~/.openclaw/workspace/` | `700` | `chmod 700 ~/.openclaw/workspace` |

### Verify Current Permissions

```bash
ls -la ~/.openclaw/
```

Expected output:
```
drwx------   gene  staff   ~/.openclaw/
-rw-------   gene  staff   ~/.openclaw/openclaw.json
-rw-------   gene  staff   ~/.openclaw/auth-profiles.json
```

---

## API Key Storage

**Never hardcode API keys directly in config values.** Always reference environment variables.

### Correct pattern:

```json5
{
  // Keys stored as env vars
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    TELEGRAM_BOT_TOKEN: "123456789:ABCdef...",
    FIRECRAWL_API_KEY: "fc-..."
  },

  // Config references env vars
  providers: {
    openrouter: {
      apiKey: "${OPENROUTER_API_KEY}"
    }
  },
  channels: {
    telegram: {
      botToken: "${TELEGRAM_BOT_TOKEN}"
    }
  }
}
```

### Why env vars in openclaw.json (not shell)?

OpenClaw's daemon runs as a LaunchAgent — it does not inherit your interactive shell's environment variables. The `env` block in `openclaw.json` is the correct place to inject secrets for the daemon process.

---

## Gateway Binding

The gateway HTTP server should **never** be exposed to the network. Bind it to the loopback interface:

```json5
{
  gateway: {
    port: 18789,
    bind: "loopback"    // binds to 127.0.0.1 only — NOT 0.0.0.0
  }
}
```

Verify the gateway is only listening on loopback:

```bash
lsof -i :18789
# Should show: 127.0.0.1:18789, NOT 0.0.0.0:18789
```

---

## Telegram Access Control

Always use `dmPolicy: "allowlist"` with your specific numeric Telegram ID:

```json5
{
  channels: {
    telegram: {
      dmPolicy: "allowlist",
      allowFrom: ["987654321"]    // your numeric ID only
    }
  }
}
```

This prevents:
- Other Telegram users who discover your bot username from controlling it
- Accidental exposure if your bot token leaks

---

## ClawHub Skill Vetting

> **Warning:** Malicious skills with credential-harvesting payloads have been found in the ClawHub registry. [Source: workspace notes — unverified independently]

### Before installing any ClawHub skill:

1. **Review the source code** — every ClawHub skill should have a public source repo
2. Check for network calls to unknown endpoints
3. Check for file system access outside the workspace
4. Look for any code that reads `~/.openclaw/` or environment variables
5. Check the skill's GitHub stars, recent commits, and issue tracker

```bash
# View skill details before installing
clawhub info <skill-name>

# Install only after reviewing
clawhub install <skill-name>
```

### Vetted skills for financial research:

| Skill | Purpose | Status |
|-------|---------|--------|
| `firecrawl` | Web scraping via remote browser | Documented, widely used |
| `crawl4ai` | Local web crawling | Check: no official OpenClaw skill as of research date [unverified] |

### Avoid:

- Skills from accounts with no GitHub history
- Skills that request filesystem permissions beyond workspace
- Skills without readable source code

---

## Security Audit Command

OpenClaw includes a built-in security audit tool:

```bash
# Run audit (report only)
openclaw security audit

# Run audit and auto-fix issues
openclaw security audit --fix
```

The audit checks [unverified — exact checks based on documented behavior]:
- File permissions on `~/.openclaw/`
- Gateway binding configuration
- Whether API keys appear to be hardcoded vs env-referenced
- LaunchAgent plist permissions
- Presence of `auth-profiles.json` with correct permissions

---

## Git Safety

If you version-control your workspace files, **never commit secrets**:

```bash
# ~/.openclaw/.gitignore
openclaw.json
auth-profiles.json
*.log
```

Only version-control the workspace content files:
```
SOUL.md        ✓ safe to commit
USER.md        ✓ safe (no secrets)
AGENTS.md      ✓ safe
HEARTBEAT.md   ✓ safe
openclaw.json  ✗ never commit — contains API keys
auth-profiles.json  ✗ never commit
```

---

## Complete Security Checklist

Run through this after every fresh installation:

- [ ] `chmod 700 ~/.openclaw`
- [ ] `chmod 600 ~/.openclaw/openclaw.json`
- [ ] `chmod 600 ~/.openclaw/auth-profiles.json`
- [ ] `openclaw security audit --fix`
- [ ] All API keys stored as `env` block variables, not hardcoded values
- [ ] `openclaw.json` and `auth-profiles.json` in `.gitignore`
- [ ] Telegram `dmPolicy` set to `"allowlist"` with your numeric ID
- [ ] Gateway `bind` set to `"loopback"` (not `"0.0.0.0"`)
- [ ] Verified with `lsof -i :18789` that port is loopback-only
- [ ] Reviewed source of every installed ClawHub skill
- [ ] Applied conditional KeepAlive fix to LaunchAgent plist (see `02_Configuration.md`)

---

## Sources

- Workspace notes: `research/setup-guide-2026-03-30.md`
- [docs.openclaw.ai/gateway/security](https://docs.openclaw.ai/gateway/security) [link unverified this session]
