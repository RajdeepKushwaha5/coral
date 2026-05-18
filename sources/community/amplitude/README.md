# Amplitude

**Version:** 1.0.0
**Backend:** HTTP
**Base URL:** `https://amplitude.com/api`

Query Amplitude Analytics as SQL tables. Browse your full event catalog, timeline annotations, and per-user activity. Join with GitHub deploys, Linear incidents, or Plausible web traffic for cross-source product intelligence.

## Tables

| Table | Description | Required filters | Optional filters |
|-------|-------------|-----------------|-----------------|
| `amplitude.events` | All event types tracked in the project with occurrence counts | — | — |
| `amplitude.annotations` | Timeline annotation markers with dates and labels | — | — |

## Source-Scoped Table Functions

| Function | Description |
|----------|-------------|
| `amplitude.user_activity(user_id => '...')` | All recorded events for a specific user — by Amplitude ID, device ID, or app user ID |

## Authentication

Requires `AMPLITUDE_API_KEY` and `AMPLITUDE_SECRET_KEY`.

**To get your keys:**

1. Log in to [amplitude.com](https://amplitude.com)
2. Go to **Settings** → **Projects** → select your project
3. Copy the **API Key** and **Secret Key** shown on the project page

Both keys are required. The connector uses HTTP Basic Auth (API Key as username, Secret Key as password).

## Install

```bash
coral source lint manifest.yaml
coral source add --file manifest.yaml
coral source test amplitude
```

Or with keys inline:

```bash
AMPLITUDE_API_KEY=your-key AMPLITUDE_SECRET_KEY=your-secret coral source add --file manifest.yaml
```

## Example Queries

Top 20 events by total occurrences:

```sql
SELECT name, totals, category
FROM amplitude.events
ORDER BY totals DESC
LIMIT 20;
```

Active (non-deleted, non-hidden) events only:

```sql
SELECT name, totals, category
FROM amplitude.events
WHERE deleted = false
  AND hidden = false
ORDER BY totals DESC;
```

All project annotations:

```sql
SELECT date, label, details
FROM amplitude.annotations
ORDER BY date DESC;
```

Activity for a specific user (by Amplitude user ID):

```sql
SELECT event_type, client_event_time, platform, country, city
FROM amplitude.user_activity(user_id => '12345678')
ORDER BY client_event_time DESC;
```

Activity for a user by your app's user ID (prefix with `user:`):

```sql
SELECT event_type, client_event_time, event_properties
FROM amplitude.user_activity(user_id => 'user:alice@example.com')
ORDER BY client_event_time DESC;
```

## Cross-Source JOIN Example

Amplitude annotations alongside GitHub deploys — correlate release activity with annotation markers (requires `github` source installed):

```sql
WITH annotations AS (
    SELECT date, label
    FROM amplitude.annotations
),
deploys AS (
    SELECT SUBSTR(merged_at, 1, 10) AS deploy_date,
           COUNT(*) AS prs_merged
    FROM github.pulls
    WHERE owner = 'your-org'
      AND repo  = 'your-repo'
    GROUP BY SUBSTR(merged_at, 1, 10)
)
SELECT
    COALESCE(a.date, d.deploy_date) AS date,
    d.prs_merged,
    a.label AS annotation
FROM deploys d
LEFT JOIN annotations a ON a.date = d.deploy_date
ORDER BY date DESC;
```

## The `user_activity` Table Function

The `user_id` argument accepts three formats:

| Format | Example | Meaning |
|--------|---------|---------|
| Numeric string | `'12345678'` | Amplitude user ID |
| `device:` prefix | `'device:abc123'` | Device ID |
| `user:` prefix | `'user:alice@acme.com'` | Your app's user ID |

`event_properties` and `user_properties` columns contain raw JSON objects. Use `json_extract` or similar SQL JSON functions to access nested fields.

## Notes

- All tables are strictly read-only.
- `amplitude.events` returns all event types for the project in a single request — no pagination.
- `amplitude.annotations` returns all annotations in a single request.
- `amplitude.user_activity` returns up to the last 1 000 events for the specified user.
- Rate limits vary by Amplitude plan. The connector handles `429` responses automatically via `Retry-After`.
