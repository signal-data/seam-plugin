# Slack company alert format

Shared formatting rules and message template for all company-level Slack alerts
(intent spike, new opp, deal stage change, etc.). Workflow files reference this
to keep formatting consistent and DRY.

---

## Slack mrkdwn rules

These apply to all alert messages sent via the Slack connector:

- **Bold:** Use `**double asterisks**` (not single `*text*` — the connector uses standard markdown)
- **Italic:** Use `_underscores_`
- **Blank lines:** Put `⠀` (U+2800 braille blank) on its own line — Slack collapses regular empty lines
- **Links:** Use `<url|link text>` format
- **Emoji:** Only in the parent message header (e.g. 🔔). No emoji in score labels or body text.
- **Contacts:** Simple `• Name — _Title_` format. No nested links or extra metadata per contact.

## Link patterns

Use these URL templates throughout. Only include a link if the ID value is present and non-empty.

| Entity          | URL template                                                        | ID field              |
|-----------------|---------------------------------------------------------------------|-----------------------|
| Seam company    | `https://app.getseam.ai/companies/{{SEAM_COMPANY_ID}}`             | SEAM_COMPANY_ID       |
| Salesforce acct | `https://seam.my.salesforce.com/{{SFDC_ACCT_ACCOUNT_ID}}`          | SFDC_ACCT_ACCOUNT_ID  |
| HubSpot company | `https://app.hubspot.com/contacts/242151588/record/0-2/{{HUBS_COMP_ID}}` | HUBS_COMP_ID   |
| Seam person     | `https://app.getseam.ai/people/{{SEAM_PEOPLE_ID}}`                 | SEAM_PEOPLE_ID        |
| Salesforce lead | `https://seam.my.salesforce.com/{{SFDC_PERSON_ID}}`                | SFDC_PERSON_ID        |

## Message template — parent (first account)

The parent message includes the alert header with @mention. Subsequent accounts
use the thread reply template (same body, no header).

```
🔔 <@{{owner_slack_id}}> - {{alert_title}}!
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
```

## Message template — thread reply (accounts #2+)

Same as parent but without the 🔔 header line. Starts directly with `**Company Name:**`.
**All sections must be included** — scores, contacts, summary, AND deep links.

```
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
```

## Message template — summary (end of thread)

Only sent when there are more accounts than the top N shown.

```
**+ {{remaining_count}} more account(s)** above {{threshold}} intent score.
⠀
<https://app.getseam.ai/companies|View all in Seam>
```

## Summary generation

The **Summary** field is an LLM-generated 1–2 sentence description composed from
real Seam data fields. It should:

1. **Include specific topic surge names and scores** from SEAM_COMPANY_INTENT_DETAILS
   (e.g. "surging on ABM (79), Predictive Lead Scoring (69)")
2. **Reference the fit explanation** from SEAM_COMPANY_FIT_EXPLANATION to explain
   why the fit score is what it is
3. **Mention engagement signals** from SEAM_COMPANY_ABM_STAGE_DESCRIPTION
   (e.g. "showing indirect engagement across 12 topics with 9 social keyword mentions")
4. **Include interests** from SEAM_COMPANY_INTERESTS if present
5. Skip any field that is null or empty
6. Write in plain English, not bullet points — this is a quick read in Slack

### Fields used for summary generation

| Field                              | What it provides                                   |
|------------------------------------|----------------------------------------------------|
| SEAM_COMPANY_FIT_EXPLANATION       | Why the fit score is rated the way it is            |
| SEAM_COMPANY_ABM_STAGE_DESCRIPTION | Engagement signals: topic surges, social mentions   |
| SEAM_COMPANY_INTERESTS             | Topics/keywords the account is researching          |
| SEAM_COMPANY_INTENT_DETAILS        | Specific topic surges with scores driving intent    |

## Required company fields

Any workflow using this format must query at minimum:

```
SEAM_COMPANY_ID, SEAM_COMPANY_NAME, SEAM_COMPANY_DOMAIN,
SEAM_COMPANY_INDUSTRY, SEAM_COMPANY_EMPLOYEES, SEAM_COMPANY_DESCRIPTION,
SEAM_COMPANY_INTENT_SCORE, SEAM_COMPANY_INTENT_LEVEL, SEAM_COMPANY_INTENT_DETAILS,
SEAM_COMPANY_FIT_SCORE, SEAM_COMPANY_FIT_NAME, SEAM_COMPANY_FIT_EXPLANATION,
SEAM_COMPANY_SIGNAL_SCORE, SEAM_COMPANY_SIGNAL_STRENGTH,
SEAM_COMPANY_REACH_SCORE, SEAM_COMPANY_REACH_LEVEL,
SEAM_COMPANY_ABM_STAGE, SEAM_COMPANY_ABM_STAGE_DESCRIPTION,
SEAM_COMPANY_INTERESTS, SEAM_COMPANY_LAST_ACTIVE_DATE,
SEAM_COMPANY_OWNER_EMAIL, SEAM_COMPANY_OWNER_FULL_NAME,
SEAM_COMPANY_LINKEDIN_URL, SFDC_ACCT_ACCOUNT_ID, HUBS_COMP_ID
```

## Required people fields

```
SEAM_PEOPLE_ID, SEAM_PEOPLE_FIRST_NAME, SEAM_PEOPLE_LAST_NAME,
SEAM_PEOPLE_TITLE, SEAM_PEOPLE_EMAIL, SEAM_PEOPLE_INTENT_SCORE,
SEAM_PEOPLE_INTENT_LEVEL, SEAM_PEOPLE_BUYING_STAGE,
SEAM_PEOPLE_CHAMPION, SEAM_PEOPLE_ASSOCIATED_TO_OPPORTUNITY,
SFDC_PERSON_ID
```

Note: `HUBS_CONTACT_ID` is NOT a valid field in the people dataset — do not include it.