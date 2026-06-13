# Running OpenAB Locally on macOS

This guide covers running OpenAB as a persistent background service on macOS using a LaunchAgent, without Docker or Kubernetes.

## Prerequisites

- OpenAB binary built via `cargo build --release`
- Your agent CLI installed (e.g. `claude-agent-acp`, `kiro-cli`)
- A Discord or Slack bot token stored in Keychain

## The PATH Problem

When macOS launches a process via LaunchAgent, it provides a minimal environment with only:

```
PATH=/usr/bin:/bin:/usr/sbin:/sbin
```

OpenAB uses `env_clear()` when spawning agent subprocesses, then re-exports its own `PATH` to the child. If OpenAB itself was launched with a minimal PATH, the agent CLI will inherit that minimal PATH too.

This causes silent failures for agent CLIs installed via Homebrew or nvm, because their binaries live in paths like `/opt/homebrew/bin` or `~/.nvm/versions/node/*/bin` that are not included in the default LaunchAgent PATH.

For example, `claude-agent-acp` uses a `#!/usr/bin/env node` shebang. If `node` is not in PATH, the process exits immediately and OpenAB reports only:

```
ERROR openab::adapter: pool error: JSON-RPC error -1: connection closed
```

**Fix:** Always set PATH explicitly in your `start.sh` before launching OpenAB:

```bash
export PATH="/opt/homebrew/bin:/usr/local/bin:$PATH"
```

## Recommended Setup

### 1. Create a working directory

```bash
mkdir -p ~/.kiro/openab-claude
cd ~/.kiro/openab-claude
```

### 2. Create `config.toml`

```toml
[discord]
bot_token = "${DISCORD_BOT_TOKEN}"
allowed_channels = ["YOUR_CHANNEL_ID"]
allow_user_messages = "involved"   # allows follow-up messages in threads without @mention

[agent]
command = "/opt/homebrew/bin/claude-agent-acp"
args = []
working_dir = "/Users/yourname/workspace"
```

> **`allow_user_messages`**: Use `"involved"` so that follow-up messages inside a bot-owned thread don't require an @mention each time. Use `"mentions"` if you want every message to require an explicit @mention.

### 3. Store secrets in Keychain

```bash
security add-generic-password -s "openab-discord-token" -a "$USER" -w "your-bot-token"
```

### 4. Create `start.sh`

```bash
#!/bin/bash

# Set PATH explicitly so agent CLIs installed via Homebrew or nvm are found.
# LaunchAgent provides only /usr/bin:/bin:/usr/sbin:/sbin by default.
export PATH="/opt/homebrew/bin:/usr/local/bin:$PATH"

export DISCORD_BOT_TOKEN=$(security find-generic-password -s "openab-discord-token" -w)

exec /path/to/openab/target/release/openab run -c config.toml
```

```bash
chmod +x start.sh
```

### 5. Create a LaunchAgent plist

Save as `~/Library/LaunchAgents/openab.claude.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>openab.claude</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>/Users/yourname/.kiro/openab-claude/start.sh</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/Users/yourname/.kiro/openab-claude</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/Users/yourname/.kiro/openab-claude/openab.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/yourname/.kiro/openab-claude/openab.log</string>
</dict>
</plist>
```

### 6. Load and start

```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/openab.claude.plist
```

### Restarting after config changes

```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/openab.claude.plist
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/openab.claude.plist
```

## Verifying the setup

Check that OpenAB is running and that PATH was set correctly:

```bash
# Confirm the process is running
ps aux | grep "openab run"

# Confirm PATH includes Homebrew
ps ewww $(pgrep openab) | tr ' ' '\n' | grep "^PATH="

# Tail the log
tail -f ~/.kiro/openab-claude/openab.log
```
