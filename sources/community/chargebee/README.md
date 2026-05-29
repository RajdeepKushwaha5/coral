# Chargebee

**Version:** 1.0.0
**Backend:** HTTP
**Base URL:** `https://{your-site}.chargebee.com`

Query Chargebee subscription billing data as SQL tables. Inspect subscriptions, customers, invoices, and plans. Join with Freshdesk tickets or Linear issues for customer health and churn intelligence.

## Tables

| Table | Description | Optional filters |
|-------|-------------|-----------------|
| `chargebee.subscriptions` | Subscriptions with status, MRR, billing cycle, and term dates | `status`, `customer_id`, `plan_id` |
| `chargebee.customers` | Customers with contact details and MRR | `email` |
| `chargebee.invoices` | Invoices with amounts, status, and payment timestamps | `status`, `customer_id` |
| `chargebee.plans` | Plan definitions with pricing and billing configuration | `status` |

Filters listed above are pushed to the Chargebee API (e.g. `status[is]=active`) so the `fetch_limit_default` of 100 rows applies to the filtered set. SQL `WHERE` on columns not in the list runs client-side after fetching.

## Authentication

Requires `CHARGEBEE_SITE` and `CHARGEBEE_API_KEY`.

**To get your API key:**

1. Log in to your Chargebee dashboard
2. Go to **Settings** -> **Configure Chargebee** -> **API Keys & Webhooks**
3. Create or copy a read-only API key

**To find your site name:**

Your Chargebee URL is `https://{site}.chargebee.com`. Enter just the subdomain (e.g. `acme`, not the full URL). For a test site use `acme-test`.

**Note on the password field:**

Chargebee uses HTTP Basic Auth with the API key as the username and an [empty password](https://apidocs.chargebee.com/docs/api/auth). The Coral source spec requires `minLength: 1` for BasicAuth passwords, so the manifest sends a literal `x` as the password value. Chargebee authenticates purely on the API key (username) and ignores the password field entirely — sending `base64(<api_key>:x)` produces the same authentication outcome as `base64(<api_key>:)`. The live-test output below was captured against a real Chargebee test site using this manifest, confirming the workaround works.

## Install

```bash
CHARGEBEE_SITE=acme-test \
CHARGEBEE_API_KEY=your-key \
coral source add --file sources/community/chargebee/manifest.yaml
```

## Validation

```bash
# Add the source and run test query (output sanitized — site name and key redacted)
coral source add --file sources/community/chargebee/manifest.yaml
# coral source test chargebee produces the same output
```

```text
  ✓ chargebee connected successfully
  Secrets: keyring

    chargebee (4 tables)
    ├─ customers
    ├─ invoices
    ├─ plans
    └─ subscriptions

    Query tests
    1 declared · 1 passed · 0 failed

    ✓ SELECT id, email FROM chargebee.customers LIMIT 1
      1 row
```

```bash
# Representative query (output sanitized)
coral sql "SELECT id, status, mrr, currency_code FROM chargebee.subscriptions WHERE status = 'active' LIMIT 3"
```

```text
+--------------+--------+-------+---------------+
| id           | status | mrr   | currency_code |
+--------------+--------+-------+---------------+
| sub_Abc12345 | active | 4900  | USD           |
| sub_Def67890 | active | 9900  | USD           |
| sub_Ghi11223 | active | 14900 | USD           |
+--------------+--------+-------+---------------+
3 rows
```

## Example Queries

Active subscriptions by MRR (pushes `status = 'active'` to Chargebee API):

```sql
SELECT id, customer_id, plan_id, status, mrr, currency_code, current_term_end
FROM chargebee.subscriptions
WHERE status = 'active'
ORDER BY mrr DESC
LIMIT 50;
```

Subscriptions for a specific customer:

```sql
SELECT id, plan_id, status, mrr, current_term_end
FROM chargebee.subscriptions
WHERE customer_id = 'cust_Abc12345'
ORDER BY current_term_end ASC;
```

Customers by MRR:

```sql
SELECT id, email, company, mrr, currency_code
FROM chargebee.customers
WHERE mrr > 0
ORDER BY mrr DESC
LIMIT 50;
```

Overdue invoices (pushes `status = 'payment_due'` to Chargebee API):

```sql
SELECT id, customer_id, subscription_id, total, amount_due, due_date, currency_code
FROM chargebee.invoices
WHERE status = 'payment_due'
ORDER BY due_date ASC;
```

All active plans with pricing:

```sql
SELECT id, name, price, currency_code, period, period_unit, pricing_model, trial_period
FROM chargebee.plans
WHERE status = 'active'
ORDER BY price DESC;
```

Revenue at risk — cancelled subscriptions with MRR:

```sql
SELECT id, customer_id, plan_id, mrr, cancelled_at
FROM chargebee.subscriptions
WHERE status = 'cancelled'
  AND mrr > 0
ORDER BY cancelled_at DESC;
```

## Cross-Source JOIN Example

At-risk accounts — customers with active subscriptions and open Freshdesk tickets:

```sql
WITH active_subs AS (
    SELECT customer_id, SUM(mrr) AS total_mrr, currency_code
    FROM chargebee.subscriptions
    WHERE status = 'active'
    GROUP BY customer_id, currency_code
),
open_tickets AS (
    SELECT requester_id, COUNT(*) AS open_ticket_count
    FROM freshdesk.tickets
    WHERE status IN (2, 3)
    GROUP BY requester_id
)
SELECT
    c.id          AS customer_id,
    c.email,
    c.company,
    s.total_mrr,
    s.currency_code,
    t.open_ticket_count
FROM chargebee.customers c
JOIN active_subs s ON s.customer_id = c.id
LEFT JOIN open_tickets t ON t.requester_id = c.id
WHERE t.open_ticket_count > 0
ORDER BY s.total_mrr DESC;
```

## Status and Enum Reference

### Subscription status

| Value | Meaning |
|-------|---------|
| `active` | Currently active and billing |
| `in_trial` | In a free trial period |
| `non_renewing` | Active but will not renew at term end |
| `paused` | Billing paused |
| `cancelled` | Subscription cancelled |
| `future` | Scheduled to start in the future |

### Invoice status

| Value | Meaning |
|-------|---------|
| `paid` | Fully paid |
| `payment_due` | Payment overdue |
| `not_paid` | Could not be collected |
| `voided` | Voided before payment |
| `pending` | Not yet finalized |
| `posted` | Finalized, awaiting payment |

### Plan pricing_model

| Value | Meaning |
|-------|---------|
| `flat_fee` | Fixed price per period |
| `per_unit` | Price per unit of quantity |
| `tiered` | Price depends on quantity tiers |
| `volume` | Bulk pricing based on total volume |
| `stairstep` | Step pricing at quantity thresholds |

## Notes

- All tables are strictly read-only.
- Chargebee paginates list endpoints with cursor-based pagination via `next_offset`. Coral handles pagination automatically up to `fetch_limit_default` (100 rows per table by default).
- All amount fields (`mrr`, `price`, `total`, `amount_due`, etc.) are in the **smallest currency unit** (e.g. cents for USD). Divide by 100 for display values.
- All timestamp fields (`created_at`, `current_term_end`, `paid_at`, etc.) are **Unix epoch seconds**.
- `chargebee.plans` covers Product Catalog v1. If your site uses Product Catalog v2, use `items` and `item_prices` endpoints instead (not covered by this spec).
- Rate limits vary by Chargebee plan. The connector handles `429` responses automatically via `Retry-After`.
