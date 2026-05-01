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
schedule:
  default: "0 9 * * 1-5"
  description: Weekdays at 9 AM in the owner's timezone
  recommended_cadences:
    - cron: "0 9 * * 1-5"
      label: Weekday mornings (recommended)
      description: Run every weekday at 9 AM — catches overnight intent spikes before the team's first meetings.
    - cron: "0 9 * * 1"
      label: Weekly (Monday morning)
      description: Run once a week on Monday at 9 AM — good for lower-volume teams or higher thresholds.
    - cron: "0 9,14 * * 1-5"
      label: Twice daily (weekdays)
      description: Run at 9 AM and 2 PM on weekdays — for high-velocity teams that want mid-day updates.
---

## References

- See `references/slack-company-alert-format.md` for Slack formatting rules,
  link patterns, message templates, and summary generation guidelines.
- All message formatting in this workflow follows that shared reference.

---

## Steps

Step 1a: Query A/B fit accounts with spiking intent (priority accounts)
  search_companies({
    filter: "SEAM_COMPANY_INTENT_SCORE>={{threshold}};SEAM_COMPANY_FIT_SCORE=in=(A,B)",
    relationFilters: [{ relation: "users", filter: "SEAM_USER_EMAIL=='{{owner_email}}'" }],
    fields: [
      "SEAM_COMPANY_ID",
      "SEAM_COMPANY_NAME",
      "SEAM_COMPANY_DOMAIN",
      "SEAM_COMPANY_INDUSTRY",
      "SEAM_COMPANY_EMPLOYEES",
      "SEAM_COMPANY_DESCRIPTION",
      "SEAM_COMPANY_INTENT_SCORE",
      "SEAM_COMPANY_INTENT_LEVEL",
      "SEAM_COMPANY_INTENT_DETAILS",
      "SEAM_COMPANY_FIT_SCORE",
      "SEAM_COMPANY_FIT_NAME",
      "SEAM_COMPANY_FIT_EXPLANATION",
      "SEAM_COMPANY_SIGNAL_SCORE",
      "SEAM_COMPANY_SIGNAL_STRENGTH",
      "SEAM_COMPANY_REACH_SCORE",
      "SEAM_COMPANY_REACH_LEVEL",
      "SEAM_COMPANY_ABM_STAGE",
      "SEAM_COMPANY_ABM_STAGE_DESCRIPTION",
      "SEAM_COMPANY_INTERESTS",
      "SEAM_COMPANY_LAST_ACTIVE_DATE",
      "SEAM_COMPANY_OWNER_EMAIL",
      "SEAM_COMPANY_OWNER_FULL_NAME",
      "SEAM_COMPANY_LINKEDIN_URL",
      "SFDC_ACCT_ACCOUNT_ID",
      "HUBS_COMP_ID"
    ],
    sort: "SEAM_COMPANY_INTENT_SCORE,desc",
    size: 10
  })
  save_as: priority_accounts

Step 1b: Query all accounts with spiking intent (for total count + backfill)
  search_companies({
    filter: "SEAM_COMPANY_INTENT_SCORE>={{threshold}}",
    relationFilters: [{ relation: "users", filter: "SEAM_USER_EMAIL=='{{owner_email}}'" }],
    fields: [same as Step 1a],
    sort: "SEAM_COMPANY_INTENT_SCORE,desc",
    size: 10,
    includeTotal: true
  })
  save_as: all_accounts
  save total count as: total_spiking

Step 2: Check if any results
  condition: all_accounts.length == 0
  if true: exit silently — no alert needed

Step 3: Merge and prioritize top 10
  Build the final list by taking A/B fit accounts first, then backfilling:
    1. Start with all accounts from priority_accounts (A/B fit), sorted by intent desc
    2. If fewer than 10, backfill from all_accounts — skip any already included
       (match on SEAM_COMPANY_ID to deduplicate)
    3. Cap at 10 total
  save_as: top_10
  remaining_count = total_spiking - top_10.length

  If priority_accounts is empty (no A/B fit accounts, or all fit scores are null),
  fall back to all_accounts sorted by intent score desc — same as before.

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
        "SFDC_PERSON_ID"
      ],
      sort: "SEAM_PEOPLE_INTENT_SCORE,desc",
      size: 3
    })
    save_as: account.top_contacts

Step 6: Generate summary for each top-10 account
  For each account in top_10, compose a 1–2 sentence **Summary** by combining
  the actual data from these fields:
    - SEAM_COMPANY_FIT_EXPLANATION (why the fit score is what it is)
    - SEAM_COMPANY_ABM_STAGE_DESCRIPTION (engagement signals and topic/social mentions)
    - SEAM_COMPANY_INTERESTS (topics/keywords the account is researching)
    - SEAM_COMPANY_INTENT_DETAILS (specific topic surges with scores driving the intent)
  Write it in plain English. Include specific topic names and surge scores from
  the intent details (e.g. "surging on ABM (79), Predictive Lead Scoring (69)").
  Reference the fit explanation to explain why fit is rated the way it is.
  Mention engagement details from the ABM stage description.
  Skip any field that is null or empty.
  save_as: account.summary

