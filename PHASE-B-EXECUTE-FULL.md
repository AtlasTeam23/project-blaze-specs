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
  OPENAI_API_KEY                 # fallback model (optional — skip if leaning lean)
  RESEND_API_KEY                 # outbound email (key labeled "Project Blaze on Leadquik")
```

Email outbound: **Resend** via the `RESEND_API_KEY` secret (key labeled "Project Blaze on Leadquik" in Supabase Secrets). Sender stays `noreply@leadquik.com` — must be a verified domain in Resend. Chosen for VPS portability; Lovable Emails would lock us to the Lovable platform.

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

<!-- Phase B execute prompt — continued, paste 2 of 3 -->

## 6. Edge Function spec: `generate_ai_summary`

**Path**: `supabase/functions/generate_ai_summary/index.ts`
**Trigger**:
- cron `30 7 * * 1` ET (Monday 7:30am) — weekly summary for all businesses
- cron `0 8 * * *` ET — daily for Internal Elite businesses only (where `service_tier = 'internal_elite'`)
- HTTP POST with body `{ "business_id": "<uuid>", "force": true }` — manual push from UI

**Logic**:
1. Determine date range: weekly trigger = last 7 days; daily = last 24h; manual = `force_range` param or default 7 days
2. Build the structured `SummaryInput` payload (schema below) by querying:
   - `marketing_daily_metrics` for ads + GMB + SEO totals
   - `marketing_keyword_performance` for top converting + wasted spend
   - `marketing_search_terms` for newly flagged terms
   - `lead_conversion_uploads` joined to `leads` for LeadQuik conversions
   - `marketing_business_settings` for business name + industry + tier
3. Call **Anthropic Claude** with the prompt template (below) and the JSON payload
4. On failure (timeout, rate limit, 5xx), retry with **OpenAI GPT-4o**
5. On total failure, send a templated non-AI summary so the operator still gets a Monday email
6. INSERT into `marketing_ai_summaries` (business_id, generated_at, date_range_start, date_range_end, trigger, summary_markdown, highlight_items, model)
7. Send email via Resend (RESEND_API_KEY) from `noreply@leadquik.com`. Use the `resend` npm SDK via esm.sh in the Edge Function. Subject: `🪵 Weekly Marketing Summary — {{business.name}} — {{date_range.start}}`

**SummaryInput TypeScript schema**:
```ts
type SummaryInput = {
  business: { name: string; industry: string };
  date_range: { start: string; end: string };
  ads: {
    spend_cents: number;
    spend_delta_pct: number;
    conversions: number;
    conversions_delta_pct: number;
    avg_cpc_cents: number;
    avg_cpc_delta_pct: number;
    cost_per_booked_cents: number;
    top_converting_keywords: Array<{ text: string; clicks: number; cost_cents: number; conv: number }>;
    wasted_spend_terms: Array<{ text: string; cost_cents: number; clicks: number; days_with_zero: number }>;
    new_search_terms_flagged: Array<{ term: string; type: 'service_mismatch' | 'high_intent' }>;
  };
  gmb: null | {                                       // null if GMB API not yet approved (current state)
    health_score: number;
    health_score_delta: number;
    unresponded_review_count: number;
    days_since_last_photo: number;
    days_since_last_post: number;
    call_count: number;
    call_count_delta_pct: number;
  };
  seo: {
    pages_crawled: number;
    total_issues: number;
    issues_delta: number;
    health_score: number;
  };
  leadquik: {
    ai_calls_handled: number;
    appointments_booked: number;
    conversion_rate: number;
    top_call_outcome_keywords: Array<{ keyword: string; calls: number; booked: number }>;
  };
};
```

**Prompt template** (versioned in `supabase/functions/generate_ai_summary/prompt.ts`):

```
You are a marketing analyst writing a weekly executive summary for {{business.name}},
a {{business.industry}} business. Audience: busy owner-operator. They have 90 seconds.

Write in this voice:
- Operator-direct, like one contractor talking to another
- Specific numbers, no vague claims
- Lead with the win, then the concern, then concrete actions
- Replace SaaS jargon with plain English

