# AI Product Shortlist 2026

## Context

You have four working personal projects (`little-mind-stories`, `nas-photo-organizer`, `streamer-tv`, `traveler-research`) plus a tools stack (ComfyUI + custom nodes, self-hosted Langfuse, planned Memex RAG, LoRA placeholder). All current projects share a common pattern:

- Multi-agent / pipeline architecture orchestrated through Claude API
- Tiered model usage (Haiku 4.5 for fast tasks → Sonnet 4.6 for judgment → Opus 4.6 only when needed)
- Cost observability via Langfuse
- Family/personal-first; no multi-user, billing, or auth

**Goal:** Identify candidate new projects that (a) deliver immediate family utility, (b) have a credible path to a paid service, and (c) realistically fit your existing stack (Claude API + Gemini Pro + ComfyUI + self-hosted infra).

The two tables below answer that, ordered by my estimate of combined utility × revenue potential.

---

## Existing project recap (so we don't duplicate)

| Project | What it does | Service-readiness gap |
|---|---|---|
| little-mind-stories | Brief → illustrated children's comic PDF | No multi-tenancy, no print-on-demand fulfillment, no billing |
| nas-photo-organizer | Vision-tiered NAS photo classification + dedup | NAS-coupled; would need cloud storage abstraction |
| streamer-tv | Self-healing IPTV channel orchestrator | Legal/licensing risk — hard to commercialize as-is |
| traveler-research | Multi-agent travel-folder → research report | Closest to product; needs UI, accounts, payments |

---

## Table 1 — New projects feasible with Claude API + Gemini Pro only

Ordered best → lowest by utility × revenue potential.

