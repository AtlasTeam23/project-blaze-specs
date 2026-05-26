# Phase B+ — Goals & Morning Briefing UI

**Backend is live (Claude has deployed everything).** This phase = UI only.

## What's already shipped on the backend

| Component | Endpoint / Resource |
|---|---|
| `marketing_goals` table | RLS via business_members. 3 default goals seeded for GA business `7fb42e54-...`. |
| `marketing_goal_evaluations` table | History of goal evals (for trend graphs later). |
| `generate_dashboard_briefing` Edge Function | `POST /functions/v1/generate_dashboard_briefing` body `{ business_id }` → returns `{ ok, briefing_id, text, model, goal_status[] }`. ~5s generation time. Stores in `marketing_ai_summaries` with `trigger='daily_briefing'`. |

## What you (Lovable) need to build

### 1. Morning Briefing Banner

**Component**: `src/components/marketing/MorningBriefingBanner.tsx`

**Where**: Top of `/marketing` dashboard, above the KPI cards row.

**Behavior**:
- On mount, query `marketing_ai_summaries` for the latest row where `business_id = activeBusinessId AND trigger = 'daily_briefing'`. Render `summary_markdown` as a single styled paragraph.
- If no briefing exists OR latest is >12 hours old, show a "Generating fresh briefing..." spinner and call `generate_dashboard_briefing` automatically.
- Manual refresh button (small icon, top-right of banner): triggers a fresh call to `generate_dashboard_briefing`, replaces card content.
- Show "Updated 2m ago" / "Updated 4h ago" relative timestamp.

**Visual**: Dark surface card with LeadQuik flame icon on the left, briefing text in body text size (slightly larger than dashboard body), timestamp + refresh icon top-right. Use the LeadQuik brand blue for the flame icon.

**Example render**:
```
🔥  Yesterday: $42 spent, 3 booked appointments at $14 CPA — under your $50 target.
    1 review needs reply on GMB. Pacing on track for May ($234 of $3,000 cap).
    [Updated 8m ago] [↻]
```

### 2. Goals — Dashboard Progress Bars

**Component**: `src/components/marketing/GoalsStrip.tsx`

**Where**: New row on `/marketing` dashboard between the briefing banner and the KPI cards (or alongside KPIs — designer's call).

**Behavior**:
- Query `marketing_goals` for the active business (where `enabled = true`).
- For each goal, render a horizontal progress bar with:
  - Goal label (human-readable, see formatter below)
  - Current value / target value
  - Progress bar fill (color-coded: green ≤alert_pct, amber between alert_pct and 100%, red >100% for max-type goals OR <alert_pct for min-type goals)
  - Period label ("this month" / "this week")
- Compute current values from `marketing_daily_metrics` (same logic as the briefing function — see `buildBriefingPayload` for reference).

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

**Route**: `/marketing/settings/goals` — new page

**Behavior**: List of goals with inline edit, plus "+ Add goal" button. For each goal:
- Goal type dropdown (the goal_type strings above)
- Target value input (number, dynamic suffix based on unit: "cents/$", "count", "★", etc.)
- Period dropdown: `daily | weekly | monthly`
- Alert threshold input: "Alert me at __% of target" (default 80)
- Channels checkboxes: email / SMS / push (only email for v1; gray out the others)
- Enabled toggle

CRUD via the `marketing_goals` table directly (RLS will handle access). After insert/update, immediately refresh the GoalsStrip on the dashboard.

### 4. Default goals — UX nicety

When a new business is added to Blaze, auto-seed these 3 defaults via the existing seeding flow:
- `monthly_ad_spend_max` = $1,000 (in cents = 100000)
- `cpa_max` = $75 (in cents = 7500)
- `bookings_per_week_min` = 5

This way new tenants see something useful in GoalsStrip from day 1.

### 5. Briefing in weekly emails (optional v1.5)

Not required for Phase B+. But conceptually: the weekly Monday summary email could open with the morning briefing text as a TL;DR, then the full 250-word executive summary below. Defer.

## ✅ Before you say done

- [ ] `MorningBriefingBanner` renders the latest `daily_briefing` row at top of `/marketing`
- [ ] If no briefing exists, the component auto-generates one on first load
- [ ] Manual refresh button triggers a new generation in ≤10 seconds, updates the banner
- [ ] `GoalsStrip` renders the 3 seeded GA goals with correct labels and progress bars
- [ ] Progress bars show: monthly_ad_spend_max = 8%, cpa_max = 12%, bookings_per_week_min = 390% (will read 100%+/green capped at full bar)
- [ ] `/marketing/settings/goals` page lists the goals, allows edit + add + delete + enable/disable toggle
- [ ] Adding/editing a goal persists to `marketing_goals` table (RLS enforced)
- [ ] Switching businesses (when other tenants are seeded) shows only that business's goals
- [ ] No regressions: existing dashboard, ads overview, reports pages still render

## Goal alert engine

**NOT required for Phase B+** — that's a Phase C edge function (`check_goals_alert`) that runs daily, evaluates each goal, writes to `marketing_goal_evaluations`, and sends Resend email when threshold crossed. Mention it in the Phase B+ ship report as "pending Phase C" and we move on.

## Backend reference for the curious

The briefing function aggregates these data points and passes them to Claude:
- Yesterday's spend / conversions / clicks
- Last 7d totals + cost per booked
- Month-to-date spend
- All enabled goals with their target/actual/pct values

If you want to replicate any of this UI-side (e.g., live goal progress without calling the edge function), the query pattern is in `supabase/functions/generate_dashboard_briefing/index.ts` `buildBriefingPayload`.
