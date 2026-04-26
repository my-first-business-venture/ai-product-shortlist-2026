# Real Estate / Rental Research Engine — MVP Implementation Plan

## Context

House-hunting and rental search are the canonical "open 50 tabs, lose track of which is which" workflow. Real Estate Research Engine is a personal-first agent that scrapes Idealista / Fotocasa (and optionally Zillow / Booking.com for the diaspora) on a daily cadence, deduplicates listings across portals, enriches each one with commute time / schools / nearby services from Google Maps, scores them with Claude against your stated preferences, and pings you when a high-scoring new listing appears.

**Goal of the MVP:** A working pipeline you and your spouse can use during a real house-hunt — daily scrape, ranked list of top 10 candidates with one-paragraph fit analysis each, and a deep-dive button that produces a per-listing PDF/Markdown report. No multi-tenancy, no user-facing public deployment, no commercialization until you've personally validated the ranking quality.

## Scope

### In scope (MVP)
- One or two **search projects** (e.g. "Madrid 2-bed under €1,800" + "Barcelona near beach under €2,200"), each with structured criteria.
- **Scraping** of Idealista + Fotocasa for Spain (configurable to swap in Zillow / Booking.com / Imovirtual).
- **Listing deduplication** via pgvector embeddings — the same flat listed on 3 portals collapses to one record with all source URLs.
- **Enrichment** per listing: commute time to a configured target address (Google Maps Distance Matrix), counts of nearby schools / supermarkets / pharmacies / hospitals, walk-score signals from POI density.
- **Claude scoring** — Sonnet 4.6 weighs the enriched listing against your criteria and returns a 0–100 fit score plus a 2–3 sentence justification.
- **Daily ranked digest** delivered via webhook (Discord / Telegram / ntfy) and rendered in a tiny FastAPI/HTML page.
- **Deep-dive reports** on demand — full Markdown report per listing with pros/cons, neighborhood profile, and follow-up questions to ask the agent.
- **Optional Gemini Flash photo analysis** — image-derived signals about condition, lighting, recent reform.

### Out of scope (deferred)
- Multi-tenant SaaS, accounts, billing — see the **Service-tier projection** section for the path.
- Auto-contacting agents / sending messages on your behalf.
- Mortgage calculator, affordability modeling, ROI for buy-to-let.
- Native mobile app.
- Saved-search sharing between users beyond a single Postgres database.
- Price history reconstruction (would need a paid Idealista API tier or long-running data accumulation).

## Architecture

```
┌──────────────────────────────────────┐
│ Cron in FastAPI (apscheduler)         │ daily at 07:00
│   → fans out one job per portal       │
└──────────────┬────────────────────────┘
               ▼
┌──────────────────────────────────────┐
│ 1. Scraper layer                     │  Apify actors per portal
│    - idealista-scraper               │  (or self-hosted Playwright)
│    - fotocasa-scraper                │
│    - (optional) zillow-scraper       │
└──────────────┬────────────────────────┘
               ▼
┌──────────────────────────────────────┐
│ 2. Normalizer + dedup                │  Postgres + pgvector
│    embed(title+address) → cosine     │  upsert by listing_key
└──────────────┬────────────────────────┘
               ▼
┌──────────────────────────────────────┐
│ 3. Enrichment                        │  Google Maps Distance Matrix
│    commute_min, school_count,        │  + Places API
│    supermarket_count, etc.           │
└──────────────┬────────────────────────┘
               ▼
┌──────────────────────────────────────┐
│ 4. AI scoring                        │  Claude Sonnet 4.6
│    fit_score 0–100 + 2-3 sentences   │  (Gemini Flash on photos, optional)
└──────────────┬────────────────────────┘
               ▼
┌──────────────────────────────────────┐
│ 5. Digest + notify                   │  Webhook → Telegram/Discord/ntfy
│    HTML/Markdown ranked list         │  + persisted in Postgres for UI
└──────────────────────────────────────┘

   On-demand: GET /listing/:id/deep-dive  →  Claude full report
```

