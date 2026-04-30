---
name: alerts
description: |
  Use when the user wants to create, set up, manage, list, view, or delete scheduled
  alerts on their Seam data. Triggers on: "set up an alert", "create an alert",
  "show my alerts", "list my alerts", "delete alert", "remove alert",
  "what alerts do I have", "manage my alerts", "alert me when", "notify me when".
---

## Overview

This skill manages scheduled remote alert agents. Each alert is:
1. Configured as a JSON file in `alerts/active/<slug>.json`
2. Executed by a remote scheduled agent that reads the config + a workflow template

See `references/alert-types.md` for available alert templates and their inputs.

---

## Determine Mode

Read the user's intent:
- **Create**: user wants to set up a new alert → follow Create Mode below
- **List**: user wants to see active alerts → follow List Mode below
- **Delete**: user wants to remove an alert → follow Delete Mode below

---

## Create Mode

Step 1: Present alert types
  Show the user the four available alert types from `references/alert-types.md`:
  - **intent-spike** — alert when accounts cross an intent score threshold
  - **stage-change** — surface accounts currently at a target ABM stage
  - **weekly-digest** — weekly rollup of top accounts by signal
  - **re-engagement** — surface accounts with no recent activity

  Ask which type they want.

Step 2: Pre-fill owner email
  Call `get_user_details` to get the current user's email.
  Use it as the default value for `owner_email` — confirm with the user before using it.

Step 3: Collect remaining inputs
  Ask for each required input for the chosen alert type (see `references/alert-types.md`).
  Ask for:
  - `channel`: "#channel-name" for Slack, or "email@domain.com" for email
  - `schedule`: accept plain English ("every weekday at 9am") and convert to cron

  Cron conversion guide:
  - "every weekday morning / 9am" → `0 9 * * 1-5`
  - "every Monday" / "weekly on Monday" → `0 9 * * 1`
  - "daily at 8am" → `0 8 * * *`
  - "twice a week" → `0 9 * * 1,4`
  - "every two weeks" → `0 9 1,15 * *`

Step 4: Generate slug and write config
  Create a slug: lowercase the alert name, replace spaces with hyphens, remove special characters.
  Example: "Intent spike — maxwell" → "intent-spike-maxwell"

  Write `alerts/active/<slug>.json`:
  ```json
  {
    "name": "<descriptive name including the user's name or identifier>",
    "workflow": "<alert type name, e.g. intent-spike>",
    "schedule": "<cron expression>",
    "routine_id": "",
    "inputs": {
      "owner_email": "<value>",
      "<other inputs>": "<values>"
    },
    "created_at": "<ISO 8601 UTC timestamp>"
  }
  ```

Step 5: Schedule the remote agent
  Invoke `/schedule` to create a recurring remote agent with:
  - Schedule: the cron expression
  - Prompt: "You are executing a Seam alert workflow in the seam-plugin repository. Your working directory is the root of the seam-plugin repository. Read the config at `alerts/active/<slug>.json`. Then load and execute the workflow at `workflows/<workflow>.workflow.md`, substituting all `{{variable}}` placeholders with the values from the config's `inputs` object. Use the Seam MCP tools to query data. Deliver results to the channel specified in the inputs."

Step 6: Save routine_id
  After `/schedule` returns the routine ID, update `alerts/active/<slug>.json` to set `routine_id` to the returned value.

Step 7: Confirm to user
  Tell the user:
  - Alert name and type
  - Schedule in plain English
  - Delivery channel
  - That it runs remotely (no need to keep their machine on)

---

## List Mode

Step 1: Read all `*.json` files in `alerts/active/`

Step 2: If none found, tell the user they have no active alerts and suggest:
  "Try: 'Set up an intent spike alert'"

Step 3: Render a table:

| Name | Type | Schedule | Channel | Created |
|---|---|---|---|---|
| <name> | <workflow> | <human-readable schedule> | <channel> | <created_at date> |

Convert cron to human-readable for display (e.g. `0 9 * * 1-5` → "Weekdays at 9am").

---

## Delete Mode

Step 1: If the user named a specific alert:
  Find the config file in `alerts/active/` matching by name or slug.
  If no match, list active alerts and ask which to delete.

  If the user did not name a specific alert:
  List active alerts and ask which to delete.

Step 2: Read the matching config file to get `routine_id`.

Step 3: Cancel the scheduled routine.
  Read `routine_id` from the config. If it is empty string or missing, skip cancellation and tell the user: "No scheduled routine was found for this alert — the alert config will be deleted but there is nothing to cancel."
  Otherwise, cancel the routine using the routine_id via the schedule management tools.

Step 4: Delete `alerts/active/<slug>.json`.

Step 5: Confirm: "Alert '<name>' has been deleted and its scheduled run cancelled."
