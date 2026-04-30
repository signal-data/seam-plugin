---
name: Intent Spike Alert
description: Alert when owned accounts cross an intent score threshold
author: seam
tags:
  - alerts
  - intent
  - slack
inputs:
  - name: owner_email
    type: string
    required: true
  - name: threshold
    type: number
    required: false
    default: 80
  - name: channel
    type: string
    required: false
    default: "#hot-accounts"
---

## Steps

Step 1: Query accounts with spiking intent
  search_companies({
    filter: "SEAM_COMPANY_INTENT_SCORE>={{threshold}}",
    relationFilters: [{ relation: "users", filter: "SEAM_USER_EMAIL=='{{owner_email}}'" }],
    fields: [
      "SEAM_COMPANY_NAME",
      "SEAM_COMPANY_INTENT_SCORE",
      "SEAM_COMPANY_FIT_SCORE",
      "SEAM_COMPANY_ABM_STAGE",
      "SEAM_COMPANY_DOMAIN"
    ],
    sort: "SEAM_COMPANY_INTENT_SCORE,desc"
  })
  save_as: spiking_accounts

Step 2: Check if any results
  condition: spiking_accounts.length == 0
  if true: exit silently — no alert needed

Step 3: Format message
  Build a single message string:

  🔥 *Intent Spike Alert* — {{spiking_accounts.length}} account(s) above {{threshold}}

  For each account in spiking_accounts:
  • *{{SEAM_COMPANY_NAME}}* — Intent: {{SEAM_COMPANY_INTENT_SCORE}} | Fit: {{SEAM_COMPANY_FIT_SCORE}}
    Stage: {{SEAM_COMPANY_ABM_STAGE}} → {{SEAM_COMPANY_DOMAIN}}

  save_as: message_text

Step 4: Deliver to channel
  Channel routing:
  - If {{channel}} starts with "#": call slack_send_message(channel: "{{channel}}", text: message_text)
  - If {{channel}} contains "@": log "Email delivery not yet implemented in v1" and exit

  On success: done (no user-facing confirmation needed — this runs headlessly)
