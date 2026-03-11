---
name: openclaw-daily-ops
description: Daily cost reporting + session hygiene for OpenClaw deployments. Tracks per-session API spend, shows 7-day trend, and wipes zombie sessions >24h old to prevent context snowball. Telegram-only notifications. Runs nightly via cron.
---

# OpenClaw Daily Ops

Two tasks in one nightly cron: **know what you spent, kill what's dead.**

- Parses all OpenClaw session JSONL files → computes today's API cost per session
- Posts a clean cost report to Telegram with 7-day trend
- Wipes sessions older than 24h with >1MB context (zombie killer)
- Logs everything to `state/cost-log.json` and `state/session-reset-log.json`

Zero AI credits spent — pure Python + one Telegram message.

---

## Setup

### 1. Configure

Copy `config.example.json` to `config.json` and fill in your values:

```bash
cp config.example.json config.json
```

Edit `config.json`:

```json
{
  "sessions_dir": "~/.openclaw/agents/main/sessions",
  "workspace_dir": "~/.openclaw/workspace",
  "telegram_bot_token": "YOUR_TELEGRAM_BOT_TOKEN",
  "telegram_chat_id": "-1001234567890",
  "telegram_user_id": "1616735985",
  "channel_names": {
    "channel:YOUR_CHANNEL_ID": "#your-channel-name"
  },
  "zombie_min_age_hours": 24,
  "zombie_min_size_mb": 1,
  "alert_high_cost": 50,
  "alert_low_cost": 10,
  "timezone_offset_hours": -6
}
```

**How to get Telegram bot token + chat id:**
1. Create bot in BotFather and copy token
2. Add bot to your target chat/topic
3. Use the chat ID/topic destination you already use in OpenClaw

**Channel names** (optional): Map your OpenClaw channel session keys to human-readable names for the report. Find session keys in `~/.openclaw/agents/main/sessions/sessions.json`.

### 2. Test it

```bash
python3 scripts/cost_report.py --config config.json --dry-run
```

This prints the report without posting to Telegram or writing logs.

### 3. Set up the cron

Add a nightly OpenClaw cron job. In your OpenClaw config or via CLI:

```
Schedule: 0 21 * * * (9 PM daily — adjust to your timezone)
Model: haiku (cost report is simple, no need for a heavy model)
Payload: Read /path/to/openclaw-daily-ops/SKILL.md and follow it exactly.
```

Or run it as a system cron:

```bash
# Add to crontab -e
0 21 * * * python3 /path/to/openclaw-daily-ops/scripts/cost_report.py --config /path/to/config.json >> /path/to/cost_report.log 2>&1
```

---

## Skill Steps (for OpenClaw cron payload)

When running as an OpenClaw cron agentTurn, the agent should:

### Step 1 — Run cost parser
Execute `scripts/cost_report.py --config /path/to/config.json` and capture output.

### Step 2 — Run zombie killer
Execute `scripts/zombie_killer.py --config /path/to/config.json` and capture output.

### Step 3 — Format and post
Combine both outputs into the report format and post to Telegram.

---

## Configuration Reference

| Key | Type | Description |
|-----|------|-------------|
| `sessions_dir` | string | Path to OpenClaw sessions directory |
| `workspace_dir` | string | Path to OpenClaw workspace (for state logs) |
| `telegram_bot_token` | string | Telegram bot token to post the report |
| `telegram_chat_id` | string | Telegram chat ID/topic destination |
| `telegram_user_id` | string | Optional Telegram user ID shown in urgent alerts |
| `channel_names` | object | Map session keys → display names (optional) |
| `zombie_min_age_hours` | number | Sessions older than this get reset (default: 24) |
| `zombie_min_size_mb` | number | Minimum file size to reset (default: 1MB) |
| `alert_high_cost` | number | Daily cost threshold for 🚨 URGENT flag (default: $50) |
| `alert_low_cost` | number | Daily cost threshold for ✅ UNDER BUDGET flag (default: $10) |
| `timezone_offset_hours` | number | Your UTC offset for date filtering (default: -6 for CST) |

---

## Files

```
openclaw-daily-ops/
├── SKILL.md                 ← this file
├── config.example.json      ← template config (copy to config.json)
├── scripts/
│   ├── cost_report.py       ← parses sessions, computes costs, posts to Telegram
│   └── zombie_killer.py     ← wipes stale sessions, logs what was cleared
└── state/                   ← created automatically
    ├── cost-log.json        ← rolling 90-day cost history
    └── session-reset-log.json ← log of all zombie kills
```
