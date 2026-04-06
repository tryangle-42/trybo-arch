# Use-Case-First Skill Strategy

## Approach

Instead of integrating dozens of MCPs generically, Trybo picks **one sharp, under-served personal problem** in each of 5 domains. For each, only the exact skills/MCPs required to solve that problem end-to-end get built. Every use case serves as a proof point that:

- The primitive-skill + orchestration architecture works in production.
- Trybo generates real user value that existing apps cannot match.
- Multi-tool orchestration is the moat — no single app solves these problems today.

**Selection criteria for use cases:**
- Must be a real, frequent personal phone problem (not enterprise, not niche).
- Must require 3+ tool/agent invocations (proves orchestration, not just a wrapper).
- Must be clearly under-served by existing apps (Google Maps, Swiggy, Paytm, etc.).
- Must be technically feasible with available APIs and MCPs.

---

## Platform Architecture Principle: No Feature-Specific Tables

A recurring pattern across all 5 use cases: **a skill produces data that should persist beyond a single graph execution and be queryable later.** Commute measurements, digest preferences, user profiles, trip plans — these are all the same abstract problem.

The current architecture already has:
- **mem0** → semantic memory (unstructured, fuzzy recall — "user prefers window seats")
- **Audit log** → observability (what happened during execution — not business data)
- **LangGraph checkpoints** → in-flight graph state (lost after graph completes)

What's missing is a **Skill Data Persistence Layer** — structured, queryable, per-user data that skills write to and read from. This must be **one common architectural module, not N tables per feature**. Creating `commute_measurements`, `digest_items`, `trip_plans`, `briefing_preferences` as separate tables is feature-developer thinking. It creates schema sprawl, requires a migration for every new use case, and fragments the data access layer.

### The Skill Data Store

One generic table. One service interface. Zero migrations for new use cases.

```sql
CREATE TABLE skill_data_store (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id),
  namespace TEXT NOT NULL,          -- skill/use-case identifier: "commute", "digest", "briefing"
  record_type TEXT NOT NULL,        -- data category: "measurement", "profile", "preference", "daily_digest"
  payload JSONB NOT NULL,           -- the actual data (schema varies by namespace+record_type)
  recorded_at TIMESTAMPTZ NOT NULL, -- when the data was observed/extracted (business time)
  created_at TIMESTAMPTZ DEFAULT now(),
  metadata JSONB                    -- optional context: source_email_id, route_config, etc.
);

-- Composite index covers all common query patterns
CREATE INDEX idx_skill_data_lookup
  ON skill_data_store(user_id, namespace, record_type, recorded_at);

-- GIN index for filtering within JSONB payload (e.g., day_of_week, vendor_name)
CREATE INDEX idx_skill_data_payload
  ON skill_data_store USING GIN (payload);
```

**How it works across use cases:**

| Use Case | namespace | record_type | payload example |
|---|---|---|---|
| UC1 Commute | `"commute"` | `"measurement"` | `{duration_sec: 3480, day_of_week: 1, time_slot: "07:15", distance_m: 28400}` |
| UC1 Commute | `"commute"` | `"profile"` | `{day_of_week: 1, time_slot: "07:15", avg_duration: 3720, stddev: 240, samples: 8}` |
| UC1 Commute | `"commute"` | `"route_config"` | `{origin: {...}, destination: {...}, collection_window: "06:30-10:30"}` |
| UC4 Briefing | `"briefing"` | `"preference"` | `{time: "07:00", sections: ["weather","calendar","email"], city: "Delhi"}` |
| UC5 Digest | `"digest"` | `"preference"` | `{role: "AI Engineer", topics: ["LLM engineering", "agent frameworks"], sources: ["hn", "arxiv", "rss"], digest_time: "08:00"}` |
| UC5 Digest | `"digest"` | `"daily_digest"` | `{date: "2026-03-06", items_count: 12, ideas_extracted: 2, written_to: ["obsidian", "linear"]}` |

**Aggregation is done in application code**, not SQL materialized views (which would require DDL per use case). For the commute use case, a scheduled Python background job reads `record_type: "measurement"` rows, computes AVG/STDDEV, and writes results back as `record_type: "profile"` — same table, different record_type.

**Service interface** (common module used by all skills):

```python
# orchestration/persistence/skill_data_store.py

class SkillDataStore:
    """Generic persistence layer for skill-produced data.
    Platform module — no skill-specific logic lives here."""

    async def write(self, user_id: str, namespace: str, record_type: str,
                    payload: dict, recorded_at: datetime, metadata: dict | None = None) -> str

    async def query(self, user_id: str, namespace: str, record_type: str,
                    since: datetime | None = None, until: datetime | None = None,
                    payload_filter: dict | None = None,  # JSONB containment filter
                    limit: int | None = None, order: str = "desc") -> list[dict]

    async def query_latest(self, user_id: str, namespace: str,
                           record_type: str) -> dict | None

    async def delete(self, user_id: str, namespace: str,
                     record_type: str | None = None, before: datetime | None = None) -> int

    async def count(self, user_id: str, namespace: str,
                    record_type: str, since: datetime | None = None) -> int
```

This sits alongside Registry, Executor, Consent, and Memory as a first-class module in the orchestration layer:

```
/orchestration
  /gid
  /planner
  /executor
  /registry
  /consent
  /memory          ← mem0 (semantic, unstructured)
  /persistence     ← Skill Data Store (structured, queryable) ← NEW
  /scheduler       ← Scheduled jobs ← NEW
  /adapters
  /models
  /audit
```

---

## Use Case 1: Smart Commute Optimizer

**Domain:** Maps / Transportation
**Priority:** P0 — First pilot
**Trigger phrase:** *"I need to reach Hauz Khas by 9 AM tomorrow. When should I leave from Noida?"*

### The Problem

Google Maps' "arrive by" / "depart at" feature returns a useless range like **55 min – 1 hr 40 min**. This is not a prediction — it's the entire theoretical range: 55 min is the empty-road best case (e.g., 5 AM), 1h40m is worst-case peak congestion. A ~100% variance tells a user nothing actionable.

No existing app answers: *"What's the best time to leave for this route?"* — a user would have to manually check Google Maps at 16 different departure times and compare the results. That's exactly what Trybo automates.

### Why It's Valuable

- Commute optimization is a **daily pain point** for millions of urban users.
- No existing app solves departure-time optimization as a first-class feature.
- High emotional stakes: missing a flight, being late to an interview, arriving late to pick up a child.
- Even with imperfect estimates, the **relative comparison** across time slots is actionable — "7 AM is 30 min faster than 8 AM" is useful even if the absolute numbers are off by 10 min.

### How Trybo Solves It — Three Stages

#### Stage 1: Manual Automation (current — shipping now)

The simplest approach: automate what a user would do manually — check Google Maps at many different departure times and compare.

