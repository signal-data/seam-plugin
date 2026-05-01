---
name: Intent Spike Alert
description: Alert when owned accounts cross an intent score threshold. Sends the top 10 accounts as individual rich messages in a Slack thread with deep links to Seam, Salesforce, and HubSpot.
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
    default: 85
  - name: channel
    type: string
    required: false
    default: "#hot-accounts"
---

## Link patterns

Use these URL templates throughout. Only include a link if the ID value is present and non-empty.

- Seam company: `https://app.getseam.ai/companies/{{SEAM_COMPANY_ID}}`
- Salesforce account: `https://seam.my.salesforce.com/{{SFDC_ACCT_ACCOUNT_ID}}`
- HubSpot company: `https://app.hubspot.com/contacts/242151588/record/0-2/{{HUBS_COMP_ID}}`
- Seam person: `https://app.getseam.ai/people/{{SEAM_PEOPLE_ID}}`
- Salesforce person: `https://seam.my.salesforce.com/{{SFDC_PERSON_ID}}`
- HubSpot contact: `https://app.hubspot.com/contacts/242151588/record/0-1/{{HUBS_CONTACT_ID}}`

---

## Steps

Step 1: Query accounts with spiking intent
  search_companies({
    filter: "SEAM_COMPANY_INTENT_SCORE>={{threshold}}",
    relationFilters: [{ relation: "users", filter: "SEAM_USER_EMAIL=='{{owner_email}}'" }],
    fields: [
      "SEAM_COMPANY_ID",
      "SEAM_COMPANY_NAME",
      "SEAM_COMPANY_DOMAIN",
      "SEAM_COMPANY_INDUSTRY",
      "SEAM_COMPANY_EMPLOYEES",
      "SEAM_COMPANY_INTENT_SCORE",
      "SEAM_COMPANY_INTENT_LEVEL",
      "SEAM_COMPANY_FIT_SCORE",
      "SEAM_COMPANY_FIT_NAME",
      "SEAM_COMPANY_SIGNAL_SCORE",
      "SEAM_COMPANY_SIGNAL_STRENGTH",
      "SEAM_COMPANY_REACH_SCORE",
      "SEAM_COMPANY_REACH_LEVEL",
      "SEAM_COMPANY_ABM_STAGE",
      "SEAM_COMPANY_BUYING_STAGE",
      "SEAM_COMPANY_LAST_ACTIVE_DATE",
      "SEAM_COMPANY_OWNER_EMAIL",
      "SEAM_COMPANY_OWNER_FULL_NAME",
      "SEAM_COMPANY_LINKEDIN_URL",
      "SFDC_ACCT_ACCOUNT_ID",
      "HUBS_COMP_ID"
    ],
    sort: "SEAM_COMPANY_INTENT_SCORE,desc",
    includeTotal: true
  })
  save_as: spiking_accounts
  save total count as: total_spiking

Step 2: Check if any results
  condition: spiking_accounts.length == 0
  if true: exit silently — no alert needed

Step 3: Split results
  top_10 = spiking_accounts.slice(0, 10)
  remaining_count = total_spiking - top_10.length

Step 4: Look up owner Slack user ID
  slack_search_users({ query: "{{owner_email}}" })
  Extract the user_id from the first result.
  save_as: owner_slack_id
  If no match found, set owner_slack_id to null (skip @mention in messages).

Step 5: Query top contacts for each top-10 account
  For each account in top_10:
    search_people({
      filter: "SEAM_PEOPLE_COMPANY_DOMAIN=='{{account.SEAM_COMPANY_DOMAIN}}'",
      fields: [
        "SEAM_PEOPLE_ID",
        "SEAM_PEOPLE_FIRST_NAME",
        "SEAM_PEOPLE_LAST_NAME",
        "SEAM_PEOPLE_TITLE",
        "SEAM_PEOPLE_EMAIL",
        "SEAM_PEOPLE_INTENT_SCORE",
        "SEAM_PEOPLE_INTENT_LEVEL",
        "SEAM_PEOPLE_BUYING_STAGE",
        "SEAM_PEOPLE_CHAMPION",
        "SEAM_PEOPLE_ASSOCIATED_TO_OPPORTUNITY",
        "SFDC_PERSON_ID",
        "HUBS_CONTACT_ID"
      ],
      sort: "SEAM_PEOPLE_INTENT_SCORE,desc",
      size: 3
    })
    save_as: account.top_contacts