- **Backend (FastAPI on NAS Docker).** Single Python service runs cron + API endpoints. Migrate to Hetzner CPX21 (€7.55/mo) only if you need the UI accessible off-network.
- **DB (Postgres 16 + pgvector).** Listings table with embedding column for cross-portal dedup.
- **Scraping (Apify).** Pay-per-result actors for Idealista/Fotocasa/Zillow. Free fallback: self-hosted Playwright with conservative throttling.
- **Maps (Google Maps APIs).** Distance Matrix for commute, Places for POI counts. Stays inside the $200/mo free credit at any sane personal scale.
- **AI scoring (Anthropic Claude Sonnet 4.6).** Per-listing scoring + on-demand deep-dive reports.
- **Image analysis (Google Gemini Flash, optional).** Cheap multimodal pass on the 3–5 hero photos per listing.
- **Notifications.** Single configurable webhook URL.
- **Observability.** Self-hosted Langfuse traces every Claude call with cost.

## Required components and choices

| Component | Choice | Reasoning | Cost |
|---|---|---|---|
| Backend runtime | **FastAPI + apscheduler on Docker** | Matches your stack; cron + API in one service | Free |
| DB | **Postgres 16 + pgvector** | Dedup needs vector similarity; everything else fits relational | Free |
| Scraping | **Apify Starter plan** (recommended baseline) | Production-grade; ~$49/mo covers personal use across 2–3 portals | **$49/mo** |
| Scraping (free alt) | Self-hosted Playwright | Fragile; breaks on portal redesigns; no CAPTCHA help | $0 (high maintenance) |
| Scraping (heavy alt) | Bright Data residential + Scraping Browser | Best for hostile portals; pay-as-you-go | $50–150/mo |
| Maps | **Google Maps Distance Matrix + Places** | $200/mo free credit ample at personal scale | $0 |
| Geocoding fallback | Nominatim (OpenStreetMap) self-host | Free, slower, lower accuracy in EU rural | $0 |
| AI scoring | **Claude Sonnet 4.6** | Quality of fit-judgment is the whole product | Pay-per-token |
| Photo analysis (optional) | **Gemini Flash 2.5** | Multimodal at a fraction of Claude's cost | Free tier covers personal scale |
| Embeddings (dedup) | `text-embedding-3-small` (OpenAI) or `voyage-3-lite` | ~$0.02 per 1M tokens; minimal cost | <$1/mo |
| Notifications | **ntfy / Telegram bot / Discord webhook** | Free; pick one | Free |
| Observability | **Self-hosted Langfuse** (existing in your stack) | Cost-per-listing traceable | Free |
| Frontend (optional) | Plain HTML + HTMX or SvelteKit static export | Personal use → minimal UI is fine | Free |

## Implementation steps

### Phase 1 — Skeleton + DB (1 day)
1. New repo `real-estate-research-engine/`. Python 3.12 + FastAPI + apscheduler + asyncpg + pgvector.
2. Docker Compose on NAS: `app` + `postgres-pgvector` + optional `pgadmin`.
3. Schema:
   - `search_projects(id, name, city, country, max_price, min_bedrooms, must_haves_json, target_address, target_lat, target_lng, max_commute_min, criteria_md, created_at)`
   - `listings(id, listing_key, search_project_id, title, address, lat, lng, price, bedrooms, bathrooms, m2, description_md, photos_json, source_urls_json, embedding vector(1536), first_seen_at, last_seen_at, status enum('active','removed','rented'))`
   - `enrichments(listing_id, commute_min, school_count, supermarket_count, pharmacy_count, hospital_count, poi_density_score, computed_at)`
   - `scores(listing_id, model, fit_score, summary_md, scored_at, cost_usd)`
   - `cost_log(id, day, source, units, cost_usd)` — Apify, Google, Anthropic, Gemini

### Phase 2 — Search-project criteria editor (0.5 day)
4. Tiny HTML form (or just direct SQL inserts for MVP) to create a search project.
5. Free-text `criteria_md` field captures soft preferences ("south-facing balcony", "near a park", "no ground floor"). Goes verbatim into the Claude scoring prompt.

### Phase 3 — Scraping (Idealista first) (1.5 days)
6. Sign up for Apify, create token, store in `.env`.
7. Pick an Idealista actor (e.g. `vaclavrut/idealista-scraper`); parameterize input (URL of search page, max pages).
8. Cron job: every day at 07:00, run the actor for each search project, write raw JSON to `data/raw/{date}/idealista/`.
9. Normalizer reads raw JSON, upserts into `listings` keyed on `listing_key = sha256(canonical_url)`.
10. Repeat the actor pattern for Fotocasa.
11. Acceptance: a manual run pulls ≥20 fresh listings from each portal and persists them deduped.

