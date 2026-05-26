# Phase C — Execute Spec

**Phase C backend is fully deployed.** This spec is for the remaining UI work in Lovable.

## What's already live (Claude deployed via Supabase MCP)

| Component | Endpoint | Status |
|---|---|---|
| `apply_recommendation` v2 | `POST /functions/v1/apply_recommendation` | ✅ Real mutation engine |
| `check_goals_alert` v1 | `POST /functions/v1/check_goals_alert` | ✅ Goals evaluator + Resend alerts |
| pg_cron `check_goals_alert_daily` | 08:00 UTC daily (~04:00 ET) | ✅ Scheduled |

### Mutation engine — handlers per recommendation type

| Type | Implementation | Notes |
|---|---|---|
| `add_negative_keyword` | ✅ Full | Phrase/exact negative across all enabled Search campaigns. Returns count added. |
| `pause_keyword` | ✅ Full | Sets `ad_group_criterion.status = PAUSED`. Returns count paused. |
| `budget_adjustment` | ✅ Full + safety rail | Updates `campaign_budget.amount_micros`. **Rejects any change >50% in either direction.** |
| `expand_geo` | ⚠️ Stubbed | Returns "needs manual review of which geos." UI should surface the action in Settings → Geo. |
| `increase_keyword_focus` | ⚠️ Stubbed | Returns "needs Manual CPC switch or ad group split." Returns guidance message. |
| `reply_review` / `upload_photo` | ⚠️ GMB API pending | Returns "GMB API approval pending (case 7-0261000041237)." |
| `fix_seo_issue` | ⚠️ Stubbed | Returns "deep-link to CMS only." |

Every action writes a row to `marketing_audit_log` with `before_value` + `after_value` + `mutation_message` regardless of success.

### Goals engine

`check_goals_alert` evaluates these goal types automatically:
- `monthly_ad_spend_max` — current month spend
- `weekly_ad_spend_max` — last 7d spend
- `cpa_max` — last 7d cost-per-booked
- `bookings_per_week_min` — last 7d conversions
- `bookings_per_month_min` — month-to-date conversions
- `seo_health_score_min` — latest SEO scan
- `gmb_*` / `roas_min` — deferred (GMB API pending, no revenue data yet)

For each goal:
- Inserts a row in `marketing_goal_evaluations` (always, builds trend history)
- If `status = warning` (≥80% of target) or `breach` (target crossed): sends Resend email to business owner
- 24h dedup per `(goal_id, status)` so the same alert doesn't fire twice in a day

Tested end-to-end: 3 GA goals → all evaluated as `on_track` → no false alerts → 3 history rows persisted.

---

## UI work for Lovable

### 1. Audit Log page

**Route**: `/marketing/ads/audit`

**Query**:
```sql
SELECT applied_at, action, resource, before_value, after_value, applied_by
FROM marketing_audit_log
WHERE business_id = :activeBusinessId
ORDER BY applied_at DESC
LIMIT 200
```

**Layout**: vertical timeline-style list. Each entry card:

```
┌─────────────────────────────────────────────────────────────┐
│  [icon]  recommendation_apply              2 minutes ago    │
│          Increase monthly budget allocation by 20%          │
│                                                              │
│  Before: $100/day                                            │
│  After:  ⚠ budget_change_too_large: 3500% > 50% safety rail │
│                                                              │
│  Actor: user (David)                                         │
└─────────────────────────────────────────────────────────────┘
```

**Action types and icons**:

| `action` value | Icon | Color |
|---|---|---|
| `recommendation_apply` | ✅ check | emerald-500 (if mutation succeeded) / amber-500 (if rejected by safety rail) |
| `recommendation_dismiss` | ❌ x | slate-400 |
| `recommendation_snooze` | ⏸ pause | blue-500 |
| `recommendation_info` | ℹ info | slate-400 |
| `sync_ads_daily_error` | 🚨 alert | red-500 |
| (any other) | ⚙ gear | slate-400 |

**Filters at top**:
- Date range (last 7d / 30d / 90d / all)
- Action type dropdown
- Actor (user / system / auto-opt)
- Search box (matches `before_value->>title` and `resource`)

