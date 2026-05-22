# Google Drive (Community)

**Version:** 0.1.0
**Backend:** HTTP (Google Drive API v3)
**Tables:** 3
**Base URL:** `https://www.googleapis.com`

Query Google Drive file listings, file metadata, and shared drives via the
Drive API v3. Authenticates with Google OAuth (authorization_code + PKCE) using
the same credential pattern as the core Gmail source.

## Install

Community sources are not bundled with the Coral binary. Add the manifest from
this directory:

```bash
coral source add --file sources/community/google_drive/manifest.yaml
```

Or copy `manifest.yaml` into your workspace and pass that path to
`coral source add --file`.

## Authentication and setup

Requires a Google OAuth client ID and client secret with the
`https://www.googleapis.com/auth/drive.readonly` scope.

1. In [Google Cloud Console](https://console.cloud.google.com), create an
   OAuth 2.0 client ID (Desktop app type) under **APIs & Services → Credentials**.
2. Enable the **Google Drive API** under **APIs & Services → Library**.
3. Run the interactive setup:

```bash
coral source add --interactive google_drive \
  --file sources/community/google_drive/manifest.yaml
```

Choose **Connect with Google**, then paste your `GOOGLE_OAUTH_CLIENT_ID` and
`GOOGLE_OAUTH_CLIENT_SECRET` when prompted. Coral completes the PKCE OAuth
flow locally and stores the access token.

To paste an access token directly instead:

```bash
export GOOGLE_DRIVE_TOKEN=ya29.a0...
coral source add --file sources/community/google_drive/manifest.yaml
```

## Tables

| Table | Required filters | Description |
| --- | --- | --- |
| `files` | none (optional `q`, `drive_id`) | Paginated file list across My Drive and shared drives |
| `file_info` | `file_id` | Full metadata for a single file or folder |
| `shared_drives` | none | All shared drives accessible to the authenticated user |

The `q` filter on `files` uses [Drive query syntax](https://developers.google.com/drive/api/guides/search-files),
not SQL — for example `mimeType = "application/pdf"` or
`modifiedTime > "2026-01-01T00:00:00"`.

## Example queries

### List recently modified files

```sql
SELECT id, name, mime_type, modified_time, web_view_link
FROM google_drive.files
WHERE q = 'modifiedTime > "2026-05-01T00:00:00"'
ORDER BY modified_time DESC
LIMIT 20;
```

### Full metadata for one file

```sql
SELECT name, mime_type, size, owner_email, last_modifier_email, web_view_link
FROM google_drive.file_info
WHERE file_id = '1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms'
LIMIT 1;
```

### List shared drives

```sql
SELECT drive_id, name, created_time
FROM google_drive.shared_drives
LIMIT 10;
```

### Files scoped to a shared drive

```sql
SELECT id, name, mime_type, modified_time
FROM google_drive.files
WHERE drive_id = '<DRIVE_ID>'
ORDER BY modified_time DESC
LIMIT 50;
```

## Cross-source examples

### File owners joined with Linear assignees

```sql
SELECT f.name, f.owner_email, li.title AS linear_issue
FROM google_drive.files f
JOIN linear.issues li ON LOWER(f.owner_email) = LOWER(li.assignee_email)
WHERE li.state_type != 'completed'
LIMIT 30;
```

### Google Docs modified by the same engineer who merged a recent PR

```sql
SELECT f.name, f.modified_time, f.web_view_link, pr.merged_by
FROM google_drive.files f
JOIN github.pull_requests pr ON LOWER(f.owner_email) = LOWER(pr.author_email)
WHERE pr.state = 'closed'
  AND pr.merged_at >= '2026-05-01T00:00:00Z'
LIMIT 20;
```

## Validation

```bash
# YAML style
make lint-sources

# Manifest structure smoke check
coral source lint sources/community/google_drive/manifest.yaml

# Add and test
coral source add --file sources/community/google_drive/manifest.yaml
coral source test google_drive
```

## Limitations

- **Read-only** — this source exposes read-only Drive API endpoints only.
- **Drive query syntax** — the `q` filter on `files` uses Drive query syntax,
  not SQL; unsupported operators cause API errors.
- **Primary owner only** — only the first owner (index 0) is surfaced as
  `owner_email`; the `raw` column contains the full owners array.
- **No file content** — binary or text content download is out of scope for v1.
- **No permissions table** — per-file ACL listing is a natural follow-on.
- **Quota** — the Drive API enforces per-user and per-project quotas; always
  use `LIMIT` on large drives.
- Community sources are maintained separately from bundled core sources.

## Contributing

Follow [CONTRIBUTING.md](../../../CONTRIBUTING.md): discuss on the linked issue
first, sign the CLA if this is your first contribution, run `make lint-sources`,
and open a focused PR titled
`feat(sources/community/google_drive): add google drive community source`.