```
User: "I need to reach Hauz Khas Delhi from Noida by 9 AM tomorrow. When should I leave?"

Agent execution:
  1. Geocode origin + destination
  2. Generate departure timestamps across a window (e.g., every 15 min, 6:00 AM – 10:00 AM)
  3. Fire ~16 Google Maps Directions API queries, one per departure time
     (Google accepts future departure_time and returns estimated duration
      based on historical traffic models)
  4. Collect all results:
     6:00 AM → 42 min (arrive 6:42)
     6:15 AM → 44 min (arrive 6:59)
     6:30 AM → 48 min (arrive 7:18)
     6:45 AM → 53 min (arrive 7:38)
     7:00 AM → 58 min (arrive 7:58)
     7:15 AM → 65 min (arrive 8:20)
     7:30 AM → 72 min (arrive 8:42)    ← latest safe departure for 9 AM arrival
     7:45 AM → 78 min (arrive 9:03)    ← too late
     8:00 AM → 85 min (arrive 9:25)
     8:30 AM → 82 min (arrive 9:52)
     9:00 AM → 68 min (arrive 10:08)
     9:30 AM → 52 min (arrive 10:22)
  5. Return:
     "Best departure: 6:00 AM (42 min, arrive 6:42 AM — fastest)
      Latest safe departure for 9 AM: 7:30 AM (72 min, arrive ~8:42 AM)
      Peak traffic: 7:45–8:30 AM (78-85 min — avoid if possible)

      ⚠ These are estimates based on Google Maps historical traffic models.
      Actual travel times may vary."
```

**Key tradeoff:** Google's future-departure-time estimates are less accurate than real-time "leaving now" queries. The absolute numbers may be off by 10-15 minutes. But the **relative ranking across slots is reliable** — if Maps says 7 AM is 58 min and 8 AM is 85 min, leaving at 7 AM is genuinely faster even if the actual times are 50 min and 75 min. This is explicitly accepted as imperfect but highly useful.

**What it does NOT do:** No background data collection, no scheduled cron jobs, no historical profiles, no statistical modeling. It's a one-shot query — stateless, instant, no setup required.

#### Stage 2: Historical Data Collection (future)

