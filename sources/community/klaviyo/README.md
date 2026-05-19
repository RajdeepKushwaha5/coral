# Klaviyo

**Version:** 1.0.0
**Backend:** HTTP
**Base URL:** `https://a.klaviyo.com`

Query Klaviyo email marketing data as SQL tables. Inspect subscriber lists, campaigns, automation flows, and event metric definitions. Join with Shopify orders or Chargebee subscriptions for e-commerce revenue intelligence.

## Tables

| Table | Description | Required filters | Optional filters |
|-------|-------------|-----------------|-----------------|
| `klaviyo.lists` | Subscriber lists with opt-in configuration | — | `filter` |
| `klaviyo.campaigns` | Email and SMS campaigns with status and send time | `filter` | — |
| `klaviyo.flows` | Automation flow definitions with status | — | `filter` |
| `klaviyo.metrics` | Event metric definitions with integration source | — | `filter` |

## Authentication

Requires `KLAVIYO_API_KEY`.

**To get your API key:**

1. Log in to your Klaviyo dashboard
2. Go to **Settings** → **API Keys**
3. Click **Create Private API Key**
4. Select read-only scopes for the resources you plan to query

The connector uses an `Authorization: Klaviyo-API-Key {key}` header and the API revision `2023-12-15`.

## Install

```bash
coral source lint manifest.yaml
coral source add --file manifest.yaml
coral source test klaviyo
```

Or with the key inline:

```bash
KLAVIYO_API_KEY=your-key coral source add --file manifest.yaml
```

## Example Queries

All subscriber lists (up to 100):

```sql
SELECT id, name, opt_in_process, created
FROM klaviyo.lists
ORDER BY created DESC;
```

Lists matching a name pattern:

```sql
SELECT id, name, opt_in_process, created
FROM klaviyo.lists
WHERE filter = 'contains(name,"newsletter")'
ORDER BY created DESC;
```

Email campaigns (channel filter is required by the Klaviyo API):

```sql
SELECT id, name, status, channel, send_time
FROM klaviyo.campaigns
WHERE filter = 'equals(messages.channel,"email")'
ORDER BY send_time DESC;
```

Sent email campaigns only (combine filters with AND):

```sql
SELECT id, name, send_time
FROM klaviyo.campaigns
WHERE filter = 'and(equals(messages.channel,"email"),equals(status,"Sent"))'
ORDER BY send_time DESC;
```

SMS campaigns:

```sql
SELECT id, name, status, send_time
FROM klaviyo.campaigns
WHERE filter = 'equals(messages.channel,"sms")'
ORDER BY send_time DESC;
```

Live automation flows:

```sql
SELECT id, name, status, created
FROM klaviyo.flows
WHERE filter = 'equals(status,"live")'
ORDER BY created DESC;
```

Event metrics from Shopify:

```sql
SELECT id, name, integration_name, integration_category
FROM klaviyo.metrics
WHERE filter = 'equals(integration.name,"Shopify")'
ORDER BY name ASC;
```

## Cross-Source JOIN Example

Campaign send volume versus support ticket volume by day — detect whether email campaigns correlate with support spikes (requires `freshdesk` source installed):

```sql
WITH campaign_days AS (
    SELECT
        SUBSTR(send_time, 1, 10) AS day,
        COUNT(*)                 AS campaigns_sent
    FROM klaviyo.campaigns
    WHERE filter = 'and(equals(messages.channel,"email"),equals(status,"Sent"))'
    GROUP BY SUBSTR(send_time, 1, 10)
),
ticket_days AS (
    SELECT
        SUBSTR(created_at, 1, 10) AS day,
        COUNT(*)                  AS tickets_opened
    FROM freshdesk.tickets
    GROUP BY SUBSTR(created_at, 1, 10)
)
SELECT
    COALESCE(c.day, t.day)        AS date,
    COALESCE(c.campaigns_sent, 0) AS campaigns_sent,
    COALESCE(t.tickets_opened, 0) AS tickets_opened
FROM campaign_days c
FULL OUTER JOIN ticket_days t ON t.day = c.day
ORDER BY date DESC;
```

## Klaviyo Filter Syntax

Klaviyo filter expressions use function-call syntax. String values must be double-quoted inside the expression. Wrap the whole expression in single quotes in SQL.

| Pattern | Example |
|---------|---------|
| Equality | `equals(field,"value")` |
| Contains | `contains(field,"value")` |
| Combine | `and(expr1,expr2)` |

Common filter fields:

- `messages.channel` — `"email"`, `"sms"`, `"mobile_push"` (campaigns only; required)
- `status` — campaign or flow status value
- `name` — list or flow name

## Status Reference

### Campaign status

| Value | Meaning |
|-------|---------|
| `Draft` | Not yet sent or scheduled |
| `Scheduled` | Queued for future delivery |
| `Sending` | Currently being delivered |
| `Sent` | Delivery complete |
| `Cancelled` | Sending was cancelled |

### Flow status

| Value | Meaning |
|-------|---------|
| `draft` | Not yet live |
| `live` | Active and triggering |
| `manual` | Paused for manual review |
| `archived` | Archived, no longer triggering |

## Notes

- All tables are strictly read-only.
- Each table returns up to 100 records per query (Klaviyo's maximum `page[size]`). Klaviyo uses URL-embedded cursor pagination which is not supported by the Coral HTTP backend. For accounts with more than 100 of any resource, use the optional `filter` parameter to narrow results to the subset you need.
- `klaviyo.campaigns` requires a `filter` with a channel expression (`equals(messages.channel,"email")` etc.) — this is enforced by the Klaviyo API, not just the connector.
- The `revision: 2023-12-15` header is sent automatically with every request as required by the Klaviyo API.
- All timestamp fields use ISO 8601 format with timezone offset (e.g. `2024-01-15T10:30:00+00:00`).
- Rate limit handling: `429` responses are retried automatically via `Retry-After`.
