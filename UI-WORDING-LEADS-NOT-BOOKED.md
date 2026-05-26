# UI Wording Update — "Leads" not "Booked"

**Operator rule (2026-05-26)**: Every conversion is a **lead**, not a confirmed booked job. Some leads are tire-kickers. The "booked" wording overstates closure.

Apply this label change everywhere user-facing.

## In `src/components/marketing/GoalsStrip.tsx` (and Goals settings)

Replace the `goalLabel` formatter:

```ts
function goalLabel(goal: { goal_type: string; target_value: number; unit: string }): string {
  switch (goal.goal_type) {
    case 'monthly_ad_spend_max':   return `Monthly ad spend ≤ $${goal.target_value/100}`;
    case 'weekly_ad_spend_max':    return `Weekly ad spend ≤ $${goal.target_value/100}`;
    case 'cpa_max':                return `Cost per lead ≤ $${goal.target_value/100}`;       // was: "Cost per booked"
    case 'bookings_per_week_min':                                                            // legacy goal_type, keep working
    case 'leads_per_week_min':     return `≥ ${goal.target_value} leads/week`;               // was: "bookings/week"
    case 'bookings_per_month_min':
    case 'leads_per_month_min':    return `≥ ${goal.target_value} leads/month`;              // was: "bookings/month"
    case 'roas_min':               return `ROAS ≥ ${goal.target_value}x`;
    case 'gmb_rating_min':         return `GMB rating ≥ ${goal.target_value}★`;
    case 'gmb_response_rate_min':  return `Review response rate ≥ ${goal.target_value}%`;
    case 'seo_health_score_min':   return `SEO health ≥ ${goal.target_value}`;
    default: return goal.goal_type;
  }
}
```

**Note**: The DB `goal_type` values stay as-is (`cpa_max`, `bookings_per_week_min`, etc.). Only the display labels change. New goals created via the UI can use the new `leads_per_*` type names if you want — both will format the same.

## In `src/components/marketing/KpiCard.tsx` (and any dashboard KPI titles)

Anywhere you see KPI labels like:
- "Cost per booked appointment" → **"Cost per lead"**
- "Booked appointments" / "Bookings" → **"Leads"**

## Goals Settings page (`/marketing/settings/goals`)

The goal type dropdown options. Keep existing values for compat, update display:

```ts
const GOAL_TYPE_OPTIONS = [
  { value: 'monthly_ad_spend_max',  label: 'Monthly ad spend (max)' },
  { value: 'weekly_ad_spend_max',   label: 'Weekly ad spend (max)' },
  { value: 'cpa_max',               label: 'Cost per lead (max)' },         // was: "Cost per booked"
  { value: 'bookings_per_week_min', label: 'Leads per week (min)' },        // was: "Bookings per week"
  { value: 'bookings_per_month_min',label: 'Leads per month (min)' },       // was: "Bookings per month"
  { value: 'seo_health_score_min',  label: 'SEO health score (min)' },
  { value: 'gmb_rating_min',        label: 'GMB rating (min)' },
];
```

## Backend already updated (Claude deployed via Supabase MCP)

| Function | Version | Change |
|---|---|---|
| `generate_dashboard_briefing` | v2 | System prompt mandates "leads" wording. Tested — output now says "39 leads last week" not "39 booked jobs". |
| `generate_ai_summary` | v17 | Prompt template updated. Payload fields renamed `conversions→leads`, `cost_per_booked_cents→cost_per_lead_cents`. Templated fallback updated. |
| `check_goals_alert` | v2 | Alert email subject + body uses "Cost per lead" / "≥ N leads/week". |

## Effect when Lovable ships the UI label update

| Surface | Before | After |
|---|---|---|
| Dashboard KPI card | "Cost per booked" / "39 booked" | "Cost per lead" / "39 leads" |
| GoalsStrip bar | "≥ 10 bookings/week" | "≥ 10 leads/week" |
| Settings dropdown | "Bookings per week" | "Leads per week" |
| Morning briefing text | "You booked 39 jobs..." | "You had 39 leads..." |
| Weekly AI summary email | "...booked 39..." | "...39 leads..." |
| Goal alert email | "≥ 10 bookings/week — BREACHED" | "≥ 10 leads/week — BREACHED" |

## Why

The whole `marketing_daily_metrics.ads_conversions` value is what Google Ads counts as a conversion event — that's a LEAD (a phone call, form submit, or appointment request). Whether the operator closes that lead into a paid job is a separate measure that LeadQuik captures downstream. Calling every conversion "booked" misleads the operator. "Leads" is accurate.