| # | App name | Short description | Family utility | Service angle |
|---|---|---|---|---|
| 1 | **Inbox Triage Agent** | Daemon that reads IMAP/Gmail, triages by urgency, summarizes threads, drafts replies, and surfaces actionable items into a daily brief. Uses Haiku for sort, Sonnet for drafts. | Reclaim 30–60 min/day of inbox load for you and your wife; shared family inbox (school, utilities) gets parsed automatically. | $10–25/mo SaaS — proven willingness-to-pay (Superhuman, Shortwave). Differentiator: cost-transparent, BYO-Claude-key tier. |
| 2 | **Document Vault & Auto-Filer** | Drop scans/PDFs/photos of bills, contracts, IDs, receipts → vision extraction → categorized, OCR-indexed, with auto-reminders for renewals (passport, insurance, warranty). Built on the same vision-tiering pattern as nas-photo-organizer. | Replaces "the kitchen drawer of paperwork" — wife/parents can drop a phone photo and forget it. | $5–15/mo per family. Wedge: extends NAS organizer beyond photos; same MCP-server pattern. |
| 3 | **Homework Helper & AI Tutor** | Photo of homework → grade-appropriate guided walkthrough (Socratic, not answer-dumping), with parent dashboard showing what was asked. Multi-modal (handwriting, diagrams) via Gemini for cheap OCR + Sonnet for pedagogy. | Direct family value as kids hit school age; parents stay in the loop. | $15–40/mo; massive existing market (Photomath, Khanmigo). Defensible niche: parent-controlled, language-localized. |
| 4 | **Family Memoir / Legacy Book Generator** | Voice notes + photos + interviews → AI-stitched illustrated family history book PDF. Reuses the little-mind-stories layout/PDF pipeline but for non-fiction with real photos. | Birthday/anniversary gift for parents/grandparents; preserves stories before they're lost. | $50–250 one-shot per book — premium emotional product. Print-on-demand via Lulu/Blurb API later. |
| 5 | **Receipt → Expense & Tax Pipeline** | Snap receipts (or auto-pull from email) → categorize, deduplicate, generate monthly expense reports + year-end tax-ready CSV/PDF. Adds Spanish/EU-VAT logic. | Wife's small purchases + your work expenses tracked without spreadsheets. | $8–20/mo; freelancers and small businesses pay readily. Lean wedge vs. Expensify because of EU tax knowledge. |
| 6 | **Meal Planner from Pantry Photo** | Photo of fridge/pantry → weekly meal plan honoring dietary prefs/allergies → grocery list deduped against current stock. Vision tiering identical to photo organizer. | Solves the daily "what's for dinner" tax; reduces food waste. | $5–10/mo; competitive space (Mealime, Whisk) but AI-native + multilingual + family-shared is differentiated. |
| 7 | **Vacation Photo-Book Auto-Generator** | Folder of trip photos → cluster by day/location → captions from EXIF + Claude → printable PDF photobook. Natural sibling to traveler-research (post-trip companion). | Every trip you take produces a memento with zero effort. | $15–50 per book; or $9/mo unlimited. Could integrate Lulu/Blurb later for physical print. |
| 8 | **Subscription & Bill Auditor** | Connect bank statement (CSV upload, no Plaid yet) → Claude identifies recurring charges, flags price hikes, drafts cancellation emails. | Most families bleed €30–80/mo on forgotten subs. | $15 one-shot scan, or $5/mo monitor. Easy viral hook: "we found €X for you in 5 minutes." |
| 9 | **Personal Newsletter / Daily Brief** | Curates RSS + saved YouTube channels + selected newsletters into one personalized morning email. Uses Memex-style RAG so you can ask follow-ups. | One coffee-time read instead of 10 tabs; tunable per household member. | $5–10/mo; thin margin alone but excellent retention/funnel into other apps. |
| 10 | **Family Calendar Coordinator** | Reads shared Google/Apple calendar → conflict detection, weekend planning suggestions (weather + traveler-research data), reminders, automated meal-plan/birthday-card triggers. | Becomes the "hub" tying every other family agent together. | $8/mo per family. Sticky once integrated, low churn. |
| 11 | **YouTube / Podcast Knowledge Library** | Subscribe to channels → auto-transcribe (Gemini Pro is cheap/free for this) → semantic index → ask questions across your watch history. Direct fit for the planned Memex tool. | Replaces the "I saw something about X last week" frustration. | $10/mo; upsell to teams (researchers, analysts). |
| 12 | **AI Birthday/Card/Gift Concierge** | Calendar-driven agent: detects upcoming family events → drafts personalized card text + ComfyUI illustration → optionally orders gift via Amazon affiliate links. | Nobody forgets a birthday again; saves 1–2 hrs per occasion. | $3/mo + affiliate commission. Low ARPU but viral — every recipient asks "where did this come from?". |
| 13 | **Wardrobe Stylist & Outfit Planner** | Photograph wardrobe once → daily outfit recommendations (weather + calendar + occasion). Vision tagging + small inference loop. | Kid school outfits, business-trip packing aid. | $5/mo; or freemium with affiliate fashion links. Niche but proven (Whering, Indyx). |
| 14 | **Language-Learning Companion** | Conversational practice with Sonnet, photo-translate vocabulary in the wild, kid-friendly stories generated on demand (reuses little-mind-stories engine). | Bilingual household support; kids practice via familiar story format. | $10–15/mo; strong LTV in language space. Differentiator: ties into your story generator. |
| 15 | **Job-Hunt Pipeline** | Paste job listing → tailored CV + cover letter + interview prep crib sheet, all versioned. Tracks applications and follow-ups. | Personal/family use whenever someone is looking. | $20 one-shot or $15/mo during search. High urgency = fast conversion. |
| 16 | **AI Code Reviewer for Indie Devs** | Self-hosted GitHub-app that reviews PRs with Sonnet + your house style. Runs your existing /review and /security-review skills against any repo. | Use it on your own projects. | $10–30/mo per dev; well-established willingness-to-pay (CodeRabbit, Greptile). |

---

## Table 2 — Higher-ceiling projects that require additional paid services

These cross a paid-API line, but the utility or revenue justifies it.