### Phase 4 — Dedup via pgvector (1 day)
12. On ingest, compute an embedding for `f"{title} | {address} | {bedrooms}br {m2}m2"` using `text-embedding-3-small`.
13. Before insert, query for existing listings within `cosine_distance <= 0.10`; if a match, **merge**: append the new portal URL to `source_urls_json`, keep the older `first_seen_at`, refresh `last_seen_at`.
14. Acceptance: feed the same flat scraped from Idealista and Fotocasa; confirm exactly one row in `listings` with two source URLs.

### Phase 5 — Enrichment (1 day)
15. Google Cloud project; enable Distance Matrix + Places APIs; create API key restricted by domain/IP.
16. For each new listing, async:
    - Distance Matrix call from `(lat,lng)` to `target_address` for commute_min (driving + transit, take min).
    - Places nearby search at the listing coords for `school`, `supermarket`, `pharmacy`, `hospital` counts within 1 km.
17. Compute a simple `poi_density_score = log(1 + supermarket + 0.5*school + 0.7*pharmacy + 0.4*hospital)`.
18. Persist to `enrichments`.
19. Acceptance: one listing has plausible commute_min and POI counts (sanity-check against Google Maps manually).

### Phase 6 — Claude scoring (1.5 days)
20. Per listing, build the prompt: criteria_md + structured listing (price, m2, bedrooms, address) + enrichments + (optional) Gemini Flash photo summary.
21. Sonnet system prompt: "Output exactly one JSON object: `{ fit_score: 0–100, headline: <12 words>, pros: [<bullet>, ...], cons: [<bullet>, ...], summary_md: <2–3 sentences> }`. Score must reflect both hard criteria (price, bedrooms) and soft `criteria_md`. **Do not invent facts**; if a criterion is unobservable from the listing, say so."
22. Cache by `(listing_id, criteria_hash, model)` so re-scoring with unchanged inputs is free.
23. **Deep-dive endpoint** `GET /listing/:id/deep-dive`: a richer Sonnet call producing a full Markdown report (neighborhood profile, fit analysis, "questions to ask the agent" section).
24. Acceptance: top-5 by fit_score are listings you'd actually click on, evaluated by you.

### Phase 7 — Daily digest + notifications (1 day)
25. After scoring completes, render an HTML email / Markdown digest of the top 10 fresh-or-rescored listings with thumbnail, fit_score, headline, source links.
26. POST it to your configured webhook (Telegram bot / Discord / ntfy).
27. The same digest is available at `GET /` (a simple page) so you can browse without leaving the message.

### Phase 8 — Photo analysis (optional, 0.5 day)
28. For each listing, take the first 3 photos; one Gemini Flash call returning JSON `{ light_quality: 1-5, condition_signals: [...], red_flags: [...], recent_reform_likelihood: 0-1 }`.
29. Pass into the Claude scoring prompt as additional context.

**Total build effort: ~5–6 days (Phases 1–7); ~6–7 days with Phase 8.**

## Cost estimate

### Fixed monthly
- **Apify Starter: ~$49/mo** — the realistic baseline for production-grade scraping across 2–3 portals.
- **Google Maps Distance Matrix + Places: $0** — at personal scale (≤5 search projects, ≤200 enrichments/day) the $200/mo free credit easily absorbs it.
- **Postgres + FastAPI on NAS: $0** marginal.
- **Notifications (ntfy / Telegram / Discord): $0**.
- **Embeddings (`text-embedding-3-small`): <$1/mo** — even at 5k embeddings/mo.

### Per listing scored (basic mode)
- Claude Sonnet score: ~2,000 input + 600 output → **~$0.015 per listing**.
- Optional Gemini Flash photo analysis: ~3 photos × ~258 tokens + 300 output → **~$0.0004 per listing**.
- Maps + embedding: rounding error.

### Per deep-dive report (on-demand)
- Sonnet, ~3,500 input + 2,500 output → **~$0.048 per report**.

### Monthly totals (typical personal use)

