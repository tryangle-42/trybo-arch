# Hero Use Cases — Testing Plan

Branch: `feat/hero-usecases` | 30 skills | 3 testers

---

## Setup (everyone)

Before starting, make sure these are in your `.env`:

```env
# Shared — everyone needs these
SUPABASE_URL=...
SUPABASE_SERVICE_ROLE_KEY=...
SUPABASE_DB_URI=...              # LangGraph checkpointer
PARALLEL_FANOUT_ENABLED=true
JOB_WORKER_ENABLED=true
JOB_WORKER_POLL_INTERVAL_S=30   # Short for testing
# LLM keys (for synthesis skills)
OPENAI_API_KEY=...               # or whichever LLM provider is configured
```

## Bug fix rules

- You own your skills. Fix and push directly to `feat/hero-usecases`.
- Shared infra bug (direct.py dispatch, seed_registry, SkillDataStore, SchedulerService, compiler_graph) → message the group before pushing.
- Commit format: `fix(commute): maps handler missing traffic_condition field`

---
---
---

# Avinash — Commute Optimizer + Morning Briefing

**Use cases:** UC1 (Commute), UC4 (Briefing)
**Skills you own (6):** maps.directions.google, maps.geocode.google, commute.optimize.departure, weather.forecast.openmeteo, briefing.synthesize, briefing.preference.save
**Files you own:** `skills_maps.py`, `skills_calendar.py`, maps/commute/weather/briefing handlers in `direct.py`

### Env vars you need

```env
GOOGLE_MAPS_API_KEY=...          # console.cloud.google.com → Routes API + Geocoding API
```

### Phase 1 — Dry run smoke tests

Call each skill with `dry_run=True` in ExecutionContext. Verify it returns `(outputs_dict, Receipt)` with `Receipt.status == "success"` and no exceptions.

| # | Skill | What to check |
|---|---|---|
| 1 | `maps.directions.google` | Returns mock duration_seconds, distance_meters |
| 2 | `maps.geocode.google` | Returns mock lat, lng, formatted_address |
| 3 | `commute.optimize.departure` | Returns mock slots array (~16 entries), best_departure, peak_window |
| 4 | `weather.forecast.openmeteo` | Returns mock current conditions + daily_summary |
| 5 | `briefing.synthesize` | Returns mock briefing_text + artifact (type=briefing_card) |
| 6 | `briefing.preference.save` | Returns mock preference_id + scheduled_job_id |

### Phase 2 — Live API tests (one call each)

| # | Skill | Test input | What to verify |
|---|---|---|---|
| 1 | `maps.directions.google` | origin="Noida Sector 62", destination="Hauz Khas Delhi", departure_time="now" | duration_seconds > 0, distance_meters > 0 |
| 2 | `maps.geocode.google` | address="Hauz Khas, New Delhi" | lat around 28.55, lng around 77.20 |
| 3 | `commute.optimize.departure` | origin="Noida Sector 62", destination="Hauz Khas Delhi", date=tomorrow, window_start="07:00", window_end="10:00", interval_minutes=30 | slots array with ~6 entries, best_departure populated |
| 4 | `weather.forecast.openmeteo` | latitude=28.55, longitude=77.20, forecast_days=1 | current temp present, daily_summary non-empty |
| 5 | `briefing.synthesize` | Pass real weather output from #4 + mock calendar_events array | briefing_text non-empty, artifact.artifact_type == "briefing_card" |
| 6 | `briefing.preference.save` | briefing_time="07:00", city="Delhi", sections=["weather","calendar"] | Row in skill_data_store (namespace=briefing), row in scheduled_jobs |

### Phase 3 — End-to-end graph tests

**Test 1 — Commute:**
```
Send: "I need to reach Hauz Khas Delhi from Noida by 9 AM tomorrow. When should I leave?"
Expect: GID → Planner → commute.optimize.departure → StructuredArtifact(commute_alert)
Check: best_departure, latest_safe_departure, slots comparison in response
```

**Test 2 — Briefing setup:**
```
Send: "Set up a daily morning briefing at 7 AM for Delhi"
Expect: GID → Planner → briefing.preference.save → confirmation
Check: skill_data_store has preference record, scheduled_jobs has daily job
```

**Test 3 — Briefing execution:**
```
Trigger the scheduled briefing manually (or wait for worker to fire it)
Expect: weather.forecast ∥ calendar → briefing.synthesize → StructuredArtifact(briefing_card)
Check: Briefing card has weather section, parallel execution timing
```

### Phase 4 — Scheduled job test

Set `briefing.preference.save` with briefing_time = 2 minutes from now. Wait for the job worker to pick it up and fire the briefing graph. Verify the full pipeline runs and produces a briefing_card.

