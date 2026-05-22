# Dynatrace (Community)

**Version:** 0.1.0
**Backend:** HTTP (Dynatrace Environment API v2)
**Tables:** 3
**Base URL:** configured via `DYNATRACE_BASE_URL` input

Query Dynatrace problems, monitored entities, and software releases via the
Environment API v2. Join with GitHub deploys, Sentry errors, and PagerDuty
incidents for cross-source SRE and observability intelligence.

## Install

```bash
coral source add --file sources/community/dynatrace/manifest.yaml
```

## Authentication and setup

Requires a Dynatrace API token with read scopes for the resources you plan to
query.

1. In Dynatrace, go to **Access Tokens → Generate new token**.
2. Enable the following scopes:
   - `problems.read`
   - `entities.read`
   - `releases.read`
3. Copy the generated token.

```bash
export DYNATRACE_BASE_URL=https://abc12345.live.dynatrace.com
export DYNATRACE_API_TOKEN=dt0s01.ST2EY72KQINMH5...
coral source add --file sources/community/dynatrace/manifest.yaml
```

For managed (on-premise) environments use the form
`https://your-host/e/your-environment-id` as the base URL.

## Tables

| Table | Description |
| --- | --- |
| `problems` | Detected problems from the last two weeks (status, severity, impact) |
| `entities` | Monitored entities: services, hosts, applications, processes |
| `releases` | Software releases tracked by Dynatrace with problem and vulnerability counts |

## Example queries

### Open problems by severity

```sql
SELECT problem_id, display_id, title, severity_level, impact_level, start_time_ms
FROM dynatrace.problems
WHERE status = 'OPEN'
ORDER BY start_time_ms DESC
LIMIT 20;
```

### All monitored services

```sql
SELECT entity_id, display_name, first_seen_ms, last_seen_ms
FROM dynatrace.entities
WHERE entity_selector = 'type(SERVICE)'
LIMIT 50;
```

### Releases with active problems

```sql
SELECT name, version, product, stage, problem_count, vulnerability_count
FROM dynatrace.releases
WHERE affected_by_problems = true
ORDER BY problem_count DESC
LIMIT 20;
```

## Cross-source examples

### Dynatrace problems overlapping with Sentry errors

```sql
SELECT dt.title AS dynatrace_problem,
       dt.severity_level,
       se.title AS sentry_error,
       se.first_seen
FROM dynatrace.problems dt
JOIN sentry.issues se
  ON se.first_seen >= TO_TIMESTAMP(dt.start_time_ms / 1000)
  AND se.first_seen <= TO_TIMESTAMP(COALESCE(dt.end_time_ms, dt.start_time_ms + 3600000) / 1000)
WHERE dt.status = 'OPEN'
LIMIT 20;
```

### GitHub deployments that opened Dynatrace problems within one hour

```sql
SELECT d.environment, d.created_at AS deploy_time,
       dt.title AS problem_title, dt.severity_level
FROM github.deployments d
JOIN dynatrace.problems dt
  ON dt.start_time_ms >= CAST(EXTRACT(EPOCH FROM CAST(d.created_at AS TIMESTAMP)) * 1000 AS BIGINT)
  AND dt.start_time_ms <= CAST(EXTRACT(EPOCH FROM CAST(d.created_at AS TIMESTAMP)) * 1000 AS BIGINT) + 3600000
ORDER BY d.created_at DESC
LIMIT 20;
```

### Dynatrace releases with problems matched against GitHub release tags

```sql
SELECT dr.version, dr.product, dr.stage, dr.problem_count,
       gr.name AS github_release
FROM dynatrace.releases dr
LEFT JOIN github.releases gr ON gr.tag_name = dr.version
WHERE dr.affected_by_problems = true
ORDER BY dr.problem_count DESC
LIMIT 20;
```

## Validation

```bash
make lint-sources
coral source lint sources/community/dynatrace/manifest.yaml
coral source add --file sources/community/dynatrace/manifest.yaml
coral source test dynatrace
```

## Limitations

- **Two-week default window** — problems and releases default to `now-2w`;
  extend with the `from` query parameter if needed.
- **Millisecond timestamps** — `start_time_ms`, `end_time_ms`, `first_seen_ms`,
  and `last_seen_ms` are Unix epoch milliseconds; use `TO_TIMESTAMP(x / 1000)`
  to convert for SQL timestamp comparisons.
- **Entity selector syntax** — the `entity_selector` filter on `entities` uses
  Dynatrace entity selector syntax, not SQL; invalid selectors return API errors.
- **Cursor reuse** — when paginating, Dynatrace encodes filters into the cursor;
  Coral passes `nextPageKey` only on subsequent pages as required.
- Community sources are maintained separately from bundled core sources.

## Contributing

Follow [CONTRIBUTING.md](../../../CONTRIBUTING.md): discuss on the linked issue
first, sign the CLA if this is your first contribution, run `make lint-sources`,
and open a focused PR titled
`feat(sources/community/dynatrace): add dynatrace community source`.
