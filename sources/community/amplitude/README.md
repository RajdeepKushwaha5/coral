# Amplitude

**Version:** 1.0.0
**Backend:** HTTP
**Base URL:** configured via `AMPLITUDE_BASE_URL` input (default: `https://amplitude.com/api`)

Query Amplitude Analytics as SQL tables. Browse your full event catalog, timeline annotations, and per-user activity. Join with GitHub deploys, Linear incidents, or Plausible web traffic for cross-source product intelligence.

## Tables

| Table | Description | Required filters | Optional filters |
|-------|-------------|-----------------|-----------------|
| `amplitude.events` | All event types tracked in the project with occurrence counts and visibility flags | — | — |
| `amplitude.annotations` | Timeline annotation markers with dates and labels | — | — |

## Source-Scoped Table Functions

| Function | Description |
|----------|-------------|
| `amplitude.user_activity(user_id => '...')` | All recorded events for a specific user — by Amplitude-assigned numeric user ID |

## Authentication

Requires `AMPLITUDE_API_KEY` and `AMPLITUDE_SECRET_KEY`.

**To get your keys:**

1. Log in to [amplitude.com](https://amplitude.com)
2. Go to **Settings** -> **Projects** -> select your project
3. Copy the **API Key** and **Secret Key** shown on the project page

Both keys are required. The connector uses HTTP Basic Auth (API Key as username, Secret Key as password).

## Regions

The default base URL targets Amplitude's US region (`https://amplitude.com/api`). For EU data residency projects set `AMPLITUDE_BASE_URL` when adding the source:

```bash
AMPLITUDE_BASE_URL=https://analytics.eu.amplitude.com/api \
AMPLITUDE_API_KEY=your-key \
AMPLITUDE_SECRET_KEY=your-secret \
coral source add --file sources/community/amplitude/manifest.yaml
```

See the [Amplitude regions docs](https://amplitude.com/docs/apis/analytics/dashboard-rest#regions) for details.

## Install

```bash
AMPLITUDE_API_KEY=your-key \
AMPLITUDE_SECRET_KEY=your-secret \
coral source add --file sources/community/amplitude/manifest.yaml
```

## Validation

```bash
# Add and run test query (output sanitized)
coral source add --file sources/community/amplitude/manifest.yaml
# coral source test amplitude produces the same output
```

```text
  ✓ amplitude connected successfully
  Secrets: keyring

    amplitude (2 tables, 1 function)
    ├─ annotations
    ├─ events
    └─ user_activity (function)

    Query tests
    1 declared · 1 passed · 0 failed

    ✓ SELECT value, totals FROM amplitude.events ORDER BY totals DESC LIMIT 1
      1 row
```

```bash
# Representative query (output sanitized)
coral sql "SELECT value, display, totals FROM amplitude.events WHERE hidden = false ORDER BY totals DESC LIMIT 3"
```

```text
+--------------------------+----------------------------------+--------+
| value                    | display                          | totals |
+--------------------------+----------------------------------+--------+
| signup_completed         | Signup Completed                 | 482100 |
| page_viewed              | Page Viewed                      | 341890 |
| button_clicked           | Button Clicked                   | 218340 |
+--------------------------+----------------------------------+--------+
3 rows
```

## Example Queries

Top 20 events by total occurrences:

```sql
SELECT value, display, totals
FROM amplitude.events
ORDER BY totals DESC
LIMIT 20;
```

Active (non-deleted, non-hidden) events only:

```sql
SELECT value, display, totals
FROM amplitude.events
WHERE deleted = false
  AND hidden = false
ORDER BY totals DESC;
```

All project annotations:

```sql
SELECT start, label, details, category_name
FROM amplitude.annotations
ORDER BY start DESC;
```

Activity for a specific user (Amplitude numeric user ID):

```sql
SELECT event_type, client_event_time, platform, country, city
FROM amplitude.user_activity(user_id => '12345678')
ORDER BY client_event_time DESC;
```

## Cross-Source JOIN Example

Amplitude annotations alongside GitHub deploys — correlate release activity with annotation markers:

```sql
WITH annotations AS (
    SELECT SUBSTR(start, 1, 10) AS annotation_date, label
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
    COALESCE(a.annotation_date, d.deploy_date) AS date,
    d.prs_merged,
    a.label AS annotation
FROM deploys d
LEFT JOIN annotations a ON a.annotation_date = d.deploy_date
ORDER BY date DESC;
```

## The `user_activity` Table Function

`user_activity` accepts the **Amplitude-assigned numeric user ID** as its argument. This is the `amplitude_id` column returned by the function itself.

To look up an Amplitude ID from a device ID or your app's user ID, use the [Amplitude User Search API](https://amplitude.com/docs/apis/analytics/dashboard-rest#user-search) first — it is not exposed as a Coral table in this source.

```sql
SELECT event_type, client_event_time, platform, country,
       event_properties, user_properties
FROM amplitude.user_activity(user_id => '12345678')
ORDER BY client_event_time DESC
LIMIT 50;
```

`event_properties` and `user_properties` contain raw JSON objects. Use `json_extract` or similar SQL JSON functions to access nested fields.

## Notes

- All tables are strictly read-only.
- `amplitude.events` uses `GET /api/2/events/list` (Dashboard REST API v2). Returns all event types in a single request. The `hidden` column marks events hidden in the Amplitude UI; use `WHERE hidden = false` to exclude them.
- `amplitude.annotations` uses `GET /api/3/annotations` (Chart Annotations API v3). Returns all annotations in a single request. `start` and `end` are ISO 8601 timestamps; `end` is null for single-day annotations.
- `amplitude.user_activity` uses `GET /api/2/useractivity`. The `user` parameter accepts the Amplitude numeric user ID only. Returns up to the last 1000 events for the specified user.
- Rate limits vary by Amplitude plan. The connector handles `429` responses automatically via `Retry-After`.
