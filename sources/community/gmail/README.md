# Gmail

**Version:** 1.0.0
**Backend:** HTTP
**Base URL:** `https://gmail.googleapis.com`

Query Gmail labels, messages, threads, and message metadata as SQL tables. Search your inbox with Gmail's native search syntax, drill into individual messages, and join with other Coral sources for cross-source intelligence.

## Tables

| Table | Description | Required filters | Optional filters |
|-------|-------------|-----------------|-----------------|
| `gmail.labels` | Gmail system and user labels | none | none |
| `gmail.messages` | Message search and list | none | `q`, `label_ids`, `max_results` |
| `gmail.message_details` | Full metadata for one message | `message_id` | none |
| `gmail.threads` | Gmail conversation threads | none | `q`, `label_ids` |
| `gmail.thread_messages` | All messages inside one thread | `thread_id` | none |

## Source-Scoped Table Functions

| Function | Description |
|----------|-------------|
| `gmail.search_messages(q => '...')` | Pushes Gmail search syntax into the API and returns compact candidate IDs |
| `gmail.get_message(message_id => '...')` | Fetches metadata for one message by ID |
| `gmail.get_thread_messages(thread_id => '...')` | Expands one conversation into message rows |

## Authentication

Requires a `GOOGLE_ACCESS_TOKEN` with the `https://www.googleapis.com/auth/gmail.readonly` scope.

**Quickest way to get a token (OAuth Playground):**

1. Go to [developers.google.com/oauthplayground](https://developers.google.com/oauthplayground)
2. Click the gear icon and check **Use your own OAuth credentials** — paste your Client ID and Secret
3. Select the scope: `https://www.googleapis.com/auth/gmail.readonly`
4. Click **Authorize APIs**, sign in, then **Exchange authorization code for tokens**
5. Copy the **Access token**

If you are connecting Gmail and Google Calendar together, authorize both scopes in the same flow:

```
https://www.googleapis.com/auth/gmail.readonly
https://www.googleapis.com/auth/calendar.readonly
```

One token covers both connectors.

**Using gcloud (recommended for daily use):**

```bash
gcloud auth login
export GOOGLE_ACCESS_TOKEN=$(gcloud auth print-access-token)
```

Access tokens expire after 1 hour. Re-run `gcloud auth print-access-token` to get a fresh one.

## Install

```bash
coral source lint manifest.yaml
coral source add --file manifest.yaml
coral source test gmail
```

Or with the token inline:

```bash
GOOGLE_ACCESS_TOKEN=ya29.your-token coral source add --file manifest.yaml
```

## Example Queries

List all labels:

```sql
SELECT id, name, type, messages_unread
FROM gmail.labels
ORDER BY name
LIMIT 20;
```

Search recent unread messages:

```sql
SELECT id, thread_id
FROM gmail.messages
WHERE q = 'is:unread newer_than:7d'
LIMIT 20;
```

Same search using a table function:

```sql
SELECT id, thread_id
FROM gmail.search_messages(q => 'is:unread newer_than:7d')
LIMIT 20;
```

Fetch metadata for one message:

```sql
SELECT subject, from_email, to_email, date, snippet
FROM gmail.message_details
WHERE message_id = 'MESSAGE_ID_HERE';
```

Expand one thread into rows:

```sql
SELECT subject, from_email, snippet, internal_datetime
FROM gmail.thread_messages
WHERE thread_id = 'THREAD_ID_HERE'
ORDER BY internal_datetime ASC;
```

## Cross-Source JOIN Example

Inbox pressure vs. meeting load in one query (requires `google_calendar` source also installed):

```sql
WITH urgent_emails AS (
    SELECT COUNT(DISTINCT id) AS urgent_unread
    FROM gmail.messages
    WHERE q = 'is:unread (urgent OR deadline OR "due date") newer_than:30d'
),
meetings AS (
    SELECT COUNT(DISTINCT id) AS meeting_count
    FROM google_calendar.events_between(
        calendar_id => 'primary',
        time_min    => '2026-05-01T00:00:00Z',
        time_max    => '2026-05-31T23:59:59Z'
    )
)
SELECT urgent_unread, meeting_count
FROM urgent_emails CROSS JOIN meetings;
```

## Notes

- `gmail.messages` returns only `id` and `thread_id` from the list API. Use `gmail.message_details` or `gmail.get_message` to fetch subject, sender, and snippet for a specific message.
- `allow_404_empty: true` is set on all single-item lookup tables — a missing or deleted message returns zero rows instead of an error.
- Rate limiting is handled automatically via `rate_limit` config in the manifest.
- Gmail timestamps (`internalDate`) are in milliseconds since epoch and are mapped to a `Timestamp` column via `format_timestamp`.
