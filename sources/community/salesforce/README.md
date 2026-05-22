# Salesforce

**Version:** 0.1.0
**Backend:** HTTP (Salesforce REST API v59.0)
**Base URL:** `https://{your-instance}.my.salesforce.com`

Query Salesforce CRM accounts, contacts, leads, opportunities, and cases as SQL tables. Join with GitHub PRs, Linear issues, and PagerDuty incidents for cross-source sales and support intelligence.

## Tables

| Table | Description | Cap |
| --- | --- | --- |
| `salesforce.accounts` | Companies and organizations with industry, revenue, and billing location | 2000 |
| `salesforce.contacts` | Individuals linked to accounts with email, phone, and title | 2000 |
| `salesforce.leads` | Prospective contacts not yet converted to accounts | 2000 |
| `salesforce.opportunities` | Deals in the sales pipeline with stage, amount, and close date | 2000 |
| `salesforce.cases` | Customer support tickets with status, priority, and origin | 2000 |

All tables return up to 2000 records ordered by `LastModifiedDate DESC`. Use SQL filters to narrow the set.

## Authentication

Requires `SALESFORCE_API_URL` and `SALESFORCE_ACCESS_TOKEN`.

**To find your instance URL:**

Go to **Setup → Company Information → Instance URL**. Use the full URL without a trailing slash, for example `https://mycompany.my.salesforce.com`.

**To get an access token:**

1. Create a Connected App under **Setup → App Manager → New Connected App**
2. Enable **OAuth Settings** with the `api` scope
3. Retrieve a token using the Salesforce CLI:

```bash
sf org display --target-org <alias>
```

Or use the [Username-Password OAuth flow](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/quickstart_oauth.htm).

## Install

```bash
coral source lint manifest.yaml
coral source add --file manifest.yaml
coral source test salesforce
```

Or with credentials inline:

```bash
SALESFORCE_API_URL=https://mycompany.my.salesforce.com \
SALESFORCE_ACCESS_TOKEN=your-token \
coral source add --file manifest.yaml
```

## Example Queries

Open opportunities by stage:

```sql
SELECT name, stage_name, amount, close_date, owner_id
FROM salesforce.opportunities
WHERE is_closed = false
ORDER BY close_date ASC
LIMIT 25;
```

High-priority open cases:

```sql
SELECT id, subject, status, priority, account_id, created_date
FROM salesforce.cases
WHERE is_closed = false
  AND priority = 'High'
ORDER BY created_date DESC
LIMIT 25;
```

Recent leads from web:

```sql
SELECT id, first_name, last_name, email, company, status
FROM salesforce.leads
WHERE lead_source = 'Web'
  AND is_converted = false
ORDER BY last_modified_date DESC
LIMIT 50;
```

Contacts at accounts in a specific industry:

```sql
SELECT c.email, c.first_name, c.last_name, a.name AS account_name
FROM salesforce.contacts c
JOIN salesforce.accounts a ON c.account_id = a.id
WHERE a.industry = 'Technology'
ORDER BY a.name ASC
LIMIT 50;
```

## Cross-Source JOIN Examples

Contacts with open GitHub PRs (requires `github` source):

```sql
SELECT c.first_name, c.last_name, c.email, COUNT(pr.id) AS open_prs
FROM salesforce.contacts c
LEFT JOIN github.pull_requests pr
  ON LOWER(pr.author_email) = LOWER(c.email)
  AND pr.state = 'open'
GROUP BY c.first_name, c.last_name, c.email
ORDER BY open_prs DESC
LIMIT 20;
```

Open Salesforce cases alongside open PagerDuty incidents for the same account (requires `pagerduty` source):

```sql
SELECT sf.subject AS case_subject, sf.priority, pd.title AS incident_title
FROM salesforce.cases sf
JOIN salesforce.contacts c ON sf.contact_id = c.id
JOIN pagerduty.incidents pd ON LOWER(pd.user_email) = LOWER(c.email)
WHERE sf.is_closed = false
  AND pd.status != 'resolved'
LIMIT 20;
```

Opportunities owned by engineers who also have open Linear issues (requires `linear` source):

```sql
SELECT o.name AS opportunity, o.stage_name, o.amount, li.title AS linear_issue
FROM salesforce.opportunities o
JOIN salesforce.contacts c ON o.account_id = c.account_id
JOIN linear.issues li ON LOWER(li.assignee_email) = LOWER(c.email)
WHERE o.is_closed = false
  AND li.state_type != 'completed'
LIMIT 20;
```

## Notes

- All tables are read-only.
- Each table is backed by a SOQL query returning up to 2000 records ordered by `LastModifiedDate DESC`. For orgs with more records, use SQL filters after fetching to scope results.
- `close_date` and `converted_date` are date-only strings (`YYYY-MM-DD`), not full timestamps.
- `amount`, `annual_revenue`, and `probability` are returned as strings to preserve decimal precision.
- Access tokens expire; re-run `sf org display` or re-authorize to refresh.
- The Salesforce API enforces per-org rate limits. Each table query counts as one API call.
