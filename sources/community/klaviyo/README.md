# Klaviyo

**Version:** 1.0.0
**Backend:** HTTP
**Base URL:** `https://a.klaviyo.com`

Query Klaviyo email marketing data as SQL tables. Inspect subscriber lists, campaigns, automation flows, and event metric definitions. Join with Shopify orders or Chargebee subscriptions for e-commerce revenue intelligence.

## Tables

| Table | Description | Required filters | Optional filters |
|-------|-------------|-----------------|-----------------|
| `klaviyo.lists` | Subscriber lists with opt-in configuration | — | `filter` |
| `klaviyo.campaigns` | Email and SMS campaigns with status and send time | `channel` | — |
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

### Lists

All subscriber lists (up to 100):

```sql
SELECT id, name, opt_in_process, created
FROM klaviyo.lists
ORDER BY created DESC;
```

Lists matching a name pattern (API-level filter — use to narrow below 100):

```sql
SELECT id, name, opt_in_process, created
FROM klaviyo.lists
WHERE filter = 'contains(name,"newsletter")'
ORDER BY created DESC;
```

Lists created after a date (combine with SQL WHERE to slice large accounts):

```sql
SELECT id, name, opt_in_process, created
FROM klaviyo.lists
WHERE filter = 'greater-than(created,"2024-01-01T00:00:00+00:00")'
ORDER BY created DESC;
```

### Campaigns

`channel` is required by the Klaviyo API. Valid values: `email`, `sms`, `mobile_push`.

Email campaigns (up to 100, most recently updated):

```sql
SELECT id, name, status, channel, send_time
FROM klaviyo.campaigns
WHERE channel = 'email'
ORDER BY send_time DESC;
```

SMS campaigns:

```sql
SELECT id, name, status, send_time
FROM klaviyo.campaigns
WHERE channel = 'sms'
ORDER BY send_time DESC;
```

Sent email campaigns only (SQL `WHERE` applied after fetch):

```sql
SELECT id, name, send_time
FROM klaviyo.campaigns
WHERE channel = 'email'
  AND status = 'Sent'
ORDER BY send_time DESC;
```

Email campaigns in a specific year (narrow below 100 with date range):

```sql
SELECT id, name, status, send_time
FROM klaviyo.campaigns
WHERE channel = 'email'
  AND send_time >= '2024-01-01T00:00:00+00:00'
  AND send_time < '2025-01-01T00:00:00+00:00'
ORDER BY send_time DESC;
```

### Flows

All automation flows (up to 100):

```sql
SELECT id, name, status, created
FROM klaviyo.flows
ORDER BY created DESC;
```

Live flows only (API-level filter — efficient for large accounts):

```sql
SELECT id, name, status, created
FROM klaviyo.flows
WHERE filter = 'equals(status,"live")'
ORDER BY created DESC;
```

Flows created after a date:

```sql
SELECT id, name, status, created
FROM klaviyo.flows
WHERE filter = 'greater-than(created,"2024-01-01T00:00:00+00:00")'
ORDER BY created DESC;
```

### Metrics

All event metrics (up to 100):

```sql
SELECT id, name, integration_name, integration_category
FROM klaviyo.metrics
ORDER BY integration_name ASC, name ASC;
```

Metrics from Shopify only:

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
    WHERE channel = 'email'
      AND status = 'Sent'
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

`klaviyo.lists`, `klaviyo.flows`, and `klaviyo.metrics` accept an optional `filter` SQL parameter whose value is a Klaviyo filter expression. String values must be double-quoted inside the expression; wrap the whole expression in single quotes in SQL.

| Pattern | Example |
|---------|---------|
| Equality | `equals(field,"value")` |
| Contains | `contains(field,"value")` |
| Greater-than | `greater-than(field,"value")` |
| Less-than | `less-than(field,"value")` |
| Combine | `and(expr1,expr2)` |

`klaviyo.campaigns` takes a plain `channel` filter (`email`, `sms`, or `mobile_push`) — the connector automatically constructs the required Klaviyo filter expression.

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
- Each table returns up to 100 records per query (Klaviyo's maximum `page[size]`). Klaviyo uses URL-embedded cursor pagination which is not supported by the Coral HTTP backend. Use the `filter` parameter on `lists`, `flows`, and `metrics` to narrow results with Klaviyo filter expressions (name, status, date range, etc.). For `campaigns`, query each channel separately and apply SQL `WHERE` conditions on `status`, `send_time`, and other columns to slice within the 100-record window.
- `klaviyo.campaigns` requires a `channel` value (`email`, `sms`, or `mobile_push`) — the Klaviyo API mandates a channel selector on this endpoint and returns an error without one. The connector automatically constructs `equals(messages.channel,"<value>")` from the `channel` filter.
- The `revision: 2023-12-15` header is sent automatically with every request as required by the Klaviyo API.
- All timestamp fields use ISO 8601 format with timezone offset (e.g. `2024-01-15T10:30:00+00:00`).
- Rate limit handling: `429` responses are retried automatically via `Retry-After`.
