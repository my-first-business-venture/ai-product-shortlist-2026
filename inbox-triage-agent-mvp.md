# Inbox Triage Agent — MVP Implementation Plan

## Context

Email is the highest-frequency daily-load tax on every adult in the household. Inbox Triage Agent is a background daemon that connects to your Gmail / IMAP account, classifies new mail by urgency with Haiku (cheap sort), summarizes important threads and drafts replies with Sonnet (judgment), and produces a single daily brief so you stop touching the inbox 40 times a day.

**Goal of the MVP:** Run unattended on your NAS for one or two real Gmail accounts (yours + the shared family account), produce an accurate triage + a useful morning digest, and never auto-send a reply (drafts only). Pure Claude-API build — no external paid services.

## Scope

### In scope (MVP)
- Connect to one or two Gmail accounts via OAuth (and/or generic IMAP).
- Poll for new mail every N minutes (or IMAP IDLE where supported).
- Per-message Haiku triage into {`urgent`, `important`, `FYI`, `newsletter`, `junk`}.
- For `urgent` / `important`: Sonnet summarizes the thread context.
- For actionable items: Sonnet drafts a reply, saved as a Gmail draft (or stored locally for IMAP).
- Daily brief: HTML email at a chosen hour with the day's summaries, action items, and links to drafts.
- Per-user daily budget cap on Claude spend (hard stop, not a warning).
- All state local on NAS (SQLite). No third-party data persistence.

