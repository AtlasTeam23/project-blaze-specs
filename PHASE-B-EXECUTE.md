# Phase B — Execute Prompt

**Goal**: Land a read-only Google Ads dashboard for Hardwood Guys GA, with a working `sync_ads_daily` Edge Function, an AI summary engine, and a polished `/marketing` UI showing real numbers from the live Ads account.

**Self-contained** — everything Lovable needs is in this prompt. No external references required.

---

## 1. Data to use

```yaml
business_id:        '7fb42e54-e6f0-4867-a801-ace2f68ef989'  # Hardwood Guys GA
ads_customer_id:    '5657725342'
mcc_customer_id:    '7990784116'
campaign_filter:    'Georgia May Leads 2026 Abhinav'

conversion_actions:
  form_submit:        '7620422282'   # Action ID only; full resource = customers/5657725342/conversionActions/7620422282
  qualified_lead:     '7620422285'
  booked_appointment: '7620422288'   # PRIMARY for bidding

env_vars_to_use_in_edge_functions:
  GOOGLE_ADS_DEVELOPER_TOKEN
  GOOGLE_ADS_CLIENT_ID
  GOOGLE_ADS_CLIENT_SECRET
  GOOGLE_ADS_REFRESH_TOKEN
  GOOGLE_ADS_LOGIN_CUSTOMER_ID   # = 7990784116 (MCC)
  ANTHROPIC_API_KEY              # AI summaries
  OPENAI_API_KEY                 # fallback model
```

Email outbound: **existing Lovable Emails pipeline**, sender `noreply@leadquik.com`. (Standing memory rule.)

---

## 2. Critical API gotchas (don't burn time re-discovering these)

Use **REST API directly via `fetch`** — the `google-ads-api` npm package isn't Deno-compatible. Base URL: `https://googleads.googleapis.com/v23/...`

Headers required on every Ads API call:
```
Authorization: Bearer <access_token>      // refresh via OAuth refresh token
developer-token: <GOOGLE_ADS_DEVELOPER_TOKEN>
login-customer-id: 7990784116             // the MCC
Content-Type: application/json
```

Gotchas you'll hit in Phase B:

- **GAQL date literals**: `LAST_30_DAYS`, `LAST_90_DAYS` are valid; arbitrary `LAST_X_DAYS` is NOT. Custom ranges use `BETWEEN '2026-04-25' AND '2026-05-25'` (single quotes, BETWEEN keyword)
- **`search_term_view` ≠ `keyword_view`**: `keyword_view` = keywords YOU bid on; `search_term_view` = actual user queries that triggered ads (the negative-mining surface)
- **Filter ENABLED status**: `WHERE campaign.status = 'ENABLED' AND campaign.advertising_channel_type = 'SEARCH'` (not `status != 'REMOVED'` — too permissive)
- **Pagination**: `SearchStream` returns up to 10k rows per batch. For larger queries use `Search` (not `SearchStream`) with `pageSize` + `pageToken`
- **`metrics.average_cpc` is in micros** (1e6). All `cost_*_micros` fields too. Divide by 1e6 to get dollars; multiply by 100 to get cents.
- **`metrics.search_impression_share` is a fraction** (0.0–1.0), not a percentage. Multiply by 100 for display.
- **Rate limit**: ~15k ops/day on Basic Access, 100 QPS peak. Pace bulk operations.

---

## 3. GAQL queries (paste-ready)

### Q1 — Daily metrics for the GA campaign over the backfill window

```sql
SELECT
  campaign.id, campaign.name, segments.date,
  metrics.impressions, metrics.clicks, metrics.cost_micros,
  metrics.conversions, metrics.all_conversions,
  metrics.average_cpc, metrics.search_impression_share
FROM campaign
WHERE segments.date BETWEEN '<start>' AND '<end>'
  AND campaign.advertising_channel_type = 'SEARCH'
  AND campaign.status = 'ENABLED'
  AND campaign.name = 'Georgia May Leads 2026 Abhinav'
ORDER BY segments.date DESC
```

### Q2 — Keyword performance (last 30d)

```sql
SELECT
  campaign.name, ad_group.name,
  ad_group_criterion.keyword.text, ad_group_criterion.keyword.match_type,
  metrics.clicks, metrics.cost_micros, metrics.conversions, metrics.all_conversions
FROM keyword_view
WHERE segments.date DURING LAST_30_DAYS
  AND ad_group_criterion.status = 'ENABLED'
  AND campaign.name = 'Georgia May Leads 2026 Abhinav'
ORDER BY metrics.conversions DESC, metrics.cost_micros DESC
LIMIT 100
```