Step 7: Send parent message (Account #1)
  Build the message for the first account in top_10.

  ---
  MESSAGE FORMAT — Parent (Account #1):
  ---

  🔔 <@{{owner_slack_id}}> - New High Intent Company Alert!
  ⠀
  **Company Name:** {{SEAM_COMPANY_NAME}}
  {{SEAM_COMPANY_INDUSTRY}} · {{SEAM_COMPANY_EMPLOYEES}} employees
  ⠀
  **Intent:** {{SEAM_COMPANY_INTENT_SCORE}} ({{SEAM_COMPANY_INTENT_LEVEL}})
  **Fit:** {{SEAM_COMPANY_FIT_SCORE}} ({{SEAM_COMPANY_FIT_NAME}})
  **Signal:** {{SEAM_COMPANY_SIGNAL_SCORE}} ({{SEAM_COMPANY_SIGNAL_STRENGTH}})
  **Reach:** {{SEAM_COMPANY_REACH_SCORE}} ({{SEAM_COMPANY_REACH_LEVEL}})
  **ABM Stage:** {{SEAM_COMPANY_ABM_STAGE}}
  **Last Active:** {{SEAM_COMPANY_LAST_ACTIVE_DATE formatted as Mon DD}}
  **Owner:** {{SEAM_COMPANY_OWNER_FULL_NAME}}
  ⠀
  {{if account.top_contacts.length > 0}}
  **Top Contacts:**
     {{for each contact in account.top_contacts}}
     • {{SEAM_PEOPLE_FIRST_NAME}} {{SEAM_PEOPLE_LAST_NAME}} — _{{SEAM_PEOPLE_TITLE}}_
     {{end for}}
  ⠀
  {{endif}}
  **Summary:** {{account.summary}}
  ⠀
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

Step 8: Send thread replies (Accounts #2–10)
  For each account in top_10[1..9] (index 1 through 9):

    ---
    MESSAGE FORMAT — Thread reply (Accounts #2–10):
    ---

    **Company Name:** {{SEAM_COMPANY_NAME}}
    {{SEAM_COMPANY_INDUSTRY}} · {{SEAM_COMPANY_EMPLOYEES}} employees
    ⠀
    **Intent:** {{SEAM_COMPANY_INTENT_SCORE}} ({{SEAM_COMPANY_INTENT_LEVEL}})
    **Fit:** {{SEAM_COMPANY_FIT_SCORE}} ({{SEAM_COMPANY_FIT_NAME}})
    **Signal:** {{SEAM_COMPANY_SIGNAL_SCORE}} ({{SEAM_COMPANY_SIGNAL_STRENGTH}})
    **Reach:** {{SEAM_COMPANY_REACH_SCORE}} ({{SEAM_COMPANY_REACH_LEVEL}})
    **ABM Stage:** {{SEAM_COMPANY_ABM_STAGE}}
    **Last Active:** {{SEAM_COMPANY_LAST_ACTIVE_DATE formatted as Mon DD}}
    **Owner:** {{SEAM_COMPANY_OWNER_FULL_NAME}}
    ⠀
    {{if account.top_contacts.length > 0}}
    **Top Contacts:**
       {{for each contact in account.top_contacts}}
       • {{SEAM_PEOPLE_FIRST_NAME}} {{SEAM_PEOPLE_LAST_NAME}} — _{{SEAM_PEOPLE_TITLE}}_
       {{end for}}
    ⠀
    {{endif}}
    **Summary:** {{account.summary}}
    ⠀
    <{{seam_company_link}}|View in Seam>{{if SFDC_ACCT_ACCOUNT_ID}} · <{{sfdc_account_link}}|Salesforce>{{endif}}{{if HUBS_COMP_ID}} · <{{hubspot_company_link}}|HubSpot>{{endif}}{{if SEAM_COMPANY_LINKEDIN_URL}} · <{{SEAM_COMPANY_LINKEDIN_URL}}|LinkedIn>{{endif}}

    ---
    END MESSAGE FORMAT
    ---

    slack_send_message({
      channel_id: "{{channel}}",
      message: <formatted message above>,
      thread_ts: "{{parent_ts}}"
    })

Step 9: Send summary reply (if remaining accounts exist)
  condition: remaining_count > 0
  if false: skip this step

  ---
  MESSAGE FORMAT — Summary:
  ---

  **+ {{remaining_count}} more account(s)** above {{threshold}} intent score.
  ⠀
  <https://app.getseam.ai/companies|View all in Seam>

  ---
  END MESSAGE FORMAT
  ---

  slack_send_message({
    channel_id: "{{channel}}",
    message: <formatted message above>,
    thread_ts: "{{parent_ts}}"
  })

  On success: done (no user-facing confirmation needed — this runs headlessly)