# NoUsage — Product Requirements Document

> Get pinged when you're rate limited. Get pinged again when you're back.

## 1. Problem Statement

Claude Code users hit rate limits and are left guessing when they can resume. There's no built-in notification when limits reset — users either check manually, wait blindly, or context-switch and forget to come back.

**NoUsage** is a lightweight, open-source Claude Code hook plugin that solves this with two simple notifications:

1. **"You've been rate limited."** — immediate alert across your chosen channels
2. **"You're good to go."** — fires when the cooldown expires so you can jump back in

Ships with **Discord**, **Slack**, and **Windows/macOS/Linux desktop toast** support out of the box. Built so anyone can add a new notification channel in under 20 lines of code.

---

## 2. Target Users

- Claude Code CLI users on Pro/Team plans who regularly hit rate limits
- Developers who want to stay productive during cooldown windows without babysitting the terminal
- Open-source contributors who want to add new notification channels

---

## 3. Design Principles

### 3.1 Zero-Friction Onboarding
- One command to install: `npm install -g nousage`
- Interactive setup wizard: `nousage setup` walks you through picking channels and pasting webhook URLs
- Works immediately after setup with sensible defaults (desktop toast enabled by default, cooldown = 5 hours)
- No accounts to create, no services to host, no Docker, no databases

### 3.2 Plugin Architecture (Open/Closed Principle)
- Adding a new notification channel should **never** require modifying core code
- Each channel is a self-contained module that implements a simple interface: `send(title, message, config)`
- Channels are auto-discovered from the `channels/` directory — drop a file in, it works
- Core logic has zero knowledge of specific channels

### 3.3 Minimal Moving Parts
- No background services or daemons — the hook fires only when Claude Code triggers it
- The cooldown timer is a single detached process, not a scheduler
- No database — state is a single lockfile
- Zero production dependencies — everything uses Node.js built-ins

### 3.4 Predictable Behavior
- Every notification includes a timestamp so you know exactly when it fired
- Duplicate suppression via lockfile — you'll never get spammed with repeated alerts
- Cooldown duration is user-configurable and clearly documented
- Dry-run mode: `nousage test` sends a test notification through all enabled channels

### 3.5 Cross-Platform from Day 1
- **Windows:** PowerShell toast notifications (native, no extra installs)
- **macOS:** `osascript` display notification
- **Linux:** `notify-send`
- **Discord/Slack:** platform-independent HTTP webhooks

---

## 4. Architecture

### 4.1 Project Structure

```
nousage/
├── bin/
│   └── nousage.js                # CLI entry point (setup, test, install, uninstall, status)
├── src/
│   ├── hook.sh                   # Bash entry point (what Claude Code invokes)
│   ├── core.js                   # Core orchestrator: detect → lock → dispatch → timer
│   ├── detector.js               # Parses hook stdin JSON, returns { isRateLimit, rawMessage }
│   ├── timer.js                  # Spawns detached background cooldown process
│   ├── cooldown-worker.js        # Background script: sleep → notify "good to go" → cleanup
│   ├── lockfile.js               # Lockfile create/check/remove (duplicate suppression)
│   └── channels/
│       ├── index.js              # Auto-discovers and loads all channel modules
│       ├── toast.js              # Windows/macOS/Linux desktop notification
│       ├── discord.js            # Discord webhook
│       └── slack.js              # Slack webhook
├── config/
│   └── default.json              # Default configuration template
├── test/
│   ├── detector.test.js          # Unit tests for rate limit detection
│   ├── lockfile.test.js          # Unit tests for lockfile behavior
│   ├── channels/                 # Per-channel integration tests
│   └── e2e.test.js               # End-to-end hook simulation
├── package.json
├── README.md
├── LICENSE                       # MIT
├── PRD.md                        # This document
└── .github/
    ├── CONTRIBUTING.md
    └── ISSUE_TEMPLATE/
        ├── bug_report.md
        └── channel_request.md
```

### 4.2 Data Flow

```
Claude Code hits rate limit
  → fires Notification hook event
  → invokes hook.sh (reads JSON from stdin)
  → pipes to core.js
  → detector.js checks for rate-limit keywords
  → if rate limit detected:
      → lockfile.js checks if timer already running
      → if no existing timer:
          → create lockfile
          → broadcast "Rate Limited" to all enabled channels
          → timer.js spawns detached cooldown-worker.js
          → cooldown-worker.js sleeps for cooldown_seconds
          → on wake: broadcast "Good to Go" to all channels
          → remove lockfile
```