| Use pattern | Listings/day after dedup | Listings/mo | $ Claude | $ Apify | **Total** |
|---|---|---|---|---|---|
| **Light hunt** (1 search, casual) | 20 | ~600 | ~$9 | $49 | **~$58/mo** |
| **Active hunt** (2 searches, both spouses) | 50 | ~1,500 | ~$22 | $49 | **~$71/mo** |
| **Heavy hunt + photo analysis + 30 deep-dives** | 50 | ~1,500 | ~$24 | $49 | **~$73/mo** |
| **Free-tier / Playwright path (high maintenance)** | 20 | ~600 | ~$9 | $0 | **~$9/mo** |

### Hosting alternatives
- **Hetzner CPX21** (€7.55/mo) — needed only if the UI must be reachable off-network during a viewing.
- **GCP** — Cloud Run jobs + Cloud SQL Postgres w/ pgvector ($9/mo) + Cloud Storage for photo cache. **~$15–25/mo + Apify metered.** Use only if you go multi-user.

### Service-tier projection (not MVP, for context)
- **Per-report pricing ($50–100):** marginal Claude cost is $0.05; Apify amortized across paying users; gross margin ~95%+ once Apify Starter is shared across ~10 users.
- **Subscription pricing ($30/mo):** breaks even on Apify Starter at **~2 paying users**; ~70% gross margin at 10+ users.
- The shortlist's "scraping fragility caps long-term margin" caveat is real — 1 portal redesign per quarter eats ~1 dev-day.

## Verification plan

After Phase 7:
1. **End-to-end fresh run.** From an empty DB, one cron tick produces ≥20 deduplicated listings with enrichments + scores within 30 minutes.
2. **Dedup sanity.** Manually find a flat listed on both portals; confirm one row in `listings` with two `source_urls`.
3. **Commute correctness.** Spot-check 5 listings; the calculated `commute_min` matches a manual Google Maps trip ±10%.
4. **Score sanity.** The top-5 by `fit_score` are listings you would actually click. The bottom-5 are clearly outside your criteria. If not, tune the prompt.
5. **Deep-dive quality.** Generate 3 deep-dive reports; the family-essentials section names real schools/supermarkets that exist (no hallucinations). Address verifiable facts only.
6. **Notification flow.** A new listing matching your criteria triggers a Telegram/Discord message within 1 hour of the cron run.
7. **Cost reconciliation.** Langfuse + Apify console + Google billing dashboard sum within 5% of the `cost_log` table over a 7-day window.
8. **Budget guard.** Force a low cap (e.g. $0.50/day) and confirm the system stops scoring once exceeded, with a clear log + notification.

## Risks and open questions

- **Scraping fragility.** Idealista in particular rate-limits aggressively and changes its DOM regularly. Apify actors absorb most of that pain; expect ~1 day of fixing per quarter regardless. If a portal blocks Apify entirely, fall back to a slower Bright Data Scraping Browser session or remove the portal.
- **ToS gray area.** Scraping public listings for personal use is generally tolerated; commercializing crosses lines. The shortlist explicitly flags this — if you ever go to per-report pricing, get legal advice for your jurisdiction(s).
- **Hallucinated neighborhood facts.** Claude can confidently invent a school name. Mitigate by feeding only verified Places API results into prompts and instructing the model: "only mention POIs present in the supplied JSON; cite their type."
- **Stale listings.** Listings get rented within hours in hot markets. Always refresh on the day before contacting an agent; mark `status='removed'` when the actor stops returning a listing for 3 consecutive days.
- **Geocoding edge cases.** Rural addresses and new developments confuse Google's geocoder. Spot-check coords for any commute_min > 90 minutes — usually wrong.
- **Cost runaway via deep-dives.** The deep-dive endpoint is the highest-cost surface. Cap to N requests/day per user and cache by `(listing_id, criteria_hash)` so multiple clicks of the same button cost nothing.
- **Photo analysis privacy.** Some listings include identifiable interiors / current tenants. Don't archive photos longer than the listing lives. If you ever multi-user, treat photo URLs as PII.
- **Embedding drift.** If you change embedding models, dedup quality breaks. Pin the model version; re-embed in a controlled migration if you ever upgrade.
- **Service-tier business risk.** Per the shortlist ranking, scraping fragility is the cap on long-term margin. Build the personal MVP first; only commit to commercialization once you've maintained the scrapers for ≥3 months and know the real maintenance cost.
