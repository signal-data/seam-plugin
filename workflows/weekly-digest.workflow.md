---
name: Weekly Digest
description: Weekly rollup of top accounts by intent score with recent activity summary
author: seam
tags:
  - alerts
  - digest
  - weekly
inputs:
  - name: owner_email
    type: string
    required: true
  - name: top_n
    type: number
    required: false
    default: 10
  - name: channel
    type: string
    required: true
---

## Steps

Step 1: Query top accounts by intent score
  search_companies({
    relationFilters: [{ relation: "users", filter: "SEAM_USER_EMAIL=='{{owner_email}}'" }],
    fields: [
      "SEAM_COMPANY_ID",
      "SEAM_COMPANY_NAME",
      "SEAM_COMPANY_INTENT_SCORE",
      "SEAM_COMPANY_FIT_SCORE",
      "SEAM_COMPANY_ABM_STAGE",
      "SEAM_COMPANY_DOMAIN"
    ],
    sort: "SEAM_COMPANY_INTENT_SCORE,desc",
    size: {{top_n}}
  })
  save_as: top_accounts

Step 2: Query recent activities for top accounts
  From the top_accounts result, collect all SEAM_COMPANY_ID values into a comma-separated list.
  search_activities({
    relationFilters: [{
      relation: "companies",
      filter: "SEAM_COMPANY_ID=in=(<comma-separated list of SEAM_COMPANY_ID values from top_accounts>)"
    }],
    fields: [
      "SEAM_ACTIVITY_TYPE",
      "SEAM_ACTIVITY_DATE",
      "SEAM_ACTIVITY_DESCRIPTION",
      "SEAM_ACTIVITY_COMPANY_NAME"
    ],
    sort: "SEAM_ACTIVITY_DATE,desc",
    size: 50
  })
  save_as: recent_activities

Step 3: Format digest message
  Build a single message:

  📋 *Weekly Account Digest* — Top {{top_n}} accounts

  For each account in top_accounts:
    Find matching entries in recent_activities where SEAM_ACTIVITY_COMPANY_NAME matches SEAM_COMPANY_NAME.
    Summarize in one line (e.g., "2 calls, 1 email this week") or "No recent activity".

  *{{SEAM_COMPANY_NAME}}* | Intent: {{SEAM_COMPANY_INTENT_SCORE}} | Fit: {{SEAM_COMPANY_FIT_SCORE}} | Stage: {{SEAM_COMPANY_ABM_STAGE}}
  Recent: <activity summary>
  → {{SEAM_COMPANY_DOMAIN}}

  save_as: message_text

Step 4: Deliver to channel
  Channel routing:
  - If {{channel}} starts with "#": call slack_send_message(channel: "{{channel}}", text: message_text)
  - If {{channel}} contains "@": log "Email delivery not yet implemented in v1" and exit
