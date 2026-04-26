# Property-Manager Co-Pilot — MVP Implementation Plan

## Context

Short-term rental hosts (Airbnb, Booking.com, VRBO) and small landlords spend hours per week on four predictable tasks: pricing nightly rates against neighborhood comps, replying to guests in the right language and tone, scheduling cleaning turnovers around the booking calendar, and monitoring reviews. Incumbents like Hostfully and Hospitable solve this for $50–100/listing — Property-Manager Co-Pilot targets the long tail of 1–10-listing operators who can't justify that price but still need the productivity gain.

**Goal of the MVP:** A working personal install for **one listing** (yours, or a family member's, or a friend's whose data you can use). Each feature must demonstrably reclaim time over a 30-day live test before declaring MVP done. Co-pilot shape only — every outbound action is host-approved, never auto-sent.

> **Build trigger:** the shortlist explicitly notes you don't currently rent anything out. Do not build this until you (or someone close enough to dogfood) has at least one active listing. Without real bookings, the pricing engine has nothing to react to and the messaging engine has no real guests to draft replies for.

## Scope

### In scope (MVP — single host, 1–10 listings)
- **Listing registry** with iCal feed URL per listing (every Airbnb / Booking.com / VRBO listing exports one).
- **Pricing engine.** Daily pull of comps via Rentcast (or AirDNA); Claude Sonnet weighs occupancy, day-of-week, lead time, local events; outputs a suggested nightly rate per listing per night with a 1-paragraph rationale.
- **Guest messaging draft queue.** Email-forwarding ingestion (host forwards Airbnb / Booking notification emails to a per-host address); Claude Sonnet drafts a reply in the guest's language matching the listing's house tone; host approves and pastes back into the Airbnb / Booking inbox.
- **Cleaning schedule.** Read iCal feeds, build a daily turnover plan, push a Telegram (or WhatsApp Cloud API) digest to the cleaning team at 18:00 with addresses, access codes, and special-request notes.
- **Review monitoring.** New-review email forwarding (or manual paste); Claude drafts a tone-matched reply; host approves; cross-listing pattern surfacer flags recurring complaints ("3 guests in 30 days mentioned AC noise").
- **Approve-and-send queue UI.** Single screen showing every pending draft (price changes, messages, review replies, cleaning broadcasts) with a one-click approve.
- **Cost transparency.** Per-listing Claude spend visible per day and per month.

### Out of scope (deferred)
- **Direct Airbnb / Booking partner-API integration** — getting Hosting Tools API access is a multi-month vendor process and may be denied for sub-50-listing operators. Email-forwarding + iCal is the practical-MVP integration shape.
- **Channel manager features** (cross-portal calendar sync, push-pricing back to portals automatically) — that's a different product class.
- **Auto-send.** Pricing and messaging mistakes are expensive (wrong price = lost revenue, wrong tone = bad review). Always co-pilot.
- **Multi-tenant SaaS** — auth, Stripe Connect billing, per-tenant isolation. Architecture allows it; MVP doesn't ship it.
- **Native mobile app.** Approve-and-send is a single web page; works on phone via PWA.
- **Payment processing for guests.** The portal (Airbnb/Booking) keeps owning that.
- **Smart-lock / IoT integration** for access codes — manual code per listing is enough for MVP.

## Architecture

```
┌────────────────────────────────────────────────────────┐
│  iCal sync worker (every 30 min)                        │
│   → bookings table; turnover events                    │
└────────────────────────────┬───────────────────────────┘
                             │
┌────────────────────────────┼───────────────────────────┐
│  Email ingest webhook (Postmark Inbound Parse)         │
│   → guest_messages | reviews | system_alerts          │
└────────────────────────────┬───────────────────────────┘
                             ▼
                    ┌──────────────────┐
                    │  Postgres        │
                    │  on Fly.io       │
                    └────────┬─────────┘
                             │
   ┌─────────────────────────┼─────────────────────────┐
   ▼                         ▼                         ▼
┌─────────────┐ ┌──────────────────────┐ ┌────────────────────┐
│ Pricing eng │ │ Messaging draft eng  │ │ Reviews + cleaning │
│ (Rentcast + │ │ (Claude Sonnet,      │ │ (Claude Sonnet     │
│  Claude)    │ │  guest-language aware)│ │  + Telegram bot)   │
│  daily 06:00│ │  on inbound webhook  │ │                    │
└──────┬──────┘ └──────────┬───────────┘ └─────────┬──────────┘
       │                   │                       │
       └───────────────────┴───────────────────────┘
                             ▼
                    ┌──────────────────┐
                    │  Approve-and-    │
                    │  send queue UI   │
                    └────────┬─────────┘
                             │ host clicks "approve"
                             ▼
                    ┌──────────────────┐
                    │  Outbound:       │
                    │  - clipboard for │
                    │    Airbnb paste  │
                    │  - Telegram push │
                    │  - email reply   │
                    └──────────────────┘
```

