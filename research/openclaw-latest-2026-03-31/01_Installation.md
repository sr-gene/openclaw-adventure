# 01 — OpenClaw Installation

> **Source notes:** Based on workspace research notes dated 2026-03-30 and prior session findings.
> Live docs: [docs.openclaw.ai/install](https://docs.openclaw.ai/install) [unverified — not fetched in this session]

<!-- IMAGE: OpenClaw CLI onboarding wizard terminal screenshot -->

---

## Node.js Version Requirements

| Requirement | Value |
|-------------|-------|
| Minimum | Node.js 22.14+ |
| Recommended | Node.js 24.x |
| Package manager | pnpm (recommended) or npm |

> Node.js 22 is the stated minimum. Node.js 24 is recommended for the Mac Mini M4 setup as it ships with the latest V8 engine and ARM64 optimizations.

---

## Platform Support

| Platform | Status |
|----------|--------|
| macOS (Apple Silicon / Intel) | Fully supported |
| Linux | Fully supported |
| WSL2 (Windows) | Supported |
| Native Windows | Not supported [unverified] |

---

## Installation Methods

### Method 1 — One-Line Installer (Recommended for Mac)

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

This script automatically:
- Installs Homebrew if not present
- Installs Node.js 24 via Homebrew
- Installs the OpenClaw CLI globally via npm
- Runs `openclaw doctor` as a post-install health check

### Method 2 — Manual via pnpm (Fastest, Most Control)

```bash
# Step 1: Install Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Step 2: Install Node.js 24
brew install node@24

# Step 3: Install pnpm
brew install pnpm

# Step 4: Install OpenClaw globally
pnpm add -g openclaw@latest
```

### Method 3 — Manual via npm

```bash
npm install -g openclaw@latest
```

### Method 4 — Docker [estimated]

```bash
docker pull openclaw/openclaw:latest
docker run -d --name openclaw -v ~/.openclaw:/root/.openclaw openclaw/openclaw:latest
```

> Docker adds a Linux VM overhead layer on macOS. Not recommended for personal Mac Mini setups — use npm instead.

### Method 5 — Build from Source [estimated]

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install
pnpm build
pnpm link --global
```

---

## Installation Method Comparison

| Method | Best For | Complexity | macOS Performance |
|--------|----------|------------|-------------------|
| One-liner | First-time setup, fastest start | Easy | Excellent |
| pnpm global | Personal Mac, most control | Easy | Excellent |
| npm global | Fallback if pnpm unavailable | Easy | Excellent |
| Docker | Isolated/sandboxed environments | Medium | Reduced (VM overhead) |
| Build from source | Developers, bleeding edge features | Hard | Excellent |

**Recommendation for Mac Mini M4:** Use the one-liner or pnpm global method.

---

## Post-Install Verification

After installing, verify everything is working:

```bash
# Check Node.js version (must be 22.14+ ideally 24.x)
node --version

# Check OpenClaw CLI is available
openclaw --version

# Run health check
openclaw doctor
```

Expected output from `openclaw --version`:

```
openclaw/x.x.x darwin-arm64 node-v24.x.x
```

---

## Onboarding Wizard

Run this immediately after install:

```bash
openclaw onboard --install-daemon
```

The `--install-daemon` flag installs OpenClaw as a **macOS LaunchAgent** so it:
- Starts automatically on every boot/login
- Keeps running after Terminal is closed
- Runs silently in the background 24/7

### What the wizard asks:

1. **Auth token** — generates a secure gateway auth token
2. **LLM provider** — select OpenRouter, paste your API key
3. **Gateway settings** — port and bind address (accept defaults: port 18789, loopback binding)
4. **Channel** — choose Telegram (recommended first channel)
5. **Workspace location** — accept default (`~/.openclaw/workspace/`)
6. **Skills** — optional, can skip and install later

---

## Post-Onboarding Verification

```bash
# Check daemon is running
openclaw gateway status

# Verify gateway responds
curl http://localhost:18789/health

# Watch live logs
openclaw logs --follow

# Open browser dashboard
openclaw dashboard
# Opens at: http://localhost:18789
```

---

## Quick Troubleshooting

| Problem | Command |
|---------|---------|
| Daemon not running | `openclaw gateway start` |
| CLI not found after install | `pnpm setup` then restart terminal |
| Node version wrong | `brew install node@24 && brew link node@24 --force` |
| Permission errors | `openclaw doctor --fix` |
| Onboarding failed midway | `openclaw onboard` (re-run without `--install-daemon`) |

---

## Sources

- Workspace notes: `research/getting-started-openclaw-2026-03-30.md`
- Workspace notes: `research/setup-guide-2026-03-30.md`
- [docs.openclaw.ai/install](https://docs.openclaw.ai/install) [link unverified this session]
- [OpenClaw macOS Install Guide — AlexHost](https://alexhost.com/faq/how-to-install-openclaw-on-macos/) [link unverified this session]
- [How to Run OpenClaw: Daemon & TUI — Dextra Labs](https://dextralabs.com/blog/how-to-run-openclaw/) [link unverified this session]
