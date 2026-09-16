# API Credentials Setup — Exact Paste Locations

**Every credential below goes into `.env.local` (not `.env` and never into source code).**

Next.js automatically loads `.env.local` at start-up and it takes precedence over `.env`.

---

## Quick start (copy-paste block)

1. Open `.env.local` (create it if it doesn't exist — see `.env.example`).
2. Paste the block below, then replace each placeholder with your real value.

```bash
# ─── SHOPIFY ───────────────────────────────────────────────────
SHOPIFY_STORE_URL="your-store.myshopify.com"
SHOPIFY_ACCESS_TOKEN="shpss_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
SHOPIFY_API_VERSION="2024-01"
SHOPIFY_WEBHOOK_SECRET=""

# ─── META ADS ──────────────────────────────────────────────────
META_ACCESS_TOKEN=""
META_AD_ACCOUNT_ID=""
META_APP_ID=""
META_APP_SECRET=""
META_SYNC_DAYS="30"
```

---

## Shopify — where to find each value

| Variable | Where to find it | Notes |
|---|---|---|
| `SHOPIFY_STORE_URL` | **Shopify Admin → Settings → Domains** (or your admin URL). Use the format `my-store.myshopify.com`. No `https://`, no trailing `/`. | This is your store's unique handle. |
| `SHOPIFY_ACCESS_TOKEN` | **Shopify Admin → Settings → Apps & sales channels → Develop apps → your app → API credentials → Admin API access token**. Click "Reveal token once" and copy the full string. | Token starts with `shpss_` (new apps) or `shpat_` (older apps). Paste it exactly as shown. |
| `SHOPIFY_API_VERSION` | Leave as `2024-01` unless you are sure about a different version. | Valid format: `YYYY-MM`. |
| `SHOPIFY_WEBHOOK_SECRET` | **Develop apps → your app → Webhooks** (optional). Leave blank if you don't plan to use webhooks. | Only needed if you configure webhooks later. |

### Shopify app permissions required
When creating or editing the app in **Develop apps**, enable these scopes under **Admin API access scopes**:

```
read_customers, write_customers
read_orders, write_orders
read_products, write_products
read_inventory, write_inventory
read_locations
read_fulfillments, write_fulfillments
```

Then **Install app** → copy the Admin API access token.

---

## Meta / Facebook & Instagram Ads — where to find each value

| Variable | Where to find it | Notes |
|---|---|---|
| `META_ACCESS_TOKEN` | **Meta for Developers** → your app → **Tools & settings → System users → Generate new token** (or **Access Tokens** panel). Select the `ads_read` and `ads_management` permissions. | Use a **System User** token (never expires). Do NOT use a User token that expires in 60 days. |
| `META_AD_ACCOUNT_ID` | **Ads Manager → your ad account → Account settings → Ad account ID**. The value looks like `1234567890`. Prefix with `act_` if missing (the code auto-adds it). | Paste as `act_1234567890` or just `1234567890`. |
| `META_APP_ID` | **Meta for Developers → your app → Settings → Basic → App ID**. | Found in the Basic settings panel. |
| `META_APP_SECRET` | **Meta for Developers → your app → Settings → Basic → App Secret → Show**. | Click "Show" to reveal it, then copy. |
| `META_SYNC_DAYS` | Not from Meta. Controls how many days of ad spend to pull per sync. Default `30`. | Set to `90` to backfill last quarter, etc. |

---

## Running the first sync

### Via the Admin UI
1. Log in → **Admin → Integrations**.
2. Click the **Sync All (from .env.local)** button (top-right).
3. Wait for the success toast — it will report how many Shopify orders/customers/products and Meta campaigns were synced.

### Via the API directly

```bash
# Full sync (both Shopify + Meta)
curl -X POST http://localhost:3000/api/integrations/sync \
  -H "Content-Type: application/json" \
  -b "auth-token=YOUR_JWT_COOKIE" \
  -d '{}'

# Shopify only
curl -X POST http://localhost:3000/api/integrations/sync \
  -H "Content-Type: application/json" \
  -b "auth-token=YOUR_JWT_COOKIE" \
  -d '{"syncShopify": true, "syncMeta": false}'

# Meta only (last 14 days)
curl -X POST http://localhost:3000/api/integrations/sync \
  -H "Content-Type: application/json" \
  -b "auth-token=YOUR_JWT_COOKIE" \
  -d '{"syncShopify": false, "syncMeta": true, "sinceDays": 14}'
```

To get the auth cookie, log in via the UI and copy the `auth-token` cookie value from your browser.

---

## What gets synced where

| Data source | Synced into | Module |
|---|---|---|
| Shopify **orders** | `Order` + `OrderItem` + `Customer` + `Payment` tables | Orders, Customers, Payments, Dashboard |
| Shopify **customers** | `Customer` table | Customers |
| Shopify **products & inventory** | `Product` + `ProductVariant` + `InventoryMovement` tables | Products, Inventory |
| Meta **Facebook ad spend** | `Expense` (category FACEBOOK_ADS) + `MarketingExpense` | Expenses, Marketing, Profit/Loss |
| Meta **Instagram ad spend** | `Expense` (category INSTAGRAM_ADS) + `MarketingExpense` | Expenses, Marketing, Profit/Loss |

Expense and Profit/Loss figures automatically reflect the real API data once sync completes — no manual entry needed.

---

## Troubleshooting

### Shopify sync returns "Shopify not configured"
- Ensure `SHOPIFY_STORE_URL` and `SHOPIFY_ACCESS_TOKEN` are non-empty in `.env.local`
- Token must NOT be `shpat_xxxxxxxx` (that's the placeholder)

### Meta sync returns "Meta Ads not configured"
- Ensure `META_ACCESS_TOKEN` and `META_AD_ACCOUNT_ID` are non-empty
- Ad account must start with `act_` (or just the number — the code auto-prefixes)

### "database is locked" error
- This can happen if two syncs run at the same time on SQLite. Wait for the current sync to finish, then try again.

### Orders sync but products/variants are missing
- The products sync runs before orders. If products sync fails (e.g. too many products, or network error), order items still get placeholder product rows so FK integrity is preserved. Re-run sync to fill in real product data.

---

## Production deployment checklist

- [ ] All credentials in `.env.local` (never in `.env`, never in git)
- [ ] `.env.local` is in `.gitignore`
- [ ] Shopify app has the correct scopes (see permissions table above)
- [ ] Meta token is a System User token (doesn't expire)
- [ ] `AUTH_SECRET` is a strong random string (32+ chars)
- [ ] Default admin credentials changed after first login