For users with regular commute routes, collect real-time `departure_time=now` measurements over 2-3 weeks via scheduled cron jobs. Build per-day-of-week, per-timeslot profiles with tight variance (±3-7 min vs Google's 45-min spread). This gives accurate predictions for recurring routes and far-in-future queries where Stage 1 estimates are too broad.

Requires: Scheduled job executor (already built), SkillDataStore measurements + profiles, nightly aggregation job.

#### Stage 3: Statistical Modeling (future)

Advanced modeling for standalone queries (no historical data available). Combine Google's estimates with weather forecasts, event calendars, and time-of-day patterns to produce tighter predictions even for one-off trips.

### API Cost Analysis

```
Stage 1 — Per query:
  ~16 Google Maps Directions API calls per optimization request
  At $10/1000 requests (Routes API with traffic): ~$0.16 per query
  Users might run 1-2 queries per day: ~$0.32/day per active user

  For 100 active users: ~$32/day = ~$960/month
  For 1,000 active users: ~$9,600/month

  Free tier: $200/month credit on new accounts = ~1,250 free queries
  (enough for ~78 optimization requests or ~39 daily users)
```

### Architecture Additions Required

| Addition | Why | Scope |
|---|---|---|
| **Google Maps Directions skill** | Core of Stage 1 — must accept both `departure_time=now` and future timestamps. Returns duration, distance, route summary. | Direct adapter skill: `maps.directions.google`. Uses Google Maps Routes API via httpx. |
| **Google Maps Geocoding skill** | Resolve addresses to lat/lng before querying directions. | Direct adapter skill: `maps.geocode.google`. Same API key. |
| **Commute optimizer skill** | The one-shot optimization loop — generates timestamps across a window, calls directions for each, compares and recommends. | Direct adapter skill: `commute.optimize.departure`. Calls `maps.directions.google` internally in a loop. Produces StructuredArtifact `"commute_alert"`. |
| **Location service** | "from here" or "from home" requires resolving device location or stored home/work addresses. | Mobile: `expo-location` → coordinates in request context. Stored locations in `skill_data_store` with `namespace: "user_profile"`. |

**Deferred to Stage 2:** Scheduled job executor for data collection crons, SkillDataStore measurement/profile records, nightly profile computation, proactive push notifications, anomaly detection.

### Skills & Integrations Required

| Skill | Adapter Type | Implementation | Side Effect | Notes |
|---|---|---|---|---|
| **Google Maps Directions** | Direct | Direct HTTP call to Google Maps Routes API. Accepts both `departure_time=now` (real-time traffic) and future timestamps (historical traffic models). | No | ~$10/1000 requests with traffic. Core skill for both Stage 1 optimization and future Stage 2 data collection. |
| **Google Maps Geocoding** | Direct | Geocoding API to resolve addresses to lat/lng. One-time per query, not per loop iteration. | No | Bundled with Maps API key. |
| **Commute Optimizer** | Direct | Loops over a time window calling Maps Directions for each slot. Compares durations, identifies best/latest-safe departure, peak window. Returns structured comparison + artifact. | No | ~16 API calls per invocation. Parallelized via asyncio.gather. |
| **Weather Forecast** | Direct | Open-Meteo API (free) — optional context for predictions ("rain forecast → expect delays"). | No | Optional enhancement. Shared with UC2 and UC4. |
| **Location Service** | Direct (mobile) | React Native `expo-location` → coordinates in API request body. | No | Not a backend skill — mobile client provides this. |
| **LLM Analysis** | Direct | Generate human-readable recommendations from comparison data. Lightweight — mostly template-based with LLM for natural language polish. | No | Existing `text.summarize.llm` or inline call. |

---

## Use Case 2: End-to-End Trip Planner

**Domain:** Travel / Itinerary
**Priority:** P1
**Trigger phrase:** *"Plan a 4-day Goa trip next month — flights from Bangalore, beachside hotel, budget 40K"*

### The Problem

Trip planning today is a multi-app juggling act: Google Flights for flights, Booking.com for hotels, Google Maps for distances, weather apps for forecasts, and a notes app to stitch it all together. Each app optimizes its own vertical but none synthesizes across them. Users spend hours switching between 4-5 apps, manually cross-referencing availability, pricing, location proximity, and weather — then building the itinerary in their head or a spreadsheet.

### Why It's Valuable

- Trip planning is **high-effort, high-frequency** — most people plan 2-6 trips per year and each takes hours of research.
- The synthesis step (combining flights + hotels + weather + logistics into a coherent plan) is exactly what LLM orchestration excels at.
- No existing app does cross-vertical trip synthesis. MakeMyTrip/Goibibo sell flights and hotels but don't factor in weather windows, local commute times, or day-wise scheduling.
- High willingness to pay — travel is a domain where users value convenience.

### How Trybo Solves It

```
User: "Plan a 4-day Goa trip, flying from Bangalore, last week of March.
       Beachside hotel, budget around 40K total."

Agent execution:
  1. Weather check: Query 7-day forecast for Goa for each candidate date range
     → Identify best weather windows (avoid rain days, prefer clear skies)
     → Recommend: "March 24-27 has the best weather (clear, 31°C, low humidity)"

  2. Flight search: Query flight APIs for BLR → GOI round-trip
     → Filter by date range, sort by price + duration
     → Return top 3 options with prices, times, airlines

  3. Hotel search: Query hotel APIs for Goa beachside properties
     → Filter by dates, budget (total minus flight cost), location, rating
     → Return top 3 options with prices, ratings, distance-to-beach

  4. Logistics: Query Google Maps for:
     → Airport (GOI) → each candidate hotel (distance + travel time)
     → Hotel → key attractions (beaches, forts, markets)

  5. Synthesis: LLM builds a day-wise itinerary:
     Day 1: Arrive 11:30 AM (IndiGo 6E-234, ₹4,200) → 35 min to hotel → check-in → Calangute Beach (10 min walk) → dinner at Britto's
     Day 2: North Goa circuit — Fort Aguada (20 min drive) → Anjuna Flea Market → Vagator Beach sunset
     Day 3: South Goa — Old Goa churches (45 min) → Palolem Beach (1h30m)
     Day 4: Check-out → airport (40 min) → 4:15 PM flight home

     Total estimate: ₹8,400 flights + ₹18,000 hotel (4 nights) + ₹8,000 food/transport = ₹34,400

  6. Consent gate: "Here's your trip plan. Approve to save, or edit any section."
```

### Architecture Additions Required

| Addition | Why | Scope |
|---|---|---|
| **Parallel fan-out in executor** | Steps 2, 3, 4 above are independent and should run concurrently. The current executor runs nodes sequentially even when no dependency exists. This use case makes parallelism a requirement — sequential execution would take 30-40s vs 8-10s parallel. | Fix in `plan_builder.py` — use LangGraph fan-out |
| **Structured Artifact System (mobile)** | An itinerary needs structured cards: flight card, hotel card, day-wise timeline. Plain chat text is unusable. | Platform module: generic artifact protocol (`artifact_type` + JSONB payload). Mobile renderer registry maps type → component. See Architecture Additions Summary. |
| **Partial Re-planning (Planner)** | User may want to swap a hotel or shift dates mid-plan without starting from scratch. This is a generic Planner capability — any multi-step plan could need mid-flight modification (change commute destination, pivot purchase comparison, refine briefing sections). | Platform enhancement to Planner: accept a completed PlanSpec + modification request → preserve already-completed node results → re-plan only affected nodes + their downstream dependents → resume graph execution from the modification point. |

### Skills & Integrations Required

| Skill | Adapter Type | Implementation | Side Effect | Notes |
|---|---|---|---|---|
| **Flight Search** | Direct | Amadeus Self-Service API (free tier: 500 calls/month) or Duffel API (pay-per-booking). Python SDK available for both. Alternatively, Tavily/web search as a fallback (scrape Google Flights results). | No | Amadeus is the most reliable programmatic flight search. Free test tier for development. |
| **Hotel Search** | Direct or MCP | Booking.com Affiliate API (requires affiliate account) or Tripadvisor Content API. Alternatively, Tavily web search + LLM extraction as fallback. | No | Booking.com API is invite-only but well-documented. Tripadvisor is more accessible. Serpapi hotel search works well as a fallback. |
| **Weather Forecast** | MCP | Open-Meteo API via `open-meteo/open-meteo-mcp` — fully free, no API key, 16-day forecast, global coverage. Alternatively, direct HTTP (Open-Meteo has a clean REST API). | No | Free. No API key. 16-day forecast horizon is sufficient for trip planning. |
| **Google Maps Directions** | MCP or Direct | Same as Use Case 1. Used here for airport↔hotel and hotel↔attraction distances. | No | Already needed for Use Case 1; shared skill. |
| **Google Maps Places** | MCP or Direct | Places API for attraction search, ratings, opening hours. Part of same Google Maps MCP server. | No | ~$17/1000 requests (Place Details). |
| **LLM Synthesis** | Direct | Planner or dedicated agent node — takes all structured data and generates human-readable day-wise itinerary. | No | Core LLM call. Uses Claude/GPT for synthesis. |
| **Web Search** | MCP (Tavily) | Fallback for flights/hotels if dedicated APIs are unavailable. Also useful for "things to do in Goa" type queries. | No | Already in v1 skill set. |

---

## Use Case 3: Smart Purchase Advisor

**Domain:** Shopping / E-commerce
**Priority:** P1
**Trigger phrase:** *"Find the best deal on a Samsung S25 Ultra — check Amazon, Flipkart, and Croma. Include bank offers."*

### The Problem

Buying a high-value product (phone, laptop, appliance) in India requires checking 3-5 platforms (Amazon, Flipkart, Croma, Reliance Digital, brand stores), comparing base prices, then layering on platform-specific discounts: bank card cashback (HDFC 10% off on Amazon, ICICI 5% on Flipkart), exchange offers, no-cost EMI terms, coupon codes, and bundled accessories. A single product can have 10+ effective price points across platforms. Users spend 30-60 minutes on this comparison manually — and still miss deals.

### Why It's Valuable

- High-value purchase decisions are **high-stakes, high-effort** — the price difference between platforms can be ₹3,000-8,000 on a phone.
- No existing app does cross-platform comparison with bank-offer layering. PriceHistory and BuyHatke track prices but don't factor in personalized bank offers.
- This is a **trust-building use case** — saving a user ₹5,000 on a phone purchase creates strong retention.
- India-specific complexity: Flipkart vs Amazon pricing wars, bank partnerships, exchange bonuses, and seasonal sales (Big Billion Days, Great Indian Festival) make this problem harder and more valuable to solve.

### How Trybo Solves It

```
User: "Best deal on Samsung S25 Ultra 256GB? I have HDFC and ICICI cards."

Agent execution:
  1. Search product across platforms:
     → Amazon.in: ₹1,29,999 (MRP) → HDFC cashback ₹10,000 → effective ₹1,19,999
     → Flipkart: ₹1,27,999 → ICICI discount ₹7,500 + ₹3,000 exchange bonus → effective ₹1,17,499
     → Croma: ₹1,29,999 → HDFC EMI cashback ₹5,000 → effective ₹1,24,999
     → Samsung.in: ₹1,29,999 → Samsung Finance+ ₹8,000 cashback → effective ₹1,21,999

  2. Enrich with context:
     → Price history: "Amazon price was ₹1,24,999 last week — current price is inflated"
     → Delivery: Amazon Prime 1-day, Flipkart 2-day, Croma store pickup available
     → Reviews summary: "4.3★ across 12K reviews. Common complaints: weight, heating"
     → Upcoming sale: "Flipkart Big Saving Days starts in 5 days — historically 8-12% drop"

  3. Recommendation:
     "Best deal right now: Flipkart at ₹1,17,499 (with ICICI + exchange).
      But consider waiting 5 days for Flipkart sale — could drop to ~₹1,12,000.
      If you need it today: Flipkart is ₹2,500 cheaper than the next best option."
```

### Architecture Additions Required

| Addition | Why | Scope |
|---|---|---|
| **Web scraping / automation skill** | Product pages on Amazon/Flipkart don't have public APIs for pricing. Need Playwright MCP or Firecrawl to scrape product pages and extract structured price + offer data. | New skill: `web.scrape.structured` via Playwright MCP or Firecrawl. Generic skill — usable by any future use case that needs web data extraction. |
| **User profile data** | "I have HDFC and ICICI cards" should be remembered across sessions. Two storage paths depending on data nature: (a) mem0 for semantic facts the LLM planner needs at planning time ("user has HDFC card" → planner knows to check HDFC offers), (b) `skill_data_store` with `namespace: "user_profile", record_type: "financial_instruments"` for structured lookups by skills at execution time. | Uses existing platform modules — mem0 (already in progress) + Skill Data Store. No new infrastructure. |

### Skills & Integrations Required

| Skill | Adapter Type | Implementation | Side Effect | Notes |
|---|---|---|---|---|
| **Web Search** | MCP (Tavily) | Search for product across platforms. Tavily's `search_depth: "advanced"` returns structured content from product pages. | No | Already in v1 skill set. Primary discovery mechanism. |
| **Web Scraping** | MCP | Playwright MCP (`anthropics/mcp-playwright` or `nicholasoxford/playwright-mcp`) for dynamic pages, or Firecrawl MCP (`nicholasgriffintn/firecrawl-mcp`) for simpler extraction. | No | Needed for extracting structured price/offer data from Amazon/Flipkart product pages where Tavily results aren't detailed enough. |
| **Price History** | Direct | PriceHistory.in API or BuyHatke API (unofficial). Alternatively, scrape price-tracking sites. | No | Optional but high-value. "This is ₹3K above last month's price" changes purchase decisions. |
| **LLM Analysis** | Direct | Agent node that takes raw price data + bank offers + user's cards → computes effective prices → generates recommendation. | No | Core analysis + synthesis step. |
| **mem0 Memory** | Direct | Retrieve user's stored financial instruments (bank cards, wallets) for personalized offer matching. | No | Already planned in architecture. |

---

## Use Case 4: Personalized Morning Briefing

**Domain:** Productivity / Daily Intelligence
**Priority:** P1
**Trigger phrase:** *"Good morning"* (auto-triggered) or *"What do I need to know today?"*

### The Problem

Starting the day means opening 5-7 apps in sequence: Weather (should I carry an umbrella?), Google Maps (how's traffic?), Google Calendar (what meetings do I have?), Gmail (anything urgent overnight?), a news app (what happened?), and maybe a trading app (how's the portfolio?). Each app shows its own silo of information. There's no unified "here's your day" view that synthesizes across sources and highlights what actually matters.

### Why It's Valuable

- **Universal daily habit** — every smartphone user does some version of this morning routine.
- The synthesis is the value: "You have a 9 AM meeting at Indiranagar — leave by 8:15 due to rain-related traffic. Also, your Amazon delivery arrives today and your HDFC credit card bill is due."
- This is the **"Jarvis" use case** — the most intuitive demonstration of what a personal agent OS can do. It shows multi-tool orchestration in a single, daily interaction.
- High retention driver: once a user relies on this, switching cost is extremely high.

### How Trybo Solves It

```
Trigger: Daily at user's preferred time (e.g., 7:00 AM), or on "Good morning"

Agent execution (parallel fan-out):
  Branch 1 — Weather:
    → Query weather for user's city
    → "29°C, partly cloudy. Rain expected after 4 PM — carry an umbrella."

  Branch 2 — Calendar:
    → Fetch today's events from Google Calendar
    → "3 meetings: 9 AM standup (Zoom), 11:30 AM lunch with Rahul (Indiranagar),
       3 PM dentist (Koramangala)"

  Branch 3 — Commute:
    → For each calendar event with a location: query Maps for travel time from home
    → "Leave by 8:15 for your 9 AM meeting (traffic heavier due to rain).
       Indiranagar lunch: leave office by 11:00 (25 min drive)."

  Branch 4 — Email digest:
    → Search Gmail for unread high-priority emails since last evening
    → "3 important emails: AWS invoice ($142), flight confirmation (DEL→BLR Mar 5),
       message from boss RE: Q1 review deck"

  Branch 5 — Portfolio snapshot (optional, if Groww connected):
    → Fetch portfolio summary
    → "Portfolio: ₹4,12,000 (+1.2% today). NIFTY 50: 22,430 (+0.3%)"

  Synthesis:
    Combine all branches into a single briefing card:
    ┌──────────────────────────────────────┐
    │ ☀ Good Morning — Thu, Mar 6         │
    │                                      │
    │ 🌤 29°C, rain after 4 PM            │
    │ 📅 3 events today                    │
    │ 🚗 Leave by 8:15 for 9 AM standup  │
    │ 📧 3 important emails                │
    │ 📈 Portfolio +1.2%                   │
    └──────────────────────────────────────┘
```

### Architecture Additions Required

| Addition | Why | Scope |
|---|---|---|
| **Scheduled job executor** | Morning briefing must fire automatically at a user-configured time without a chat trigger. Same scheduler infrastructure as Use Case 1. | Shared with Use Case 1: `orchestration/scheduler/` |
| **Proactive push notifications** | The briefing result must be pushed to the user's phone. Same push infra as Use Case 1. | Shared with Use Case 1: `notification.push.send` skill |
| **Parallel fan-out in executor** | All 5 branches are independent and must execute concurrently — sequential execution would take 15-20s vs 3-5s parallel. Same fix as Use Case 2. | Shared with Use Case 2: LangGraph fan-out fix |
| **User preference config** | User must configure: briefing time, which sections to include, which email labels to scan, whether to include portfolio. Structured config data — not fuzzy semantic memory. | Uses Skill Data Store: `namespace: "briefing", record_type: "preference"`. No new table. Same platform module as UC1/UC5. |

### Skills & Integrations Required

| Skill | Adapter Type | Implementation | Side Effect | Notes |
|---|---|---|---|---|
| **Weather** | MCP or Direct | Open-Meteo API (free, no key) — current conditions + today's hourly forecast. | No | Shared with Use Case 2. |
| **Google Calendar** | MCP | Official Google Calendar MCP or `anthropics/google-workspace-mcp` (covers Calendar + Gmail + Drive). OAuth 2.0 per user required. | No (read) | P1 in current v1 skill list. |
| **Gmail Search** | MCP | Same Google Workspace MCP server. Search unread since last briefing, filter by priority. | No (read) | P1 in current v1 skill list. |
| **Google Maps Directions** | MCP or Direct | Same as Use Case 1. Compute travel time from home to each calendar event location. | No | Shared with Use Case 1. |
| **Groww Portfolio** | Direct | Groww API — `get_holdings` (read-only). Per-user OAuth. | No (read) | Optional section. Already planned in `3-skills.md`. |
| **Push Notification** | Direct | FCM/APNs. Same as Use Case 1. | Yes (low risk) | Shared with Use Case 1. |
| **LLM Synthesis** | Direct | Combine all branch results into a concise briefing. Prioritize: flag urgent items, suppress noise. | No | Core LLM call. |
| **Location Service** | Direct (mobile) | User's home/work locations stored in profile for commute calculations. | No | Configured once, reused daily. |

---

## Use Case 5: Professional Intelligence Digest

**Domain:** Professional Development / Knowledge Management
**Priority:** P1
**Trigger phrase:** *"What's new in AI today?"* (on-demand) or auto-triggered daily at configured time

### The Problem

Staying current in fast-moving fields (AI/ML, startups, developer tools) requires monitoring 10+ sources daily: Hacker News, ArXiv, Twitter/X, tech newsletters (The Batch, Import AI, TLDR AI, Ben's Bites), Product Hunt, Reddit, and company blogs. Most professionals either spend 45-60 minutes context-switching across these sources, subscribe to too many newsletters they never read, or simply miss important developments. Worse, even when they do find something relevant, the insight evaporates — there's no structured capture into their knowledge system (Obsidian, Notion) or product backlog (Linear, GitHub Issues).

### Why It's Valuable

- **Daily professional need** — every knowledge worker in tech does some version of "morning news scan." The problem is universal, high-frequency, and deeply personal to the user's role.
- **Role-aware personalization is the moat**: An AI engineer needs ArXiv papers and model release announcements. A PM needs product launches and competitor moves. A founder needs funding news and market shifts. No existing news aggregator personalizes by professional role and active projects — they personalize by broad topic at best.
- **The "product idea extractor" is the killer differentiator**: Using the user's product context from mem0 ("building a personal assistant app on FastAPI"), the digest doesn't just summarize news — it flags items relevant to the user's work and suggests actionable product ideas. This turns passive reading into active ideation. No newsletter does this.
- **Write side-effects make it actionable**: Instead of "interesting, I'll remember that" (they won't), Trybo writes extracted insights directly to Obsidian/Notion and product ideas to Linear/GitHub Issues — with consent. This closes the loop from discovery to capture.
- **Builds a proprietary data asset**: Over time, the digest history + relevance signals become a personalized interest graph per user. mem0 learns what the user engages with, improving future ranking. This data cannot be replicated by any general-purpose news app.

### Why Not Twitter/LinkedIn Scraping?

Twitter and LinkedIn are tempting sources but impractical as primary channels:
- **Twitter/X API**: $100/month minimum for Basic tier (read-only). Free tier is write-only. Rate limits are aggressive. Content quality is highly variable — most tweets are noise.
- **LinkedIn**: No public API for feed access. Scraping violates ToS and requires auth cookies that expire. Professional insights are better sourced from the publications and repos that LinkedIn users link to anyway.

**Better approach:** Source the same insights from where they originate — HN, ArXiv, RSS newsletters, Product Hunt, Reddit — which have free, reliable APIs. Twitter/LinkedIn can be added later via Tavily web search as a supplementary signal if specific accounts are high-value.

### How Trybo Solves It

```
Trigger: Daily at configured time (e.g., 8:00 AM) or on "What's new in AI?"

Setup (one-time, via chat):
  User: "Set up a daily AI digest for me. I'm an AI engineer building a
         personal assistant startup. Focus on LLM engineering, agent frameworks,
         mobile dev, and startup funding. Write insights to my Obsidian vault
         and product ideas to Linear."

  Agent:
    1. Store preferences in skill_data_store:
       namespace: "digest", record_type: "preference"
       payload: {
         role: "AI Engineer",
         product_context: "Personal assistant startup on FastAPI",
         topics: ["LLM engineering", "agent frameworks", "mobile dev", "startup funding"],
         sources: ["hn", "arxiv", "rss:the-batch", "rss:import-ai", "rss:tldr-ai",
                    "rss:bens-bites", "producthunt", "reddit:MachineLearning",
                    "reddit:LocalLLaMA"],
         output_targets: ["obsidian", "linear"],
         digest_time: "08:00"
       }
    2. Register scheduled job: daily at 08:00 → full graph execution
    3. OAuth setup for Notion/Linear (if chosen as output targets)
    4. Confirm: "Digest configured. First delivery tomorrow at 8 AM."

Agent execution (parallel fan-out, 6 branches):

  Branch 1 — Hacker News:
    → Fetch top 30 stories from HN Algolia API (past 24h)
    → Filter by relevance to user's topics (LLM scoring with mem0 context)
    → Return: [{title, url, points, comments, relevance_score}]

  Branch 2 — ArXiv:
    → Search ArXiv API for papers in cs.AI, cs.CL, cs.LG (last 24-48h)
    → Filter by keyword relevance to user's configured topics
    → Return: [{title, authors, abstract_summary, url, relevance_score}]

  Branch 3 — RSS/Newsletter Feeds:
    → Fetch configured RSS feeds (The Batch, Import AI, TLDR AI, Ben's Bites, etc.)
    → Parse entries published in last 24h
    → Return: [{title, source, summary, url}]

  Branch 4 — Product Hunt:
    → Fetch today's top launches from Product Hunt API
    → Filter by relevance to user's domain
    → Return: [{name, tagline, url, votes, relevance_score}]

  Branch 5 — Reddit:
    → Fetch top posts from configured subreddits (past 24h, score > threshold)
    → Return: [{title, subreddit, score, url, relevance_score}]

  Branch 6 — Tavily Web Search (supplementary):
    → Search for breaking news in user's topics that other sources may miss
    → "latest AI engineering news today", "new LLM releases this week"
    → Return: [{title, url, snippet}]

Synthesis (LLM — multi-stage):

  Stage 1 — Dedup:
    Merge all results, detect duplicate stories
    (same news covered on HN + newsletter + Reddit → keep highest-signal version)

  Stage 2 — Rank:
    Score each item by:
    → Topic relevance to user's configured interests
    → Professional relevance via mem0 (past engagement, product context)
    → Recency + community signal (HN points, Reddit score, PH votes)

  Stage 3 — Categorize & Summarize:
    Group into sections:
    → "Must Read" (top 3-5 highest-relevance items)
    → "Research" (ArXiv papers with one-sentence summaries)
    → "Product Launches" (Product Hunt + new tools)
    → "Industry News" (funding, acquisitions, partnerships)
    → "Community Discussion" (notable HN/Reddit threads)
    2-3 sentence summary per item, one-paragraph executive summary at top.

  Stage 4 — Product Idea Extraction (LLM with mem0 context):
    Prompt: "Given that user is building [product context from mem0], which
             items suggest features, integrations, or market opportunities?"
    Output: [{idea, source_item, rationale}]
    Example: "LangGraph 0.3 ships parallel state branches → upgrade Trybo's
              fan-out from Send API to native parallel states"
              (from HN post about LangGraph release)

Write side-effects (with consent):

  → Obsidian/Notion: Write daily note with "Must Read" summaries + research
    highlights. Consent gate: approve on first use, then auto-approve for
    daily digest writes (low-risk, append-only).

  → Linear/GitHub Issues: Create one issue per extracted product idea with
    source link + rationale. Consent gate: per-item approval — each idea
    shown in a consent card before issue creation (medium-risk, creates
    external artifacts).

Delivery:

  → Push notification: "Your AI digest is ready — 12 items, 2 product ideas"
  → In-app structured digest card:

  ┌──────────────────────────────────────────────────┐
  │ AI Digest — Thu, Mar 6                           │
  │                                                  │
  │ Must Read (3):                                   │
  │  • GPT-5 benchmarks leaked — 40% jump on MATH   │
  │  • LangGraph 0.3 ships parallel state branches   │
  │  • Anthropic raises $5B Series E                 │
  │                                                  │
  │ Research (2):                                    │
  │  • "Scaling Test-Time Compute" (DeepMind)        │
  │  • "MoE without routing" (Meta FAIR)             │
  │                                                  │
  │ Product Launches (2):                            │
  │  • Cursor 0.45 — multi-file agent editing        │
  │  • Devin 2.0 — async task execution              │
  │                                                  │
  │ Product Ideas (2):                               │
  │  • LangGraph parallel branches → upgrade         │
  │    Trybo's fan-out implementation                 │
  │  • Test-time compute → could improve GID         │
  │    confidence scoring                             │
  │                                                  │
  │ Written to Obsidian · 2 Linear issues created    │
  └──────────────────────────────────────────────────┘
```

### Data Model (uses generic Skill Data Store)

No digest-specific tables. All data flows through the platform's `skill_data_store`:

```
User preferences → write to skill_data_store:
  namespace: "digest", record_type: "preference"
  payload: {role, topics, sources, output_targets, digest_time}

Each daily digest run → write to skill_data_store:
  namespace: "digest", record_type: "daily_digest"
  payload: {
    date: "2026-03-06",
    items: [{title, url, source, category, relevance_score, summary}],
    ideas: [{idea, source_item, rationale, linear_issue_id}],
    stats: {total_fetched: 87, after_dedup: 42, after_ranking: 12}
  }
  metadata: {sources_queried: ["hn", "arxiv", "rss", "ph", "reddit", "tavily"]}

Engagement tracking (future) → write to skill_data_store:
  namespace: "digest", record_type: "engagement"
  payload: {item_url: "...", action: "clicked|saved|dismissed", timestamp: "..."}
  → Feeds back into mem0 for improved relevance scoring over time
```

### Architecture Additions Required

| Addition | Why | Scope |
|---|---|---|
| **RSS/Feed aggregation skill** | Core data collection for newsletters and blogs. No existing skill fetches and parses RSS/Atom feeds. This is a generic skill — any future use case needing periodic content ingestion from web sources can reuse it. | New Direct adapter skill: `content.feed.fetch`. Takes a list of feed URLs, returns parsed entries with title, content, date, link. Uses Python `feedparser` library. Registered in skill registry. |
| **External knowledge base write skills** | Digest insights must flow to the user's knowledge system (Obsidian, Notion). These are side-effect skills requiring consent gates. Not digest-specific — any use case that produces reference content (trip itineraries, research results) can write to the same targets. | Notion: via MCP (`makenotion/notion-mcp`) or Composio. Obsidian: via community MCP (`obsidian-mcp`) or Obsidian Local REST API plugin. Register as `knowledge.note.write` in skill registry with `requires_consent: true`. |
| **External issue tracker write skills** | Product ideas must flow to the user's task tracker (Linear, GitHub Issues). Side-effect skills requiring per-item consent. Generic capability — any use case that produces actionable tasks can use this. | Linear: via Composio (`composio-langgraph`). GitHub Issues: via MCP (`github/github-mcp`) or Composio. Register as `tasks.issue.create` in skill registry with `requires_consent: true`. |
| **Scheduled job executor** | Daily digest must fire automatically at user-configured time. Same scheduler infrastructure as UC1/UC4. | Shared with UC1, UC4: `orchestration/scheduler/` |
| **Proactive push notifications** | Digest results must be pushed to user's phone. Same push infra as UC1/UC4. | Shared with UC1, UC4: `notification.push.send` skill |
| **Parallel fan-out in executor** | All 6 source-fetching branches are independent and must execute concurrently. Sequential execution would take 20-30s vs 4-6s parallel. Same fix as UC2/UC4. | Shared with UC2, UC4: LangGraph fan-out fix |
| **User preference config** | User must configure: role, topics, sources, output targets, digest time. Structured config data — not fuzzy semantic memory. | Uses Skill Data Store: `namespace: "digest", record_type: "preference"`. No new table. Same platform module as UC1/UC4. |

### Skills & Integrations Required

| Skill | Adapter Type | Implementation | Side Effect | Notes |
|---|---|---|---|---|
| **RSS Feed Fetcher** | Direct | Python `feedparser` + `httpx` — fetch and parse RSS/Atom feeds. Generic skill usable for any feed source. | No | Handles: newsletters (The Batch, Import AI, TLDR AI, Ben's Bites), blog feeds, podcast feeds. ~0.5s per feed. |
| **Hacker News API** | Direct | HN Algolia API (`hn.algolia.com/api/v1`) — search and top stories. Free, no auth, no rate limit concerns. | No | Filter by topic keywords, date range, minimum points. |
| **ArXiv API** | Direct | ArXiv API (`export.arxiv.org/api/query`) — search papers by category and keywords. Free, no auth. | No | Categories: cs.AI, cs.CL, cs.LG, cs.CV. Rate limit: 1 req/3s. |
| **Product Hunt API** | Direct | Product Hunt GraphQL API — today's launches, filtered by topic. Free tier available. OAuth for full access. | No | Daily launch data sufficient for digest. |
| **Reddit API** | Direct | Reddit JSON API (`reddit.com/r/{sub}/top.json`) — no auth needed for public read. PRAW for authenticated access if rate-limited. | No | Subreddits: r/MachineLearning, r/LocalLLaMA, r/artificial, r/startups, etc. |
| **Web Search (Tavily)** | MCP | Supplementary search for breaking news that primary sources may miss. Already in v1 skill set. | No | Shared with UC2, UC3. |
| **Notion Write** | MCP or Marketplace | Create/append pages in user's Notion workspace. `makenotion/notion-mcp` or Composio. OAuth per user. | Yes | Consent: approve on first digest write, then auto-approve (low risk, append-only). |
| **Obsidian Write** | MCP or Direct | Append to daily note in user's Obsidian vault. Community MCP (`obsidian-mcp`) or Obsidian Local REST API plugin. | Yes | Requires Obsidian running with REST API plugin. Alternative: use Notion as primary target. |
| **Linear Issue Create** | Marketplace (Composio) | Create issues in user's Linear project with title, description, labels. `composio-langgraph` SDK. | Yes | Per-item consent — each product idea shown in consent card before issue creation. |
| **GitHub Issues Create** | MCP or Marketplace | Alternative to Linear. `github/github-mcp` or Composio. | Yes | Same consent pattern as Linear. User chooses Linear or GitHub, not both. |
| **LLM Analysis** | Direct | Multi-stage: dedup + relevance scoring + categorization + summarization + product idea extraction. Heaviest LLM usage of all UCs (~6 LLM calls per digest run). | No | Use GPT-4o-mini for filtering/dedup, Claude/GPT-4o for synthesis + idea extraction. |
| **mem0 Memory** | Direct | Read: user's role, interests, product context for relevance scoring. Write: engagement patterns (what user clicks/reads) for improved future ranking. | No | Already planned. Critical for personalization quality — digest improves over time. |
| **Push Notification** | Direct | Deliver digest notification. FCM/APNs. Same as UC1/UC4. | Yes (low risk) | Shared with UC1, UC4. |

---

## Future Use Cases (Deferred)

Use cases validated but deferred due to higher infrastructure dependencies. These reuse the same platform modules built for UC1-UC5.

### Email Expense Tracker

**Domain:** Personal Finance / Document Intelligence
**Priority:** P3 (deferred — depends on Document Intelligence pipeline)
**Trigger phrase:** *"How much did I spend on subscriptions last quarter?"*

**Problem:** Subscription and purchase data is scattered across hundreds of emails (AWS invoices, Netflix receipts, Swiggy orders, Uber rides, credit card statements). No aggregation, no categorization, no trend analysis. Users who want to know total monthly spend have to manually search Gmail and add up numbers.

**Why deferred:** Requires Document Intelligence pipeline (Kreuzberg + LLMWhisperer for PDF parsing, Instructor + Pydantic for entity extraction) — a significant infrastructure piece not needed by any P0/P1 use case. Also requires Parallel-Map node in executor (process 40-60 emails concurrently). Both are platform modules that will be built when this use case is prioritized.

**Key skills needed:** Gmail Search/Attachment Download (shared with UC4), Document Text Extraction (Kreuzberg — new), Structured Entity Extraction (Instructor — new), LLM Analysis, mem0 (vendor→category mappings). All persistent data through Skill Data Store (`namespace: "expenses"`).

**Platform modules it would add:**
- Document Parsing Pipeline (Kreuzberg + Instructor) — generic skill for any document→structured data use case
- Parallel-Map Node in Executor — generic `map` node type for processing variable-length lists (also benefits UC3 product page scraping, UC2 flight option checking)

---

## Architecture Additions Summary

Cross-cutting **platform modules** required to support the 5 use cases. Each addition is generic infrastructure — no module is built for a single use case. Mapped against the current planned architecture from `0-main.md`:

| Platform Module | Required By | Current State | Implementation Scope |
|---|---|---|---|
| **Skill Data Persistence Layer** (`orchestration/persistence/`) | UC1, UC3, UC4, UC5 (and any future use case that produces data outliving a single graph execution) | **Not planned.** Current architecture has mem0 (semantic memory) and audit log (observability) but nothing for structured, queryable, per-user skill-produced data. | Single `skill_data_store` table + `SkillDataStore` service class. See "Platform Architecture Principle" section above. One table, one module, zero migrations for new use cases. Namespace isolation between skills. Aggregation done in application code, results written back to the same table. |
| **Scheduled Job Executor** (`orchestration/scheduler/`) | UC4 (daily briefing), UC5 (daily digest), UC1 Stage 2+ (future commute data collection crons) | **Not planned.** Current architecture is one-shot only (user message → graph → result). | New module. Supabase table `scheduled_jobs(user_id, job_type, cron_expression, config JSONB, next_run, enabled)`. Background worker polls and triggers full graph executions (UC4 briefing, UC5 digest). APScheduler or Celery Beat as runtime. Generic — any skill can register a scheduled job. |
| **Proactive Push Notifications** (new skill: `notification.push.send`) | UC4 (briefing delivery), UC5 (digest notification), UC1 Stage 2+ (future departure alerts) | **Not planned.** All current output goes to chat only. | New Direct adapter skill registered in the skill registry. FCM Admin SDK for Android, APNs for iOS. React Native push token registration at app startup, stored in user profile via Skill Data Store (`namespace: "user_profile", record_type: "push_token"`). Generic — any skill or scheduled job can push. |
| **Parallel Fan-Out in Executor** | UC2 (flight+hotel+maps concurrent), UC4 (all briefing branches concurrent), UC5 (6 source-fetching branches concurrent) | **Planned but not implemented.** Current executor serializes everything via topological sort. Known P1 issue from `4-other-implementation-comparison.md`. | Fix `compile_plan_to_stategraph()` in `plan_builder.py` to use LangGraph's `Send` API or parallel state branches for independent nodes. Pure platform fix — benefits every multi-step plan, not just these use cases. |
| **Structured Artifact System** (mobile) | UC2 (itinerary), UC3 (comparison table), UC4 (briefing), UC5 (digest card) | **Not planned.** Mobile UI only renders plain chat text + consent cards. | Generic artifact protocol: backend returns `{artifact_type: string, payload: JSONB}`. Mobile app has a renderer registry mapping `artifact_type` → React Native component. Adding a new card type = adding one React Native component + registering it, no backend schema change. Initial types: `itinerary_card`, `comparison_table`, `briefing_card`, `digest_card`. |
| **RSS/Feed Aggregation Skill** | UC5 (newsletter + blog feed collection) | **Not planned.** No existing skill for RSS/Atom feed parsing. | New Direct adapter skill: `content.feed.fetch`. Python `feedparser` + `httpx`. Takes list of feed URLs, returns parsed entries. Generic — any future use case needing content ingestion from web feeds can reuse. |
| **External Write Adapters** (Notion, Obsidian, Linear, GitHub Issues) | UC5 (write insights to knowledge base + product ideas to issue tracker) | **Not planned.** Current skills are read-only or internal. No skills write to external user-owned systems. | Notion: MCP (`makenotion/notion-mcp`) or Composio. Obsidian: community MCP. Linear/GitHub: Composio or MCP. All registered with `requires_consent: true`. Generic — any use case producing reference content or actionable tasks can use these write targets. |
| **Web Scraping Skill** | UC3 (product page extraction) | **Not planned.** Current skills are API-based only. | Playwright MCP for dynamic pages, Firecrawl MCP for static extraction. Register as `web.scrape.structured` skill in registry. Generic — usable for any future web data extraction need. |
| **Partial Re-planning** (Planner) | UC2 (swap hotel mid-plan), and any complex plan where the user wants to modify one step without restarting | **Not planned.** Current planner generates a full plan from scratch each time. | Enhancement to Planner: accept a completed/in-progress PlanSpec + a modification request → identify affected nodes → preserve completed results for unaffected nodes → re-plan only the changed subtree → resume executor from modification point. |
| **Location Service Passthrough** | UC1 (origin from device), UC4 (home/work locations) | **Not planned explicitly.** | Mobile: `expo-location` → coordinates in API request body, passed into `TaskContext`. Stored locations (home/work) persisted in Skill Data Store (`namespace: "user_profile", record_type: "saved_location"`). |

**Deferred platform modules** (needed only for Future use cases):

| Platform Module | Required By | Implementation Scope |
|---|---|---|
| **Document Parsing Pipeline** (skill) | Future: Email Expense Tracker | Kreuzberg + LLMWhisperer (Layer 1) + Instructor + Pydantic (Layer 2). Generic skill for any document→structured data extraction. |
| **Parallel-Map Node** (Executor) | Future: Email Expense Tracker, UC3 (product pages), UC2 (flight options) | New `map` node type in PlanSpec. Executor spawns N parallel invocations with configurable concurrency limit. Distinct from fan-out — handles variable-length input lists. |

---

## Complete Skills Inventory

All skills/integrations required across the 5 active use cases, deduplicated and tagged by which use cases need them:

### Already Planned (in v1 roadmap or `3-skills.md`)

| Skill | Used By | Status |
|---|---|---|
| Web Search (Tavily MCP) | UC2, UC3, UC5 | P0 — in v1 skill set |
| Gmail Search/Read (Google Workspace MCP) | UC4 | P1 — in v1 skill set |
| Google Calendar Read (Google Workspace MCP) | UC4 | P1 — in v1 skill set |
| Groww Portfolio Read (Direct) | UC4 | Planned in `3-skills.md` |
| LLM Summarize/Analyze (Direct) | UC1-5 | In v1 skill set (`text.summarize.llm`) |
| mem0 Memory (Direct) | UC3, UC5 | Workstream 1 — in progress |

### New Skills to Build

| Skill | Adapter Type | Used By | Priority | Est. Effort |
|---|---|---|---|---|
| Google Maps Directions (real-time + future departure times) | Direct (preferred) | UC1, UC2, UC4 | **P0** | 1-2 days — direct Routes API call; MCP optional |
| Google Maps Geocoding | MCP or Direct | UC1, UC2 | **P0** | Bundled with Maps Directions (same API/MCP) |
| Google Maps Places | MCP or Direct | UC2 | P1 | Bundled with Maps MCP |
| Weather (Open-Meteo) | MCP or Direct | UC2, UC4 | **P0** | 1 day — Open-Meteo MCP exists, or direct REST (no key needed) |
| Push Notification (FCM/APNs) | Direct | UC1, UC4, UC5 | **P0** | 2-3 days — FCM Admin SDK + mobile push token registration |
| RSS Feed Fetcher | Direct | UC5 | **P1** | 1 day — Python `feedparser` + `httpx`, straightforward |
| Hacker News API | Direct | UC5 | **P1** | 0.5 days — HN Algolia API, free, no auth |
| ArXiv API | Direct | UC5 | **P1** | 0.5 days — ArXiv search API, free, no auth |
| Product Hunt API | Direct | UC5 | P1 | 1 day — GraphQL API, OAuth for full access |
| Reddit API | Direct | UC5 | P1 | 0.5 days — JSON API (no auth) or PRAW (authenticated) |
| Notion Write | MCP or Marketplace | UC5 | P1 | 1-2 days — `makenotion/notion-mcp` or Composio, OAuth setup |
| Obsidian Write | MCP or Direct | UC5 | P1 | 1-2 days — community MCP, requires Obsidian REST API plugin |
| Linear Issue Create | Marketplace (Composio) | UC5 | P1 | 1 day — `composio-langgraph` SDK |
| GitHub Issues Create | MCP or Marketplace | UC5 | P1 | 1 day — `github/github-mcp` or Composio (alternative to Linear) |
| Flight Search (Amadeus / Duffel) | Direct | UC2 | P1 | 2-3 days — Amadeus Python SDK, free test tier |
| Hotel Search (Booking.com / Serpapi) | Direct | UC2 | P1 | 2-3 days — Serpapi hotel search as MVP, Booking.com API as upgrade |
| Web Scraping (Playwright / Firecrawl) | MCP | UC3 | P1 | 1-2 days — Playwright MCP exists |
| Location Service | Mobile + Backend | UC1, UC4 | P1 | 1 day — expo-location on mobile, coordinate passthrough on backend |

### Deferred Skills (Future Use Cases)

| Skill | Adapter Type | Used By | Est. Effort |
|---|---|---|---|
| Gmail Attachment Download | MCP or Direct | Future: Expense Tracker | Bundled with Gmail MCP |
| Document Text Extraction (Kreuzberg) | Direct | Future: Expense Tracker | 2-3 days — Python library |
| Structured Entity Extraction (Instructor) | Direct | Future: Expense Tracker | 1-2 days — Python library, schema definition |

---

## Implementation Priority & Sequencing

**Phase 1 — Platform foundations + Commute Optimizer pilot (P0):**

Platform modules built here are reused by every subsequent phase.

1. **Skill Data Persistence Layer** — `orchestration/persistence/` module + `skill_data_store` table. Foundation for all structured data across all use cases.
2. **Scheduled Job Executor** — `orchestration/scheduler/` module + `scheduled_jobs` table. Foundation for all recurring/proactive behavior (UC4/UC5 daily triggers, future UC1 Stage 2 data collection).
3. Google Maps Directions skill (Direct adapter — supports both real-time and future departure times)
4. Google Maps Geocoding skill (resolve addresses to lat/lng)
5. Commute Optimizer skill (one-shot: loop over time window → ~16 Maps queries → compare → recommend best departure)
6. Structured Artifact System — `commute_alert` artifact type (first artifact registered)

**Phase 2 — Morning Briefing (proves parallel fan-out + multi-source synthesis):**

Reuses Phase 1 platform modules: Scheduler (daily trigger), Push Notifications (delivery), Skill Data Store (user preferences).

1. Weather skill (Open-Meteo)
2. Google Calendar MCP integration
3. Gmail MCP integration
4. Parallel fan-out fix in executor (platform fix — benefits all plans)
5. Structured Artifact System on mobile (generic renderer registry)
6. Briefing card artifact type (first artifact registered)
7. Daily scheduled trigger (reuses Phase 1 scheduler)

**Phase 3 — Professional Intelligence Digest (proves external write side-effects + personalization):**

Reuses Phase 2 platform modules: Scheduler (daily trigger), Push Notifications (delivery), Parallel Fan-Out (6 concurrent branches), Structured Artifact System (digest card), Skill Data Store (preferences + digest history). This is the natural successor to Morning Briefing — same infrastructure pattern (scheduled → fan-out → synthesize → deliver), different data sources and the addition of write side-effects.

1. RSS Feed Fetcher skill (Python `feedparser` — generic content ingestion)
2. Hacker News API, ArXiv API, Product Hunt API, Reddit API skills (all simple Direct adapters — free APIs, no auth for basic access)
3. Digest card artifact type (registered in Phase 2's artifact system)
4. Notion Write skill (MCP or Composio — first external write adapter, requires OAuth setup)
5. Obsidian Write skill (community MCP — alternative to Notion)
6. Linear Issue Create skill (Composio — first issue tracker write adapter)
7. Product idea extraction logic (LLM with mem0 context — leverages existing mem0 from Phase 1)
8. Engagement tracking (write-back to Skill Data Store for future relevance improvement)

**Phase 4 — Trip Planner (proves complex multi-step orchestration):**

Reuses: parallel fan-out, artifact system, Maps skill, Weather skill.

1. Flight search skill (Amadeus)
2. Hotel search skill (Serpapi → Booking.com)
3. Partial Re-planning in Planner (platform capability — modify PlanSpec mid-flight, preserve completed results)
4. Itinerary card artifact type (registered in Phase 2's artifact system)

**Phase 5 — Purchase Advisor (proves web scraping + personalization):**

Reuses: Skill Data Store (user financial profile), mem0 (semantic user facts), artifact system, Notion/Obsidian write adapters (from Phase 3).

1. Web scraping skill (Playwright MCP)
2. User financial profile storage (mem0 for semantic facts + Skill Data Store for structured lookup)
3. Comparison table artifact type

**Future — Email Expense Tracker (deferred — depends on Document Intelligence pipeline):**

Reuses: Gmail MCP, Skill Data Store, artifact system, Parallel-Map node.

1. Document text extraction skill (Kreuzberg)
2. Structured entity extraction skill (Instructor + Pydantic)
3. Gmail attachment download
4. Parallel-Map node in Executor (platform pattern — process N items concurrently with concurrency limit)
5. Expense persistence (writes to Skill Data Store — `namespace: "expenses"`)