Format (250 words max):

**[Business name] — Week of {{date_range.start}} to {{date_range.end}}**

**The big win:** [single biggest positive story from the data, with specific number]

**What needs attention:** [up to 3 bulleted items, each with the dollar impact]

**What's working — keep doing:** [up to 3 bulleted items, each with the dollar impact]

**Recommended actions** (each must be applyable from the dashboard):
- [Concrete action 1]
- [Concrete action 2]
- [Concrete action 3]

DATA:
{{json}}

If `gmb` is null, do not mention GMB in the summary — the integration is pending approval.
```

---

## 7. UI specs — Phase B pages

### `/marketing` Dashboard

**Top row** — 4 KPI cards with 7d/30d toggle:
- **Spend** (with delta % vs prior period) — emerald if down with conv stable, red if up with conv flat
- **Conversions** (booked appointments, with delta)
- **Cost per booked appointment** (with target line if `marketing_business_settings.features_enabled.defaults.target_cpa_cents` set)
- **GMB health score** — placeholder card showing "Awaiting Google API approval" until GMB approved

**Middle row** — 2 charts (Recharts):
- Spend vs Conversions stacked area (60 days)
- Cost per booking trend (60 days, with target reference line)

**Bottom row** — 3 cards:
- **Top recommendations** (latest 5 from `marketing_recommendations` where `status = 'pending'`, ORDER BY priority DESC)
- **Recent changes** (last 10 from `marketing_audit_log`, ORDER BY applied_at DESC)
- **This week's AI summary** (latest row from `marketing_ai_summaries`; "Read full →" link to `/marketing/reports`)

**Empty states**: For Phase B, recommendations + audit log will be empty. Show friendly "Coming soon as you take actions in Blaze" placeholder copy, not a blank box.

### `/marketing/ads/overview`

Account selector (only 1 option for Phase B: the GA account just linked), then:
- KPI strip: Spend / Clicks / Impressions / CTR / Avg CPC / Conversions / Cost-per-Conv (last 30d, with deltas)
- Spend over time chart (60 days)
- **Top 10 spend keywords** table (from `marketing_keyword_performance`, last 30d)
- **Top 10 converting keywords** table (same, sorted by conversions DESC)
- **Wasted spend keywords** table — keywords with `cost_cents >= 800 AND conversions = 0` in last 30d. Each row has `[Add as Negative]` button (Phase C will wire the button; Phase B can leave it as a disabled tooltip-explained button: "Available in Phase C")

### `/marketing/reports`

- **Latest AI Summary card** at top — renders `marketing_ai_summaries.summary_markdown` (markdown→HTML) for the most recent row for this business
- **[Generate New Summary] button** (primary CTA, LeadQuik blue) — calls `generate_ai_summary` Edge Function with `{ business_id, force: true }`. Shows spinner during generation (target <30s). On success, reload card.
- **List of previous summaries** — table of date_range_start / date_range_end / trigger / model with [View] action that expands inline

(Continued in next paste...)

<!-- Phase B execute prompt — continued, paste 3 of 3 -->

## 8. File structure expected

```
src/
├── pages/marketing/
│   ├── MarketingDashboard.tsx          (was the Phase A placeholder; replace with real dashboard)
│   ├── ads/
│   │   └── AdsOverview.tsx             (NEW — /marketing/ads/overview)
│   ├── reports/
│   │   └── ReportsLatest.tsx           (NEW — /marketing/reports)
│   └── settings/
│       └── AccountsSettings.tsx        (NEW — /marketing/settings/accounts, where OAuth link button lives)
├── components/marketing/
│   ├── KpiCard.tsx                     (reusable KPI tile with delta + sparkline)
│   ├── TrendChart.tsx                  (Recharts line/area wrapper)
│   ├── KeywordTable.tsx                (reusable for top-spend / top-conv / wasted-spend)
│   ├── AISummaryCard.tsx               (renders summary_markdown)
│   ├── AccountSwitcher.tsx             (dropdown — for Phase B always shows 1 GA account)
│   └── LinkAdsAccountModal.tsx         (OAuth flow modal)
├── hooks/
│   ├── useBlazeMarketing.ts            (already exists from Phase A)
│   ├── useAdsAccountMetrics.ts         (NEW — queries marketing_daily_metrics)
│   ├── useTopKeywords.ts               (NEW — queries marketing_keyword_performance)
│   └── useAISummary.ts                 (NEW — queries marketing_ai_summaries + triggers generate)
└── lib/marketing/
    ├── adsApi.ts                       (REST API client — used in Edge Functions, NOT client-side)
    └── formatters.ts                   (cents→dollars, micros→cents, delta percentages, etc.)