### 4.3 Hook Integration

Claude Code hooks are configured in `~/.claude/settings.json`. The `nousage install` command adds:

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash \"$(npm root -g)/nousage/src/hook.sh\"",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

The `Notification` hook fires when Claude Code sends user-facing notifications, including rate limit messages. The hook is observability-only — exit codes are ignored, so it cannot break Claude Code.

---

## 5. Channel Interface

Every channel module exports a single object with this shape:

```js
module.exports = {
  name: 'discord',

  // Config keys this channel needs (drives the setup wizard)
  configSchema: {
    webhook_url: { type: 'string', required: true, prompt: 'Discord webhook URL' }
  },

  // Send a notification. Returns true on success, false on failure.
  async send(title, message, channelConfig) {
    const res = await fetch(channelConfig.webhook_url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        content: `**${title}**\n${message}`
      })
    });
    return res.ok;
  }
};
```

### Adding a new channel

1. Create a new `.js` file in `src/channels/` (e.g., `telegram.js`)
2. Export an object with `name`, `configSchema`, and `send(title, message, config)`
3. That's it — auto-discovery picks it up, the setup wizard prompts for its config keys

No imports to update. No registration. No core code changes.

---

## 6. Configuration

### 6.1 Location

User config is stored at `~/.nousage/config.json` — outside the repo to keep secrets out of version control. Created by `nousage setup`.

### 6.2 Schema

```json
{
  "cooldown_seconds": 18000,
  "channels": {
    "toast": {
      "enabled": true
    },
    "discord": {
      "enabled": true,
      "webhook_url": "https://discord.com/api/webhooks/..."
    },
    "slack": {
      "enabled": false,
      "webhook_url": ""
    }
  }
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `cooldown_seconds` | number | `18000` (5h) | How long to wait before sending "good to go" |
| `channels.<name>.enabled` | boolean | varies | Whether this channel is active |
| `channels.<name>.*` | string | `""` | Channel-specific config (webhook URLs, etc.) |

---

## 7. CLI Commands

```bash
nousage setup          # Interactive setup wizard — pick channels, paste URLs
nousage test           # Send a test notification to all enabled channels
nousage install        # Add hook to ~/.claude/settings.json (merges, won't clobber)
nousage uninstall      # Remove hook from ~/.claude/settings.json
nousage status         # Show config, enabled channels, active timer state
nousage channels       # List all available channels and their status
```

### 7.1 Setup Wizard

```
$ nousage setup

  NoUsage — Setup

  Detected platform: Windows 11
  Desktop toast notifications enabled by default.

  Enable Discord notifications? (y/n): y
  Paste your Discord webhook URL: https://discord.com/api/webhooks/...
  Discord configured.

  Enable Slack notifications? (y/n): n

  Rate limit cooldown duration (default: 5h): 5h
  Cooldown set to 18000 seconds.

  Config saved to ~/.nousage/config.json
  Run 'nousage install' to add the hook to Claude Code.
  Run 'nousage test' to verify everything works.
```

---

## 8. Rate Limit Detection

```js
const RATE_LIMIT_PATTERNS = [
  /rate.?limit/i,
  /429/,
  /too many requests/i,
  /usage.?limit/i,
  /capacity/i,
  /throttl/i
];

function isRateLimit(hookInput) {
  const raw = JSON.stringify(hookInput);
  return RATE_LIMIT_PATTERNS.some(p => p.test(raw));
}
```

**Why grep the entire JSON payload?** The exact fields in the `Notification` hook input aren't fully documented. Matching against the whole payload is more resilient to schema changes. False positives are extremely unlikely with these patterns, and a false alert is harmless.

---

## 9. Cooldown Timer

The hook process has a 10-second timeout imposed by Claude Code, so the timer must outlive it:

```js
const { spawn } = require('child_process');