**Before/After display**:
- Parse `before_value` and `after_value` JSONB
- If `mutation_message` exists in `after_value`, show it prominently
- Show structured diff if before/after have comparable keys (e.g., `amount_cents: 10000 → 12000`)
- For dismissed/snoozed: just show the status change

### 2. Recommendation Apply UI polish (already shipped, minor tweaks)

The `RecommendationsQueue` is already calling `apply_recommendation` correctly. Two enhancements:

**A.** When Yes button returns success — show the actual mutation message in the toast, not the generic "applied" text:

```ts
// Old: "Marked applied — Phase C will execute"
// New: 
toast.success(response.mutation_status.message);
// e.g., "added 'laminate' as PHRASE negative to 3/3 campaigns"
// or:   "budget updated: $100.00 → $120.00"
```

**B.** When Yes button returns success=false BUT attempted=true (safety rail rejection, or stubbed type), show as **amber warning** not red error:

```ts
if (response.mutation_status.attempted && !response.mutation_status.succeeded) {
  toast.warning(response.mutation_status.message, {
    description: "Recommendation remains in queue — adjust or dismiss."
  });
}
```

This makes the safety rails feel like protective features, not bugs.

### 3. Goal alert email preview (optional)

Add a "Preview alert email" button on `/marketing/settings/goals` that triggers:
```
POST /functions/v1/check_goals_alert
body: { business_id, force_send: true }
```

`force_send: true` bypasses the 24h dedup so the operator can verify their alerts are wired correctly.

### 4. Apply via UI — what the user experiences after Phase C

| Recommendation | Click Yes → |
|---|---|
| "Add 'laminate' as negative keyword" | ✅ Actually adds it. Toast: "added 'laminate' as PHRASE negative to N campaigns" |
| "Increase monthly budget 20%" | ✅ Actually changes budget if ≤50% change. Otherwise: ⚠ "Safety rail rejected — change is too large. Adjust manually in Settings → Budgets." |
| "Pause keyword X" | ✅ Actually pauses. Toast: "paused 1 instance of 'X'" |
| "Increase bid 25% on Y" | ⚠ Stub message: "needs Manual CPC switch or ad group split — see Settings → Keywords" |
| "Expand geo to 40mi" | ⚠ Stub message: "needs manual review of which geos to add" |
| "Reply to review" | ⚠ "GMB API approval pending" |

## Acceptance checklist (✅ Before you say done)

- [ ] `/marketing/ads/audit` route exists, gated behind blaze_marketing_tab flag
- [ ] Renders most recent 200 audit log entries for the active business
- [ ] Each entry shows action icon + title + before/after diff + actor + relative timestamp
- [ ] Filters work: date range, action type, actor, search
- [ ] Empty state when no entries: "No changes yet. Blaze will log every Yes/No/Wait action you take."
- [ ] `RecommendationsQueue` Yes button now displays the actual `mutation_status.message` in the toast
- [ ] Safety-rail rejections show as amber warnings, not red errors
- [ ] Recommendations remain in the queue when `attempted: true, succeeded: false` (only fade out on `succeeded: true`)
- [ ] Optional: "Preview alert email" button on `/marketing/settings/goals`

## Test data already in production

Recent audit log entries you can render against immediately:

```sql
SELECT applied_at, action, before_value->>'title' AS title,
       after_value->>'mutation_message' AS msg, applied_by
FROM marketing_audit_log
WHERE business_id = '7fb42e54-e6f0-4867-a801-ace2f68ef989'
ORDER BY applied_at DESC LIMIT 10;
```

Sample row:
```
applied_at:  2026-05-26 20:30:37
action:      recommendation_apply
title:       Increase monthly budget allocation by 20% for campaign 23865868508
msg:         budget_change_too_large: 3500% > 50% safety rail. Adjust the recommendation or apply via Settings → Budgets directly.
applied_by:  recommendation:32af955a-...|actor:system
```

That's a real safety-rail-rejection event to render — perfect UX test case.

## What's deferred to Phase D (future, not Phase C)

- `expand_geo` real implementation (needs UX picker for geos)
- `increase_keyword_focus` real implementation (needs bid strategy switch or auto split-to-new-ad-group logic)
- All GMB mutations (waiting on API approval ~Jun 4-8)
- SEO automation (CMS-specific, won't auto-mutate)
- Goal-eval trend graphs (use `marketing_goal_evaluations` history; nice-to-have, not required)
