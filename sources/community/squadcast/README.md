# Squadcast (Community)

**Version:** 0.1.0
**Backend:** HTTP (Squadcast API v3)
**Tables:** 3
**Base URL:** `https://api.squadcast.com`

Query Squadcast incidents, services, and users via the API v3. Join with
GitHub PRs, Sentry errors, PagerDuty incidents, and Linear issues for
cross-source SRE and incident management intelligence.

## Install

```bash
coral source add --file sources/community/squadcast/manifest.yaml
```

## Authentication and setup

Requires a Squadcast access token. Tokens expire after 1 hour; exchange your
refresh token for an access token before each session.

1. Log in to Squadcast, go to **Profile → Refresh Token**, and copy your
   refresh token.
2. Exchange it for an access token:

```bash
curl -X POST https://auth.squadcast.com/oauth/access-token \
  -H "X-Refresh-Token: <your-refresh-token>"
```

3. Copy the `access_token` value from the response.

```bash
export SQUADCAST_TOKEN=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...
coral source add --file sources/community/squadcast/manifest.yaml
```

For EU accounts, update the manifest `base_url` to
`https://api.eu.squadcast.com`.

## Tables

| Table | Required filters | Description |
| --- | --- | --- |
| `incidents` | none (optional `status`, `service_id`) | Incidents with severity, assignee, and timestamps |
| `services` | none | All services with escalation policy |
| `users` | none | All account users with email and role |

## Example queries

### Open incidents

```sql
SELECT id, title, severity, service_name, assignee_name, triggered_at
FROM squadcast.incidents
WHERE status = 'triggered'
ORDER BY triggered_at DESC
LIMIT 20;
```

### All services

```sql
SELECT id, name, escalation_policy, status
FROM squadcast.services
ORDER BY name;
```

### Incidents for a specific service

```sql
SELECT id, title, status, severity, triggered_at, resolved_at
FROM squadcast.incidents
WHERE service_id = '<SERVICE_ID>'
ORDER BY triggered_at DESC
LIMIT 20;
```

### MTTR per service

```sql
SELECT service_name,
       COUNT(*) AS incident_count,
       AVG(
         EXTRACT(EPOCH FROM (
           CAST(resolved_at AS TIMESTAMP) - CAST(triggered_at AS TIMESTAMP)
         )) / 60
       ) AS avg_mttr_minutes
FROM squadcast.incidents
WHERE status = 'resolved'
  AND resolved_at IS NOT NULL
GROUP BY service_name
ORDER BY avg_mttr_minutes DESC
LIMIT 20;
```

## Cross-source examples

### Squadcast incidents with Sentry errors on the same day

```sql
SELECT sq.title AS incident, sq.severity, sq.triggered_at,
       se.title AS sentry_error, se.first_seen
FROM squadcast.incidents sq
JOIN sentry.issues se
  ON SUBSTR(se.first_seen, 1, 10) = SUBSTR(sq.triggered_at, 1, 10)
WHERE sq.status = 'triggered'
LIMIT 20;
```

### GitHub PRs merged within one hour before a triggered incident

```sql
SELECT pr.title AS pr_title, pr.merged_at, pr.author_login,
       sq.title AS incident, sq.triggered_at
FROM squadcast.incidents sq
JOIN github.pull_requests pr
  ON pr.merged_at >= CAST(sq.triggered_at AS TIMESTAMP) - INTERVAL '1 hour'
  AND pr.merged_at <= CAST(sq.triggered_at AS TIMESTAMP)
WHERE sq.status IN ('triggered', 'acknowledged')
ORDER BY sq.triggered_at DESC
LIMIT 20;
```

### Engineers on-call with open Linear issues

```sql
SELECT u.name, u.email, COUNT(li.id) AS open_issues
FROM squadcast.users u
JOIN linear.issues li ON LOWER(li.assignee_email) = LOWER(u.email)
WHERE li.state_type != 'completed'
GROUP BY u.name, u.email
ORDER BY open_issues DESC
LIMIT 20;
```

## Validation

```bash
make lint-sources
coral source lint sources/community/squadcast/manifest.yaml
coral source add --file sources/community/squadcast/manifest.yaml
coral source test squadcast
```

## Limitations

- **Short-lived tokens** — access tokens expire after 1 hour; re-exchange your
  refresh token and update `SQUADCAST_TOKEN` before each session.
- **US region only in v1** — hard-coded to `api.squadcast.com`; EU customers
  must update `base_url` to `https://api.eu.squadcast.com`.
- **No webhook or postmortem tables** — only incidents, services, and users are
  exposed in v1.
- Community sources are maintained separately from bundled core sources.

## Contributing

Follow [CONTRIBUTING.md](../../../CONTRIBUTING.md): discuss on the linked issue
first, sign the CLA if this is your first contribution, run `make lint-sources`,
and open a focused PR titled
`feat(sources/community/squadcast): add squadcast community source`.