function startCooldown(seconds, configPath) {
  const child = spawn(process.execPath, [
    path.join(__dirname, 'cooldown-worker.js'),
    String(seconds),
    configPath
  ], {
    detached: true,
    stdio: 'ignore'
  });
  child.unref();
}
```

`cooldown-worker.js`:
1. Sleeps for `cooldown_seconds`
2. Loads config and channels
3. Broadcasts "You're good to go!" to all enabled channels
4. Removes the lockfile

---

## 10. Duplicate Suppression (Lockfile)

**Path:** `~/.nousage/.lock`

When a rate limit is detected, the lockfile is checked before dispatching:

```js
function isTimerRunning() {
  if (!fs.existsSync(LOCK_PATH)) return false;
  const age = Date.now() - fs.statSync(LOCK_PATH).mtimeMs;
  const config = loadConfig();
  return age < config.cooldown_seconds * 1000;
}
```

If the lockfile exists and is younger than the cooldown, a timer is already running — skip. This prevents duplicate notifications when Claude Code fires multiple events in quick succession.

---

## 11. Cross-Platform Desktop Toast

```js
const commands = {
  win32: (title, msg) =>
    `powershell -Command "Add-Type -AssemblyName System.Windows.Forms; ` +
    `[System.Windows.Forms.MessageBox]::Show('${msg}','${title}')"`,

  darwin: (title, msg) =>
    `osascript -e 'display notification "${msg}" with title "${title}"'`,

  linux: (title, msg) =>
    `notify-send "${title}" "${msg}"`
};
```

Windows primary, macOS and Linux supported. No extra dependencies on any platform.

---

## 12. Technology Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Runtime | Node.js | Cross-platform, `fetch` built-in (18+), Claude Code users already have it |
| Distribution | npm (`npm install -g`) | Familiar, one command, handles PATH |
| Dependencies | Zero | `fetch`, `child_process`, `fs`, `readline` are all built-in |
| State | Lockfile | Simplest possible — no database, no service |
| Timer | Detached child process | Outlives the hook, no scheduler needed |
| Config location | `~/.nousage/config.json` | Outside repo, secrets stay local |

---

## 13. Testing Strategy

| Layer | Target | Method |
|-------|--------|--------|
| Unit | `detector.js` | Feed various JSON payloads, assert detection |
| Unit | `lockfile.js` | Create, check, age out, verify |
| Integration | Each channel | Mock HTTP server, verify POST body |
| E2E | Full pipeline | `echo '{"message":"rate limit"}' \| bash hook.sh` |
| Manual | `nousage test` | Real notifications to verify config |

---

## 14. Implementation Roadmap

| Phase | What | Files |
|-------|------|-------|
| 1 | Project scaffold + package.json | `package.json`, dirs |
| 2 | Config module + defaults | `src/config.js`, `config/default.json` |
| 3 | Channel auto-discovery | `src/channels/index.js` |
| 4 | Toast channel | `src/channels/toast.js` |
| 5 | Discord channel | `src/channels/discord.js` |
| 6 | Slack channel | `src/channels/slack.js` |
| 7 | Rate limit detector | `src/detector.js` |
| 8 | Lockfile module | `src/lockfile.js` |
| 9 | Timer + cooldown worker | `src/timer.js`, `src/cooldown-worker.js` |
| 10 | Core orchestrator | `src/core.js` |
| 11 | Bash hook entry point | `src/hook.sh` |
| 12 | CLI (setup, test, install, status) | `bin/nousage.js` |
| 13 | Tests | `test/` |
| 14 | README + CONTRIBUTING | docs |

---

## 15. Acceptance Criteria

- [ ] `nousage setup` creates config interactively with channel selection
- [ ] `nousage test` sends notifications to all enabled channels
- [ ] `nousage install` adds hook to `~/.claude/settings.json` without clobbering existing settings
- [ ] `nousage uninstall` cleanly removes the hook
- [ ] Rate limit event triggers immediate "Rate Limited" notification on all enabled channels
- [ ] Lockfile prevents duplicate timers and notification spam
- [ ] Background timer fires "Good to Go" notification after configured cooldown
- [ ] Works on Windows 11, macOS, and Linux
- [ ] Zero production dependencies
- [ ] New channels can be added with a single file, no core changes needed

---

## 16. Future Considerations (Not in v1)

- **Actual API polling** — if Anthropic exposes usage data, poll it instead of estimating cooldown
- **Browser extension** — for claude.ai web users, not just CLI
- **Usage dashboard** — track rate limit frequency over time
- **Smart cooldown** — parse the rate limit response for actual reset time if provided
- **Community channel registry** — npm packages that auto-install as channels
