# Google Sheets (Community)

**Version:** 0.1.0
**Backend:** HTTP (Google Sheets API v4)
**Tables:** 3 · **Functions:** 1
**Base URL:** `https://sheets.googleapis.com`

Query Google Sheets spreadsheet metadata, sheet structure, named ranges, and
cell values via the Sheets API v4. Authenticates with Google OAuth
(authorization_code + PKCE) using the same credential pattern as the core
Gmail source.

## Install

Community sources are not bundled with the Coral binary. Add the manifest from
this directory:

```bash
coral source add --file sources/community/google_sheets/manifest.yaml
```

Or copy `manifest.yaml` into your workspace and pass that path to
`coral source add --file`.

## Authentication and setup

Requires a Google OAuth client ID and client secret with the
`https://www.googleapis.com/auth/spreadsheets.readonly` scope.

1. In [Google Cloud Console](https://console.cloud.google.com), create an
   OAuth 2.0 client ID (Desktop app type) under **APIs & Services → Credentials**.
2. Enable the **Google Sheets API** under **APIs & Services → Library**.
3. Run the interactive setup:

```bash
coral source add --interactive --file sources/community/google_sheets/manifest.yaml
```

Choose **Connect with Google**, then paste your `GOOGLE_OAUTH_CLIENT_ID` and
`GOOGLE_OAUTH_CLIENT_SECRET` when prompted. Coral completes the PKCE OAuth
flow locally and stores the token.

To paste an access token directly instead:

```bash
export GOOGLE_SHEETS_TOKEN=ya29.a0...
coral source add --file sources/community/google_sheets/manifest.yaml
```

## Tables and functions

### Metadata tables

| Table | Required filters | Description |
| --- | --- | --- |
| `spreadsheet_info` | `spreadsheet_id` | Title, locale, time zone, and recalc interval |
| `sheets` | `spreadsheet_id` | All sheet tabs with ID, index, type, and grid size |
| `named_ranges` | `spreadsheet_id` | Named ranges with sheet location and row/column bounds |

### Table functions

| Function | Required args | Description |
| --- | --- | --- |
| `get_values(spreadsheet_id, range)` | both | Cell values for an A1-notation range |

The `spreadsheet_id` is the value between `/d/` and `/edit` in the spreadsheet
URL: `https://docs.google.com/spreadsheets/d/<SPREADSHEET_ID>/edit`.

## Example queries

### Spreadsheet metadata

```sql
SELECT spreadsheet_id, title, locale, time_zone
FROM google_sheets.spreadsheet_info
WHERE spreadsheet_id = '1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms'
LIMIT 1;
```

### List all sheet tabs

```sql
SELECT sheet_id, title, index, sheet_type, row_count, column_count
FROM google_sheets.sheets
WHERE spreadsheet_id = '1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms'
ORDER BY index;
```

### Cell values for a range

```sql
SELECT row_data
FROM google_sheets.get_values('1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms', 'Sheet1!A1:D10')
LIMIT 10;
```

Extract individual cells by column index:

```sql
SELECT
  json_get_str(row_data, 0) AS col_a,
  json_get_str(row_data, 1) AS col_b,
  json_get_str(row_data, 2) AS col_c
FROM google_sheets.get_values('1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms', 'Sheet1!A2:C20')
LIMIT 20;
```

### Named ranges

```sql
SELECT name, sheet_id, start_row_index, end_row_index, start_column_index, end_column_index
FROM google_sheets.named_ranges
WHERE spreadsheet_id = '1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms';
```

### Join sheet structure with named range bounds

```sql
SELECT nr.name, s.title AS sheet_title, nr.start_row_index, nr.end_row_index
FROM google_sheets.named_ranges nr
JOIN google_sheets.sheets s
  ON nr.sheet_id = s.sheet_id AND nr.spreadsheet_id = s.spreadsheet_id
WHERE nr.spreadsheet_id = '1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms';
```

## Cross-source examples

### Correlate spreadsheet roster with Linear issue assignments

```sql
SELECT
  json_get_str(rv.row_data, 0) AS engineer_name,
  json_get_str(rv.row_data, 1) AS team,
  COUNT(li.id) AS open_issues
FROM google_sheets.get_values('SPREADSHEET_ID', 'Roster!A2:B50') rv
JOIN linear.issues li ON LOWER(json_get_str(rv.row_data, 0)) = LOWER(li.assignee_name)
WHERE li.state_type != 'completed'
GROUP BY 1, 2
ORDER BY open_issues DESC
LIMIT 20;
```

### Match a spreadsheet task list against open GitHub pull requests

`github.pulls` requires constant `owner` and `repo` filters, so scope it to a
single repo and join on the PR title against a spreadsheet column:

```sql
SELECT
  json_get_str(rv.row_data, 0) AS task_name,
  pr.title AS matching_pr,
  pr.state
FROM google_sheets.get_values('SPREADSHEET_ID', 'Tasks!A2:A30') rv
JOIN github.pulls pr
  ON LOWER(pr.title) LIKE '%' || LOWER(json_get_str(rv.row_data, 0)) || '%'
WHERE pr.owner = 'your-org'
  AND pr.repo = 'your-repo'
  AND pr.state = 'open'
ORDER BY task_name
LIMIT 20;
```

## Validation

```bash
# YAML style check
make lint-sources

# Add interactively (output sanitized — real IDs and token redacted)
coral source add --interactive --file sources/community/google_sheets/manifest.yaml
# coral source test google_sheets produces the same output
```

```text
  ✓ google_sheets connected successfully
  Secrets: keyring

    google_sheets (3 tables, 1 function)
    ├─ named_ranges
    ├─ sheets
    ├─ spreadsheet_info
    └─ get_values (function)

    Query tests
    2 declared · 2 passed · 0 failed
```

```bash
# List the tabs in a spreadsheet (output sanitized)
coral sql "SELECT sheet_id, title, index, row_count, column_count FROM google_sheets.sheets WHERE spreadsheet_id = '1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms' ORDER BY index"
```

```text
+----------+-----------+-------+-----------+--------------+
| sheet_id | title     | index | row_count | column_count |
+----------+-----------+-------+-----------+--------------+
| 0        | Roster    | 0     | 1000      | 26           |
| 1845...  | Projects  | 1     | 500       | 12           |
| 9920...  | Budget    | 2     | 200       | 8            |
+----------+-----------+-------+-----------+--------------+
3 rows
```

```bash
# Fetch cell values from a range (output sanitized)
coral sql "SELECT json_get_str(row_data, 0) AS name, json_get_str(row_data, 1) AS team FROM google_sheets.get_values('1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms', 'Roster!A2:B4')"
```

```text
+----------------+-------------+
| name           | team        |
+----------------+-------------+
| Ada Lovelace   | Platform    |
| Alan Turing    | Security    |
| Grace Hopper   | Compiler    |
+----------------+-------------+
3 rows
```

## Limitations

- **Read-only** — this source exposes read-only Sheets API endpoints only.
- **Cell values as JSON arrays** — `get_values` returns each row as a JSON
  array of strings; individual cells require `json_get_str(row_data, N)`.
- **No formula expressions** — `valueRenderOption: FORMATTED_VALUE` returns
  displayed strings, not underlying formulas.
- **No batch ranges** — `get_values` fetches one range per call.
- **Quota** — the Sheets API enforces per-project and per-user request quotas.
- Community sources are maintained separately from bundled core sources.

## Contributing

Follow [CONTRIBUTING.md](../../../CONTRIBUTING.md): discuss on the linked issue
first, sign the CLA if this is your first contribution, run `make lint-sources`,
and open a focused PR titled
`feat(sources/community/google_sheets): add google sheets community source`.
