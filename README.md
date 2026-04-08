<p align="center">
  <img src="banner.png" alt="NoUsage" width="100%" />
</p>

# NoUsage

Rate limit notifications for Claude Code. Know when you're capped. Know when you're back.

---

## What it does

When you hit a rate limit in Claude Code, NoUsage does two things:

1. Sends you a notification immediately — "You've been rate limited."
2. Starts a cooldown timer and notifies you again when it expires — "You're good to go."

That's it. No dashboard, no account, no background service running 24/7. It hooks into Claude Code's event system, fires when needed, and stays out of your way.

## Why it exists

Claude Code doesn't tell you when your usage resets. You're left refreshing, guessing, or losing your train of thought. NoUsage removes the guesswork so you can context-switch cleanly and come back at the right time.

---

## How it works

```
Claude Code hits rate limit
  -> fires a Notification hook event
  -> NoUsage reads the event, detects rate limit keywords
  -> sends "Rate Limited" to all your enabled channels
  -> spawns a background timer (default: 5 hours)
  -> timer expires -> sends "Good to Go" to all channels
```

NoUsage is a Claude Code hook plugin. Hooks are shell commands that Claude Code runs in response to events. NoUsage listens to the `Notification` event, checks if it's a rate limit, and dispatches alerts. The background timer is a single detached process — no daemon, no scheduler, no database. State is one lockfile that prevents duplicate notifications.

---

## Supported channels

| Channel | How | Setup |
|---------|-----|-------|
| Desktop toast | Native OS notification (Windows, macOS, Linux) | Nothing — enabled by default |
| Discord | Webhook POST | Paste a webhook URL |
| Slack | Webhook POST | Paste a webhook URL |

Adding a new channel is one file. See [Contributing](#adding-a-channel).

---

## Get started

### Install

```bash
npm install -g nousage
```

### Setup

```bash
nousage setup
```

Walks you through choosing channels and pasting webhook URLs. Creates your config at `~/.nousage/config.json`.

### Hook into Claude Code

```bash
nousage install
```

Adds the hook to `~/.claude/settings.json`. Merges cleanly with your existing config — nothing gets overwritten.

### Verify

```bash
nousage test
```

Sends a test notification to every enabled channel. If it shows up, you're done.

---

## Configuration

Config lives at `~/.nousage/config.json`. You can edit it directly or re-run `nousage setup`.

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

`cooldown_seconds` controls how long NoUsage waits before sending the "good to go" notification. Default is 18000 (5 hours), matching the typical Claude Pro reset window. Adjust it to whatever works for your plan.

---

## CLI reference

```
nousage setup        Interactive config wizard
nousage install      Add hook to Claude Code settings
nousage uninstall    Remove hook from Claude Code settings
nousage test         Send test notification to all enabled channels
nousage status       Show current config and timer state
nousage channels     List available channels
```

---

## How channels work

Every channel is a standalone file in `src/channels/` that exports three things:

```js
module.exports = {
  name: 'discord',

  configSchema: {
    webhook_url: { type: 'string', required: true, prompt: 'Discord webhook URL' }
  },

  async send(title, message, channelConfig) {
    const res = await fetch(channelConfig.webhook_url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ content: `**${title}**\n${message}` })
    });
    return res.ok;
  }
};
```

Channels are auto-discovered at runtime. Drop a file in the directory and it works. No imports to update, no registration, no core code changes.

---

## Adding a channel

1. Create `src/channels/yourChannel.js`
2. Export `name`, `configSchema`, and `async send(title, message, config)`
3. Open a PR

The setup wizard will automatically prompt for any config keys you define in `configSchema`. That's the entire integration surface.

---

## Technical details

- **Runtime:** Node.js (18+). Zero production dependencies — uses built-in `fetch`, `fs`, `child_process`, `readline`.
- **Distribution:** npm global install.
- **State:** Single lockfile at `~/.nousage/.lock`. Prevents duplicate notifications when multiple rate limit events fire in quick succession.
- **Timer:** Detached child process that outlives the hook. No scheduler, no cron, no service.
- **Detection:** Pattern matching against the hook's JSON input for rate limit keywords (`rate limit`, `429`, `too many requests`, `usage limit`, `capacity`, `throttle`). Intentionally broad — the Notification hook schema isn't fully documented, so we match against the entire payload.

---

## Uninstall

```bash
nousage uninstall
npm uninstall -g nousage
rm -rf ~/.nousage
```

Clean removal. Nothing left behind.

---

## License

MIT
