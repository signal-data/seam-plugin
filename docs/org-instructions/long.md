# Seam Plugin — Full Setup Guide

## Prerequisites
- Claude Code installed
- Access to a Seam org
- Slack workspace (for Slack alerts)

## Installation

1. Install the plugin:
   ```
   claude plugin install seam@latest
   ```

2. Connect your Seam MCP server:
   ```
   claude mcp add seam --transport http https://mcp.getseam.ai/mcp
   ```

3. Verify the connection: ask Claude "Who am I in Seam?"

## Creating Alerts

Say "Set up an alert" — Claude will guide you through:
1. Choosing an alert type
2. Setting your parameters (threshold, owner, channel)
3. Choosing a schedule ("every weekday morning", "weekly on Monday")

Your alert runs as a **remote scheduled agent** — it works even when your machine is off.

## Managing Alerts

- **List**: "Show my alerts" or "What alerts do I have running?"
- **Delete**: "Delete my intent spike alert" or "Remove all my alerts"

## Available Alert Types

| Alert | What it does | Default schedule |
|---|---|---|
| Intent Spike | Fires when accounts cross your intent score threshold | Weekdays at 9am |
| Stage Monitor | Shows accounts currently at a target ABM stage | Weekdays at 9am |
| Weekly Digest | Top accounts by signal + recent activity | Monday at 9am |
| Re-engagement | Accounts with no activity in N days | Weekly on Monday |

## Delivery Channels

- **Slack**: use a channel name like `#hot-accounts`
- **Email**: use an email address like `maxwell@getseam.ai` *(planned for v2)*

## Test Prompts

```
"Set up an alert for when any of my accounts hits an intent score over 75,
 post to #hot-accounts every weekday morning"

"Show me my active alerts"

"Delete my weekly digest"

"What are my top 10 accounts by intent score right now?"

"Which of my accounts haven't had any activity in the last 30 days?"
```

## Troubleshooting

**"I can't find any accounts"**
Check your MCP server is connected: `claude mcp list`

**"No active alerts"**
Alert configs live in `alerts/active/` — if deleted externally, recreate them with "Set up an alert"

**Alert not firing**
Verify the routine is active: ask Claude "List my scheduled routines"