---
---
---

# Rishav — Professional Intelligence Digest

**Use case:** UC5 (Digest)
**Skills you own (10):** content.feed.fetch, content.hn.top, content.arxiv.search, content.producthunt.today, content.reddit.top, digest.synthesize, digest.preference.save, knowledge.note.write.notion, tasks.issue.create.linear, tasks.issue.create.github
**Files you own:** `skills_feeds.py`, `direct_feeds.py`, marketplace skill definitions

### Env vars you need

```env
PRODUCT_HUNT_API_TOKEN=...       # Optional — falls back to Tavily search
# For marketplace writes (Notion/Linear/GitHub) — test last:
COMPOSIO_API_KEY=...
COMPOSIO_BASE_URL=...
COMPOSIO_USER_ID=...
COMPOSIO_CONNECTED_ACCOUNT_ID=...
```

### Phase 1 — Dry run smoke tests

Call each skill with `dry_run=True`. Verify `(outputs_dict, Receipt)` with no exceptions.

| # | Skill | What to check |
|---|---|---|
| 1 | `content.feed.fetch` | Returns mock entries array with title/link/published |
| 2 | `content.hn.top` | Returns mock stories array with title/url/points |
| 3 | `content.arxiv.search` | Returns mock papers array with title/abstract/url |
| 4 | `content.producthunt.today` | Returns mock launches array |
| 5 | `content.reddit.top` | Returns mock posts array with title/score/url |
| 6 | `digest.synthesize` | Returns mock sections + executive_summary + artifact (type=digest_card) |
| 7 | `digest.preference.save` | Returns mock preference_id + scheduled_job_id |
| 8-10 | Notion/Linear/GitHub | Skip dry_run — these are marketplace skills, test live in Phase 2 |

### Phase 2 — Live API tests (one call each)

| # | Skill | Test input | What to verify |
|---|---|---|---|
| 1 | `content.feed.fetch` | feed_urls=["https://hnrss.org/frontpage"], since_hours=24 | entries array non-empty, each has title + link |
| 2 | `content.hn.top` | since_hours=24, min_points=50, limit=5 | stories array ≤5 items, each has title + url + points |
| 3 | `content.arxiv.search` | categories=["cs.AI"], max_results=3 | papers array ≤3, each has title + abstract + url |
| 4 | `content.producthunt.today` | limit=3 | launches array non-empty (if PH token missing, verify Tavily fallback works) |
| 5 | `content.reddit.top` | subreddits=["MachineLearning"], limit_per_sub=3 | posts array non-empty, each has title + score |
| 6 | `digest.synthesize` | Feed real outputs from #1-5 + mock user_preferences={role:"AI Engineer", topics:["LLM"]} | sections.must_read array non-empty, artifact.artifact_type == "digest_card" |
| 7 | `digest.preference.save` | role="AI Engineer", topics=["LLM engineering"], digest_time="08:00" | Row in skill_data_store (namespace=digest), row in scheduled_jobs |
| 8 | `knowledge.note.write.notion` | Test only if Composio connected. Create a test page. | successful==true, page_url returned |
| 9 | `tasks.issue.create.linear` | Test only if Composio connected. Create test issue. | successful==true, issue_url returned |
| 10 | `tasks.issue.create.github` | Test only if Composio connected. Create test issue. | successful==true, issue_url returned |

### Phase 3 — End-to-end graph tests

**Test 1 — Digest setup:**
```
Send: "Set up a daily AI digest. I'm an AI engineer focused on LLM engineering and agent frameworks."
Expect: GID → Planner → digest.preference.save → confirmation
Check: skill_data_store has preference record, scheduled_jobs has daily job
```

**Test 2 — Digest execution:**
```
Trigger the digest manually (or via scheduled job)
Expect: content.hn.top ∥ content.arxiv.search ∥ content.feed.fetch ∥ content.reddit.top ∥ content.producthunt.today → digest.synthesize → StructuredArtifact(digest_card)
Check:
  - All 5 content fetchers run in parallel (check execution timing)
  - digest_card artifact has sections (must_read, research, launches, etc.)
  - product_ideas array present (may be empty if no relevant items)
```

**Test 3 — Digest with writes (only if Composio is set up):**
```
Same as Test 2 but with output_targets=["notion","linear"] in preferences
Expect: After digest synthesis → consent gates for Notion write + Linear issue creation
Check: Consent cards generated, approval flow works, items written after approval
```

### Phase 4 — Scheduled job test

Set `digest.preference.save` with digest_time = 2 minutes from now. Wait for worker to fire the full digest graph. Verify all 5 fetchers run, synthesis produces digest_card.

---
---
---

# Krishna — Trip Planner + Purchase Advisor