### Out of scope (deferred)
- Auto-send replies — drafts only; human stays in the loop.
- Calendar/event extraction (handled by **Family Calendar Coordinator** — Table 1 #10).
- Multi-tenant SaaS (auth, billing, per-tenant Langfuse projects).
- Web UI for triage rule training / feedback loop.
- Mobile app — daily brief lands in your existing inbox.
- Search / RAG over historical mail (separate Memex project).
- Slack / Teams / chat integrations.

## Architecture

```
┌──────────────────────┐
│  Cron / IMAP IDLE    │   every N min or push
└──────────┬───────────┘
           ▼
┌──────────────────────┐
│ 1. Fetcher           │   Gmail API or IMAP → new message envelopes + bodies
├──────────────────────┤
│ 2. Dedup + persist   │   SQLite: have we seen this UID? store raw + headers
├──────────────────────┤
│ 3. Haiku triage      │   classify {urgent, important, FYI, newsletter, junk}
├──────────────────────┤
│ 4. Sonnet summarize  │   only for urgent/important; thread-aware
├──────────────────────┤
│ 5. Sonnet draft      │   only when actionable; saved as Gmail draft
├──────────────────────┤
│ 6. Budget guard      │   per-user daily $ cap → hard stop
└──────────┬───────────┘
           ▼
┌──────────────────────┐
│  Daily brief job     │   cron at 07:00 → render HTML → send to user
└──────────────────────┘

   All steps traced via self-hosted Langfuse.
```

## Required components and choices

| Component | Choice | Reasoning | Cost |
|---|---|---|---|
| Mail connection | **Gmail API** (primary), `aioimaplib` (fallback) | Gmail API gives labels, drafts API, and OAuth refresh — simpler than raw IMAP | Free |
| Auth | **Google Cloud Console OAuth client** | Required for Gmail API; install-mode credentials for personal use | Free |
| Triage model | **Claude Haiku 4.5** | Cheap, fast, good enough for 5-class sort | ~$0.80/$4 per MTok in/out |
| Summary + draft model | **Claude Sonnet 4.6** | Judgment work, thread-aware reasoning | ~$3/$15 per MTok in/out |
| Orchestration | Python 3.12 + asyncio | Matches existing stack | Free |
| HTTP client | `httpx` | Async, retry-friendly | Free |
| Anthropic SDK | `anthropic` Python | Official | Free |
| State store | **SQLite** | Single-user; zero ops | Free |
| Email templating | `jinja2` | Daily brief HTML | Free |
| Brief delivery | Send via Gmail API as a draft sent to self | Avoids needing SMTP creds | Free |
| Observability | **Self-hosted Langfuse** | Already running in your stack | Free |
| Container | Docker Compose on existing NAS | Reuses NAS infrastructure | $0 marginal |
| Backups | Restic → Hetzner Storage Box (optional) | SQLite is small; nightly restic snapshot | €3.81/mo if used |

## Implementation steps

### Phase 1 — Mail connection + auth (1 day)
1. Create a Google Cloud project; enable Gmail API; create OAuth 2.0 desktop-app credentials.
2. Run a one-time consent flow to get a refresh token; store in `secrets/google_oauth.json` on NAS.
3. Wrap Gmail API in a thin `MailClient` interface (`list_new`, `get_thread`, `create_draft`) so IMAP can swap in later.
4. Smoke test: list last 10 unread message IDs.

### Phase 2 — Persistence + dedup (0.5 day)
5. SQLite schema:
   - `messages(id, account_id, gmail_id, thread_id, from_, subject, snippet, received_at, raw_path, triage_label, triage_at, summary, draft_id, processed_at)`
   - `accounts(id, email, refresh_token_path, daily_budget_usd, created_at)`
   - `cost_log(id, account_id, day, model, input_tokens, output_tokens, cost_usd)`
6. Idempotent insert: skip if `gmail_id` already present.

### Phase 3 — Haiku triage (1 day)
7. Build a triage prompt: classify into `{urgent, important, FYI, newsletter, junk}` with a one-line justification. Few-shot 6 examples drawn from your real inbox.
8. Single Haiku call per message (envelope + first 1k chars of body).
9. Write label + justification to `messages` row.
10. Acceptance: ≥85% agreement with hand-labeled set of 30 messages from your real inbox.

### Phase 4 — Sonnet summary + draft (1.5 days)
11. For `urgent` / `important`: fetch the full thread (`get_thread`), strip quoted history, send to Sonnet with a `summarize_for_morning_brief` prompt → 2–3 sentence summary + a `needs_reply: bool` field.
12. For `needs_reply: true`: second Sonnet call with `draft_reply` prompt that takes the thread + your previous-reply style examples (from Sent folder, sampled once at install).
13. Save the draft via Gmail API `users.drafts.create`.
14. Store summary + `draft_id` on the message row.

### Phase 5 — Budget guard (0.5 day)
15. Before each model call, check `cost_log` for today's running total; abort and label the message `deferred_budget_exceeded` if cap hit.
16. After each call, log token usage + computed cost.
17. Default cap: $1.00/day per account. Tunable in `accounts` row.

### Phase 6 — Daily brief (1 day)
18. Cron job at 07:00 (timezone-aware) per account:
    - Pull yesterday's triage rows.
    - Render Jinja2 HTML: urgent block at top with summary + draft link, important block, FYI/newsletter counts collapsed.
    - Create a draft addressed to yourself titled `Morning brief — YYYY-MM-DD`, then send it (Gmail API `users.drafts.send`).
19. Acceptance: brief lands in inbox by 07:05 with no rendering errors over 7 consecutive days.

### Phase 7 — Observability + polish (0.5 day)
20. Wrap every Claude call with Langfuse trace (input, output, model, cost, latency).
21. Add structured logs (JSON) → NAS log volume.
22. Health-check endpoint (`/healthz`) returning last-poll timestamp + last-brief timestamp.
23. README with one-line restart command.

**Total build effort: ~5–6 days of focused work.**

## Cost estimate

### Assumptions (heavy personal user)
- ~50 messages/day reach the inbox
- ~10/day get Sonnet summaries (urgent + important)
- ~5/day get Sonnet drafts

### Per-account daily Claude spend

| Step | Volume/day | Tokens/call (in/out) | Model | $/day |
|---|---|---|---|---|
| Triage | 50 msgs | 500 / 50 | Haiku 4.5 | ~$0.03 |
| Summarize | 10 threads | 2,000 / 200 | Sonnet 4.6 | ~$0.09 |
| Draft | 5 threads | 2,500 / 300 | Sonnet 4.6 | ~$0.06 |
| **Total** | | | | **~$0.18/day** |

### Monthly totals

| Scope | Cost |
|---|---|
| Claude API (1 heavy account) | **~$5–6/mo** |
| Claude API (2 accounts: you + shared family) | **~$10–12/mo** |
| Infra on NAS | **$0 marginal** |
| Optional offsite backup (Hetzner Storage Box 1 TB) | **+€3.81/mo** |
| **MVP total (family use, 2 accounts, NAS-hosted)** | **~$10–12/mo + €3.81/mo backup** |

### Hosting alternatives (if you ever expose it publicly)
- **Hetzner CX22** (€4.51/mo) — Docker + SQLite stays simple; same daemon, just on a public box.
- **GCP Cloud Run + Cloud SQL micro** (~$10–15/mo at family scale, ~$25–40/mo at 100 users) — only worth it if you go multi-tenant SaaS.

### Service-tier projection (not MVP, for context)
- At $15/mo per user and ~$5/mo Claude cost per user → ~67% gross margin.
- Break-even on the SaaS tier: ~3 paying users covers infra (Cloud Run + Cloud SQL).

## Verification plan

1. **Auth smoke test:** complete OAuth flow, list last 10 unread, confirm refresh-token rotation works.
2. **Triage accuracy:** hand-label 30 real emails, run triage, confirm ≥85% agreement. Iterate on the prompt + few-shot until stable.
3. **Summary quality:** generate summaries for 10 known threads; have a human (you) score 1–5; mean ≥4.
4. **Draft quality:** generate 5 drafts; verify tone matches your prior replies; verify no hallucinated facts (no commitments you didn't make).
5. **Budget guard:** force a low cap ($0.05) and confirm the daemon stops calling Claude and labels remaining messages `deferred_budget_exceeded`.
6. **Daily brief:** schedule for 07:00, confirm delivery for 7 consecutive days with no rendering errors.
7. **Cost audit:** compare Langfuse-recorded spend to Anthropic console — agreement within 5%.

## Risks and open questions

- **Privacy.** Emails are the most sensitive personal data you process. Keep raw bodies on NAS only; do not log full bodies; redact PII before sending to Langfuse cloud (use self-hosted Langfuse, which you already have).
- **Gmail API quota.** Default 1B quota units/day per project — generous, but be aware of `users.messages.list` and `users.threads.get` costs if you ever scale to many users.
- **Hallucinated drafts.** Sonnet can invent facts. Mitigate with a strict system prompt: "draft must contain only information present in the source thread." Human review (drafts not auto-sent) is the real safety net.
- **OAuth verification.** If you ever go past 100 users, Google requires app verification — plan for that before announcing a SaaS tier.
- **Triage misses on urgent items.** A false-negative on a truly urgent email is worse than a false-positive. Bias the triage prompt toward over-classifying as urgent.
- **IMAP fallback.** Generic IMAP lacks Gmail's labels and drafts API — if you support non-Gmail accounts, drafts have to be stored locally and shown in the brief instead.
