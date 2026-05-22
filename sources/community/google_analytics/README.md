# Google Analytics GA4 (Community)

**Version:** 0.1.0
**Backend:** HTTP (Analytics Admin API v1beta)
**Tables:** 3
**Base URL:** `https://analyticsadmin.googleapis.com`

Query Google Analytics GA4 account structure, properties, and data streams via
the Analytics Admin API v1beta. Authenticates with Google OAuth
(authorization_code + PKCE) using the same credential pattern as the core Gmail
source.

## Install

Community sources are not bundled with the Coral binary. Add the manifest from
this directory:

```bash
coral source add --file sources/community/google_analytics/manifest.yaml
```

Or copy `manifest.yaml` into your workspace and pass that path to
`coral source add --file`.

## Authentication and setup

Requires a Google OAuth client ID and client secret with the
`https://www.googleapis.com/auth/analytics.readonly` scope.

1. In [Google Cloud Console](https://console.cloud.google.com), create an
   OAuth 2.0 client ID (Desktop app type) under **APIs & Services → Credentials**.
2. Enable the **Google Analytics Admin API** under **APIs & Services → Library**.
3. Run the interactive setup:

```bash
coral source add --interactive google_analytics \
  --file sources/community/google_analytics/manifest.yaml
```

Choose **Connect with Google**, then paste your `GOOGLE_OAUTH_CLIENT_ID` and
`GOOGLE_OAUTH_CLIENT_SECRET` when prompted. Coral completes the PKCE OAuth
flow locally and stores the access token.

To paste an access token directly instead:

```bash
export GOOGLE_ANALYTICS_TOKEN=ya29.a0...
coral source add --file sources/community/google_analytics/manifest.yaml
```

## Tables

| Table | Required filters | Description |
| --- | --- | --- |
| `accounts` | none | All GA4 accounts accessible to the authenticated user |
| `properties` | `account_id` | GA4 properties under an account |
| `data_streams` | `property_id` | Data streams attached to a property |

IDs use the resource name form:

- `account_id` → `accounts/123456789`
- `property_id` → `properties/987654321`

## Example queries

### List all GA4 accounts

```sql
SELECT account_id, display_name, region_code, create_time
FROM google_analytics.accounts
LIMIT 10;
```

### Properties under an account

```sql
SELECT property_id, display_name, time_zone, currency_code, industry_category, service_level
FROM google_analytics.properties
WHERE account_id = 'accounts/123456789'
ORDER BY display_name
LIMIT 20;
```

### Data streams for a property

```sql
SELECT stream_id, stream_type, display_name, web_uri, measurement_id
FROM google_analytics.data_streams
WHERE property_id = 'properties/987654321'
LIMIT 10;
```

### All web streams across all properties under one account

```sql
SELECT p.display_name AS property_name, ds.display_name AS stream_name, ds.web_uri, ds.measurement_id
FROM google_analytics.properties p
JOIN google_analytics.data_streams ds ON ds.property_id = p.property_id
WHERE p.account_id = 'accounts/123456789'
  AND ds.stream_type = 'WEB_DATA_STREAM'
ORDER BY p.display_name;
```

## Cross-source examples

### GA4 web streams matched against domains with recent GitHub deploys

```sql
SELECT ds.web_uri, ds.measurement_id, COUNT(d.id) AS recent_deploys
FROM google_analytics.data_streams ds
JOIN github.deployments d ON LOWER(d.environment_url) LIKE '%' || LOWER(ds.web_uri) || '%'
WHERE ds.stream_type = 'WEB_DATA_STREAM'
  AND d.created_at >= '2026-05-01T00:00:00Z'
GROUP BY 1, 2
ORDER BY recent_deploys DESC
LIMIT 20;
```

### GA4 properties with open Sentry projects (join on slug/name similarity)

```sql
SELECT p.display_name AS ga4_property, sp.slug AS sentry_project
FROM google_analytics.properties p
JOIN sentry.projects sp
  ON LOWER(sp.slug) = LOWER(REPLACE(p.display_name, ' ', '-'))
WHERE p.account_id = 'accounts/123456789'
LIMIT 20;
```

## Validation

```bash
# YAML style
make lint-sources

# Manifest structure smoke check
coral source lint sources/community/google_analytics/manifest.yaml

# Add and test
coral source add --file sources/community/google_analytics/manifest.yaml
coral source test google_analytics
```

## Limitations

- **Admin API only (v1)** — account, property, and stream metadata only. GA4
  reporting data (sessions, events, conversions) requires the GA4 Data API
  (`analyticsdata.googleapis.com`) with a POST-based report body; that is a
  natural follow-on for v2.
- **No UA properties** — Universal Analytics (UA-*) properties are sunset and
  not supported; this source targets GA4 only.
- **Soft-deleted items hidden** — `showDeleted: false` is the default; deleted
  accounts and properties are excluded from results.
- **Resource name IDs** — all IDs use the API resource name format
  (`accounts/123456789`, `properties/987654321`), not bare numeric IDs.
- Community sources are maintained separately from bundled core sources.

## Contributing

Follow [CONTRIBUTING.md](../../../CONTRIBUTING.md): discuss on the linked issue
first, sign the CLA if this is your first contribution, run `make lint-sources`,
and open a focused PR titled
`feat(sources/community/google_analytics): add google analytics GA4 community source`.
