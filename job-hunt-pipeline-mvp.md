# Job-Hunt Pipeline — MVP Implementation Plan

## Context

Job hunting is high-urgency, transient work — every applicant produces 10–40 applications over a few weeks, each requiring a tailored CV, a cover letter, and a quick interview-prep crib sheet. The default path (manually editing the CV per role) is the single biggest time sink in a job hunt. Job-Hunt Pipeline turns the loop into: paste a job listing → out comes a tailored CV + cover letter + interview prep, versioned, and the application is logged for follow-up.

**Goal of the MVP:** Get a single-user (you or anyone in the family who's currently job-hunting) tool working end-to-end on Cloudflare's free tier with a Supabase free Postgres for state. Tailoring quality must be high enough that the human only edits, never rewrites. No auto-submission, no scraping, no LinkedIn integration — those are out-of-MVP business surfaces.

## Scope

### In scope (MVP)
- One-user account (or two — you + spouse), magic-link auth.
- Persisted **base profile**: master CV, achievements list, optional portfolio links.
- **Per-application flow:** paste job listing text (or paste a URL we fetch and clean) → generate three artefacts in one click:
  1. Tailored CV (Markdown + downloadable PDF).
  2. Cover letter (Markdown + downloadable PDF).
  3. Interview-prep crib sheet (Markdown).
- **Versioning:** every regeneration produces a new revision attached to the same application; the prior revisions remain accessible.
- **Application tracker:** status enum (`drafted`, `applied`, `interviewing`, `rejected`, `offer`, `withdrawn`), free-text notes, optional follow-up date with a daily digest.
- **Cost transparency:** every generation shows the actual Claude spend in the UI; lifetime per-application cost shown on the application card.

### Out of scope (deferred)
- Auto-submission to ATS / Greenhouse / Workday / LinkedIn (legal + technical risk).
- LinkedIn / Indeed scraping (ToS minefield — paste-text + URL-fetch on public listings only).
- Browser extension or native mobile app.
- Multi-user SaaS, Stripe billing, team workspaces.
- Calendar integration for interview scheduling, automated email follow-ups.
- ATS keyword-scoring dashboards / "match scores".
- A vector index of past applications (nice-to-have, not MVP).

## Architecture

```
┌──────────────────────────────────┐
│  SPA on Cloudflare Pages         │   SvelteKit or Next.js static export
│  (login, base profile, app list, │
│   generate flow, tracker)        │
└────────────┬─────────────────────┘
             │  fetch
             ▼
┌──────────────────────────────────┐
│  Cloudflare Workers (Hono)       │   /api/generate, /api/applications/*,
│  Free tier: 100k req/day         │   /api/profile, /api/auth/*
└────────────┬─────────────────────┘
             │
   ┌─────────┼──────────────────┐
   ▼         ▼                  ▼
┌──────┐ ┌─────────┐ ┌─────────────────┐
│Anth. │ │Supabase │ │ PDF render      │
│Claude│ │ Postgres│ │ (Phase 2 only)  │
│Sonnet│ │ + Auth  │ │ Docker WeasyPrint│
└──────┘ │ + Stor. │ │ on NAS or Fly.io│
         └─────────┘ └─────────────────┘
```

- **Frontend (Cloudflare Pages, free).** SvelteKit static export keeps the bundle tiny; Next.js works equally well — pick whichever you prefer.
- **API (Cloudflare Workers, free tier 100k req/day).** Hono framework for the few endpoints we need. Workers cannot run WeasyPrint (it's Python), so PDF rendering is split out (see Phase 6).
- **AI (Anthropic Claude Sonnet 4.6).** Three calls per generate-click — CV, cover letter, interview prep — done in parallel from the Worker. Optional company-research pre-step adds one Sonnet call with `web_search` tool when the user opts in.
- **State (Supabase free tier).** Postgres for `profiles`, `applications`, `revisions`, `cost_log`. Supabase Auth for magic-link login. Supabase Storage (1 GB free) for any PDFs you cache.
- **PDF rendering.** Phase 1: emit Markdown only and let the user export client-side. Phase 2 (optional): a `weasyprint`-running Docker container on the existing NAS or Fly.io ($0–$5/mo).

## Required components and choices

| Component | Choice | Reasoning | Cost |
|---|---|---|---|
| Frontend hosting | **Cloudflare Pages** | Free static; SPA fits | Free |
| API runtime | **Cloudflare Workers** | Free 100k req/day; close to user; cheap to scale | Free at MVP scale |
| Web framework | **Hono** | Workers-native; tiny | Free |
| AI | **Claude Sonnet 4.6** | Quality tailoring is the whole product | Pay-per-token |
| Optional cheaper variant | **Claude Haiku 4.5** for interview-prep only | Trades depth for cost on the lowest-stakes artefact | Pay-per-token |
| DB / auth | **Supabase free tier** | Postgres + magic-link auth + 1 GB storage in one product | Free at MVP scale |
| ORM / SQL | `postgres.js` direct or **Drizzle ORM** | Workers-compatible | Free |
| Markdown renderer (UI) | `markdown-it` or `marked` | Sandboxed render in browser | Free |
| PDF rendering (Phase 2) | **WeasyPrint** in a tiny Python Docker container on NAS / Fly.io | Best CSS support, MIT-friendly | $0 NAS / ~$5/mo Fly.io |
| Job-listing URL fetch | Worker `fetch()` + Mozilla Readability port | Public listings only; paste-text always available as fallback | Free |
| Observability | **Self-hosted Langfuse** (already in your stack) | Cost + trace per generation | Free |
| Email (digest, magic link) | **Resend free tier (3k/mo)** | Magic link + daily follow-up digest | Free |

## Implementation steps

### Phase 1 — Skeleton + auth (1 day)
1. `pnpm create svelte@latest` (or Next.js) — pick TypeScript, Tailwind.
2. `wrangler init` for the Workers backend; set up `wrangler.toml` with `[[d1_databases]]` (or skip — we use Supabase) and a `secrets`-driven `ANTHROPIC_API_KEY`, `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`.
3. Create a Supabase project; enable Email magic-link auth; record `anon` and `service_role` keys.
4. Schema:
   - `profiles(id, user_id, full_name, headline, base_cv_md, achievements_md, links_json, updated_at)`
   - `applications(id, user_id, company, role, source_url, listing_text, status, follow_up_at, notes_md, created_at, updated_at)`
   - `revisions(id, application_id, kind enum('cv','cover','prep'), content_md, model, prompt_hash, cost_usd, created_at)`
   - `cost_log(id, user_id, day, model, input_tokens, output_tokens, cost_usd)`
5. Wire magic-link login flow front-to-back; deploy to Pages + Workers; confirm round-trip.

### Phase 2 — Base profile (0.5 day)
6. Profile editor page: textarea for base CV (Markdown), achievements bullet list, optional links (LinkedIn / portfolio / GitHub).
7. Validation: base CV ≥200 chars before any generation is allowed.

### Phase 3 — Job-listing intake (0.5 day)
8. New-application modal accepts either:
   - Pasted listing text (preferred, robust).
   - A URL — Worker fetches with a 10 s timeout, runs Readability to strip nav/footer/script, returns the cleaned plain text.
9. On submit, insert an `applications` row in status `drafted`.

### Phase 4 — Generation pipeline (2 days)
10. POST `/api/generate` with `{ application_id }`. Worker:
    - Loads the user's `profile` and the application's `listing_text`.
    - Optional pre-step (toggle): Sonnet call with `web_search` tool to gather 5–10 facts about the company (used as context for cover letter + interview prep).
    - Three Sonnet calls in parallel via `Promise.all`:
      - **Tailored CV** — strict prompt: "Reorganize, re-emphasize, and trim the base CV to match the role. **Use only facts present in the base CV** — invent nothing."
      - **Cover letter** — uses base CV + listing + (optional) company facts.
      - **Interview prep crib sheet** — likely questions, suggested STAR-format answers grounded in achievements, gotchas tailored to the JD.
    - Inserts three `revisions` rows; logs token usage to `cost_log`.
    - Returns the three Markdown bodies + the cost figure.
11. Show all three side-by-side in the UI with copy-to-clipboard, regenerate (creates a new revision), and download-as-`.md`.
12. Tag the prompt with a `prompt_hash` so identical inputs don't burn budget on accidental double-clicks (return the cached revision instead).

### Phase 5 — Tracker (1 day)
13. Application list view: card per application with company, role, status badge, follow-up date, lifetime cost.
14. Status changes are direct enum updates — no workflow engine.
15. Daily cron (Cloudflare Cron Triggers, free): collect applications with `follow_up_at <= today` and `status in ('applied', 'interviewing')`; render an HTML digest; send via Resend to each user's email.

### Phase 6 — PDF rendering (1 day, optional for MVP)
16. Stand up a tiny Python service (FastAPI + WeasyPrint) in Docker on the NAS:
    ```dockerfile
    FROM python:3.12-slim
    RUN apt-get update && apt-get install -y libpango-1.0-0 libpangoft2-1.0-0 fonts-dejavu && rm -rf /var/lib/apt/lists/*
    RUN pip install fastapi uvicorn weasyprint markdown
    COPY app.py /app/app.py
    CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
    ```
17. Endpoint: `POST /pdf` with `{ markdown, css }` → returns `application/pdf`.
18. Worker forwards the user's signed request to it; result piped back to the browser.
19. If you skip Phase 6 for MVP, the UI offers a "Print to PDF" button using the browser's native print dialog with a print-only stylesheet — this is good enough.

### Phase 7 — Polish + observability (0.5 day)
20. Wrap every Claude call with Langfuse trace (input redaction for the listing PII, output, model, cost, latency).
21. Per-user daily budget cap (default $1.00/day) — hard stop with a clear UI banner.
22. Keyboard shortcuts: `g a` go to applications, `g n` new application, `c` copy current artefact.

**Total build effort: ~5–6 days for Phases 1–5 + 7 (no PDF service); ~6–7 days with Phase 6.**

## Cost estimate

### Per generation (one application = three artefacts)

| Artefact | Input tokens | Output tokens | Model | $ per call |
|---|---|---|---|---|
| Tailored CV | ~3,500 | ~2,000 | Sonnet 4.6 | ~$0.041 |
| Cover letter | ~3,500 | ~800 | Sonnet 4.6 | ~$0.022 |
| Interview prep | ~3,500 | ~2,500 | Sonnet 4.6 | ~$0.048 |
| **Subtotal — basic generate** | | | | **~$0.11/app** |
| (Optional) Company-research pre-step | ~1,000 | ~1,200 | Sonnet 4.6 + web_search | ~$0.05 |
| (Optional) Multi-pass refinement (2 passes) | doubles output | | Sonnet 4.6 | ~+$0.10 |
| **Subtotal — full quality** | | | | **~$0.25–0.30/app** |

The shortlist's headline figure of **$0.30–$0.80 per CV** assumes the higher-quality path (company research + a refinement pass) and a generous output budget. The MVP basic path is closer to **~$0.10/app**; switch on the optional steps when the role really matters.

### Monthly totals at realistic use

| Use pattern | Apps / month | $ Claude | $ Infra | **Total** |
|---|---|---|---|---|
| **Light browse** (you sometimes look) | 5 | ~$0.55 | $0 | **~$0.55/mo** |
| **Active hunt** (you or spouse, full effort) | 30 | ~$3.30 | $0 | **~$3.30/mo** |
| **Heavy hunt + company research on every app** | 30 | ~$9 | $0 | **~$9/mo** |
| Optional NAS-hosted PDF service | n/a | n/a | $0 marginal | $0 |
| Optional Fly.io PDF service | n/a | n/a | ~$5/mo | +$5/mo |

### Hosting alternatives if Cloudflare doesn't fit
- **NAS Docker** (€0 marginal) — Hono can run on Bun/Node on the NAS instead of Workers; same architecture, you keep the data on-prem. Trade-off: needs a public domain + reverse proxy if accessed off-network.
- **Hetzner CX22** (€4.51/mo) — same Hono backend on a single VPS with Postgres locally; replaces Supabase entirely if you want one-vendor simplicity.

### Service-tier projection (not MVP, for context)
- At **$15/mo** during-search SaaS pricing and ~$3 of Sonnet cost per active user/month → ~80% gross margin.
- Break-even on Cloudflare + Supabase free tiers: **1 paying user**.
- LTV is genuinely tiny (search lasts 4–8 weeks) — the unit economics work but acquisition cost has to be near-zero (organic, viral, or affiliate).

## Verification plan

After Phase 5:
1. **End-to-end success.** Paste a real job listing for a role you'd actually apply to, generate all three artefacts, read them critically. The CV must contain only facts from your base CV — no inventions.
2. **Versioning.** Edit your base profile; regenerate the same application; confirm the prior revision is preserved and the new one shows up.
3. **Tracker.** Mark an application `applied`, set a follow-up date for tomorrow, run the cron manually, confirm the digest email lands.
4. **Cost dashboard.** UI shows per-application lifetime cost; sum across the month matches Langfuse + Anthropic console within 5%.
5. **Budget cap.** Set the cap to $0.05 in the DB; trigger a fourth generation; confirm a hard stop with a clear UI message.
6. **Cold start.** Sign up as a brand-new user, fill in profile, generate one application — total time from signup to printed CV must be <5 minutes.
7. **Quality bar.** Have someone in the family read three real generations and grade 1–5 on "would you submit this without rewriting?". Mean ≥4 before declaring MVP done.

## Risks and open questions

- **Hallucinated achievements.** The biggest failure mode: the model invents experience you don't have. Mitigate with a strict system prompt ("only facts present in the base CV"), a final guard prompt that asks the model to list any facts it added that weren't in the source, and the human review step (drafts only, never auto-submit).
- **Job-listing scraping.** Many sites (LinkedIn, Indeed) actively block scraping. Treat the URL-fetch path as best-effort and **always** keep the paste-text path. Don't bake site-specific scrapers — they break.
- **Privacy.** CVs contain home addresses, phone numbers, sometimes ID numbers. Use Supabase row-level security so each user only reads their own rows; redact PII before sending to Langfuse if you ever centralize observability.
- **Cost runaway on heavy users.** A bored user clicking "regenerate" 50× burns budget fast. Per-user daily cap is the safety net; the `prompt_hash` cache prevents the most common abuse.
- **Quality drift between roles.** A CV tailored for SRE work may strip context that matters for a Data Eng role. Encourage the user to keep a long, comprehensive base CV — the tailoring is what trims, not what expands.
- **Workers + WeasyPrint mismatch.** Workers can't run Python — don't try. Either ship a separate Docker container or just emit Markdown and let the user print-to-PDF.
- **Transient LTV is the business risk, not the build risk.** A search lasts weeks. Plan for low ARPU, high churn, low CAC if commercializing — see the shortlist row for the rationale.
