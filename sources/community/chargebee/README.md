# Chargebee

**Version:** 1.0.0
**Backend:** HTTP
**Base URL:** `https://{your-site}.chargebee.com`

Query Chargebee subscription billing data as SQL tables. Inspect subscriptions, customers, invoices, and plans. Join with Freshdesk tickets or Linear issues for customer health and churn intelligence.

## Tables

| Table | Description | Required filters | Optional filters |
|-------|-------------|-----------------|-----------------|
| `chargebee.subscriptions` | Subscriptions with status, MRR, billing cycle, and term dates | — | — |
| `chargebee.customers` | Customers with contact details and MRR | — | — |
| `chargebee.invoices` | Invoices with amounts, status, and payment timestamps | — | — |
| `chargebee.plans` | Plan definitions with pricing and billing configuration | — | — |

## Authentication

Requires `CHARGEBEE_SITE` and `CHARGEBEE_API_KEY`.

**To get your API key:**

1. Log in to your Chargebee dashboard
2. Go to **Settings** → **Configure Chargebee** → **API Keys & Webhooks**
3. Create or copy a read-only API key

**To find your site name:**

Your Chargebee URL is `https://{site}.chargebee.com`. Enter just the subdomain (e.g. `acme`, not the full URL). For a test site use `acme-test`.

**Note on the password field:**

Chargebee uses HTTP Basic Auth with the API key as the username and an [empty password](https://apidocs.chargebee.com/docs/api/auth). The Coral source spec requires `minLength: 1` for BasicAuth passwords, so the manifest sends a literal `x` as the password value. Chargebee authenticates purely on the API key (username) and ignores the password field entirely — the connector behaves identically to sending `base64(<api_key>:)`. This is the same pattern used by other community sources in this repository that work around the same schema constraint.

## Install

```bash
coral source lint manifest.yaml
coral source add --file manifest.yaml
coral source test chargebee
```

Or with credentials inline:

```bash
CHARGEBEE_SITE=acme CHARGEBEE_API_KEY=your-key coral source add --file manifest.yaml
```

## Example Queries

Active subscriptions by MRR:

```sql
SELECT id, customer_id, plan_id, status, mrr, currency_code, current_term_end
FROM chargebee.subscriptions
WHERE status = 'active'
ORDER BY mrr DESC
LIMIT 50;
```

Customers with their MRR:

```sql
SELECT id, email, company, mrr, currency_code
FROM chargebee.customers
WHERE mrr > 0
ORDER BY mrr DESC;
```

Overdue invoices:

```sql
SELECT id, customer_id, subscription_id, total, amount_due, due_date, currency_code
FROM chargebee.invoices
WHERE status = 'payment_due'
ORDER BY due_date ASC;
```

All active plans with pricing:

```sql
SELECT id, name, price, currency_code, period, period_unit, charge_model, trial_period
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

Customers with open Freshdesk tickets and active subscriptions — at-risk accounts (requires `freshdesk` source installed):

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

### Plan charge_model

| Value | Meaning |
|-------|---------|
| `flat_fee` | Fixed price per period |
| `per_unit` | Price per unit of quantity |
| `tiered` | Price depends on quantity tiers |
| `volume` | Bulk pricing based on total volume |
| `stairstep` | Step pricing at quantity thresholds |

## Notes

- All tables are strictly read-only.
- Chargebee paginates all list endpoints with cursor-based pagination. Coral handles pagination automatically.
- All amount fields (`mrr`, `price`, `total`, `amount_due`, etc.) are in the **smallest currency unit** (e.g. cents for USD, pence for GBP). Divide by 100 for display values.
- All timestamp fields (`created_at`, `current_term_end`, `paid_at`, etc.) are **Unix epoch seconds**.
- `chargebee.plans` covers Product Catalog v1. If your site uses Product Catalog v2, use `items` and `item_prices` endpoints instead (not covered by this spec).
- Rate limits vary by Chargebee plan. The connector handles `429` responses automatically via `Retry-After`.
