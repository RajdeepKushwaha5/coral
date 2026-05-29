# Squadcast (Community)

**Version:** 0.1.0
**Backend:** HTTP (Squadcast API v3)
**Tables:** 3
**Base URL:** `https://api.squadcast.com` (US) or `https://api.eu.squadcast.com` (EU)

Query Squadcast incidents, services, and users via the API v3. Join with
GitHub PRs, Sentry errors, PagerDuty incidents, and Linear issues for
cross-source SRE and incident management intelligence.

## Install

```bash
coral source add --file sources/community/squadcast/manifest.yaml
```

## Authentication and setup

Requires a Squadcast access token, your account owner ID, and the API base
URL for your region (US or EU).

Exchange your refresh token for an access token:

```bash
curl -X POST https://auth.squadcast.com/oauth/access-token \
  -H "X-Refresh-Token: <your-refresh-token>"
```

Copy the `access_token` from the response, note your owner ID from
**Settings -> General Settings -> Organization ID**, then run:

```bash
SQUADCAST_TOKEN=eyJ0eXAi... \
SQUADCAST_API_BASE_URL=https://api.squadcast.com \
SQUADCAST_OWNER_ID=your-owner-id \
coral source add --file sources/community/squadcast/manifest.yaml
```

For EU accounts set `SQUADCAST_API_BASE_URL=https://api.eu.squadcast.com`.

Tokens expire after 1 hour. Re-run the command above when the token needs
to be refreshed.

## Tables

| Table | Required filters | Optional filters | Description |
| --- | --- | --- | --- |
| `incidents` | `start_time`, `end_time` | `status` | Incidents in a time window via the export endpoint |
| `services` | none | — | All services with escalation policy |
| `users` | none | — | All account users with email and role |

`incidents` uses the `GET /v3/incidents/export` endpoint and requires a time
window. `SQUADCAST_OWNER_ID` (set at source-add time) is passed automatically
as the required `owner_id` parameter on `incidents` and `services` requests.

## Example queries

### Services overview

```sql
SELECT id, name, escalation_policy, status
FROM squadcast.services
ORDER BY name;
```

### Incidents in a time window

```sql
SELECT id, title, severity, service_name, assignee_name, triggered_at
FROM squadcast.incidents
WHERE start_time = '2025-01-01T00:00:00Z'
  AND end_time = '2025-03-31T23:59:59Z'
  AND status = 'triggered'
ORDER BY triggered_at DESC
LIMIT 20;
```

### Incidents for a specific service

```sql
SELECT id, title, status, severity, triggered_at, resolved_at
FROM squadcast.incidents
WHERE start_time = '2025-01-01T00:00:00Z'
  AND end_time = '2025-03-31T23:59:59Z'
  AND service_id = '<SERVICE_ID>'
ORDER BY triggered_at DESC
LIMIT 20;
```

### MTTR per service over a quarter

```sql
SELECT service_name,
       COUNT(*) AS incident_count,
       AVG(
         EXTRACT(EPOCH FROM (
           CAST(resolved_at AS TIMESTAMP) - CAST(triggered_at AS TIMESTAMP)
         )) / 60
       ) AS avg_mttr_minutes
FROM squadcast.incidents
WHERE start_time = '2025-01-01T00:00:00Z'
  AND end_time = '2025-03-31T23:59:59Z'
  AND status = 'resolved'
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
WHERE sq.start_time = '2025-01-01T00:00:00Z'
  AND sq.end_time = '2025-03-31T23:59:59Z'
  AND sq.status = 'triggered'
LIMIT 20;
```

### GitHub PRs merged within one hour before a triggered incident

`github.pulls` requires `owner` and `repo` filters. Run one query per repo
you want to correlate.

```sql
SELECT pr.title AS pr_title, pr.merged_at,
       sq.title AS incident, sq.triggered_at
FROM squadcast.incidents sq
JOIN github.pulls pr
  ON pr.merged_at >= CAST(sq.triggered_at AS TIMESTAMP) - INTERVAL '1 hour'
  AND pr.merged_at <= CAST(sq.triggered_at AS TIMESTAMP)
WHERE sq.start_time = '2025-01-01T00:00:00Z'
  AND sq.end_time = '2025-03-31T23:59:59Z'
  AND sq.status IN ('triggered', 'acknowledged')
  AND pr.owner = 'myorg'
  AND pr.repo = 'myapp'
ORDER BY sq.triggered_at DESC
LIMIT 20;
```

### Engineers with open Linear issues

```sql
SELECT u.username_for_display, u.email, COUNT(li.id) AS open_issues
FROM squadcast.users u
JOIN linear.issues li ON LOWER(li.assignee_email) = LOWER(u.email)
WHERE li.state_type != 'completed'
GROUP BY u.username_for_display, u.email
ORDER BY open_issues DESC
LIMIT 20;
```

## Validation

```bash
# YAML style check
make lint-sources

# Add the source (output sanitized)
coral source add --file sources/community/squadcast/manifest.yaml
```

```text
  ✓ squadcast connected successfully
  Secrets: keyring

    squadcast (3 tables)
    ├─ incidents
    ├─ services
    └─ users

    Query tests
    1 declared · 1 passed · 0 failed

    ✓ SELECT id, name FROM squadcast.services LIMIT 1
      1 row
```

```bash
# Run the built-in test query against the live account
coral source test squadcast
```

```text
  ✓ squadcast connected successfully
  Secrets: keyring

    squadcast (3 tables)
    ├─ incidents
    ├─ services
    └─ users

    Query tests
    1 declared · 1 passed · 0 failed

    ✓ SELECT id, name FROM squadcast.services LIMIT 1
      1 row
```

```bash
# Representative query (output sanitized)
coral sql "SELECT id, name, escalation_policy FROM squadcast.services LIMIT 3"
```

```text
+----------+-------------------+-----------------------------+
| id       | name              | escalation_policy           |
+----------+-------------------+-----------------------------+
| svc-0001 | Payment Service   | Engineering On-Call Policy  |
| svc-0002 | Auth Service      | Backend SRE                 |
| svc-0003 | API Gateway       | Platform Team               |
+----------+-------------------+-----------------------------+
3 rows
```

## Limitations

- **Short-lived tokens** — access tokens expire after 1 hour; re-exchange your
  refresh token and rerun the source-add command when the token needs refreshing.
- **incidents uses the export endpoint** — `GET /v3/incidents/export` requires
  `start_time`, `end_time`, and `owner_id`. There is no paginated list endpoint
  in the published API docs. Scope queries to reasonable time windows to avoid
  large result sets.
- **No webhook or postmortem tables** — only incidents, services, and users are
  exposed in v1.
- Community sources are maintained separately from bundled core sources.

## API reference

- [Incidents export](https://developers.incidents.cloud.solarwinds.com/api-reference/incidents#get-v3-incidents-export)
- [Services list](https://developers.incidents.cloud.solarwinds.com/api-reference/services#get-v3-services)
- [Users list](https://developers.incidents.cloud.solarwinds.com/api-reference/users#get-v3-users)

## Contributing

Follow [CONTRIBUTING.md](../../../CONTRIBUTING.md): discuss on the linked issue
first, sign the CLA if this is your first contribution, run `make lint-sources`,
and open a focused PR titled
`feat(sources/community/squadcast): add squadcast community source`.
