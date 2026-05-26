# Phase B+ — Goals, Morning Briefing & Recommendations Queue

**All backend is live (Claude has deployed everything).** This phase = UI only.

## What's already shipped on the backend

| Component | Endpoint / Resource |
|---|---|
| `marketing_goals` table | RLS via business_members. 3 default goals seeded for GA business `7fb42e54-...`. |
| `marketing_goal_evaluations` table | History of goal evals (for trend graphs later). |
| `marketing_recommendations` table | Already from Phase A. Has `snooze_until` column added. |
| `generate_dashboard_briefing` | `POST /functions/v1/generate_dashboard_briefing` body `{ business_id }` → `{ ok, text, goal_status[], briefing_id, model }`. ~5s. Stores in `marketing_ai_summaries` with `trigger='daily_briefing'`. |
| `generate_recommendations` | `POST /functions/v1/generate_recommendations` body `{ business_id, replace?: true }`. Analyzes ads data + goals, calls Claude, inserts 3-7 rows into `marketing_recommendations`. ~30s. `replace=true` marks all current pending rows as expired first. |
| `apply_recommendation` | `POST /functions/v1/apply_recommendation` body `{ recommendation_id, action: 'apply'\|'dismiss'\|'snooze'\|'info', snooze_hours?: 24 }`. Returns `{ ok, new_status, mutation_status, ... }`. **Note: `apply` only marks intent for Phase B+; actual Google Ads / GMB mutations execute in Phase C.** Writes to `marketing_audit_log` on every action. |

## What you (Lovable) need to build

### 1. Morning Briefing Banner

**Component**: `src/components/marketing/MorningBriefingBanner.tsx`

**Where**: Top of `/marketing` dashboard, above the KPI cards row.

**Behavior**:
- On mount, query `marketing_ai_summaries` for the latest row where `business_id = activeBusinessId AND trigger = 'daily_briefing'`. Render `summary_markdown` as a single styled paragraph.
- If no briefing exists OR latest is >12 hours old, auto-call `generate_dashboard_briefing` with `{ business_id }` and refresh.
- Manual refresh button (small ↻ icon, top-right): re-runs the generation, replaces card content. Show spinner while ~5s wait.
- Show "Updated 2m ago" / "Updated 4h ago" relative timestamp.

**Visual**: Dark surface card with LeadQuik flame icon left, briefing text body, ↻ + timestamp top-right. Brand blue flame.

**Example render**:
```
🔥  You booked 39 jobs last week at $6 each—nearly 4x your 10-job target.
    Zero spend yesterday means your ads are off; turn them back on or you'll
    have nothing in the pipeline by Friday.
    [Updated 8m ago] [↻]
```

### 2. Goals — Dashboard Progress Bars

**Component**: `src/components/marketing/GoalsStrip.tsx`

**Where**: New row on `/marketing` between briefing banner and KPI cards.

**Behavior**:
- Query `marketing_goals` for active business (where `enabled = true`).
- For each goal, render horizontal progress bar:
  - Goal label (formatter below)
  - Current value / target value
  - Progress bar fill — green ≤alert_pct, amber alert_pct → 100%, red >100% for max-type goals OR <alert_pct for min-type goals
  - Period label ("this month" / "this week")