Step 6: Send parent message (Account #1)
  Build the message for the first account in top_10. Use Slack mrkdwn formatting.

  ---
  MESSAGE FORMAT — Parent (Account #1):
  ---

  🔥 *Intent Spike Alert* — {{total_spiking}} account(s) at or above {{threshold}} intent
  {{if owner_slack_id}} <@{{owner_slack_id}}> {{endif}}

  ━━━━━━━━━━━━━━━━━━━━━━━━

  *{{SEAM_COMPANY_NAME}}* — _{{SEAM_COMPANY_INDUSTRY}}_
  {{SEAM_COMPANY_EMPLOYEES}} employees

  *Scores*
  🎯 Intent: *{{SEAM_COMPANY_INTENT_SCORE}}* ({{SEAM_COMPANY_INTENT_LEVEL}})  •  💎 Fit: *{{SEAM_COMPANY_FIT_SCORE}}* ({{SEAM_COMPANY_FIT_NAME}})
  📡 Signal: *{{SEAM_COMPANY_SIGNAL_SCORE}}* ({{SEAM_COMPANY_SIGNAL_STRENGTH}})  •  📣 Reach: *{{SEAM_COMPANY_REACH_SCORE}}* ({{SEAM_COMPANY_REACH_LEVEL}})

  *Stage:* {{SEAM_COMPANY_ABM_STAGE}}  •  *Buying Stage:* {{SEAM_COMPANY_BUYING_STAGE}}
  *Last Active:* {{SEAM_COMPANY_LAST_ACTIVE_DATE}}  •  *Owner:* {{SEAM_COMPANY_OWNER_FULL_NAME}}

  {{if account.top_contacts.length > 0}}
  *Top Contacts*
  {{for each contact in account.top_contacts}}
  • *{{SEAM_PEOPLE_FIRST_NAME}} {{SEAM_PEOPLE_LAST_NAME}}* — {{SEAM_PEOPLE_TITLE}}
    Intent: {{SEAM_PEOPLE_INTENT_SCORE}} ({{SEAM_PEOPLE_INTENT_LEVEL}}) · Buying: {{SEAM_PEOPLE_BUYING_STAGE}} {{if SEAM_PEOPLE_CHAMPION}} · 🏆 Champion{{endif}}
    <{{seam_person_link}}|Seam>{{if SFDC_PERSON_ID}} · <{{sfdc_person_link}}|Salesforce>{{endif}}{{if HUBS_CONTACT_ID}} · <{{hubspot_person_link}}|HubSpot>{{endif}}
  {{end for}}
  {{endif}}

  <{{seam_company_link}}|View in Seam>{{if SFDC_ACCT_ACCOUNT_ID}} · <{{sfdc_account_link}}|Salesforce>{{endif}}{{if HUBS_COMP_ID}} · <{{hubspot_company_link}}|HubSpot>{{endif}}{{if SEAM_COMPANY_LINKEDIN_URL}} · <{{SEAM_COMPANY_LINKEDIN_URL}}|LinkedIn>{{endif}}

  ---
  END MESSAGE FORMAT
  ---

  slack_send_message({
    channel_id: "{{channel}}",
    message: <formatted message above>
  })
  Capture the returned message timestamp.
  save_as: parent_ts

Step 7: Send thread replies (Accounts #2–10)
  For each account in top_10[1..9] (index 1 through 9):
    Build the message using the same format as Step 6 but WITHOUT the header line (no 🔥 line and no @mention). Start directly from the company name.

    ---
    MESSAGE FORMAT — Thread reply (Accounts #2–10):
    ---

    *{{SEAM_COMPANY_NAME}}* — _{{SEAM_COMPANY_INDUSTRY}}_
    {{SEAM_COMPANY_EMPLOYEES}} employees

    *Scores*
    🎯 Intent: *{{SEAM_COMPANY_INTENT_SCORE}}* ({{SEAM_COMPANY_INTENT_LEVEL}})  •  💎 Fit: *{{SEAM_COMPANY_FIT_SCORE}}* ({{SEAM_COMPANY_FIT_NAME}})
    📡 Signal: *{{SEAM_COMPANY_SIGNAL_SCORE}}* ({{SEAM_COMPANY_SIGNAL_STRENGTH}})  •  📣 Reach: *{{SEAM_COMPANY_REACH_SCORE}}* ({{SEAM_COMPANY_REACH_LEVEL}})

    *Stage:* {{SEAM_COMPANY_ABM_STAGE}}  •  *Buying Stage:* {{SEAM_COMPANY_BUYING_STAGE}}
    *Last Active:* {{SEAM_COMPANY_LAST_ACTIVE_DATE}}  •  *Owner:* {{SEAM_COMPANY_OWNER_FULL_NAME}}

    {{if account.top_contacts.length > 0}}
    *Top Contacts*
    {{for each contact in account.top_contacts}}
    • *{{SEAM_PEOPLE_FIRST_NAME}} {{SEAM_PEOPLE_LAST_NAME}}* — {{SEAM_PEOPLE_TITLE}}
      Intent: {{SEAM_PEOPLE_INTENT_SCORE}} ({{SEAM_PEOPLE_INTENT_LEVEL}}) · Buying: {{SEAM_PEOPLE_BUYING_STAGE}} {{if SEAM_PEOPLE_CHAMPION}} · 🏆 Champion{{endif}}
      <{{seam_person_link}}|Seam>{{if SFDC_PERSON_ID}} · <{{sfdc_person_link}}|Salesforce>{{endif}}{{if HUBS_CONTACT_ID}} · <{{hubspot_person_link}}|HubSpot>{{endif}}
    {{end for}}
    {{endif}}

    <{{seam_company_link}}|View in Seam>{{if SFDC_ACCT_ACCOUNT_ID}} · <{{sfdc_account_link}}|Salesforce>{{endif}}{{if HUBS_COMP_ID}} · <{{hubspot_company_link}}|HubSpot>{{endif}}{{if SEAM_COMPANY_LINKEDIN_URL}} · <{{SEAM_COMPANY_LINKEDIN_URL}}|LinkedIn>{{endif}}

    ---
    END MESSAGE FORMAT
    ---

    slack_send_message({
      channel_id: "{{channel}}",
      message: <formatted message above>,
      thread_ts: "{{parent_ts}}"
    })

Step 8: Send summary reply (if remaining accounts exist)
  condition: remaining_count > 0
  if false: skip this step

  Build the summary message:

  ---
  MESSAGE FORMAT — Summary:
  ---

  📊 *Plus {{remaining_count}} more account(s)* at or above {{threshold}} intent score.

  <https://app.getseam.ai/companies|View all accounts in Seam>

  ---
  END MESSAGE FORMAT
  ---

  slack_send_message({
    channel_id: "{{channel}}",
    message: <formatted message above>,
    thread_ts: "{{parent_ts}}"
  })

  On success: done (no user-facing confirmation needed — this runs headlessly)