| # | App name | Required paid services | Description & why it's worth the spend |
|---|---|---|---|
| 1 | **Audiobook Studio for Kids' Stories** | ElevenLabs (~$22/mo Creator) or Cartesia | Narrates little-mind-stories output with cloned grandparent/parent voice. Massive emotional uplift; converts a free PDF into a $20–40 keepsake MP3+PDF bundle. Premium tier could ship audio + printed book. |
| 2 | **Smart Home Camera Vigilant** | Cloud storage (Cloudflare R2 ~$0.015/GB) + RTSP/HLS proxy; optional Twilio for SMS alerts | 24/7 vision monitoring of existing home cameras: package delivered, kid home from school, unknown person at door. Gemini Pro multimodal handles video frames cheaply. Replaces Ring/Nest subscriptions ($10/mo). Service revenue $10–20/mo per household. |
| 3 | **Restaurant / Activity Booking Agent** | Browserbase or Steel ($30–100/mo) for headless browser, optional 2Captcha | Books hard-to-get reservations (OpenTable, TheFork, Resy), kid activities, travel. Tied to traveler-research output. Concierge-tier service: $30–50/mo or per-booking fee. High willingness-to-pay among professionals. |
| 4 | **Real Estate / Rental Research Engine** | Apify or Bright Data scraping (~$50/mo entry); optional Google Maps API | Scrape Idealista/Fotocasa/Zillow → Claude scoring on commute, schools, noise, neighborhood. Personal house-hunt utility now; service to relocators/expats later at $50–100 per report or $30/mo. |
| 5 | **AI Receptionist for Small Businesses** | Twilio Voice + Programmable SMS (~pay-per-min); Stripe for billing | Answer calls/SMS with Claude voice agent, book appointments, capture leads. Sells to your wife's contacts (clinics, salons, autonomous workers) at €80–200/mo per business. Very high LTV, low CAC via word-of-mouth. |
| 6 | **Personal Health Records Aggregator** | Azure Document Intelligence or AWS Textract ($1.50/1k pages) for medical OCR; HIPAA-grade hosting if EU/US service | Photos of lab reports, prescriptions, vaccine cards → structured timeline + AI-flagged anomalies + appointment reminders. Personal value enormous (chronic conditions, kids' pediatrics). Service: $15–25/mo; regulatory care required (GDPR/HIPAA). |
| 7 | **Real-Time Conversation Translator / Note-Taker** | Deepgram or AssemblyAI streaming STT (~$0.40/hr); optional ElevenLabs TTS | In-person meetings/calls → live translation + summary + action items. Useful for any bilingual family situation (school meetings, doctor visits). Service: $15/mo; B2B upsell to small consultancies $50–100/mo. |
| 8 | **Property-Manager Co-Pilot** | Rentcast/Airbnb APIs (paid tiers), Stripe Connect | If you ever rent a room/apartment: dynamic pricing, guest-message auto-reply (multilingual), cleaning schedule, review monitoring. Niche but $30–80/mo per listing, sticky. |
| 9 | **AI Soundtrack / Score Generator for Stories** | Suno API (~$10/mo) or Udio | Adds custom music to little-mind-stories audiobook. Premium-tier upsell on top of #1 in this table. Margin-heavy. |
| 10 | **Trading / Investing Research Assistant** | Polygon.io or Alpha Vantage paid tier ($30–80/mo); optional Tiingo for fundamentals | Personal portfolio: news digest, earnings-call summaries, anomaly alerts, dividend tracker. Service: $25–50/mo retail tier. Compliance: must clearly disclaim non-advice. |

---

## Themes & cross-project leverage

Three platform plays would amplify everything above:

- **Shared "household agent" runtime.** Every Table 1 project (#1, #2, #5, #6, #10, #12) has an "always-on for one family" shape. Build one persistent agent harness on your NAS (Docker + SQLite + Langfuse) and ship apps as plugins. Lowers per-project cost massively.
- **Reuse the vision-tiering ladder.** Document Vault (#2), Receipt Tracker (#5), Meal Planner (#6), Wardrobe (#13) all reuse the Haiku→Sonnet→Opus escalation pattern from nas-photo-organizer. Extract that into an internal library.
- **Memex as the substrate.** Newsletter (#9), YouTube Library (#11), Health Records (Table 2 #6) all depend on grounded RAG. Ship Memex first; everything else gets cheaper.

## Service-readiness checklist (applies to whichever you pick first)

When converting any of these from family tool → service, you'll need (none of which exist yet in any of your projects):
- Auth & multi-tenancy (Clerk or Supabase Auth — free tier viable)
- Per-tenant Langfuse projects + cost caps to avoid runaway API spend
- Stripe billing + usage-based metering hook
- Web UI (Next.js + your existing FastAPI patterns)
- Privacy/GDPR posture: data residency in EU, deletion APIs, sub-processor list

## My recommendation

If picking *one* to start: **#1 Inbox Triage Agent** — highest immediate daily utility for you and family, fastest path to a paid SaaS, and validates the "shared household agent runtime" before you scale into the other apps. Closely followed by **#2 Document Vault** because it directly extends your strongest existing project.

## Verification plan

This is a research deliverable, not code. To validate, you would:
1. Pick 2–3 ideas from Table 1 and run a 1-week "kitchen test" — i.e., manually do the workflow for your family and time it.
2. For each pick, draft a 1-page scope doc using the Plan agent before committing build effort.
3. For Table 2 picks, get pricing quotes from the listed paid services and compute unit economics at 100 users before building.