- Compute current values from `marketing_daily_metrics` (same logic as the briefing function's `buildBriefingPayload`).

**Goal label formatter**:
```ts
function goalLabel(goal: { goal_type: string; target_value: number; unit: string }): string {
  switch (goal.goal_type) {
    case 'monthly_ad_spend_max':   return `Monthly ad spend ≤ $${goal.target_value/100}`;
    case 'weekly_ad_spend_max':    return `Weekly ad spend ≤ $${goal.target_value/100}`;
    case 'cpa_max':                return `Cost per booked ≤ $${goal.target_value/100}`;
    case 'bookings_per_week_min':  return `≥ ${goal.target_value} bookings/week`;
    case 'bookings_per_month_min': return `≥ ${goal.target_value} bookings/month`;
    case 'roas_min':               return `ROAS ≥ ${goal.target_value}x`;
    case 'gmb_rating_min':         return `GMB rating ≥ ${goal.target_value}★`;
    case 'gmb_response_rate_min':  return `Review response rate ≥ ${goal.target_value}%`;
    case 'seo_health_score_min':   return `SEO health ≥ ${goal.target_value}`;
    default: return goal.goal_type;
  }
}
```

### 3. Goals Settings page

**Route**: `/marketing/settings/goals`

**Behavior**: CRUD list with "+ Add goal" button. For each goal:
- Goal type dropdown (all `goal_type` strings)
- Target value input (number, dynamic suffix based on unit)
- Period dropdown: `daily | weekly | monthly`
- Alert threshold input: "Alert me at __% of target" (default 80)
- Channels checkboxes: email / SMS / push (only email enabled for v1; gray the others)
- Enabled toggle

Direct table CRUD via Supabase client. RLS handles access. Refresh `GoalsStrip` after save.

### 4. ⭐ Recommendations Queue (the marketing agent)

**Component**: `src/components/marketing/RecommendationsQueue.tsx`

**Where**: Bottom of `/marketing` dashboard, full-width card.

**Behavior**:
- Query `marketing_recommendations` for active business where `status = 'pending'` AND (`snooze_until` is null OR `snooze_until <= now()`), ordered by `priority DESC, surfaced_at DESC`.
- Show top 5 by default with "Show more" if N > 5.
- Each row renders as a card with:
  - Priority badge (color-coded: red ≥80, amber 50-79, blue <50)
  - Type icon (different per `type`: target = budget, magnify = keyword, location = geo, etc.)
  - Title (bold, 1 line)
  - Detail (2-3 sentences, body text)
  - **4 action buttons**: ✅ Yes  ❌ No  ⏸️ Wait  ℹ️ More info

**Action button wiring**:

| Button | API call | Result |
|---|---|---|
| ✅ Yes (Apply) | `POST /functions/v1/apply_recommendation { recommendation_id, action: 'apply' }` | Optimistic UI: fade out the row. Show toast: "Marked applied — actual Google Ads change ships in Phase C." On error revert. |
| ❌ No (Dismiss) | `POST .../apply_recommendation { recommendation_id, action: 'dismiss' }` | Fade out, no toast. |
| ⏸️ Wait (Snooze) | Open small dropdown: "1 hour / 4 hours / 1 day (default) / 3 days / 1 week". Then `POST .../apply_recommendation { recommendation_id, action: 'snooze', snooze_hours: N }`. | Fade out. Optionally show "Will resurface in 1 day" toast. |
| ℹ️ More info | `POST .../apply_recommendation { recommendation_id, action: 'info' }` | No state change. Returns `{ info: { type, payload, reasoning, ... } }`. Expand the card inline showing the `reasoning` text + a syntax-highlighted JSON view of the `payload` (small, monospace, for transparency). |

**Refresh button** (top-right of the queue card): "Generate fresh recommendations" — calls `POST /functions/v1/generate_recommendations { business_id, replace: true }`. Shows spinner for ~30s. Then re-queries the recommendations and renders.

**Empty state**: "🎯 No pending recommendations. Blaze is monitoring — check back tomorrow."

**Visual hierarchy**: Priority 80+ rows get a subtle red left-border; 50-79 amber; <50 default.

**Example card render**:
```
┌─────────────────────────────────────────────────────────────────┐
│ [85] 🎯 Increase bid 25% on 'engineered hardwood flooring prices'│
│                                                                   │
│  This keyword drives 41% of all conversions (16 of 39) at a       │
│  $5.87 CPA—well below your $50 target. With a 84% conversion     │
│  rate, there's clear demand being missed.                         │
│                                                                   │
│  [ ✅ Yes ]  [ ❌ No ]  [ ⏸️ Wait ▾ ]  [ ℹ️ More info ]            │
└─────────────────────────────────────────────────────────────────┘
```

### 5. Default goals — UX nicety

When a new business is added to Blaze, auto-seed these 3 defaults via the existing seeding flow:
- `monthly_ad_spend_max` = $1,000 (100000 cents)
- `cpa_max` = $75 (7500 cents)
- `bookings_per_week_min` = 5

## ✅ Before you say done

- [ ] `MorningBriefingBanner` renders latest `daily_briefing`; auto-generates if stale; manual refresh works
- [ ] `GoalsStrip` renders 3 seeded GA goals with progress bars; colors correct
- [ ] `/marketing/settings/goals` CRUD page works (add/edit/delete/enable goal)
- [ ] `RecommendationsQueue` renders the 6 currently-seeded recommendations for GA
- [ ] All 4 action buttons work: Yes / No / Wait (with hours dropdown) / More info (inline expand)
- [ ] Snooze hides recommendation until `snooze_until` passes
- [ ] Refresh button regenerates the queue (replaces existing pending with fresh set)
- [ ] Empty state renders when no recommendations
- [ ] Audit log shows entries for every Yes / No / Wait action (verify in `marketing_audit_log`)
- [ ] Switching businesses (when other tenants seeded) shows only that business's queue
- [ ] No regressions on existing pages

## Phase C handoff (NOT in Phase B+)

When Yes is clicked, `apply_recommendation` currently just marks status — it does NOT execute the Google Ads mutation yet. Phase C will add the actual mutation handlers:
- `add_negative_keyword` → call Ads API to add the phrase negative
- `pause_keyword` → toggle status on the ad group criterion
- `budget_adjustment` → update campaign_budget.amount_micros
- `increase_keyword_focus` → bid modifier or new ad group split
- `expand_geo` → add geo target criterion
- `reply_review` / `upload_photo` → GMB API (pending API approval)
- `fix_seo_issue` → deep-link to CMS (no auto-mutation possible)

For Phase B+, the UI should just show "Marked applied — actual Google Ads change ships in Phase C" in the toast and visually fade the row out. That's enough for testing the loop end-to-end.

## Backend test data — already in DB

Recommendations seeded RIGHT NOW for GA (6 rows pending). You can query:
```sql
SELECT id, priority, title, status, payload->>'reasoning' as reasoning
FROM marketing_recommendations
WHERE business_id = '7fb42e54-e6f0-4867-a801-ace2f68ef989'
ORDER BY priority DESC;
```

These give you real content to render against immediately.
