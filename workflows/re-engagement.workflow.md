---
name: Re-engagement Alert
description: Surface owned accounts with no activity in the last N days
author: seam
tags:
  - alerts
  - re-engagement
  - pipeline
inputs:
  - name: owner_email
    type: string
    required: true
  - name: days_inactive
    type: number
    required: false
    default: 30
  - name: channel
    type: string
    required: true
---

## Steps

Step 1: Calculate cutoff date
  Compute the date that is {{days_inactive}} days before today.
  Format as ISO 8601 date string (e.g., "2026-03-31").
  save_as: cutoff_date

Step 2: Query cold accounts
  search_companies({
    filter: "SEAM_COMPANY_LAST_ACTIVE_DATE=le='{{cutoff_date}}'",
    relationFilters: [{ relation: "users", filter: "SEAM_USER_EMAIL=='{{owner_email}}'" }],
    fields: [
      "SEAM_COMPANY_NAME",
      "SEAM_COMPANY_LAST_ACTIVE_DATE",
      "SEAM_COMPANY_FIT_SCORE",
      "SEAM_COMPANY_INTENT_SCORE",
      "SEAM_COMPANY_ABM_STAGE",
      "SEAM_COMPANY_DOMAIN"
    ],
    sort: "SEAM_COMPANY_FIT_SCORE,desc"
  })
  save_as: cold_accounts

Step 3: Check if any results
  condition: cold_accounts.length == 0
  if true: exit silently — no alert needed

Step 4: Format message
  Build a single message:

  🧊 *Re-engagement Alert* — {{cold_accounts.length}} account(s) inactive for {{days_inactive}}+ days

  For each account in cold_accounts:
  • *{{SEAM_COMPANY_NAME}}* — Last active: {{SEAM_COMPANY_LAST_ACTIVE_DATE}} | Fit: {{SEAM_COMPANY_FIT_SCORE}}
    Stage: {{SEAM_COMPANY_ABM_STAGE}} → {{SEAM_COMPANY_DOMAIN}}
    💡 [Based on the account's fit score and stage, suggest one specific re-engagement action]

  save_as: message_text

Step 5: Deliver to channel
  Channel routing:
  - If {{channel}} starts with "#": call slack_send_message(channel: "{{channel}}", text: message_text)
  - If {{channel}} contains "@": log "Email delivery not yet implemented in v1" and exit