### Q3 — Search terms (what users typed, last 30d)

```sql
SELECT
  campaign.name, search_term_view.search_term,
  segments.date, metrics.clicks, metrics.cost_micros, metrics.conversions
FROM search_term_view
WHERE segments.date DURING LAST_30_DAYS
  AND metrics.clicks > 0
  AND campaign.name = 'Georgia May Leads 2026 Abhinav'
ORDER BY metrics.cost_micros DESC
LIMIT 200
```

### Q4 — Current campaign budget + bid strategy

```sql
SELECT
  campaign.id, campaign.name, campaign.status, campaign.serving_status,
  campaign.bidding_strategy_type,
  campaign.target_spend.cpc_bid_ceiling_micros,
  campaign_budget.amount_micros
FROM campaign
WHERE campaign.name = 'Georgia May Leads 2026 Abhinav'
```

---

## 4. Edge Function spec: `sync_ads_daily`

**Path**: `supabase/functions/sync_ads_daily/index.ts`
**Trigger**: cron `0 3 * * *` ET (daily 3am Eastern); also callable via HTTP POST for manual backfill
**Auth**: service role (bypasses RLS)

**Logic**:
1. SELECT all `marketing_accounts` where `type = 'google_ads' AND status = 'active'`
2. **Dedup by `external_id`** — multiple rows may share a `customer_id` (GA + CT/NY will eventually). Fetch the API once per unique `external_id`, partition results by `campaign_id` server-side.
3. For each unique `external_id`:
   - Refresh OAuth access token using refresh_token
   - Run Q1 with `start = 7 days ago`, `end = today` (daily incremental). If `marketing_daily_metrics` has fewer than 7 rows for any account, treat as backfill and use `start = 90 days ago`.
   - For each row returned, UPSERT into `marketing_daily_metrics` keyed on `(account_id, date)`. Convert micros→cents (cost_micros / 10000 = cents).
   - Run Q2, upsert into `marketing_keyword_performance` keyed on `(account_id, campaign_id, keyword_text, match_type, date_range_start)`.
   - Run Q3, INSERT into `marketing_search_terms` (no upsert — log of what happened).
4. Log success/failure summary to console.

**Manual backfill mode**: HTTP POST with body `{ "account_id": "<uuid>", "days_back": 90 }` triggers a full backfill for one account.

**Error handling**:
- 401 from Ads API → refresh token failed; mark account `status = 'error'`, surface in alerts
- 403 → MCC linkage broken; same treatment
- 429 → backoff 60s, retry once
- Other → log to `marketing_audit_log` with `action = 'sync_ads_daily_error'`

---

## 5. OAuth linking flow

**Path**: `/marketing/settings/accounts`

**UX**:
1. User clicks **[Link Google Ads Account]**
2. Modal: "Enter your Google Ads customer ID (format XXX-XXX-XXXX or 10 digits)"
3. User enters `5657725342` (or `565-772-5342`)
4. Backend sends `CustomerManagerLinkOperation` via Ads API attaching the customer to MCC `7990784116`
5. UI shows: "Pending — go approve from your Ads account: Tools → Setup → Access & Security → Linked accounts → Accept"
6. After approval (poll every 30s for 10 minutes, or until user clicks [Refresh]):
   - Detect via querying `customer_client` from MCC → confirms link is `ACTIVE`
   - INSERT into `marketing_accounts`:
     ```sql
     INSERT INTO marketing_accounts (
       business_id, type, external_id, manager_customer_id,
       campaign_filter, display_name, status, meta
     ) VALUES (
       '7fb42e54-e6f0-4867-a801-ace2f68ef989',
       'google_ads',
       '5657725342',
       '7990784116',
       'Georgia May Leads 2026 Abhinav',
       'Hardwood Guys GA',
       'active',
       '{"linked_via":"oauth_v1"}'::jsonb
     );
     ```
   - Then INSERT 3 rows into `marketing_conversion_actions` for the GA account (form_submit, qualified_lead, booked_appointment with `is_primary = true`).
7. Trigger `sync_ads_daily` with 90-day backfill for this new account.

**For Phase B's GA only**: if OAuth flow is too much work for Phase B, **stub it**: hardcode-insert the `marketing_accounts` row + `marketing_conversion_actions` rows via a Supabase seed migration, and ship OAuth in a follow-up. Document the stub clearly.

---

(Continued in next paste...)
