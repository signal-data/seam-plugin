# Alert Types

## intent-spike

Alert when owned accounts cross an intent score threshold.

### Inputs
| Input | Type | Required | Default | Description |
|---|---|---|---|---|
| owner_email | string | yes | — | Email of the rep who owns the accounts |
| threshold | number | no | 80 | Intent score threshold (0–100) |
| channel | string | no | #hot-accounts | Delivery: "#channel" for Slack, "email@domain.com" for email |

### Schedule suggestions
- "every weekday morning" → `0 9 * * 1-5`
- "twice a week" → `0 9 * * 1,4`
- "daily at 8am" → `0 8 * * *`

---

## stage-change

Surface owned accounts currently at a target ABM stage. Note: this reports accounts *at* the stage right now, not ones that recently moved to it — true change detection requires persisted state and is planned for v2.

### Inputs
| Input | Type | Required | Default | Description |
|---|---|---|---|---|
| owner_email | string | yes | — | Email of the rep who owns the accounts |
| target_stage | string | yes | — | ABM stage value to filter on |
| channel | string | no | #pipeline-alerts | Delivery channel |

### Finding valid stage values
Ask Claude: "What are the valid ABM stage values?" — Claude will call `get_company_fields` to list them.

---

## weekly-digest

Weekly rollup of top accounts by intent score with recent activity summary.

### Inputs
| Input | Type | Required | Default | Description |
|---|---|---|---|---|
| owner_email | string | yes | — | Email of the rep who owns the accounts |
| top_n | number | no | 10 | Number of top accounts to include |
| channel | string | yes | — | Delivery channel |

### Schedule suggestions
- "every Monday morning" → `0 9 * * 1`
- "every Friday afternoon" → `0 15 * * 5`

---

## re-engagement

Surface owned accounts with no activity in the last N days.

### Inputs
| Input | Type | Required | Default | Description |
|---|---|---|---|---|
| owner_email | string | yes | — | Email of the rep who owns the accounts |
| days_inactive | number | no | 30 | Days without activity to qualify |
| channel | string | yes | — | Delivery channel |

### Schedule suggestions
- "weekly on Monday" → `0 9 * * 1`
- "every two weeks" → `0 9 1,15 * *`
