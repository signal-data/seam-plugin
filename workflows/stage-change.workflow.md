---
name: Stage Monitor Alert
description: Surface owned accounts currently at a target ABM stage. Reports accounts at the stage right now — not ones that recently moved to it (change detection is v2).
author: seam
tags:
  - alerts
  - abm
  - pipeline
inputs:
  - name: owner_email
    type: string
    required: true
  - name: target_stage
    type: string
    required: true
  - name: channel
    type: string
    required: false
    default: "#pipeline-alerts"
---

## Steps

Step 1: Query accounts at target stage
  search_companies({
    filter: "SEAM_COMPANY_ABM_STAGE=='{{target_stage}}'",
    relationFilters: [{ relation: "users", filter: "SEAM_USER_EMAIL=='{{owner_email}}'" }],
    fields: [
      "SEAM_COMPANY_NAME",
      "SEAM_COMPANY_ABM_STAGE",
      "SEAM_COMPANY_FIT_SCORE",
      "SEAM_COMPANY_INTENT_SCORE",
      "SEAM_COMPANY_DOMAIN"
    ],
    sort: "SEAM_COMPANY_FIT_SCORE,desc"
  })
  save_as: stage_accounts

Step 2: Check if any results
  condition: stage_accounts.length == 0
  if true: exit silently — no alert needed

Step 3: Format message
  Build a single message string:

  📊 *Stage Monitor: {{target_stage}}* — {{stage_accounts.length}} account(s)

  For each account in stage_accounts:
  • *{{SEAM_COMPANY_NAME}}* — Fit: {{SEAM_COMPANY_FIT_SCORE}} | Intent: {{SEAM_COMPANY_INTENT_SCORE}}
    → {{SEAM_COMPANY_DOMAIN}}

  save_as: message_text

Step 4: Deliver to channel
  Channel routing:
  - If {{channel}} starts with "#": call slack_send_message(channel: "{{channel}}", text: message_text)
  - If {{channel}} contains "@": log "Email delivery not yet implemented in v1" and exit