**Use cases:** UC2 (Trip), UC3 (Purchase)
**Skills you own (7):** booking.flights.search, booking.hotels.search, trip.itinerary.synthesize, trip.plan.save, web.scrape.structured, ecommerce.price.compare, user.financial.save
**Files you own:** `skills_booking.py`, `skills_ecommerce.py`, `booking_handlers.py`, `ecommerce_handlers.py`

### Env vars you need

```env
AMADEUS_API_KEY=...              # developers.amadeus.com (free test tier)
AMADEUS_API_SECRET=...
AMADEUS_ENV=test                 # Keep "test" for now
SERPAPI_API_KEY=...              # serpapi.com (free 100 searches/month)
TAVILY_API_KEY=...               # For web scraping + search fallbacks
```

### Phase 1 — Dry run smoke tests

Call each skill with `dry_run=True`. Verify `(outputs_dict, Receipt)` with no exceptions.

| # | Skill | What to check |
|---|---|---|
| 1 | `booking.flights.search` | Returns mock flights array with airline/price/times |
| 2 | `booking.hotels.search` | Returns mock hotels array with name/rating/price |
| 3 | `trip.itinerary.synthesize` | Returns mock itinerary + budget_breakdown + artifact (type=itinerary_card) |
| 4 | `trip.plan.save` | Returns mock plan_id |
| 5 | `web.scrape.structured` | Returns mock extracted_data object |
| 6 | `ecommerce.price.compare` | Returns mock comparisons array + recommendation + artifact (type=comparison_table) |
| 7 | `user.financial.save` | Returns mock saved_count |

### Phase 2 — Live API tests (one call each)

| # | Skill | Test input | What to verify |
|---|---|---|---|
| 1 | `booking.flights.search` | origin_iata="BLR", destination_iata="GOI", departure_date=(2 weeks out) | flights array non-empty. If Amadeus fails, verify Tavily fallback works |
| 2 | `booking.hotels.search` | location="Calangute, Goa", check_in/out=(2 weeks out, 3 nights) | hotels array non-empty. If Serpapi fails, verify Tavily fallback works |
| 3 | `trip.itinerary.synthesize` | Feed real outputs from #1 + #2 + mock weather data | itinerary array with 3 days, artifact.artifact_type == "itinerary_card" |
| 4 | `trip.plan.save` | Pass output from #3 + trip_dates | Row in skill_data_store (namespace=travel, record_type=trip_plan) |
| 5 | `web.scrape.structured` | url="https://www.amazon.in/dp/B0DFD5S398", extract_fields=["price","title","rating"] | extracted_data has price + title fields |
| 6 | `ecommerce.price.compare` | product_query="Samsung S25 Ultra 256GB", platforms=["amazon.in"], user_bank_cards=["HDFC"] | comparisons array ≥1 entry, recommendation present |
| 7 | `user.financial.save` | instruments=[{type:"credit_card", provider:"HDFC", network:"visa"}] | Row in skill_data_store (namespace=user_profile, record_type=financial_instruments) |

### Phase 3 — End-to-end graph tests

**Test 1 — Trip planner:**
```
Send: "Plan a 3-day Goa trip from Bangalore next month, budget 30K"
Expect: GID → Planner → booking.flights.search ∥ booking.hotels.search ∥ weather.forecast → trip.itinerary.synthesize → consent gate → trip.plan.save
Check:
  - Flights + hotels + weather run in parallel (check timing)
  - Itinerary card has 3 days with activities
  - Budget breakdown present
  - Consent gate appears before saving
  - Trip saved to skill_data_store after approval
```

**Test 2 — Purchase advisor:**
```
Send: "Find the best deal on Samsung S25 Ultra. I have an HDFC credit card."
Expect: GID → Planner → ecommerce.price.compare → StructuredArtifact(comparison_table)
Check:
  - Multiple platforms compared
  - HDFC bank offer layered into effective price
  - Recommendation with reasoning
  - comparison_table artifact generated
```

**Test 3 — Financial profile save:**
```
Send: "I have HDFC and ICICI credit cards"
Expect: GID → Planner → user.financial.save → confirmation
Check: skill_data_store has record, mem0 has semantic fact (if mem0 enabled)
```

---
---
---

# Sync Points

After each phase, quick sync (Slack message is fine):

| Phase | Sync |
|---|---|
| Phase 1 done | "All dry_runs pass" or "X skill failing, fixing" |
| Phase 2 done | "All live APIs working" or "X API returning errors, investigating" |
| Phase 3 done | "E2E flow works" or "Planner not generating correct plan for X" |
| Phase 4 done | "Scheduled jobs firing correctly" |

If you hit a bug in shared infrastructure, post in group with the error + which file before fixing.
