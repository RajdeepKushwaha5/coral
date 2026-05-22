# Mailchimp (Community)

**Version:** 0.1.0
**Backend:** HTTP (Mailchimp Marketing API v3)
**Tables:** 4
**Base URL:** `https://{server-prefix}.api.mailchimp.com`

Query Mailchimp audiences, campaigns, subscribers, and campaign performance
reports as SQL tables. Join with Salesforce contacts or Linear issues for
cross-source marketing and engineering intelligence.

## Install

```bash
coral source add --file sources/community/mailchimp/manifest.yaml
```

Or copy `manifest.yaml` into your workspace and pass that path to
`coral source add --file`.

## Authentication and setup

Requires a Mailchimp API key and your account's server prefix.

1. In Mailchimp, go to **Account > Extras > API Keys > Create A Key**.
2. Note the server prefix from the end of your API key (e.g. if the key ends
   in `-us6`, your prefix is `us6`).
3. Run:

```bash
MAILCHIMP_SERVER_PREFIX=us6 \
MAILCHIMP_API_KEY=your-api-key \
coral source add --file sources/community/mailchimp/manifest.yaml
```

Or pass them interactively when prompted.

## Tables

| Table | Required filters | Description |
| --- | --- | --- |
| `lists` | none | All audiences with subscriber counts and engagement averages |
| `campaigns` | none (optional `status`, `list_id`) | Campaigns with status, subject, send time, and summary stats |
| `members` | `list_id` | Subscribers in an audience with email, status, and signup times |
| `reports` | none | Per-campaign performance: opens, clicks, bounces, unsubscribes |

## Example queries

### List all audiences

```sql
SELECT id, name, subscriber_count, open_rate, click_rate, date_created
FROM mailchimp.lists
ORDER BY subscriber_count DESC
LIMIT 20;
```

### Sent campaigns with performance summary

```sql
SELECT id, subject_line, emails_sent, open_rate, click_rate, send_time
FROM mailchimp.campaigns
WHERE status = 'sent'
ORDER BY send_time DESC
LIMIT 25;
```

### Active subscribers in an audience

```sql
SELECT email_address, full_name, source, timestamp_signup
FROM mailchimp.members
WHERE list_id = 'YOUR_LIST_ID'
  AND status = 'subscribed'
ORDER BY timestamp_signup DESC
LIMIT 50;
```

### Top campaigns by open rate

```sql
SELECT campaign_title, emails_sent, open_rate, click_rate,
       hard_bounces, unsubscribed
FROM mailchimp.reports
ORDER BY open_rate DESC
LIMIT 20;
```

### Campaigns joined with full report stats

```sql
SELECT c.subject_line, c.send_time, r.open_rate, r.click_rate,
       r.hard_bounces, r.unsubscribed
FROM mailchimp.campaigns c
JOIN mailchimp.reports r ON c.id = r.id
WHERE c.status = 'sent'
ORDER BY r.open_rate DESC
LIMIT 20;
```

## Cross-source examples

### Subscribers who also have open Linear issues

```sql
SELECT m.email_address, m.full_name, li.title AS issue_title, li.state_type
FROM mailchimp.members m
JOIN linear.issues li ON LOWER(li.assignee_email) = LOWER(m.email_address)
WHERE m.list_id = 'YOUR_LIST_ID'
  AND m.status = 'subscribed'
  AND li.state_type != 'completed'
LIMIT 30;
```

### Salesforce contacts who are active subscribers

```sql
SELECT c.first_name, c.last_name, c.email, m.status AS mailchimp_status
FROM salesforce.contacts c
JOIN mailchimp.members m ON LOWER(m.email_address) = LOWER(c.email)
WHERE m.list_id = 'YOUR_LIST_ID'
  AND m.status = 'subscribed'
LIMIT 50;
```

## Validation

```bash
# YAML style
make lint-sources

# Manifest structure smoke check
coral source lint sources/community/mailchimp/manifest.yaml

# Add and test
coral source add --file sources/community/mailchimp/manifest.yaml
coral source test mailchimp
```

## Limitations

- **Read-only** — this source exposes read-only Marketing API endpoints only.
- **members requires list_id** — get a list ID from `mailchimp.lists` first.
- **report_summary columns are null for unsent campaigns** — `opens`, `clicks`,
  `open_rate`, and `click_rate` on the `campaigns` table are only populated
  after a campaign has been sent. Use `mailchimp.reports` for full stats.
- **timestamp_signup and timestamp_opt** — returned as strings; Mailchimp
  returns an empty string (not null) when these are not set, so they are typed
  as `Utf8` rather than `Timestamp`.
- **API key expiry** — Mailchimp API keys do not expire by default but can be
  revoked. Re-add the source if you regenerate your key.
- **Rate limits** — Mailchimp allows 10 simultaneous connections per account.
  Always use `LIMIT` on large audiences.
- Community sources are maintained separately from bundled core sources.

## Contributing

Follow [CONTRIBUTING.md](../../../CONTRIBUTING.md): discuss on the linked issue
first, sign the CLA if this is your first contribution, run `make lint-sources`,
and open a focused PR titled
`feat(sources/community/mailchimp): add mailchimp community source`.
