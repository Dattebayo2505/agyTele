# Antigravity Telegram Bridge — Operations Runbook

Step-by-step guide and operational reference for installing, configuring, running, monitoring, and troubleshooting the Telegram ↔ Antigravity (`agy`) bridge.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Setup & Configuration](#setup--configuration)
   - [Get a Telegram Bot Token](#get-a-telegram-bot-token)
   - [Find Your Telegram User ID](#find-your-telegram-user-id)
   - [Configuration Reference](#configuration-reference)
3. [Installation](#installation)
4. [Service Management](#service-management)
   - [Quick Cheat-Sheet](#quick-cheat-sheet)
   - [Systemd Commands](#systemd-commands)
   - [Control via agy Plugin](#control-via-agy-plugin)
   - [Headless Boot (Lingering)](#headless-boot-lingering)
5. [Health Checks & Verification](#health-checks--verification)
   - [End-to-End Smoke Test](#end-to-end-smoke-test)
   - [Health & Diagnostic Commands](#health--diagnostic-commands)
6. [Hardening & Security](#hardening--security)
   - [Systemd Hardening](#systemd-hardening)
   - [File Permissions](#file-permissions)
   - [Lock Bot to DM / Single Chat](#lock-bot-to-dm--single-chat)
   - [Token Hygiene & Rotation](#token-hygiene--rotation)
   - [Backups](#backups)
7. [Log Rotation](#log-rotation)
8. [Metrics & Observability](#metrics--observability)
9. [Troubleshooting](#troubleshooting)
   - [Memory Growth & Limits](#memory-growth--limits)
   - [Bot Not Responding Checklist](#bot-not-responding-checklist)
   - [agy CLI Errors](#agy-cli-errors)
   - [Webhook Issues](#webhook-issues)
   - [Common Failure Modes & Solutions](#common-failure-modes--solutions)
10. [Upgrading](#upgrading)
11. [Uninstall](#uninstall)
12. [Pushing to GitHub & Pre-Push Checklist](#pushing-to-github--pre-push-checklist)
13. [Architecture Refresher & File Index](#architecture-refresher--file-index)

---

## Prerequisites

| Requirement | Why | How to verify |
|---|---|---|
| Linux with `systemd --user` | Daemon supervisor | `systemctl --user status` returns without error |
| Python ≥ 3.11 | Runtime | `python3 --version` |
| `uv` ≥ 0.4 | Venv & dependency management | `uv --version` (install: `curl -LsSf https://astral.sh/uv/install.sh \| sh`) |
| `agy` CLI on `PATH` | The backend CLI | `agy --version` |
| Authenticated `agy` session | Required for CLI execution | `~/.gemini/antigravity-cli/` exists |
| Outbound HTTPS to `api.telegram.org` | Telegram polling / API access | `curl -sSI https://api.telegram.org/` returns 200 |

If `agy` is missing, install it from the official Antigravity distribution and run `agy` once interactively to complete OAuth.

---

## Setup & Configuration

### Get a Telegram Bot Token

1. Open Telegram and message [@BotFather](https://t.me/BotFather).
2. Send `/newbot` and follow the prompts to choose a display name and username (must end in `bot`).
3. Copy the API token provided (format: `<digits>:<base64-ish>`). Keep it secret.
4. Recommended hardening in BotFather:
   - `/setprivacy` → choose your bot → `Disable`.
   - `/setjoingroups` → choose your bot → `Disable`.

### Find Your Telegram User ID

The bridge is **default-deny**.

- **Option A:** Message [@userinfobot](https://t.me/userinfobot) to get your numeric ID.
- **Option B:** Start the bridge once, send any message to the bot, and check the log:
  ```sh
  tail -f ~/.antigravity/bridge/logs/bridge.log
  # Look for: "drop unauth user=123456789"
  ```
  Add that ID to `allowed_user_ids` in `config.json` and restart.

### Configuration Reference

Location: `~/.gemini/extensions/antigravity-telegram-bridge/config.json`

```sh
cd ~/.gemini/extensions/antigravity-telegram-bridge
cp config.example.json config.json
chmod 600 config.json
$EDITOR config.json
```

Example `config.json`:

```json
{
  "telegram": {
    "bot_token": "1234567890:ABCDEFghijklmnopqrstuvwxyz_-1234567890",
    "allowed_user_ids": [123456789],
    "allowed_chat_ids": [123456789]
  },
  "agy": {
    "chats_root": "",
    "default_workdir": "",
    "model": "",
    "mode": "code"
  }
}
```

| Key | Required? | Description |
|---|---|---|
| `telegram.bot_token` | **Yes** | Telegram Bot API token from BotFather. |
| `telegram.allowed_user_ids` | **Yes** | Non-empty list of allowed numeric user IDs. |
| `telegram.allowed_chat_ids` | Optional | Whitelist of allowed chat IDs (e.g. your DM user ID to prevent group use). |
| `agy.chats_root` | Optional | Directory root for per-chat sessions (default: `~/.antigravity/bridge/chats`). |
| `agy.default_workdir` | Optional | Base working directory for `agy`. |
| `agy.model` | Optional | Override model ID (empty string = agy default). |
| `agy.mode` | Optional | `code` (auto-approve) or `plan` (instructs agent to only plan; no OS-level isolation). |

> [!CAUTION]
> **Never commit `config.json`**. It contains sensitive secrets and is gitignored.

---

## Installation

Run the idempotent install script from the extension repository:

```sh
cd ~/.gemini/extensions/antigravity-telegram-bridge
bash install.sh
```

What `install.sh` performs:
1. Creates runtime directories: `~/.antigravity/bridge/{logs,runtime,chats}/`.
2. Creates `~/.config/systemd/user/` if missing.
3. Creates a Python 3.11 venv at `.venv/` (if not already present).
4. Installs the project in editable mode via `uv pip install -e .`.
5. Renders the systemd user service template with paths to `~/.config/systemd/user/antigravity-telegram-bridge.service`.
6. Executes `systemctl --user daemon-reload` and enables `antigravity-telegram-bridge.service`.
7. Installs the `logrotate` config to `/etc/logrotate.d/antigravity-telegram-bridge` (if writable).

### Pre-flight Verification

Validate configuration, credentials, and dependencies:

```sh
echo '{"action":"setup"}' | .venv/bin/python -m src.control | python3 -m json.tool
```

---

## Service Management

### Quick Cheat-Sheet

| Action | Command |
|---|---|
| **Start** | `systemctl --user start antigravity-telegram-bridge.service` |
| **Stop** | `systemctl --user stop antigravity-telegram-bridge.service` |
| **Restart** | `systemctl --user restart antigravity-telegram-bridge.service` |
| **Status** | `systemctl --user status antigravity-telegram-bridge.service` |
| **Live Logs** | `tail -f ~/.antigravity/bridge/logs/bridge.log` |
| **Recent Journal** | `journalctl --user -u antigravity-telegram-bridge.service -n 100 --no-pager` |
| **Self-Test** | `echo '{"action":"setup"}' \| .venv/bin/python -m src.control` |
| **Health Check** | `curl -s http://127.0.0.1:9100/health \| jq .` |
| **Prometheus Metrics** | `curl -s http://127.0.0.1:9100/metrics` |

### Systemd Commands

```bash
# Check status
systemctl --user status antigravity-telegram-bridge

# Restart service (required after editing config.json)
systemctl --user restart antigravity-telegram-bridge

# Stop service
systemctl --user stop antigravity-telegram-bridge
```

### Control via agy Plugin

You can manage the daemon directly from an interactive `agy` session:

```sh
agy -p "use the bridge tool with action=start"
agy -p "use the bridge tool with action=status"
agy -p "use the bridge tool with action=logs and lines=200"
agy -p "use the bridge tool with action=setup"
```

Or via direct tool pipe:

```sh
echo '{"action":"status"}' | ~/.gemini/extensions/antigravity-telegram-bridge/.venv/bin/python -m src.control
```

### Headless Boot (Lingering)

To keep the user systemd instance running without an active SSH session:

```sh
sudo loginctl enable-linger $USER
```

---

## Health Checks & Verification

### End-to-End Smoke Test

1. Open Telegram and send `/start` or `hello` to your bot.
2. Within ~10s, verify the typing indicator appears, followed by an `agy` reply.
3. Check the log for a turn event:
   ```sh
   tail -n 20 ~/.antigravity/bridge/logs/bridge.log
   ```
   You should see:
   ```
   INFO antigravity_telegram_bridge: turn chat=123456789 cwd=... exit=0 ms=... reply_len=...
   ```
4. Send a follow-up (e.g. `what did I just say?`) to confirm session continuity.
5. From an unauthorized account, send a message. Verify that the message is ignored and the log displays:
   ```
   INFO antigravity_telegram_bridge: drop unauth user=999999999
   ```

### Health & Diagnostic Commands

```bash
# 1. Process memory usage
ps -o pid,rss,comm -p $(systemctl --user show -p MainPID --value antigravity-telegram-bridge)

# 2. Telegram Bot API connectivity & token validity
curl -s "https://api.telegram.org/bot<TOKEN>/getMe" | jq .ok

# 3. Backend agy CLI status
agy --version

# 4. Recent turns log
tail -20 ~/.antigravity/bridge/logs/bridge.log

# 5. Local health HTTP endpoint
curl -s http://127.0.0.1:9100/health | jq .

# 6. Prometheus metrics endpoint
curl -s http://127.0.0.1:9100/metrics
```

> [!NOTE]
> The health and metrics server listens on port **9100** by default (to avoid conflicts with ports like 9099). You can override this using `AGY_BRIDGE_HEALTH_PORT=...` in the systemd unit.

---

## Hardening & Security

### Systemd Hardening

The bridge runs under systemd with applied sandboxing and defense-in-depth directives:

| Directive | Value | Purpose |
|---|---|---|
| `PrivateTmp` | `true` | Isolated `/tmp` and `/var/tmp` directory |
| `ProtectKernelTunables` | `true` | Kernel variables (`/proc/sys`, `/sys`) made read-only |
| `ProtectKernelModules` | `true` | Module loading explicitly denied |
| `ProtectControlGroups` | `true` | Control group hierarchies made read-only |
| `LockPersonality` | `true` | Prevents changing execution domain / personality |
| `RestrictNamespaces` | `true` | Restricts access to Linux namespaces |
| `RestrictRealtime` | `true` | Prevents realtime scheduling priority escalation |
| `UMask` | `0077` | New files created are 0600 / 0700 (inaccessible to group/others) |
| `Restart` | `on-failure` / `always` | Automatic recovery from unexpected exits |
| `RestartSec` | `5s` | Backoff delay before restart |
| `MemoryMax` | `512M` *(optional override)* | Hard kill if exceeded |
| `MemoryHigh` | `384M` *(optional override)* | Soft throttle |
| `RuntimeMaxSec` | `86400` *(optional override)* | Daily restart to clear memory growth |

> [!NOTE]
> `RestrictAddressFamilies`, `RestrictSUIDSGID`, `NoNewPrivileges`, and `SystemCallFilter` are intentionally omitted in the service template because `agy` is run with `--dangerously-skip-permissions` to perform sysadmin and networking operations (such as `apt`, `sudo`, `nmap`, and `ping`) that need raw socket access, setuid binaries, and dynamic system calls.

### File Permissions

Verify runtime directories and sensitive files are restricted:

```sh
chmod 700 ~/.antigravity/bridge ~/.antigravity/bridge/logs ~/.antigravity/bridge/runtime ~/.antigravity/bridge/chats
chmod 600 ~/.gemini/extensions/antigravity-telegram-bridge/config.json ~/.antigravity/bridge/state.json ~/.antigravity/bridge/logs/bridge.log 2>/dev/null
```

### Lock Bot to DM / Single Chat

Set `allowed_chat_ids` in `config.json` to your direct message user ID:

```json
{
  "telegram": {
    "allowed_user_ids": [123456789],
    "allowed_chat_ids": [123456789]
  }
}
```

This prevents the bot from responding even if added to a Telegram group.

### Token Hygiene & Rotation

The Telegram bot token resides in `config.json` (mode `0600`, gitignored) and in daemon memory. To rotate a token:

1. In Telegram, message [@BotFather](https://t.me/BotFather) → `/token` → select your bot → `/revoke`.
2. Copy the new token and update `config.json`.
3. Restart the service:
   ```sh
   systemctl --user restart antigravity-telegram-bridge.service
   ```

### Backups

Exclude `~/.antigravity/bridge/logs/bridge.log` and `config.json` from unencrypted backups, or ensure backups are encrypted at rest.

---

## Log Rotation

The log file at `~/.antigravity/bridge/logs/bridge.log` receives all stdout and stderr. A logrotate configuration is provided at `systemd/antigravity-telegram-bridge.logrotate` and installed to `/etc/logrotate.d/antigravity-telegram-bridge`:

```
/home/YOUR_USER/.antigravity/bridge/logs/bridge.log {
    weekly
    rotate 8
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    create 600 YOUR_USER YOUR_USER
}
```

*`copytruncate` is required because systemd maintains the open file descriptor across turns.*

---

## Metrics & Observability

### Turn Log Entries

Each completed turn writes a structured INFO entry to `bridge.log`:

```
2026-06-26 12:00:00 INFO antigravity_telegram_bridge: turn chat=123456789 cwd=/home/user/.antigravity/bridge/chats/123456789 exit=0 ms=1420 reply_len=85
```

Key fields:
- `chat`: Telegram chat ID
- `cwd`: Project working directory
- `exit`: `agy` process exit code
- `ms`: Execution duration in milliseconds
- `reply_len`: Response length in characters

### Prometheus Endpoint

Available at `http://127.0.0.1:9100/metrics`:

- `antigravity_bridge_turns_total`: Total turns handled by the bridge
- `antigravity_bridge_errors_total`: Total failed turns / error responses

---

## Troubleshooting

### Memory Growth & Limits

- **Normal cold-start:** ~20 MB
- **Expected growth:** Up to ~150 MB over multiple days of active turns
- **Soft throttling (`MemoryHigh`):** Triggered at >384 MB
- **Hard kill & auto-restart (`MemoryMax`):** Triggered at >512 MB

To manually clear memory:

```sh
systemctl --user restart antigravity-telegram-bridge.service
```

### Bot Not Responding Checklist

Follow the 5-step diagnostic chain:

1. **Check bridge status:**
   ```sh
   systemctl --user status antigravity-telegram-bridge.service
   ```
2. **Verify bot token validity:**
   ```sh
   curl -s "https://api.telegram.org/bot<TOKEN>/getMe" | jq .
   ```
3. **Verify outbound network connectivity:**
   ```sh
   curl -sSI "https://api.telegram.org"
   ```
4. **Inspect live logs:**
   ```sh
   tail -n 50 ~/.antigravity/bridge/logs/bridge.log
   ```
5. **Run the pre-flight self-test:**
   ```sh
   echo '{"action":"setup"}' | .venv/bin/python -m src.control
   ```

### agy CLI Errors

1. **Verify executable on PATH:**
   ```sh
   agy --version
   ```
2. **Verify OAuth auth directory:**
   ```sh
   ls -la ~/.gemini/antigravity-cli/
   ```
3. **Re-authenticate:**
   Run `agy` interactively in your terminal and complete the login/OAuth flow.
4. **Restart bridge service:**
   ```sh
   systemctl --user restart antigravity-telegram-bridge.service
   ```

### Webhook Issues

If running in webhook receiver mode:

1. Verify `CTI_WEBHOOK_URL` and `CTI_WEBHOOK_PORT` are exported in the environment.
2. Confirm the host port is reachable externally from Telegram IP addresses.
3. Check `bridge.log` for webhook registration results.
4. Verify the secret token matches between Telegram registration and local receiver.

### Common Failure Modes & Solutions

| Symptom / Log Error | Root Cause | Solution |
|---|---|---|
| `telegram getUpdates failed: Unauthorized` | Token is invalid, expired, or revoked. | Retrieve a new token from BotFather, update `config.json`, and restart. |
| `agy CLI not found on PATH` | `agy` is missing from systemd PATH. | Ensure `agy` is in `~/.local/bin` or update `Environment=PATH=...` in the service file and run `daemon-reload`. |
| `agy error (exit <code>)` | Backend CLI failed (expired OAuth, invalid model, prompt error). | Check full stderr in `bridge.log`; re-run `agy` interactively or fix model string in `config.json`. |
| `agy timed out after 900.0s` | Turn ran for >15 minutes and was terminated. | If long turns are expected, adjust `AGY_TIMEOUT_S` in `src/turn.py`. |
| Bridge replies with `(empty reply)` | `agy` exited 0 without printing text. | Test the command manually in `~/.antigravity/bridge/chats/<chat_id>`. |
| Telegram replies truncated | Output exceeds Telegram's 4096 character limit. | The bridge auto-splits messages at newlines. Extremely long blocks are hard-chunked. |
| Leaked token in git history | Token was accidentally committed. | Revoke token in BotFather, update `config.json`, and scrub git history with `git filter-repo`. |

---

## Upgrading

To update from the repository:

```sh
cd ~/.gemini/extensions/antigravity-telegram-bridge
git pull
bash install.sh   # Idempotent: updates dependencies and re-renders service unit
systemctl --user restart antigravity-telegram-bridge.service
```

---

## Uninstall

### Complete Removal

```sh
# Stop and unregister service
systemctl --user disable --now antigravity-telegram-bridge.service
rm -f ~/.config/systemd/user/antigravity-telegram-bridge.service
systemctl --user daemon-reload

# Remove extension code and runtime state
rm -rf ~/.gemini/extensions/antigravity-telegram-bridge ~/.antigravity/bridge
```

### Temporary Pause

To keep files but stop running:

```sh
systemctl --user disable --now antigravity-telegram-bridge.service
```

Re-enable when ready:

```sh
systemctl --user enable --now antigravity-telegram-bridge.service
```

---

## Pushing to GitHub & Pre-Push Checklist

### Publishing

```sh
cd ~/.gemini/extensions/antigravity-telegram-bridge
git remote add origin https://github.com/YOUR_USER/agy-to-im.git
git push -u origin main
```

### Pre-Push Security Checklist

Before pushing, verify that no secrets or tokens are committed:

```sh
# 1. Verify config.json is ignored
git ls-files | grep -E '^config\.json$'

# 2. Check for leaked secrets
git ls-files | xargs grep -l 'bot_token\|api_key\|password' 2>/dev/null
# (Should only match examples and documentation templates)
```

---

## Architecture Refresher & File Index

### Two-Layer Architecture

```
                               ┌─────────────────────────────────────────┐
                               │  ~/.gemini/extensions/                  │
                               │   antigravity-telegram-bridge/          │
                               │   antigravity-extension.json            │   ← agy plugin manifest
       agy CLI session ───────▶│   src/control.py "bridge"               │     (start/stop/status/logs)
                               │     tool dispatcher                     │
                               └────────────────────┬────────────────────┘
                                                    │ systemctl --user
                                                    ▼
                               ┌─────────────────────────────────────────┐
                               │  systemd user daemon                    │
                               │  python -m src.daemon                   │
                               │                                         │
       Telegram Phone ────────▶│  • Polls getUpdates / Webhook receiver  │
                               │  • Authentication & Rate Limiting       │
                               │  • Spawns agy subprocess                │── agy -p "<prompt>"
                               │  • Formats & sends replies              │           --continue/--new-project
                               │                                         │
                               │  State: ~/.antigravity/bridge/          │
                               │    state.json (chat sessions)           │
                               └─────────────────────────────────────────┘
```

### Source File Index

| Path | Responsibility |
|---|---|
| `src/config.py` | Parse and validate `config.json` |
| `src/state.py` | Atomic JSON state management (`state.json`) |
| `src/telegram.py` | Telegram HTTP client, message chunking, authorization helpers |
| `src/agy_runner.py` | Builds argv and spawns `agy` CLI subprocess |
| `src/commands.py` | Slash command dispatching and inline keyboard callbacks |
| `src/turn.py` | Per-turn execution lifecycle with typing indicator heartbeat |
| `src/daemon.py` | Main polling/webhook loop and message orchestration |
| `src/control.py` | CLI plugin tool entry point (`start`, `stop`, `status`, `logs`, `setup`) |
| `src/health.py` | `/health` and `/metrics` HTTP server |
| `src/media.py` | Photo and document download and inbox handling |
| `src/queue.py` | Sliding-window rate limiting and FIFO turn queue |
| `src/webhook.py` | Webhook HTTP receiver with HMAC secret verification |