- **Backend:** FastAPI on Fly.io (~$5/mo) — public webhook reliability matters more than NAS-internal hosting here. Postgres on Fly.io (free dev tier or $7/mo prod).
- **Inbound email:** Postmark Inbound Parse ($15/mo) or SendGrid Inbound Parse (free tier). Each host gets a unique forwarding address (`host-{slug}@in.yourdomain.com`).
- **iCal sync:** apscheduler cron every 30 min; standard `icalendar` Python lib.
- **AI:** Claude Sonnet 4.6 for all drafting (messages, prices, review replies). Optional Haiku for cleaning-digest summarization.
- **Comps data:** Rentcast Pro (~$74/mo) — short-term rental comps + occupancy. AirDNA is the alternative; pick one.
- **Cleaning notifications:** Telegram Bot (free) or WhatsApp Cloud API (free for first 1k convos/mo).
- **Frontend:** SvelteKit static export, served from Fly.io app; or plain HTMX page if you want zero build step.
- **Observability:** Self-hosted Langfuse (existing in your stack).

## Required components and choices

| Component | Choice | Reasoning | Cost |
|---|---|---|---|
| Backend hosting | **Fly.io** | Public webhook reliability + EU/US regions | ~$5/mo |
| DB | **Fly.io Postgres** (or Supabase free) | Single tenant, low volume | $0–7/mo |
| Comps data | **Rentcast Pro** | STR-focused, US + select EU coverage | ~$74/mo |
| Comps alt | AirDNA | Wider EU coverage, more expensive | $50–200/mo |
| Inbound email | **Postmark Inbound Parse** | Reliable; per-host addresses; webhook-friendly | ~$15/mo |
| Inbound email (free) | SendGrid Inbound Parse | Free tier exists; less polished | $0 |
| AI | **Claude Sonnet 4.6** | Quality of drafted reply is the whole product | Pay-per-token |
| Optional | Claude Haiku 4.5 for cleaning digest | Bulk lower-stakes formatting | Pay-per-token |
| Cleaning notify | **Telegram Bot** | Free; cleaners likely already have Telegram | Free |
| Cleaning notify alt | WhatsApp Cloud API | Better LatAm reach; 1k free convos/mo then per-msg | $0–10/mo |
| iCal parser | `icalendar` Python lib | Standard | Free |
| Translation | Claude native multilingual | No separate translation API needed | Pay-per-token |
| Optional PMS integration | **Hospitable.com API** | If host already uses Hospitable, message-API access becomes legal | Host pays Hospitable |
| Observability | **Self-hosted Langfuse** | Cost-per-listing traceable | Free |
| Future billing (post-MVP) | Stripe Connect | Pay-out flow when going multi-tenant | 2.9% + €0.25/txn |

## Implementation steps

### Phase 1 — Skeleton + Fly.io deploy (1 day)
1. New repo. Python 3.12 + FastAPI + apscheduler + asyncpg + SQLAlchemy + pydantic.
2. `flyctl launch`; provision Postgres; configure `secrets` for `ANTHROPIC_API_KEY`, `RENTCAST_API_KEY`, `POSTMARK_TOKEN`, `TELEGRAM_BOT_TOKEN`.
3. Schema:
   - `hosts(id, email, telegram_chat_id, created_at)`
   - `listings(id, host_id, name, address, lat, lng, bedrooms, base_nightly_price, currency, ical_url, portal enum('airbnb','booking','vrbo','direct'), house_tone_md, house_rules_md, access_code_template, created_at)`
   - `bookings(id, listing_id, guest_first_name, check_in, check_out, source_event_uid, status, raw_ical_md, created_at)`
   - `messages(id, listing_id, booking_id, direction enum('inbound','draft','sent'), language, body_md, draft_response_md, status enum('pending','approved','sent','dismissed'), claude_cost_usd, received_at, drafted_at, approved_at)`
   - `price_suggestions(id, listing_id, date, current_price, suggested_price, rationale_md, status enum('pending','approved','dismissed'), claude_cost_usd, created_at)`
   - `reviews(id, listing_id, booking_id, rating, body_md, draft_reply_md, status, claude_cost_usd, received_at)`
   - `cleaning_tasks(id, listing_id, turnover_at, access_code, special_notes_md, dispatched_at)`
   - `cost_log(id, listing_id, day, source enum('claude','rentcast','postmark'), units, cost_usd)`
4. Health endpoint + Langfuse client wired.

### Phase 2 — Listing registry + iCal sync (1 day)
5. Web UI: add a listing — name, address, iCal URL, base price, house tone (free Markdown), access code template.
6. Cron every 30 min: fetch each iCal, upsert bookings, derive turnover events (checkout day → next check-in day).
7. Acceptance: paste a real Airbnb iCal URL → bookings populate within 1 cycle.

