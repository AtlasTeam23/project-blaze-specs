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
7. Send email via existing Lovable Emails pipeline from `noreply@leadquik.com`. Subject: `🪵 Weekly Marketing Summary — {{business.name}} — {{date_range.start}}`

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
