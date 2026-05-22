# Freshdesk

**Version:** 1.0.0
**Backend:** HTTP
**Base URL:** `https://{your-domain}.freshdesk.com`

Query Freshdesk support tickets, contacts, agents, and groups as SQL tables. Filter tickets by status, priority, or recency. Expand ticket conversation threads. Join with Linear issues, GitHub PRs, or Notion pages for cross-source support intelligence.

## Tables

| Table | Description | Required filters | Optional filters |
|-------|-------------|-----------------|-----------------|
| `freshdesk.tickets` | Support tickets from the last 30 days with status, priority, SLA, and assignment | — | `filter`, `updated_since`, `order_by`, `order_type`, `include` |
| `freshdesk.search_tickets` | Search all tickets by status, priority, agent, or group using Freshdesk filter query syntax | `query` | — |
| `freshdesk.contacts` | Customer contacts with email, phone, and company | — | `updated_since`, `state` |
| `freshdesk.agents` | Support agents with availability and contact details | — | — |
| `freshdesk.groups` | Agent groups for ticket routing | — | — |

## Source-Scoped Table Functions

| Function | Description |
|----------|-------------|
| `freshdesk.conversations(ticket_id => N)` | Agent replies and private notes for a specific ticket (does not include the original ticket description) |

## Authentication

Requires `FRESHDESK_DOMAIN` and `FRESHDESK_API_KEY`.

**To get your API key:**

1. Log in to your Freshdesk account
2. Click your avatar (top right) → **Profile Settings**
3. Your API key is shown on the right side of the page

**To find your domain:**

Your Freshdesk URL is `https://{domain}.freshdesk.com`. Enter just the subdomain part (e.g. `acme`, not the full URL).

## Install

```bash
coral source lint manifest.yaml
coral source add --file manifest.yaml
coral source test freshdesk
```

Or with credentials inline:

```bash
FRESHDESK_DOMAIN=acme FRESHDESK_API_KEY=your-key coral source add --file manifest.yaml
```

## Example Queries

New and open tickets assigned to the current agent:

```sql
SELECT id, subject, status, priority, requester_id, created_at
FROM freshdesk.tickets
WHERE filter = 'new_and_my_open'
ORDER BY created_at DESC
LIMIT 25;
```

Tickets updated in the last 24 hours:

```sql
SELECT id, subject, status, responder_id, updated_at
FROM freshdesk.tickets
WHERE updated_since = '2024-01-01T00:00:00Z'
ORDER BY updated_at DESC;
```

Fetch a ticket with its original description:

```sql
SELECT id, subject, status, description_text
FROM freshdesk.tickets
WHERE include = 'description'
LIMIT 25;
```

Unresolved high-priority tickets using `search_tickets` (searches across all ticket history):

```sql
SELECT id, subject, priority, due_by, fr_escalated
FROM freshdesk.search_tickets
WHERE query = '"(status:2 AND priority:3)"'
ORDER BY due_by ASC;
```

All urgent open tickets assigned to a specific agent:

```sql
SELECT id, subject, status, priority, due_by
FROM freshdesk.search_tickets
WHERE query = '"(status:2 AND priority:4 AND agent_id:12345)"'
ORDER BY due_by ASC;
```

All verified contacts:

```sql
SELECT id, name, email, company_id
FROM freshdesk.contacts
WHERE state = 'verified'
ORDER BY name ASC;
```

Available agents:

```sql
SELECT id, name, email, type, available, occasional
FROM freshdesk.agents
WHERE available = true
ORDER BY name ASC;
```

Expand a ticket's conversation thread (replies and notes only):

```sql
SELECT id, incoming, private, body_text, created_at
FROM freshdesk.conversations(ticket_id => 12345)
ORDER BY created_at ASC;
```

## Cross-Source JOIN Example

Open tickets alongside related Linear issues (requires `linear` source installed):

```sql
WITH tickets AS (
    SELECT id, subject, status, created_at
    FROM freshdesk.tickets
    WHERE filter = 'new_and_my_open'
),
issues AS (
    SELECT id, title, state_name, created_at
    FROM linear.issues
    WHERE team_id = 'your-team-id'
)
SELECT
    t.id    AS ticket_id,
    t.subject,
    i.id    AS linear_issue_id,
    i.title AS linear_title,
    i.state_name
FROM tickets t
LEFT JOIN issues i
    ON i.title ILIKE '%' || t.subject || '%'
ORDER BY t.created_at DESC;
```

## `filter` Values

| Value | Meaning |
|-------|---------|
| `new_and_my_open` | New and open tickets assigned to the current agent |
| `watching` | Tickets the current agent is watching |
| `spam` | Tickets marked as spam |
| `deleted` | Deleted tickets |

When `filter` is omitted, Freshdesk returns all tickets from the last 30 days.

## `search_tickets` Filter Query Syntax

The `query` filter uses [Freshdesk filter query syntax](https://developers.freshdesk.com/api/#filter_tickets). Wrap the value in double quotes and enclose field conditions in parentheses:

```sql
WHERE query = '"(status:2)"'
WHERE query = '"(status:2 AND priority:3)"'
WHERE query = '"(priority:4 AND group_id:12345)"'
WHERE query = '"(agent_id:67890 AND status:3)"'
```

The API returns at most 300 results; narrow your query if you expect more.

## Status and Priority Reference

| Status value | Meaning |
|---|---|
| `2` | Open |
| `3` | Pending |
| `4` | Resolved |
| `5` | Closed |

| Priority value | Meaning |
|---|---|
| `1` | Low |
| `2` | Medium |
| `3` | High |
| `4` | Urgent |

## Agent Type Reference

| Type value | Meaning |
|---|---|
| `support_agent` | Handles tickets directly |
| `field_agent` | Field service agent |
| `collaborator` | Read-only access to tickets |

The `occasional` column is `true` for part-time agents who are billed differently from full-time agents.

## Notes

- All tables are strictly read-only.
- `freshdesk.tickets` returns tickets from the **last 30 days** by default (Freshdesk API behavior). Use `freshdesk.search_tickets` to query across all ticket history with status, priority, agent, or group filters.
- `freshdesk.conversations` returns agent replies and private notes only. The original ticket description is not included; use `WHERE include = 'description'` on `freshdesk.tickets` to populate the `description_text` column.
- Freshdesk paginates all list endpoints at up to 100 records per page; `search_tickets` uses 30 per page (Freshdesk limit). Coral handles pagination automatically.
- `freshdesk.agents` returns contact details (name, email, phone) from the nested `contact` object in each agent record.
- `updated_since` accepts ISO 8601 datetime strings (e.g. `2024-01-01T00:00:00Z`).
- All timestamp columns (`created_at`, `updated_at`, `due_by`, `fr_due_by`) are native `Timestamp` values.
- Rate limit varies by Freshdesk plan. The connector handles `429` responses automatically via `Retry-After`.