### Phase 3 — Email ingestion + message drafting (2 days)
8. Configure Postmark domain; create per-host forwarding address pattern `host-{slug}@in.yourdomain.com`.
9. Inbound webhook handler classifies the email: **guest message**, **review**, or **system alert** (cancellation, new booking, etc.) by parsing Airbnb/Booking notification email patterns.
10. For guest messages: extract guest text + booking reference; insert `messages` row with `direction='inbound'`.
11. Draft generator: Claude Sonnet system prompt =
    > "You are the host's assistant. Draft a reply matching the listing's house tone (provided). Reply in the guest's language. Keep it warm, concrete, and short. **Do not invent house facts** beyond what is in the supplied house rules + access code template + listing description. If the guest asks something you cannot answer from the supplied context, mark `[NEEDS_HOST_INPUT]` in the draft."
12. Insert `direction='draft'` row pointing back to the inbound message.
13. Acceptance: forward 5 real Airbnb guest emails, get 5 plausible drafts in different languages.

### Phase 4 — Pricing engine (1.5 days)
14. Rentcast API integration: pull comparable nightly rates for the listing's coords + bed count.
15. Daily 06:00 cron per listing: fetch comps + occupancy + 30-day forward calendar; build a per-night context (DOW, lead time, local-events flag from a free API like PredictHQ free tier or skipped for MVP).
16. Claude Sonnet call: outputs JSON `{ date, suggested_price, confidence, rationale_md }` for each of the next 30 nights.
17. Persist as `price_suggestions`. Skip nights already booked.
18. Acceptance: suggested prices are within ±15% of host's intuitive baseline; rationale references real comp data.

### Phase 5 — Cleaning schedule + Telegram bot (1 day)
19. Telegram bot (BotFather → token). `/start` flow: cleaning team adds the bot to their group chat → host enters that `chat_id` in their listing.
20. Daily 18:00 cron: collect tomorrow's turnovers; build per-listing message with address, checkout-time, check-in-time, access code, special notes (from iCal description if any).
21. Send to the cleaning chat.
22. Acceptance: cleaning team confirms they received the right info for tomorrow.

### Phase 6 — Review intake + reply drafting (0.5 day)
23. Inbound email classifier already routes new-review notifications. Extract rating + text.
24. Claude Sonnet drafts a reply: thank the guest, address any specific feedback gracefully, end on a note that fits the house tone. **Never** rebut a complaint.
25. Persist as `reviews` row with `draft_reply_md`.
26. Cross-listing complaint surfacer: weekly cron; cluster review-bodies by topic via embeddings; if any cluster has ≥3 reviews in 30 days, flag in the UI.

### Phase 7 — Approve-and-send queue UI (1 day)
27. Single page showing all pending items grouped by listing:
    - Price suggestions (per night)
    - Message drafts (per guest)
    - Review replies
    - Cleaning broadcast preview
28. Approve = mark sent + copy reply to clipboard (host pastes into Airbnb/Booking inbox); Dismiss = mark dismissed.
29. Show today's per-listing Claude spend at the top of the page.

### Phase 8 — Optional Hospitable / PMS integration (1 day, only if host already uses one)
30. If the host has a Hospitable / Hostfully / Lodgify account, swap email-forwarding for the official messages API. Eliminates copy-paste.
31. Approve = direct API send; status updates flow back via webhook.

**Total build effort: ~7–8 days for shippable MVP without Phase 8; ~8–9 days with.**

## Cost estimate

### Fixed monthly (single host, 1–10 listings)

| Item | Cost |
|---|---|
| Fly.io app | ~$5/mo |
| Fly.io Postgres (prod tier — recommended over free dev tier for backups) | ~$7/mo |
| **Rentcast Pro** | **~$74/mo** |
| Postmark Inbound Parse (or SendGrid free) | ~$15/mo (or $0) |
| Telegram bot | $0 |
| Self-hosted Langfuse | $0 |
| **Subtotal — fixed** | **~$85–100/mo** |

### Per listing per month (Claude usage at typical activity)

| Workflow | Calls/mo | $/call | $/listing/mo |
|---|---|---|---|
| Message drafts (10–20 guest interactions × 2 sides) | 20–40 | ~$0.012 | $0.25–0.50 |
| Pricing (1× daily forward 30-day window) | 30 | ~$0.020 | $0.60 |
| Review replies | 1–3 | ~$0.020 | $0.02–0.06 |
| Cleaning digest (Haiku-class) | 30 | ~$0.005 | $0.15 |
| Pattern surfacer (weekly) | 4 | ~$0.020 | $0.08 |
| **Subtotal — Claude** | | | **~$1.10–1.40/listing** |
| Busy-season multiplier (more messages, more pricing recalcs) | | | up to **~$3–5/listing** |