supabase/
├── functions/
│   ├── sync_ads_daily/
│   │   └── index.ts                    (NEW — see §4)
│   └── generate_ai_summary/
│       ├── index.ts                    (NEW — see §6)
│       └── prompt.ts                   (versioned prompt template)
└── manual_migrations/
    └── 20260527_blaze_phase_b_seed_ga_account.sql
        (manual_migrations/20260527_blaze_phase_b_seed_ga_account.sql:
         INSERT marketing_accounts + marketing_conversion_actions for GA — see §5 OAuth fallback,
         if implementing the stub instead of the full OAuth flow)
```

---

## 9. Reference: existing Python implementations to learn from

The operator's local toolkit at `/Users/davidbrannan/THG Adwords Billing/` has battle-tested Python that proves every API pattern Phase B needs. **Read these and port the logic to TypeScript:**

- `triweekly_check.py` — full daily metrics pull + anomaly detection + email send (THE direct reference for `sync_ads_daily`)
- `apply_negatives.py` — bulk mutation pattern (saves for Phase C)
- `create_conversion_actions.py` — how conversion action IDs are created (saves for understanding `marketing_conversion_actions` seeding)

The operator can paste any of these via clipboard if you want them in your context. Ask if you do.

---

## 10. ✅ Acceptance checklist — Phase B is done when

- [ ] **OAuth linking works** OR **stub seed is in place** for the GA Ads account (`5657725342` with `campaign_filter = 'Georgia May Leads 2026 Abhinav'`)
- [ ] `marketing_accounts` has 1 active row for the GA account
- [ ] `marketing_conversion_actions` has 3 rows for the GA account (form_submit, qualified_lead, booked_appointment with `is_primary = true`)
- [ ] `sync_ads_daily` Edge Function deployed and **invocable**
- [ ] Backfill ran successfully — `marketing_daily_metrics` has **≥ 30 rows** for the GA account
- [ ] `marketing_keyword_performance` has rows from the last 30 days
- [ ] `/marketing` dashboard renders with **real numbers** that match what's visible in Google Ads admin for the GA campaign over the last 7 + 30 days (operator will spot-check 3-5 numbers)
- [ ] `/marketing/ads/overview` shows the keyword tables + wasted spend table populated with real data
- [ ] `[Generate AI Summary]` button on `/marketing/reports` works:
  - Triggers `generate_ai_summary` Edge Function
  - Returns a populated summary in **< 30 seconds**
  - Saves to `marketing_ai_summaries`
  - Sends email via Resend from `noreply@leadquik.com` to `david@hardwood-guys.com`
- [ ] Emailed summary contains specific numbers from the GA campaign (spend, top keywords, wasted terms) — not generic boilerplate
- [ ] **No data leakage**: if logged in as `david@woodtilepros.com` viewing CT/NY, no GA data is visible anywhere; `/marketing` 404s (CT/NY still not seeded into Blaze)
- [ ] All mutations done via service role inside Edge Functions; nothing client-side touches the developer token
- [ ] Migration files reported with one-line description each

---

## 11. Stop here

After hitting all checkboxes, **report back** with:
1. Migration files created (paths + 1-line description)
2. New Edge Function URLs (for invocation testing)
3. Any deviations from the spec (be explicit about what you stubbed vs. shipped)
4. Any unanswered questions

Then operator runs the spot-check + green-lights Phase C.

**Do not proceed to Phase C until operator confirms Phase B ✅.**
