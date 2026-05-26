# Project Blaze — Lovable Bootstrap Spec (v1.2)

**Paste sections of this file into a fresh chat inside the existing LeadQuik Lovable project.** Do not paste the entire file at once — see "How to use this spec" below.

This file is self-contained. No prior conversation context is required.

---

## 0. Read this first (instructions for the AI doing the build)

You are building a new module called **Blaze** **inside the existing LeadQuik Lovable + Supabase project** (live at https://leadquik.com). This is NOT a separate project. Same auth, same database, same billing layer.

LeadQuik already has:
- Supabase Auth
- A `businesses` table as the primary tenancy entity (NOT `tenants` — see Schema Naming below)
- An AI receptionist (Retell + Twilio) capturing inbound calls and texts
- `leads`, `calls`, `bookings`, `appointments` tables — these already store `gclid`, `utm_*`, `referrer`, and `landing_path` (do not duplicate this data into Blaze)
- Stripe billing
- **Lovable Emails pipeline** for all outbound transactional mail. Sender is `noreply@leadquik.com` (standing memory rule — do not split sender subdomains). SendGrid in the project is **Inbound Parse only** — do not send through it.

**Blaze adds a new nav tab** at `/marketing` and a small number of new tables prefixed `marketing_`. Do not modify or rename existing LeadQuik tables.

**Atlas (the founder's separate CRM) is explicitly out of scope for v1.** Do not add Atlas integration.

### Schema naming convention

LeadQuik's primary tenant entity is the `businesses` table. Every FK in this spec uses `business_id` referencing `businesses(id)`. If your local LeadQuik schema diverges (e.g., uses `tenants`), do a find-and-replace **before applying migrations**: `business_id` → your column name, `businesses(id)` → your table.

---

## How to use this spec with Lovable

**Do not paste the entire file as one prompt.** A spec this size triggers Lovable's "half-build and claim done" failure mode.

Paste in this order, **one prompt at a time**:

1. **Foundation context** (paste together as a single context-priming message, no build requested):
   - Section 0 (instructions)
   - Section 1 (founding tenant seed data)
   - Section 2 (Internal Elite tier)
   - Section 3 (brand colors + voice)
   - Section 4 (architecture overview)
   - Section 14 (API Implementation Appendix — read-before-write reference)

2. **Day 0 build** (after foundation is loaded, paste Section 10 Phase A only):
   > "Execute Section 10 Phase A. Stop when the ✅ checklist passes. Do not move to Phase B."

3. **Day 1-7 build** (after Phase A passes):
   > "Execute Section 10 Phase B. Stop when the ✅ checklist passes."

4. ...and so on through Phase C, D, E, F.

Each phase has its own ✅ acceptance checklist. Lovable must hit every checkbox before claiming the phase is done.

**Reference sections** (Lovable consults but doesn't "build"):
- Section 5 (Supabase schema)
- Section 6 (Edge Function specs)
- Section 7 (UI page specs)
- Section 8 (AI summary spec)
- Section 9 (public grader spec)
- Section 11 (Elite-tier defaults)
- Section 14 (API appendix)

---

## 1. Initial seed data — the four founding customers

These are the founder's own four home-services businesses, used as dogfood + demo. All owned by **david@hardwood-guys.com**. **Setup order: Hardwood Guys GA first, then the other three in parallel** once GA is stable.

All four get the **Internal Elite** service tier (Section 2) — all features unlocked, no billing enforced.

### Tenant 1 — Hardwood Guys GA *(SEED FIRST)*

```yaml
slug: thg-ga
display_name: "Hardwood Guys — Georgia"
owner_email: david@hardwood-guys.com
service_tier: internal_elite
website: https://www.hardwoodguys.co
billing_status: internal_no_charge
industry: home_services
service_category: hardwood_floor_installation
service_area:
  country: US
  state: GA
  primary_zip: "30189"
  radius_miles: 30
google_ads:
  manager_customer_id: "7990784116"           # MCC
  customer_id: "5657725342"
  campaign_filter: "Georgia May Leads 2026 Abhinav"
  conversion_action_ids:                       # see §14.8.P on full-resource-name construction
    form_submit: "7620422282"
    qualified_lead: "7620422285"
    booked_appointment: "7620422288"           # PRIMARY for bidding
google_business_profile:
  location_id: TBD                             # operator fills in /marketing/settings
leadquik_business_id: TBD                      # match to existing businesses table
brand_voice: |
  Operator-direct. Real GA contractor. No SaaS slop.
  Use "we" not "the company". Numbers, not adjectives.
default_alerts_enabled:
  - spend_spike
  - conv_drop
  - neg_review
  - billing_block
  - low_impr_share
```

### Tenant 2 — Hardwood Guys CT/NY

```yaml
slug: thg-ctny
display_name: "Hardwood Guys — CT / NY"
owner_email: david@hardwood-guys.com
service_tier: internal_elite
website: https://www.hardwoodguys.co           # shares site with GA
billing_status: internal_no_charge
industry: home_services
service_category: hardwood_floor_installation
service_area:
  country: US
  states: [CT, NY]
google_ads:
  manager_customer_id: "7990784116"
  customer_id: "5657725342"                    # SAME account as GA — filter by campaign
  campaign_filter: "CT/NY May Leads 2026 Abhinav"
  conversion_action_ids:
    form_submit: "7620422291"
    qualified_lead: "7620422294"
    booked_appointment: "7620422297"
google_business_profile:
  location_id: TBD
leadquik_business_id: TBD
```

**Note for sync layer:** GA + CT/NY share the same Ads `customer_id`. `sync_ads_daily` must fetch the Ads API **once per unique `customer_id`** and partition the result rows by `campaign_id` server-side before writing to per-business tables. Don't make two API calls.

### Tenant 3 — Sunshine FL

```yaml
slug: sunshine-fl
display_name: "Sunshine Interior Remodeling — Lake County FL"
owner_email: david@hardwood-guys.com           # founder is owner of record
operator_email: TBD                            # Wally — set in Settings once invited
service_tier: internal_elite
website: https://www.sunshinehomefl.com
billing_status: internal_no_charge
industry: home_services
service_category: interior_remodeling          # built-ins, custom cabinets, kitchens — NOT refacing
service_area:
  country: US
  state: FL
  primary_metro: orlando
  approximate_zips: [32703, 32712, 32757, 32778, 32798, 32819, 32836,
                     34711, 34714, 34715, 34737, 34756, 34761, 34786, 34787]
  excludes: ["The Villages"]
google_ads:
  manager_customer_id: "7990784116"
  customer_id: "1682174850"
  campaign_filter: "Sunshine Interior Remodeling - Lake County FL"
  conversion_action_ids:
    form_submit: "7620809497"
    qualified_lead: "7620809500"
    booked_appointment: "7620809503"
google_business_profile:
  location_id: TBD
leadquik_business_id: TBD
service_keywords_yes: [custom cabinets, built-in cabinets, tile, kitchen remodel, bathroom remodel]
service_keywords_no:  [refacing, refinishing countertops, vinyl wrap, cheap remodel]
```

### Tenant 4 — Cabinet Painting Guys

```yaml
slug: cabinet-painting-guys
display_name: "Cabinet Painting Guys — Woodstock GA"
owner_email: david@hardwood-guys.com
service_tier: internal_elite
website: https://www.kitchencabinetpaintingguys.com
billing_status: internal_no_charge
industry: home_services
service_category: kitchen_cabinet_painting     # KITCHEN cabinets only
service_area:
  country: US
  state: GA
  primary_zip: "30189"
  radius_miles: 20
google_ads:
  manager_customer_id: "7990784116"
  customer_id: "9493341241"
  campaign_filter: "Cabinet Painting Guys - Woodstock"
  conversion_action_ids:
    form_submit: "7620422057"
    qualified_lead: "7620422060"
    booked_appointment: "7620422063"
google_business_profile:
  location_id: TBD
leadquik_business_id: TBD
service_keywords_yes: [kitchen cabinet painting, kitchen cabinet refinishing,
                       paint kitchen cabinets, spray paint cabinets]
service_keywords_no:  [exterior painting, interior painting, house painting, refacing,
                       bathroom cabinet, bathroom vanity, commercial painting,
                       furniture painting, wallpaper, DIY, tutorial]
```

---

## 2. What "Internal Elite" means

These four businesses are the founder's own. They get every feature, no billing enforced, and double as case-study material. For external customers, Elite is $199/mo with the same feature set.

Behavior for Internal Elite:
- All modules enabled by default (Section 11)
- No usage limits
- Daily AI summary (vs weekly for paid)
- All anomaly alerts enabled
- Unlimited audit-log retention
- Auto-opt-in to new beta features

---

## 3. Brand identity (use these exactly)

### Colors

```
Primary blue:     #2563EB   (Tailwind blue-600 — main CTAs, links)
Hover blue:       #60A5FA   (blue-400)
Deep navy:        #0B1F4A   (hero gradient base)
Success/Live:     #10B981   (emerald-500 — booked, working, positive deltas)
Attention/CTA:    #F97316   (orange-500 — secondary actions, "Upgrade")
Accent purple:    #A855F7   (sparing — premium/AI features)
Body text:        #64748B   (slate-500)
Muted text:       #94A3B8   (slate-400)
Borders:          #E2E8F0   (slate-200)
Light bg:         #F8F9FB   (page background, light mode)
Dark bg:          #0A0A0F   (dark mode)
Dark surface:     #15161B   (dark mode cards)
Transparency:     /5, /10, /15, /20, /30, /35, /50 layers heavily used
```

### Typography

Match LeadQuik — system font stack `-apple-system, Segoe UI, Arial, sans-serif`.

### Voice

Operator-direct. One contractor talking to another. Numbers always over adjectives.

- ❌ "Improve ROI" → ✅ "Saves $342/mo vs CallRail"
- ❌ "Optimize campaigns" → ✅ "Cut waste 67%, conversions up 12×"
- Avoid: "AI-powered", "ecosystem", "platform", "transform", "leverage", "synergy"

---

## 4. Architecture

### Stack
- **Frontend**: Lovable React + TypeScript, Tailwind, Recharts for charts
- **Backend**: Supabase (existing) — Postgres + Auth + Edge Functions + Storage
- **External APIs**: Google Ads API v23 (REST from Edge Functions), Google Business Profile API, Anthropic Claude (AI summaries), Resend (outbound email), Twilio (SMS)

### Nav

```
LeadQuik (existing nav)
├── Calls / Calendar / Photos / Customers / Settings   (existing — unchanged)
└── Marketing  (NEW — Blaze)
    ├── Dashboard           /marketing
    ├── Google Ads          /marketing/ads
    │   ├── Overview        /marketing/ads
    │   ├── Keywords        /marketing/ads/keywords
    │   ├── Search Terms    /marketing/ads/search-terms
    │   ├── Negatives       /marketing/ads/negatives
    │   ├── Budgets         /marketing/ads/budgets
    │   └── Audit Log       /marketing/ads/audit
    ├── Local Pulse         /marketing/local            (may ship empty pre-GMB-approval)
    │   ├── Health
    │   ├── Reviews
    │   ├── Photos
    │   └── To-Do
    ├── SEO                 /marketing/seo
    │   ├── Overview
    │   ├── Pages
    │   └── History
    ├── Pulse               /marketing/pulse            (LeadQuik conversion feed)
    ├── Reports             /marketing/reports
    │   ├── Latest
    │   ├── History
    │   └── Improvements
    └── Settings            /marketing/settings
        ├── Companies
        ├── Linked Accounts
        ├── Alerts
        └── Integrations
```

### Company switcher

Dropdown at the top of every Blaze page listing all businesses the logged-in user has access to. Selecting one filters every chart, table, and metric. "All companies" option shows cross-business totals. **Driven by a feature flag — see Section 13 rollback plan.**

---

## 5. Data model

All new tables prefixed `marketing_`. **Every `marketing_*` table has RLS enabled with `business_id`-based access** (matching existing LeadQuik RLS pattern). The only exception is `grader_scans` — see note below.

```sql
-- Per-business settings (extends existing businesses table)
create table marketing_business_settings (
  business_id uuid primary key references businesses(id) on delete cascade,
  display_name text not null,
  website_url text,
  service_tier text not null default 'solo',     -- 'solo'|'managed'|'internal_elite'
  industry text,
  service_category text,
  service_area jsonb,
  service_keywords_yes text[],
  service_keywords_no text[],
  brand_voice text,
  default_alerts_enabled text[],
  billing_status text default 'active',          -- 'active'|'past_due'|'internal_no_charge'
  auto_review_request_enabled boolean default true,
  features_enabled jsonb default '{}',           -- see §11 for Elite defaults
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- Linked external accounts (Ads / GMB / website)
create table marketing_accounts (
  id uuid primary key default gen_random_uuid(),
  business_id uuid not null references businesses(id) on delete cascade,
  type text not null check (type in ('google_ads','gmb','website')),
  external_id text not null,                     -- ads customer_id, gmb location_id, site domain
  manager_customer_id text,                      -- MCC, for Ads accounts
  campaign_filter text,                          -- nullable: restrict to specific campaign
  display_name text not null,
  status text not null default 'active',
  meta jsonb default '{}',
  linked_at timestamptz default now(),
  unique (business_id, type, external_id, campaign_filter)
);

-- Conversion action mapping (Blaze → Ads conversion action IDs)
create table marketing_conversion_actions (
  id uuid primary key default gen_random_uuid(),
  business_id uuid references businesses(id) on delete cascade,
  account_id uuid references marketing_accounts(id),
  action_name text not null,                     -- 'form_submit'|'qualified_lead'|'booked_appointment'
  external_action_id text not null,              -- just the ID portion; full resource constructed at upload (see §14.8.P)
  is_primary boolean default false,
  unique (business_id, account_id, action_name)
);

-- Daily metrics snapshot (one row per account per day)
create table marketing_daily_metrics (
  id bigserial primary key,
  account_id uuid references marketing_accounts(id) on delete cascade,
  date date not null,
  ads_spend_cents int,
  ads_clicks int,
  ads_impressions int,
  ads_conversions numeric(8,2),
  ads_avg_cpc_cents int,
  ads_impression_share numeric(5,2),
  gmb_searches int,
  gmb_calls int,
  gmb_direction_requests int,
  gmb_website_clicks int,
  gmb_messages int,
  gmb_review_count int,
  gmb_avg_rating numeric(2,1),
  gmb_unresponded_count int,
  seo_pages_crawled int,
  seo_total_issues int,
  seo_health_score int,
  unique (account_id, date)
);

create table marketing_keyword_performance (
  id bigserial primary key,
  account_id uuid references marketing_accounts(id) on delete cascade,
  campaign_id text,
  keyword_text text not null,
  match_type text,
  date_range_start date not null,
  date_range_end date not null,
  clicks int default 0,
  cost_cents int default 0,
  conversions numeric(8,2) default 0,
  conversion_value_cents int default 0,
  unique (account_id, campaign_id, keyword_text, match_type, date_range_start)
);

create table marketing_search_terms (
  id bigserial primary key,
  account_id uuid references marketing_accounts(id) on delete cascade,
  campaign_id text,
  search_term text not null,
  date date not null,
  clicks int,
  cost_cents int,
  conversions numeric(8,2),
  flagged_as text,                               -- 'wasted_spend'|'service_mismatch'|'high_intent'|null
  action_taken text,                             -- 'added_negative'|'added_keyword'|null
  action_taken_at timestamptz
);

create table marketing_negatives (
  id uuid primary key default gen_random_uuid(),
  business_id uuid not null references businesses(id) on delete cascade,
  account_id uuid references marketing_accounts(id),   -- nullable: apply to all
  term text not null,
  match_type text not null default 'PHRASE',
  reason text,
  added_by text,
  added_at timestamptz default now()
);

-- Conversion UPLOAD tracking only.
-- Source data (gclid, utm_*, referrer, landing_path) lives on existing leads/calls/bookings tables.
-- This table tracks the Ads-side upload status, not the lead itself.
create table marketing_conversion_uploads (
  id uuid primary key default gen_random_uuid(),
  business_id uuid not null references businesses(id) on delete cascade,
  account_id uuid references marketing_accounts(id),
  source_table text not null,                    -- 'leads'|'bookings'|'appointments'|'calls'
  source_row_id uuid not null,                   -- FK by convention to the source table
  conversion_action_id uuid references marketing_conversion_actions(id),
  gclid text,                                    -- copied for fast lookup; canonical lives on source
  occurred_at timestamptz not null,
  uploaded_to_ads_at timestamptz,
  upload_status text not null default 'pending', -- pending|uploaded|failed|skipped_no_gclid
  upload_error text,
  value_cents int,
  unique (source_table, source_row_id, conversion_action_id)
);

create table marketing_gmb_reviews (
  id uuid primary key default gen_random_uuid(),
  account_id uuid references marketing_accounts(id) on delete cascade,
  external_review_id text not null,
  rating int not null,
  text text,
  reviewer_name text,
  posted_at timestamptz not null,
  response_text text,
  response_posted_at timestamptz,
  sentiment text,
  matched_call_id uuid,                          -- FK by convention to existing calls table
  matched_lead_id uuid,                          -- FK by convention to existing leads table
  unique (account_id, external_review_id)
);

create table marketing_gmb_uploads (
  id uuid primary key default gen_random_uuid(),
  account_id uuid references marketing_accounts(id) on delete cascade,
  upload_type text not null,                     -- 'photo'|'post'
  category text,
  uploaded_at timestamptz not null,
  external_id text
);

create table marketing_seo_scans (
  id uuid primary key default gen_random_uuid(),
  account_id uuid references marketing_accounts(id) on delete cascade,
  scanned_at timestamptz default now(),
  pages_crawled int,
  total_issues int,
  health_score int,
  issues_by_category jsonb,
  raw_data_path text                             -- Supabase Storage path
);

create table marketing_recommendations (
  id uuid primary key default gen_random_uuid(),
  business_id uuid not null references businesses(id) on delete cascade,
  account_id uuid references marketing_accounts(id),
  type text not null,
  priority int not null default 50,
  title text not null,
  detail text,
  payload jsonb,
  status text not null default 'pending',
  surfaced_at timestamptz default now(),
  resolved_at timestamptz,
  resolved_by text
);

create table marketing_audit_log (
  id bigserial primary key,
  business_id uuid not null references businesses(id) on delete cascade,
  account_id uuid references marketing_accounts(id),
  action text not null,
  resource text,
  before_value jsonb,
  after_value jsonb,
  applied_by text,
  applied_at timestamptz default now()
);

create table marketing_ai_summaries (
  id uuid primary key default gen_random_uuid(),
  business_id uuid not null references businesses(id) on delete cascade,
  generated_at timestamptz default now(),
  date_range_start date not null,
  date_range_end date not null,
  trigger text not null,                         -- 'weekly_cron'|'daily_cron'|'manual_push'
  summary_markdown text not null,
  highlight_items jsonb,
  model text
);

create table marketing_alerts (
  id uuid primary key default gen_random_uuid(),
  business_id uuid not null references businesses(id) on delete cascade,
  alert_type text not null,
  threshold jsonb,
  channel text not null,
  enabled boolean default true
);
```

### Public grader table — explicitly NOT in the `marketing_` namespace

The public grader has no auth. It cannot use `business_id`-based RLS. Carve it out as its own table with service-role-only writes and signed-slug public reads:

```sql
-- NOT prefixed marketing_ — public schema, no RLS by business_id
create table grader_scans (
  id uuid primary key default gen_random_uuid(),
  public_slug text not null unique,              -- short random slug for public report URL
  url text not null,
  email text,                                    -- captured for lead drip if user opted in
  ads_customer_id text,                          -- if they OAuthed Ads in the grader
  health_score int,
  scored_at timestamptz default now(),
  payload jsonb,                                 -- full scan result (issues, etc.)
  converted_to_signup boolean default false,
  converted_business_id uuid                     -- nullable FK to businesses after signup
);

-- RLS policies (do NOT use business_id):
alter table grader_scans enable row level security;
-- Reads: anyone can read by public_slug
create policy grader_scans_public_read on grader_scans for select using (true);
-- Writes: service role only
-- (no INSERT/UPDATE policies for anon role — service role bypasses RLS)
```

The grader Edge Function (Section 9) runs with the service role to write; the public report page reads by `public_slug` only.

### RLS pattern for everything in `marketing_*`

```sql
alter table marketing_<table> enable row level security;
create policy marketing_<table>_business_access on marketing_<table>
  for all using (
    business_id in (
      select business_id from business_members where user_id = auth.uid()
    )
  );
```

(Match whatever `business_members` join table LeadQuik already uses for user-to-business membership.)

---

## 6. Edge Functions

| Function | Trigger | Job |
|---|---|---|
| `sync_ads_daily` | cron `0 3 * * *` ET | For each unique `(external_id)` in `marketing_accounts` of type=google_ads, fetch **once** and partition results by campaign_id when writing per-business rows (Section 1 note on GA+CT/NY sharing customer_id). Upsert daily metrics, keyword perf, search terms. |
| `sync_gmb_daily` | cron `30 3 * * *` ET | For each GMB account: insights, new reviews, photo cadence. Invoke `match_review_to_call` for each new review. **No-op gracefully if GMB API approval not yet granted** — return early with status 'awaiting_api_approval'. |
| `seo_scan_weekly` | cron `0 7 * * 0` ET (Sunday 7am) | Crawl each linked website, score, persist. Raw data goes to Supabase Storage. |
| `generate_ai_summary` | cron `30 7 * * 1` (Monday) + on-demand HTTP POST | Compose structured prompt, call Claude, store summary, **send email via existing Lovable Emails pipeline** (sender `noreply@leadquik.com`). Body `{business_id, force: true}` triggers manual push. |
| `generate_daily_summary` | cron `0 8 * * *` ET, Internal Elite only | Same as weekly but covers last 24h. Filters to `marketing_business_settings.service_tier = 'internal_elite'`. |
| `apply_recommendation` | HTTP POST from UI | Validate user permission. Execute the action via the appropriate API (Ads or GMB). Write audit log. Mark recommendation applied. |
| `detect_anomalies` | cron `0 4 * * *` ET (daily) | Compare current 7d to prior 7d per account. Fire alerts via configured channel (Lovable Emails for email, Twilio for SMS). **Hourly frequency available as opt-in via `marketing_business_settings.features_enabled.hourly_anomaly_check = true`.** |
| `match_review_to_call` | Invoked inline by `sync_gmb_daily` on each new review | Match reviewer name + posted_at to LeadQuik calls within 30d prior. Update review row. |
| `auto_request_review` | cron `*/15 * * * *` (every 15 min) | Poll `bookings` (or equivalent existing LeadQuik table) for rows where `created_at` is between `now() - 25 hours` and `now() - 23 hours` AND no review request sent. Send SMS via Twilio with the customer's GMB review link. **Polling pattern, not a DB trigger** — LeadQuik doesn't expose a booking-created event hook. |
| `upload_leadquik_conversion` | Invoked from existing LeadQuik booking-flow code (add one line) OR pg_cron polling fallback | Pulls gclid from the source row's existing column. Constructs full conversionAction resource name. POSTs to Ads `uploadClickConversions`. Writes to `marketing_conversion_uploads`. See §14.12 for full pattern. |
| `public_grader_scan` | HTTP POST from grader.leadquik.com | Crawl URL, optionally score Ads if OAuth token provided, persist to `grader_scans`, return shareable slug. Service role only (no auth). |

### Outbound email — use the existing Lovable Emails pipeline

All email sends in Blaze use the existing LeadQuik transactional mailer:
- From: **`noreply@leadquik.com`** (standing memory rule — single sender domain, do not split SPF/DKIM by introducing a second sender subdomain)
- Pipeline: existing Lovable Emails queue (same pattern the rest of LeadQuik already uses for transactional sends)
- SendGrid is **Inbound Parse only** — do not send through it.

---

## 7. UI page specs

### `/marketing` — Dashboard

Top row — 4 KPI cards with 7d/30d toggle:
- Spend (with delta % vs prior period)
- Conversions (booked appointments, with delta)
- Cost per booked appointment (with target line if set)
- GMB health score (with delta vs last week, or "Awaiting GMB API approval" placeholder)

Middle row — 2 charts:
- Spend vs Conversions stacked area (60d)
- Cost per booking trend (60d, with target line)

Bottom row — 3 cards:
- Top recommendations (5 highest-priority pending items)
- Recent changes (last 10 audit log entries)
- This week's AI summary (latest row)

### `/marketing/ads/overview`

Account selector (if multiple), then:
- Spend / Clicks / Impressions / CTR / Avg CPC / Conversions / Cost/Conv KPI strip
- Spend over time chart
- Top 10 spend keywords table
- Top 10 converting keywords table
- Wasted-spend keywords table (≥$8, 0 conv, last 30d) — each row has [Add Negative] button

### `/marketing/ads/keywords` — Full keyword table

Columns: keyword / match type / clicks / cost / conv / CPA / impression share

Sortable, filterable.

**Bulk actions** (no manual bid editing — modern campaigns use Smart Bidding where per-keyword bid sliders are inert):
- Pause / unpause
- Add as negative
- **For Smart Bidding campaigns**: row-level "Set tCPA target for this keyword's ad group" (the only meaningful bid lever)
- **For Manual CPC campaigns** (rare in 2026): show a max CPC editor

Detect bid strategy per row via the campaign's `bidding_strategy_type` and render the appropriate control.

### `/marketing/ads/search-terms`

Last 30d search terms that triggered ads. Auto-flagged: 🟢 high intent / 🟡 service-mismatch / 🔴 wasted spend.

One-click [Negative] [Add as keyword] [Dismiss].

Smart filters: "service-mismatch only", "≥$5 spent zero conv only".

### `/marketing/ads/negatives` — Negatives library

Per-business negatives table with:
- "Apply to all my businesses" toggle for shared blocks
- Bulk paste-import
- Show which campaigns each negative is applied to

### `/marketing/ads/budgets`

Per-campaign current budget + 7d actual + month-end projection. ±50% safety rail on edits, larger changes require typed confirmation.

For Smart Bidding campaigns, additionally show the tCPA / tROAS target if set, with edit controls.

### `/marketing/ads/audit`

Append-only log of every change Blaze made. Filterable by user / action type / date.

### `/marketing/local` — Local Pulse health

**If GMB API not yet approved**: show a "Setup pending" state with a [Check status] button that pings the GMB API and updates if approval came through. Otherwise:

- Health score ring (0-100, animated)
- Top 3 score-blocking issues with [Fix] buttons
- This week's accountability checklist (photo cadence, unresponded reviews, post cadence, Q&A)
- Performance KPIs with 30d trend

### `/marketing/local/reviews`

Live feed of all reviews. Sentiment auto-tag.

[Reply] button opens drawer with AI-drafted reply (editable). Submits via GMB API where available; otherwise deep-links to the Business Profile Manager UI pre-filled.

KPI: response rate.

### `/marketing/local/photos`

Grid of last 30 photos by category. Flag any category >30d stale.

[Upload from LeadQuik Quick Capture] pulls from existing LeadQuik photo library, lets user select + push to GMB.

### `/marketing/local/todo`

Single prioritized list of every Local Pulse action item. Drag-to-reorder, check-off.

### `/marketing/seo` — SEO overview

- Health score ring per linked site
- Chart: health score over time
- Latest scan summary by category
- [Run scan now] manual trigger

### `/marketing/seo/pages`

Per-URL issue table. Click row → drawer with all issues + suggested fixes + deep-link to CMS.

### `/marketing/seo/history`

Trends + audit-log annotations (when changes were made).

### `/marketing/pulse` — LeadQuik conversion feed

Live feed of every call/text/booking. Per row: timestamp / caller / outcome / matched keyword / campaign / gclid / estimated $value. Filterable.

KPI strip: AI calls / booked / conversion rate / avg job $.

### `/marketing/reports`

Latest AI summary card. Big [Generate New Summary] button → calls `generate_ai_summary` with `force=true`.

List of previous summaries.

### `/marketing/reports/history`

Calendar/list view of every summary.

### `/marketing/reports/improvements`

Reads `marketing_audit_log`. Each change → before/after metric chart. "Blaze made N changes this month — here's the cumulative impact."

### `/marketing/settings/companies`

List + edit display name, website, service area, brand voice, default alerts per business.

### `/marketing/settings/accounts`

Linked Ads accounts (with [Reconnect OAuth] each). Linked GMB locations. Linked websites.

[Link new Google Ads account] kicks off the MCC link request flow.

### `/marketing/settings/alerts`

Toggleable list of alerts + threshold inputs + per-channel routing (email/SMS/push).

### `/marketing/settings/integrations`

API key entry for CallRail (fallback conv source), Stripe, Twilio. LeadQuik connection status.

---

## 8. AI Summary specification

### Trigger
- **Weekly** Monday 07:30 ET, all businesses
- **Daily** 08:00 ET, Internal Elite only
- **Manual** push from `/marketing/reports` button → POST to `generate_ai_summary` with `force: true`

### Input — structured

```typescript
type SummaryInput = {
  business: { name: string, industry: string };
  date_range: { start: string, end: string };
  ads: {
    spend_cents: number;
    spend_delta_pct: number;
    conversions: number;
    conversions_delta_pct: number;
    avg_cpc_cents: number;
    avg_cpc_delta_pct: number;
    cost_per_booked_cents: number;
    top_converting_keywords: Array<{ text: string, clicks: number, cost_cents: number, conv: number }>;
    wasted_spend_terms: Array<{ text: string, cost_cents: number, clicks: number, days_with_zero: number }>;
    new_search_terms_flagged: Array<{ term: string, type: 'service_mismatch'|'high_intent' }>;
  };
  gmb: {
    health_score: number | null;                  // null if API not yet approved
    health_score_delta: number | null;
    unresponded_review_count: number | null;
    days_since_last_photo: number | null;
    days_since_last_post: number | null;
    ranking_changes: Array<{ query: string, before_rank: number, after_rank: number }> | null;
    call_count: number | null;
    call_count_delta_pct: number | null;
  };
  seo: {
    pages_crawled: number;
    total_issues: number;
    issues_delta: number;
    new_issues: Array<{ type: string, url: string }>;
    health_score: number;
  };
  leadquik: {
    ai_calls_handled: number;
    appointments_booked: number;
    conversion_rate: number;
    top_call_outcome_keywords: Array<{ keyword: string, calls: number, booked: number }>;
  };
};
```

### Prompt template (versioned in code as `supabase/functions/generate_ai_summary/prompt.ts`)

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

If GMB data is null, do not mention GMB in the summary — the integration is pending approval.
```

### Model

- **Primary**: Anthropic Claude (latest Sonnet)
- **Fallback**: OpenAI GPT-4o
- **Failover**: if both fail, send templated non-AI summary via Resend so the operator still gets the Monday email

### Email delivery

Via the existing Lovable Emails pipeline, sender `noreply@leadquik.com`. Subject: `🪵 Weekly Marketing Summary — {{business.name}} — {{date_range.start}}`.

---

## 9. Public Website Grader

### Domain: `grader.leadquik.com`

### Scoring rubric (transparent — show in the report)

```
SEO Technical (30 pts)
  - HTTPS                     5
  - Mobile-friendly           5
  - Page speed < 3s avg       5
  - No broken links           5
  - HTML size < 500KB avg     5
  - Sitemap + robots.txt OK   5

SEO Content (20 pts)
  - Meta descriptions present 5
  - Titles 30-60 chars        5
  - H1 on every page          5
  - Image alt text coverage   5

Local Presence (30 pts — degrades gracefully if GMB API not approved)
  - GMB verified              5
  - GMB completeness >80%     5
  - Reviews ≥20               5
  - Avg rating ≥4.5           5
  - Response rate ≥80%        5
  - Photos uploaded last 30d  5

Ads Health (20 pts, only if user OAuthed)
  - Conversion tracking on    5
  - Negative keyword count ≥20  5
  - No "wasted spend" ≥$50/term 5
  - Geo + ad-schedule sane    5
```

### UX flow

1. Landing: URL input + optional Ads OAuth → [Get my free score]
2. Animated 30-sec scan (real activity, not fake progress)
3. Score reveal: ring, score, competitor benchmark, top-3 issues
4. Email gate: "Want the full report? Drop email." Capture → drip into LeadQuik signup nurture
5. Public shareable URL: `grader.leadquik.com/report/{public_slug}` — SEO-indexable

---

## 10. Build sequence (phased)

Each phase is its own paste-prompt. Lovable must hit every checkbox in the phase's ✅ checklist before moving on.

### Phase A — Day 0 setup *(paste this prompt first)*

> Execute Phase A. Stop when the ✅ checklist passes.

- Add a Marketing tab to LeadQuik nav, gated behind a feature flag (`features_enabled.blaze_marketing_tab`). Default: off for everyone except businesses with `service_tier='internal_elite'`.
- Create all `marketing_*` tables from Section 5 in Supabase, with RLS policies matching the Section 5 pattern.
- Create the `grader_scans` table per Section 5 (public schema, separate RLS rules).
- Seed `marketing_business_settings` for **Hardwood Guys GA only** using Section 1 Tenant 1 data.
- Apply for Google Business Profile API access (Section 14.4). Track that the application is submitted — do not block on approval.

**✅ Before you say done:**
- [ ] All `marketing_*` tables exist in Supabase with RLS enabled
- [ ] `grader_scans` exists with the public-read / service-role-write policy
- [ ] `marketing_business_settings` has exactly 1 row, for `slug='thg-ga'`
- [ ] `/marketing` route exists in the app, gated by feature flag, and is visible when logged in as david@hardwood-guys.com (otherwise 404)
- [ ] GMB API access application is submitted; record submission date in `marketing_business_settings.meta`

### Phase B — Day 1-7 GA dogfood read-only

> Execute Phase B. Stop when the ✅ checklist passes.

- Implement OAuth flow for linking the GA Ads account (customer_id `5657725342`, campaign filter `"Georgia May Leads 2026 Abhinav"`)
- Populate `marketing_accounts` with the linked row
- Populate `marketing_conversion_actions` with the 3 GA conversion action IDs from Section 1 Tenant 1
- Build the `sync_ads_daily` Edge Function (port logic from `triweekly_check.py` — see Section 14.9). Backfill last 90 days into `marketing_daily_metrics`.
- Build the `/marketing` dashboard with the Section 7 layout, reading from `marketing_daily_metrics` (no GMB or SEO data yet — show "Not yet linked" placeholders for those KPI cards)
- Build the `generate_ai_summary` Edge Function (Section 8). Wire the manual push button on `/marketing/reports`. Email goes via Resend from `notifications@leadquik.com`.

**✅ Before you say done:**
- [ ] OAuth flow connects the GA Ads account successfully; `marketing_accounts` shows 1 active row
- [ ] `sync_ads_daily` runs successfully against the linked account; `marketing_daily_metrics` has ≥30 rows
- [ ] `/marketing` dashboard displays real spend / clicks / conv numbers matching what's in Google Ads admin
- [ ] [Generate AI Summary] button produces a summary in <30s, saves to `marketing_ai_summaries`, and delivers via Resend to david@hardwood-guys.com
- [ ] No data from other businesses (CT/NY, Sunshine, Cabinet Painting) appears anywhere (the other 3 aren't seeded yet)

### Phase C — Day 8-14 Mutations + audit

> Execute Phase C. Stop when the ✅ checklist passes.

- Build `/marketing/ads/negatives` with one-click apply, writing to `marketing_negatives` + Ads API
- Build `/marketing/ads/budgets` with the ±50% safety rail
- Implement `apply_recommendation` Edge Function
- Implement `marketing_audit_log` writes on every mutation
- **First-time auto-optimization confirmation**: before the FIRST auto-applied negative for a business, surface a modal: "Blaze will start auto-applying safe junk negatives (jobs, career, tutorial, etc.). One-time confirmation. [Enable auto-opt] [Keep approval-only]." Store the choice in `marketing_business_settings.features_enabled.auto_optimize_negatives_confirmed_at`.

**✅ Before you say done:**
- [ ] Adding a negative from the UI propagates to Google Ads admin within 60s
- [ ] Changing a budget from the UI propagates within 60s, with ±50% confirmation on bigger changes
- [ ] Every mutation appears in `marketing_audit_log` with `before_value` and `after_value` populated
- [ ] First-time auto-opt modal shows on first triggered junk-negative; user choice is persisted

### Phase D — Day 15-21 Seed the other 3 businesses

> Execute Phase D. Stop when the ✅ checklist passes.

- Seed `marketing_business_settings` for CT/NY, Sunshine FL, Cabinet Painting Guys (Section 1 Tenants 2-4)
- Implement company switcher in the Blaze header
- Verify GA + CT/NY data partitioning (shared customer_id, different `campaign_filter`)
- Verify `sync_ads_daily` makes ONE API call per unique `external_id`, partitions results by `campaign_id`

**✅ Before you say done:**
- [ ] Company switcher shows all 4 businesses for david@hardwood-guys.com
- [ ] Switching to Hardwood Guys GA shows GA data only; switching to CT/NY shows CT/NY data only — verified by comparing campaign names against Google Ads admin
- [ ] "All companies" view sums across all 4
- [ ] Audit `sync_ads_daily` logs to confirm only 3 API calls per run (3 unique customer_ids), not 4

### Phase E — Day 22-30 SEO + Pulse

> Execute Phase E. Stop when the ✅ checklist passes.

- Port `seo_scan.py` to `seo_scan_weekly` Edge Function
- Build `/marketing/seo` and `/marketing/seo/pages` reading from `marketing_seo_scans`
- Build `/marketing/pulse` reading from existing LeadQuik `calls`/`leads`/`bookings` joined to `marketing_conversion_uploads`
- Implement `upload_leadquik_conversion` Edge Function (Section 14.12). Wire it into the existing LeadQuik booking-creation code path (single function call addition).
- Add `marketing_conversion_uploads` row for every booking with gclid

**✅ Before you say done:**
- [ ] SEO scan runs weekly via cron, populates `marketing_seo_scans` with at least the 4 founding websites
- [ ] `/marketing/seo` shows historical trend graphs
- [ ] `/marketing/pulse` shows a real call → keyword → campaign → ROI path for at least one actual booked appointment with a gclid
- [ ] `marketing_conversion_uploads.uploaded_to_ads_at` is non-null for at least 1 row, and the corresponding conversion appears in Google Ads conversion reports

### Phase F — Day 31-45 Local Pulse (gated on GMB approval)

> Execute Phase F **only after** GMB API access is approved.

- Implement `sync_gmb_daily` (returns early if API access still pending)
- Build `/marketing/local`, `/local/reviews`, `/local/photos`, `/local/todo`
- Implement `match_review_to_call` and `auto_request_review`
- Auto review-request SMS via Twilio

**If GMB API isn't approved by Day 45**: `/marketing/local` ships visible but in "Awaiting Google approval" empty state with a [Check status] button. Other modules ship anyway.

**✅ Before you say done:**
- [ ] If approved: `sync_gmb_daily` populates review/photo/insight data for all 4 founding businesses
- [ ] If approved: review reply works (or deep-links cleanly to Business Profile Manager where API restricted)
- [ ] If not approved: `/marketing/local` shows graceful empty state and does not crash other pages

### Phase G — Day 46-60 Recommendations + grader

> Execute Phase G. Stop when the ✅ checklist passes.

- Recommendations engine + one-click apply
- Anomaly detector (daily by default per Section 6)
- Public grader live at grader.leadquik.com

**✅ Before you say done:**
- [ ] Recommendations queue populates with at least 3 real recommendations across the 4 businesses
- [ ] Anomaly detector fires an alert when a metric crosses threshold (tested with synthetic data)
- [ ] grader.leadquik.com produces a public report URL for any entered website in <60s
- [ ] Grader-scan → email-capture → LeadQuik signup nurture loop works end-to-end

### Phase H — Day 61-90 Polish + paying customers

- Bring 5 external paying beta customers onboard
- Refine Stripe SKU integration for the Blaze add-on
- Marketing site copy + launch

---

## 11. Internal Elite feature defaults

Apply to all 4 founding businesses:

```yaml
features_enabled:
  blaze_marketing_tab: true            # gates the whole /marketing nav
  ads_module: true
  local_pulse: true                    # may render empty pre-GMB-approval
  seo_module: true
  pulse_module: true
  reports_module: true
  grader_module: true
  auto_optimize_negatives: true        # ⚠ requires first-time confirmation modal (Phase C)
  auto_optimize_negatives_confirmed_at: null   # set by user on first trigger
  auto_review_request: true
  daily_ai_summary: true               # vs weekly for paid
  manual_push_summary: true
  history_view: true
  improvements_view: true
  beta_features: true
  hourly_anomaly_check: false          # opt-in only

alerts_enabled:
  - spend_spike
  - cpc_spike
  - conv_drop
  - billing_block
  - negative_review                    # gated on GMB approval
  - gmb_photo_overdue                  # gated on GMB approval
  - gmb_unresponded_review             # gated on GMB approval
  - low_impression_share
  - anomaly_search_term

defaults:
  ads_max_cpc_warning_threshold_cents: 1500
  daily_budget_safety_rail_pct: 50
  scan_frequency_hours: 24
  audit_log_retention_days: -1
```

### Auto-optimize negatives — explicit safety pattern

Even at Internal Elite tier, **the first** auto-applied negative for any business surfaces a one-time confirmation modal:

> Blaze is about to start automatically applying "safe junk" negative keywords (jobs, career, tutorial, youtube, how to, free, diy, etc.) to your Google Ads campaigns. These never overlap with real customer intent in your industry.
>
> [ Enable auto-opt for this business ]
> [ Keep approval-only (recommend each one to me) ]

Store the user's choice in `marketing_business_settings.features_enabled.auto_optimize_negatives_confirmed_at` (timestamp) or leave null (declined → fallback to approval-required).

---

## 12. Out of scope for v1

- **Atlas integration** (the founder's separate CRM)
- Direct posting to Google Business Profile via API (deprecated)
- Bing / Microsoft Advertising
- Facebook / Meta ads
- Multi-currency or non-English UI
- White-label / agency sub-account mode
- Mobile native app (web responsive only)

---

## 13. Acceptance criteria + rollback plan

### Acceptance criteria for v1 launch

Signed in as `david@hardwood-guys.com`, all of the following must work:

- [ ] All four businesses appear in the company switcher
- [ ] Switching to each one shows correctly-partitioned data (Ads + SEO + Pulse + Reports)
- [ ] "All companies" view shows correct cross-business totals
- [ ] [Generate AI Summary now] works and emails via Resend to david@hardwood-guys.com within 30s
- [ ] Apply a negative from UI → confirmed in Google Ads admin within 60s
- [ ] Change a budget from UI → confirmed in Google Ads admin
- [ ] All mutations appear in `/marketing/ads/audit`
- [ ] At least one gclid → call → keyword → booked attribution path is visible in `/marketing/pulse`
- [ ] SEO scan runs weekly for all 4 websites and populates history
- [ ] Historical trend graphs render with ≥30 days of data
- [ ] Public grader scores any URL in <60s and emails a report
- [ ] Local Pulse ships either functional (if GMB API approved) OR with a graceful empty state (if not)
- [ ] **Feature flag respects existing LeadQuik businesses that haven't been seeded into Blaze** — non-Blaze businesses see no `/marketing` nav, no data leakage, no errors

### Rollback plan

If anything breaks for non-Blaze tenants:

- The `features_enabled.blaze_marketing_tab` flag in `marketing_business_settings` (or the businesses table — match LeadQuik convention) controls visibility
- For businesses without a `marketing_business_settings` row, the flag is treated as `false`, the Marketing tab does not render, and no `marketing_*` queries fire
- To roll back globally: set `blaze_marketing_tab = false` for all Internal Elite rows. The tab disappears, the existing LeadQuik features continue functioning. No DB rollback required.

---

## 14. API Implementation Appendix — read before writing backend code

### 14.1 Google Cloud Console setup (one-time, ~15 min)

1. Use the existing LeadQuik Google Cloud project, OR create a new one
2. Enable in `console.cloud.google.com/apis/library`:
   - Google Ads API
   - My Business Business Information API
   - My Business Account Management API
   - My Business Business Calls API
   - My Business Notifications API
3. Create OAuth 2.0 credentials:
   - **Application type: Desktop app** (Web app refresh tokens expire after 7 days during dev; Desktop tokens don't)
   - Download the JSON
4. OAuth consent screen:
   - Add the scopes from 14.2
   - Add operator email as test user
   - Stay in "Testing" mode — Ads API doesn't require app verification

### 14.2 OAuth scopes

```
https://www.googleapis.com/auth/adwords
https://www.googleapis.com/auth/business.manage
```

(Gmail scope not needed — outbound email goes through Resend, which has its own API key.)

### 14.3 Google Ads developer token

- Apply at: ads.google.com → Tools & Settings → API Center
- **Basic Access** tier (Standard is unnecessary for v1; Test is read-only)
- Operator already has a Basic Access token. Reuse via `GOOGLE_ADS_DEVELOPER_TOKEN` env var.

### 14.4 Business Profile API access — realistic timeline

- Apply via: `support.google.com/business/contact/api_default` → "Request API access"
- **Realistic timeline: 4-12 weeks. Can be rejected and need to re-apply.** Plan UI accordingly — Local Pulse must ship with a graceful empty state if not approved by Phase F.
- Required for: reading reviews, posting review replies, photo uploads, performance data
- Read-only basic profile info works without approval (Business Information API)
- **Submit on Day 0 of build (Phase A) — do not wait**

### 14.5 Required environment variables (Supabase Secrets)

```
GOOGLE_ADS_DEVELOPER_TOKEN
GOOGLE_ADS_CLIENT_ID
GOOGLE_ADS_CLIENT_SECRET
GOOGLE_ADS_REFRESH_TOKEN
GOOGLE_ADS_LOGIN_CUSTOMER_ID = 7990784116    # MCC

GMB_REFRESH_TOKEN                              # if separate from Ads token

ANTHROPIC_API_KEY                              # AI summaries
OPENAI_API_KEY                                 # fallback only

# (no separate email API key — use the existing Lovable Emails pipeline already wired into LeadQuik)

TWILIO_ACCOUNT_SID                             # review-request SMS
TWILIO_AUTH_TOKEN
TWILIO_FROM_NUMBER
```

**Never** put secrets in the spec, in code, or in client bundles. Access via `Deno.env.get(...)` in Edge Functions only.

### 14.6 MCC linking flow for new external customers

The 4 founding businesses are already linked under MCC `7990784116`.

For new external customers joining Blaze:
1. They land on `/marketing/settings/accounts`, click "Link Google Ads Account"
2. Two paths:
   - **Existing Ads account**: They enter their customer_id. Your MCC sends a `CustomerManagerLinkOperation`. They approve via Ads UI (Tools → Setup → Access & Security → Linked accounts → Accept).
   - **No Ads account yet**: Walk them to ads.google.com → Expert Mode → placeholder $1/day paused campaign → return to existing flow.
3. Once linked, your MCC's refresh token can call the Ads API for them via the `login-customer-id` header. No per-customer OAuth.

### 14.7 Library recommendations

**Supabase Edge Functions (Deno runtime):**
- **Google Ads**: REST via `fetch`. The `google-ads-api` npm package isn't Deno-compatible.
- **Business Profile**: `googleapis` via esm.sh: `import { google } from "https://esm.sh/googleapis@131"`
- **Anthropic**: `@anthropic-ai/sdk` via esm.sh
- **Email**: existing Lovable Emails pipeline (sender `noreply@leadquik.com`) — do not introduce Resend or any other sender
- **Twilio**: REST via `fetch`

**Lovable React frontend (Node):**
- Same libraries work natively
- **Never call Google Ads API from the client.** Developer token must stay server-side.

### 14.8 Critical API gotchas

**A. Campaign creation requires EU political declaration:**
```ts
"containsEuPoliticalAdvertising": "DOES_NOT_CONTAIN_EU_POLITICAL_ADVERTISING"
```

**B. Maximize Clicks (TARGET_SPEND) with no CPC cap:**
- Create: `"targetSpend": {}` (empty object). Don't set `cpc_bid_ceiling_micros = 0` (errors `TOO_LOW`).
- Clear ceiling: explicit FieldMask path + omit the field value.

**C. Campaign deletion uses `remove`, not status change:**
```ts
op = { remove: "customers/{cid}/campaigns/{campaignId}" }
```
Don't set `status = REMOVED` — errors `"Enum value REMOVED cannot be used"`.

**D. Partial failure goes on the Request, not the operation:**
```ts
{ customerId, operations, partialFailure: true }
```

**E. Every update needs a FieldMask:**
```ts
op.updateMask = { paths: ["status", "campaignBudget.amountMicros"] }
```
Without it, the API silently no-ops.

**F. Conversion action gotchas:**
- `countingType` = `"ONE_PER_CLICK"` (not `"ONE"`)
- `includeInConversionsMetric` is immutable on create (derived from `primaryForGoal`)
- Use `"GOOGLE_ADS_LAST_CLICK"` until 30+ conversions, then `"GOOGLE_SEARCH_ATTRIBUTION_DATA_DRIVEN"`
- `clickThroughLookbackWindowDays` max is 90
- `type` must be `"UPLOAD_CLICKS"` for offline-uploaded conversions

**G. GAQL date literals:**
- Built-in: `LAST_30_DAYS`, `LAST_90_DAYS` (valid)
- Custom ranges: `WHERE segments.date BETWEEN '2026-04-01' AND '2026-04-30'` (single quotes, BETWEEN)

**H. Ad schedule:**
- `startHour`, `endHour`: 0-23 integers
- `startMinute`, `endMinute`: enums (`ZERO`, `FIFTEEN`, `THIRTY`, `FORTY_FIVE`)
- `dayOfWeek`: enum
- One `AdScheduleInfo` per day — Mon-Fri 9-4 needs 5 separate criteria

**I. Use `search_term_view`, not `keyword_view`, for actual user queries:**
- `keyword_view` = your bid keywords
- `search_term_view` = what users actually typed (the negative-mining surface)

**J. Conversion upload datetime format:**
```ts
conversionDateTime: "2026-05-25 14:30:00-04:00"   // space, not 'T'; offset REQUIRED
```

**K. Filter to enabled-only on most queries:**
```sql
WHERE campaign.status = 'ENABLED'
  AND campaign.advertising_channel_type = 'SEARCH'
```

**L. Negative keywords are campaign-level criteria, not ad-group-level.** Use `CampaignCriterionOperation`. Default match type: `PHRASE`.

**M. Proximity vs geo_target_constant:**
- `proximity`: lat/lng + radius — for service-area businesses
- `location.geo_target_constant`: predefined geos — for KeywordPlanner queries (planner doesn't accept proximity)

**N. KeywordPlanner needs geo_target_constants resolved by name first:**
- Cherokee County GA = `9057337`, Cobb = `9057342`, Forsyth = `9057366`, Fulton = `9057368`
- Query `geo_target_constant` resource by `name` LIKE pattern, then pass the IDs.

**O. Billing diagnosis:**
- Campaign `serving_status = SERVING` + 0 impressions → check `billing_setup.status`
- `status != 'APPROVED'` = billing blocked
- The card change happens in Google Pay center (pay.google.com), NOT via the Ads API

**P. Conversion action resource name construction:**
- The IDs in Section 1 (`7620422282` etc.) are the **action ID portion only** — Google's standard 10-digit IDs
- Full resource name for API calls: `customers/{customerId}/conversionActions/{actionId}`
- Example for GA booked appointment: `customers/5657725342/conversionActions/7620422288`
- Construct at call time; don't store the full string in `marketing_conversion_actions.external_action_id`

### 14.9 Existing Python reference implementations

Operator has a battle-tested Python toolkit at `/Users/davidbrannan/THG Adwords Billing/`. Every pattern above is proven against the live API in these scripts. **Read and port to TypeScript:**

| File | What it teaches |
|---|---|
| `triweekly_check.py` | Daily report: 7d metrics, search terms, anomaly detection, junk-negative auto-apply, HTML email |
| `apply_negatives.py` | Bulk negative pattern, partial_failure handling |
| `set_business_hours.py` | Ad schedule criteria (5-weekday pattern) |
| `set_monday_schedule.py` | Holiday/single-day schedule override |
| `create_conversion_actions.py` | Conversion action create with all gotchas |
| `deploy_cabinet_painting.py` | Full Search campaign create including EU political flag |
| `cleanup_campaigns.py` | Removal via `op.remove` |
| `seo_scan.py` | SEO crawler — port directly to Edge Function |
| `unpause_holiday_campaigns.py` | Bulk re-enable from saved list |

When stuck on an API pattern, grep these files first.

### 14.10 Reference GAQL queries (paste-ready)

**Campaigns with state + budget:**
```sql
SELECT campaign.id, campaign.name, campaign.status, campaign.serving_status,
       campaign.advertising_channel_type, campaign.bidding_strategy_type,
       campaign.target_spend.cpc_bid_ceiling_micros,
       campaign_budget.amount_micros
FROM campaign
WHERE campaign.status != 'REMOVED'
```

**Daily metrics over a range:**
```sql
SELECT campaign.id, campaign.name, segments.date,
       metrics.impressions, metrics.clicks, metrics.cost_micros,
       metrics.conversions, metrics.all_conversions,
       metrics.average_cpc, metrics.search_impression_share
FROM campaign
WHERE segments.date BETWEEN '2026-04-25' AND '2026-05-25'
  AND campaign.advertising_channel_type = 'SEARCH'
ORDER BY segments.date DESC
```

**Keyword performance (30d):**
```sql
SELECT campaign.name, ad_group.name,
       ad_group_criterion.keyword.text, ad_group_criterion.keyword.match_type,
       metrics.clicks, metrics.cost_micros, metrics.conversions, metrics.all_conversions
FROM keyword_view
WHERE segments.date DURING LAST_30_DAYS
  AND ad_group_criterion.status = 'ENABLED'
ORDER BY metrics.conversions DESC, metrics.cost_micros DESC
LIMIT 100
```

**Search terms:**
```sql
SELECT campaign.name, search_term_view.search_term,
       segments.date, metrics.clicks, metrics.cost_micros, metrics.conversions
FROM search_term_view
WHERE segments.date DURING LAST_30_DAYS
  AND metrics.clicks > 0
ORDER BY metrics.cost_micros DESC
LIMIT 200
```

**Negatives:**
```sql
SELECT campaign.name, campaign_criterion.keyword.text,
       campaign_criterion.keyword.match_type
FROM campaign_criterion
WHERE campaign_criterion.negative = TRUE
  AND campaign_criterion.type = 'KEYWORD'
  AND campaign.status = 'ENABLED'
```

**Ad schedules:**
```sql
SELECT campaign.name, campaign_criterion.ad_schedule.day_of_week,
       campaign_criterion.ad_schedule.start_hour, campaign_criterion.ad_schedule.end_hour
FROM campaign_criterion
WHERE campaign_criterion.type = 'AD_SCHEDULE'
  AND campaign.status = 'ENABLED'
ORDER BY campaign.name, campaign_criterion.ad_schedule.day_of_week
```

**Geo targets:**
```sql
SELECT campaign.name,
       campaign_criterion.location.geo_target_constant,
       campaign_criterion.proximity.geo_point.latitude_in_micro_degrees,
       campaign_criterion.proximity.geo_point.longitude_in_micro_degrees,
       campaign_criterion.proximity.radius,
       campaign_criterion.proximity.radius_units,
       campaign_criterion.proximity.address.postal_code,
       campaign_criterion.negative
FROM campaign_criterion
WHERE campaign.status = 'ENABLED'
  AND campaign_criterion.type IN ('LOCATION', 'PROXIMITY')
```

**Conversion actions:**
```sql
SELECT conversion_action.id, conversion_action.name, conversion_action.status,
       conversion_action.category, conversion_action.type,
       conversion_action.primary_for_goal,
       conversion_action.attribution_model_settings.attribution_model,
       metrics.all_conversions, metrics.all_conversions_value
FROM conversion_action
WHERE conversion_action.status != 'REMOVED'
```

**Customer info:**
```sql
SELECT customer.id, customer.descriptive_name, customer.currency_code,
       customer.time_zone, customer.status, customer.test_account, customer.manager
FROM customer
```

**Billing setup:**
```sql
SELECT billing_setup.id, billing_setup.status,
       billing_setup.start_date_time, billing_setup.end_date_time,
       billing_setup.payments_account_info.payments_account_name,
       billing_setup.payments_account_info.payments_profile_id
FROM billing_setup
ORDER BY billing_setup.start_date_time DESC
```

**Ad approval state:**
```sql
SELECT ad_group.name, ad_group.status, ad_group_ad.status,
       ad_group_ad.policy_summary.approval_status,
       ad_group_ad.policy_summary.review_status
FROM ad_group_ad
WHERE ad_group_ad.status != 'REMOVED'
```

### 14.11 Anomaly detection rules

Compare current 7d to prior 7d per account:

| Signal | Threshold | Action |
|---|---|---|
| Spend Δ > +30% | flag | "Spend spike on {business}" |
| Spend Δ < -30% AND clicks Δ < -10% | flag | "Account starved — check billing/auction" |
| Avg CPC Δ > +50% | flag | "CPC spike" |
| Conversions Δ < -30% | flag | "Conversion drop" |
| Impression share < 30% top campaigns | flag | "Under-bidding or budget-capped" |
| Search term ≥ $8 cost, 0 conv in 7d | flag for review | "Wasted spend" recommendation |
| Search term matches `service_keywords_no` | flag for review | Pre-fill phrase-negative recommendation |

**Auto-applied (the "safe junk" tier, requires first-time confirmation per Section 11):**
```
jobs, career, salary, hiring, tutorial, youtube, how to, diy, do it yourself,
free, cheap, training, course, wholesale, manufacturer, wikipedia, reddit
```

Plus the per-business `service_keywords_no` list from `marketing_business_settings`.

**Human-approval required for everything else.**

### 14.12 Conversion upload pattern

**Trigger:** `upload_leadquik_conversion` Edge Function, invoked either:
- (preferred) by adding one call to the existing LeadQuik booking-creation code path, OR
- (fallback) by pg_cron polling `bookings` / `leads` / `appointments` every 5 min for rows with non-null `gclid` and no corresponding `marketing_conversion_uploads` row

**Flow:**

```ts
async function uploadConversion(input: {
  businessId: string,
  sourceTable: 'leads'|'bookings'|'appointments'|'calls',
  sourceRowId: string,
  conversionActionName: 'form_submit'|'qualified_lead'|'booked_appointment'
}) {
  // 1. Find the source row and its gclid (lives on the existing LeadQuik table, NOT duplicated in marketing_)
  const source = await fetchSourceRow(input.sourceTable, input.sourceRowId);
  if (!source.gclid) {
    await supabase.from('marketing_conversion_uploads').insert({
      business_id: input.businessId,
      source_table: input.sourceTable,
      source_row_id: input.sourceRowId,
      upload_status: 'skipped_no_gclid'
    });
    return;
  }

  // 2. Find the matching conversion action
  const action = await supabase
    .from('marketing_conversion_actions')
    .select('external_action_id, account_id, marketing_accounts!inner(external_id, manager_customer_id)')
    .eq('business_id', input.businessId)
    .eq('action_name', input.conversionActionName)
    .single();

  const customerId = action.marketing_accounts.external_id;
  const fullResource = `customers/${customerId}/conversionActions/${action.external_action_id}`;

  // 3. Format datetime in Ads-required format (space, not 'T', with timezone offset)
  const dt = formatAdsDatetime(source.occurred_at, source.timezone);

  // 4. POST to Ads API
  const accessToken = await getAdsAccessToken();
  const resp = await fetch(
    `https://googleads.googleapis.com/v23/customers/${customerId}:uploadClickConversions`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${accessToken}`,
        'developer-token': Deno.env.get('GOOGLE_ADS_DEVELOPER_TOKEN'),
        'login-customer-id': Deno.env.get('GOOGLE_ADS_LOGIN_CUSTOMER_ID'),
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        conversions: [{
          gclid: source.gclid,
          conversionAction: fullResource,
          conversionDateTime: dt,
          conversionValue: (source.value_cents || 0) / 100,
          currencyCode: 'USD'
        }],
        partialFailure: true
      })
    }
  );

  // 5. Persist result
  const result = await resp.json();
  const success = !result.partialFailureError;
  await supabase.from('marketing_conversion_uploads').insert({
    business_id: input.businessId,
    account_id: action.account_id,
    source_table: input.sourceTable,
    source_row_id: input.sourceRowId,
    conversion_action_id: action.id,
    gclid: source.gclid,
    occurred_at: source.occurred_at,
    uploaded_to_ads_at: success ? new Date().toISOString() : null,
    upload_status: success ? 'uploaded' : 'failed',
    upload_error: success ? null : JSON.stringify(result.partialFailureError),
    value_cents: source.value_cents
  });
}
```

**Why this matters:** every other Google Ads management tool requires CallRail ($45/mo) or Google's own forwarding numbers (which kill the operator's brand caller ID). Blaze uses the existing LeadQuik gclid capture and AI receptionist data — real-time, free, with full transcript context. **This is the unfair advantage.** Nobody else can build it without owning the receptionist layer.

### 14.13 Pagination + rate limits

- Ads `SearchStream` returns up to 10k rows per batch. Use `Search` + `pageToken` for larger.
- Basic Access: ~15k ops/day, 100 QPS peak. Pace large jobs.
- Business Profile: 60 reads/min, lower on writes. Batch.
- Anthropic API: respect rate limits. For weekly summaries × N businesses, sequence sequentially with 1s spacing.

### 14.14 Error handling

| Error | Meaning | Fix |
|---|---|---|
| `AUTHENTICATION_ERROR` | Refresh token expired/revoked | Re-OAuth, update secret |
| `AUTHORIZATION_ERROR` | Customer not under MCC | Re-link via `CustomerManagerLink` |
| `QUOTA_ERROR` | Daily ops limit hit | Pace ops, request Standard Access |
| `INVALID_ARGUMENT` on campaign create | Usually EU political flag missing | See 14.8.A-J |
| `PARTIAL_FAILURE` returned | Some ops failed | Read `partial_failure_error.message`; common = dup keywords |
| `RESOURCE_EXHAUSTED` | Account over budget cap | Verify budget, contact customer |
| 403 from Business Profile | Not approved or scope missing | Confirm API access + `business.manage` scope |

Always log failed mutations to `marketing_audit_log` with the error message.

### 14.15 Testing without burning ad spend

- Use Supabase preview branches with isolated `marketing_accounts`
- Stub the Ads API in preview env — return mock data
- For mutation testing: use the founding GA account's paused placeholder if any exists; OR create a $1/day paused test campaign. Never enable.
- **Never run mutations against external customer accounts during dev.** Always confirm the `business_id` belongs to a founding or test row.

---

## 15. Pre-build checklist

Before kicking off Phase A:

- [ ] Google Cloud project exists with all 5 APIs enabled
- [ ] OAuth Desktop credentials downloaded
- [ ] Ads developer token confirmed Basic Access (test on a real query)
- [ ] Business Profile API access application **submitted** (don't wait — 4-12 week clock)
- [ ] Supabase Secrets populated (Section 14.5)
- [ ] MCC `7990784116` confirmed linked to all 3 founding customer_ids
- [ ] `marketing_*` table migrations written and tested in a preview branch
- [ ] LeadQuik existing `businesses`, `calls`, `bookings`, `leads` tables identified
- [ ] Confirmed the actual LeadQuik tenant table name is `businesses` (or do find-replace to whatever it actually is)
- [ ] Operator has `david@hardwood-guys.com` LeadQuik user account ready
- [ ] Resend API key in Supabase Secrets, sender `notifications@leadquik.com` verified

When all boxes are checked, paste the Phase A prompt (Section 10) to Lovable.

---

*End of bootstrap spec.*

*Doc version 1.2 — 2026-05-25. Changes from v1.1:*
- *Schema FKs: `tenants(id)` → `businesses(id)`, `tenant_id` → `business_id` throughout*
- *Added "How to use this spec with Lovable" with phased paste-prompts*
- *`grader_scans` moved out of marketing_ namespace, service-role-only writes documented*
- *`marketing_conversions` refactored to `marketing_conversion_uploads` (Ads-upload tracking only); source data references existing LeadQuik tables*
- *Email path specified: Resend from `notifications@leadquik.com`. SendGrid is Inbound only*
- *Bid editing on keywords: dropped manual CPC editor as default; render tCPA editor for Smart Bidding, max-CPC only when bid strategy is Manual CPC*
- *`sync_ads_daily` dedup note for shared customer_id (GA + CT/NY)*
- *`auto_request_review` changed from DB trigger to cron polling (LeadQuik doesn't expose booking-created event)*
- *Conversion action ID format clarified (just the action ID; full resource constructed at call time)*
- *Anomaly detector: daily by default, hourly opt-in*
- *First-time auto-opt confirmation modal pattern added to Phase C and Section 11*
- *Feature flag + rollback plan added to Section 13*
- *GMB API realistic timeline (4-12 weeks) and graceful-empty-state pattern documented*
- *Each phase in Section 10 now has its own ✅ checklist*
