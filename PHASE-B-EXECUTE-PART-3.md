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
  - Sends email via Lovable Emails from `noreply@leadquik.com` to `david@hardwood-guys.com`
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