### Monthly totals (typical)

| Listings | Fixed | Claude | **Total** |
|---|---|---|---|
| 1 listing (personal dogfood) | ~$95 | ~$1–5 | **~$95–100/mo** |
| 5 listings (small operator) | ~$95 | ~$5–25 | **~$100–120/mo** |
| 10 listings (early SaaS) | ~$95 | ~$10–50 | **~$105–145/mo** |

**The Rentcast subscription dominates at small scale** — that's why this only makes economic sense once you're either dogfooding a real listing or charging customers.

### Hosting alternatives
- **AWS** — Lambda + API Gateway + DynamoDB + EventBridge cron. **~$5–15/mo + Rentcast.** Sensible if you expect bursty webhook traffic.
- **Hetzner CX22** (€4.51/mo) + Docker + Postgres + Caddy. **~€4.51/mo + Rentcast.** Cheapest if you want one fixed-cost box.

### Service-tier projection (not MVP, for context)
- Per-listing pricing **$30–80/mo** — proven by Hostfully/Hospitable.
- Per listing gross margin: $30–80 − ~$5 Claude − amortized Rentcast = **~$20–70/listing/mo**.
- Break-even on the ~$95/mo fixed costs: **2–4 paying listings**.
- 10+ listings makes a real business; the shortlist's caution about scraping fragility doesn't apply here (this product uses official iCal + email + Rentcast + optional PMS APIs — all stable).

## Verification plan

After Phase 7, run a **30-day live test on one real listing**:
1. **Pricing.** Compare your suggested prices to (a) what you actually charged and (b) the booking outcome. Suggested price must be within ±15% of what felt right ≥80% of the time, and never recommend an obviously-wrong (e.g. peak-season holiday) low price.
2. **Messages.** Forward at least 30 real guest messages over 30 days. Drafted reply usable as-is (≤2 word edits) ≥70% of the time. Marked `[NEEDS_HOST_INPUT]` when the model genuinely can't answer.
3. **Multilingual.** At least one full conversation in a non-English language; drafts must read as native to a fluent speaker.
4. **Cleaning digest.** Cleaning team reports 0 misses across the 30 days; access codes always correct.
5. **Reviews.** Drafted reply approved with ≤2 edits ≥80% of the time; pattern surfacer correctly identifies a recurring complaint by week 4 (or correctly says nothing recurs).
6. **Cost reconciliation.** Langfuse + Rentcast dashboard + Postmark dashboard sum within 5% of `cost_log` over 30 days.
7. **Time-saved benchmark.** Self-report (or cleaning team report) total minutes saved per week vs the prior workflow. Aim for ≥3 hours saved/week per listing — below that, the product isn't earning its $95/mo fixed cost.

## Risks and open questions

- **No-host risk.** This MVP only earns its keep when there's a real listing producing real bookings. Without one, the pricing engine has no occupancy signal, the messaging engine has no real guests, and you're paying $95/mo to test prompts. **Build trigger gate this entirely on having a dogfood listing.**
- **Email-format fragility.** Airbnb / Booking notification emails change format occasionally. The classifier will break — keep a fallback that surfaces unparseable emails to the host's queue with a "manual classify" action.
- **Partner-API access.** If you ever want to skip the email-forwarding hack and use the official Airbnb API, the application takes months and may be denied. Plan around email forwarding indefinitely.
- **Hallucinated facts in messages.** Top failure mode: "the WiFi password is …" with a made-up password. Strict prompt + the `[NEEDS_HOST_INPUT]` escape hatch + house-rules + access-code-template injected into every prompt as the only allowed source of facts.
- **Pricing wrong-direction errors.** Suggesting a low price during a hidden local event (concert, conference) costs the host real money. Either integrate an events API or visibly warn the host that the model has no event signal so they sanity-check during peak weekends.
- **Multilingual tone drift.** Claude is good but not perfect at register/formality across languages. Have the host hold the bar — co-pilot, not auto-pilot.
- **Cleaning team adoption.** If cleaners ignore Telegram, the digest is wasted. Plan a 7-day onboarding window where you also send the digest by SMS (Twilio) until they've confirmed they're checking the chat.
- **Review-reply liability.** A poorly worded reply to a 1-star review can amplify damage. The MVP requires host approval; never ship auto-send for reviews regardless of how good the drafts get.
- **Privacy / GDPR.** Guest messages contain PII (names, dates, sometimes phone numbers). TLS in transit, encryption at rest, retention policy (delete inbound emails after 90 days), and a clear DPA when you go multi-tenant.
- **Service-tier business risk.** Per the shortlist, dogfooding is weak because you don't currently rent. The validation loop is slower than #1–7 in Table 2; don't commit to commercialization until you (or a close collaborator) has run it on one real listing for ≥90 days and felt the wins.
