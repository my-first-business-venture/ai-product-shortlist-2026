# traveler-research — Install / Run MVP Plan

## Context

`traveler-research` is an existing 9-agent pipeline (per the [shortlist](ai-product-shortlist-2026.md)) that ingests a folder of unstructured travel notes, itineraries, bookings, and PDFs and produces a consolidated multi-section research report — costs, safety, quality, day-by-day itinerary, and family-essentials (hospitals, pharmacies, supermarkets, child-amenable hotel features). Output is one timestamped Markdown file plus seven JSON artefacts under `output/{slug}_{yyyymmdd_hhmm}/`. The core engine is production-grade for one-off personal use; the shortlist gaps ("UI, accounts, payments") are deliberately **out of scope** for this install/run MVP.

**Goal of the MVP:** Get the existing pipeline running on a developer machine end-to-end — fresh checkout to first successful report — with realistic per-run cost expectations and observability wired up.

## Scope

### In scope (MVP)
- Local install on a Windows or Unix workstation: Python 3.10+ venv, dependencies, Claude Code CLI, secrets.
- Configure `config.json` to point at a real travel folder.
- Run `python run_research.py` end-to-end and produce one valid report.
- Optional Langfuse self-hosted instance for cost tracing across runs.
- Optional Windows Task Scheduler (or cron) for automated runs.
- Validate the multi-agent retry / two-tier validation behaviour on a real input.

### Out of scope (deferred — they are the commercial-tier work)
- Web UI, mobile app, or any user-facing frontend.
- User accounts, multi-tenancy, project workspaces.
- Stripe / billing / per-report cost gating.
- Tests (none exist today — adding them is a separate effort).
- Docker Compose for the pipeline itself (Langfuse runs in its own Compose stack; the pipeline is a CLI).
- Non-Windows ergonomics (the project hardcodes a Windows Git Bash path; macOS / Linux work with one env var change).

## Architecture (recap)

Nine Claude Code agents under `D:/Repositories/traveler-research/.claude/agents/` orchestrated by `orchestrator.md`:

1. **`local-categorizer`** (Haiku 4.5) — reads every file in the travel folder → `data/categories.json`.
2. **`local-analyst`** (Haiku 4.5) — extracts destinations, flights, hotels, activities, children's ages → `data/analysis.json`.
3. **Five web agents in parallel** (all Sonnet 4.6):
   - `web-cost`, `web-security`, `web-quality`, `web-itinerary`, `web-family`.
   - Each uses Claude's built-in **WebSearch + WebFetch** tools — no separate search API keys.
4. **Two-tier validation per web agent:**
   - Tier 1: JSON-schema check by the orchestrator.
   - Tier 2: semantic gate by the `validator` agent (Haiku for non-cost, Sonnet for `web-cost`).
   - On failure, the orchestrator feeds back targeted feedback and retries (max **3** retries — `pipeline.max_validation_retries`).
5. **Consolidation** — orchestrator (Sonnet 4.6) reconciles cross-agent name drift, computes a 5-input weighted composite score (`0.25 safety + 0.25 quality + 0.20 family + 0.15 medical_access + 0.15 cost_inverse`), and renders the final Markdown.

Run shape: one-shot CLI. Either invoke the slash command `/research <folder>` interactively in Claude Code, or run `python run_research.py` (which wraps `claude -p "/research ..."` and logs the resulting cost to Langfuse).

## Required components and choices

| Component | Required? | Version / details | Cost |
|---|---|---|---|
| **Python** | Required | 3.10+ | Free |
| **Claude Code CLI** | **Required** | Installed and on `PATH` (`claude --version` works) | Free CLI; Anthropic API spend below |
| **Anthropic API key** | **Required** | `ANTHROPIC_API_KEY=sk-ant-…` | Pay-per-use, ~$1.50–$5 per report |
| **Git Bash for Windows** | Required on Windows | Default path `D:\Program Files\Git\bin\bash.exe` (hardcoded in [`run_research.py:30`](D:\Repositories\traveler-research\run_research.py#L30) — overridable via `CLAUDE_CODE_GIT_BASH_PATH`) | Free |
| **Travel folder** | Required | Any directory with notes / itineraries / bookings / PDFs; path set in `config.json` → `paths.travel_folder` | Free |
| **Langfuse (self-hosted)** | Optional | Existing self-hosted instance; default expected at `http://host.docker.internal:3000` per [`config.json:29`](D:\Repositories\traveler-research\config.json#L29) | Free |
| **`LANGFUSE_PUBLIC_KEY` / `LANGFUSE_SECRET_KEY`** | Optional | Generated in your Langfuse project | Free |
| **Windows Task Scheduler** (or `cron`) | Optional | For automated periodic runs | Free |

Python dependencies (per [`requirements.txt`](D:\Repositories\traveler-research\requirements.txt)):

| Package | Min version | Purpose |
|---|---|---|
| `anthropic` | ≥0.40.0 | Claude API client (used by `langfuse_logger.py`) |
| `langfuse` | ≥2.0.0 | Cost / trace observability |
| `opentelemetry-sdk` | ≥1.20.0 | OTLP tracing |
| `opentelemetry-exporter-otlp-proto-http` | ≥1.20.0 | OTLP HTTP exporter for Langfuse |
| `python-dotenv` | ≥1.0.0 | `.env` loading |

**Why no search API keys?** The pipeline uses Claude Code's built-in `WebSearch` and `WebFetch` tools rather than Tavily / SerpAPI / Brave / Browserbase. The cost of all web sourcing is therefore folded into the Sonnet token bill.

## Install steps

### Phase 1 — Workstation prerequisites
1. Install **Python 3.10+** (`python --version`).
2. Install **Git** for Windows; confirm `D:\Program Files\Git\bin\bash.exe` exists, or note your install path so you can override `CLAUDE_CODE_GIT_BASH_PATH`.
3. Install the **Claude Code CLI** (`claude --version`); log in (`claude login`) and confirm an interactive `claude -p "hello"` returns text.

### Phase 2 — Secrets and accounts
4. Create an Anthropic API key at <https://console.anthropic.com>; store the `sk-ant-…` value.
5. (Optional) Stand up self-hosted Langfuse via its own Docker Compose; create a project; generate `pk-lf-…` + `sk-lf-…`.
6. Create `.env` at the repo root:
    ```
    ANTHROPIC_API_KEY=sk-ant-...
    LANGFUSE_PUBLIC_KEY=pk-lf-...      # optional
    LANGFUSE_SECRET_KEY=sk-lf-...      # optional
    LANGFUSE_HOST=http://host.docker.internal:3000   # optional
    ```

### Phase 3 — Repo + virtual env
7. Clone or refresh `D:\Repositories\traveler-research`.
8. Create the venv:
    ```bash
    cd D:\Repositories\traveler-research
    python -m venv .venv
    .\.venv\Scripts\pip install -r requirements.txt
    ```
    (Unix: `python -m venv .venv && .venv/bin/pip install -r requirements.txt`.)

### Phase 4 — Configure the run
9. Edit [`config.json`](D:\Repositories\traveler-research\config.json):
   - `paths.travel_folder` → absolute path to your real travel folder.
   - `pipeline.max_validation_retries` → leave at `3` for first run.
   - `langfuse.enabled` → `true` if step 5 done, otherwise `false`.
   - `langfuse.host` → match your self-hosted instance.
   - `models.*` → leave defaults (Haiku for local agents + validator, Sonnet for web agents + orchestrator + validator-cost).
10. Sanity-check that the travel folder contains at least one note, one booking confirmation, and any PDFs you care about — agent #1 reads everything in there.

### Phase 5 — First run
11. Either run the slash command interactively from a Claude Code session opened in this repo:
    ```
    /research d:/path/to/your/travel-folder
    ```
    …or invoke the wrapped automated run:
    ```bash
    python run_research.py
    ```
12. Watch the progress ticker (default every 20 s — `pipeline.progress_interval_seconds`).
13. Expect ~**20 minutes** wall time (~970 s API time) for a typical 2-destination trip with 5 web agents in parallel.

### Phase 6 — Validate the output
14. Confirm a fresh directory exists at `output/{slug}_{yyyymmdd_hhmm}/` containing:
    - `{slug}_{yyyymmdd}_{hhmm}.md` — the consolidated report (~400+ lines).
    - Seven JSON files: `categories.json`, `analysis.json`, `costs.json`, `security.json`, `quality.json`, `itinerary.json`, `family.json`.
15. Open the Markdown report and check the four pillars:
    - Cross-agent name reconciliation worked (no duplicate hotel entries with slightly different names).
    - Composite score is present and ranks options consistently.
    - Family section has hospitals / pharmacies / supermarkets near each candidate stay.
    - Itinerary section has at least 3 day-by-day options with stroller / nap-time annotations if children's ages were detected.
16. If `langfuse.enabled: true`, open the Langfuse dashboard and confirm one trace per run with per-agent spans and a USD cost figure.

### Phase 7 — Optional automation
17. To run it on a schedule, point **Windows Task Scheduler** (or `cron`) at:
    ```
    D:\Repositories\traveler-research\.venv\Scripts\python.exe D:\Repositories\traveler-research\run_research.py
    ```
    A common cadence: weekly during active trip planning, otherwise on demand.
18. (Optional) Wire the existing Langfuse alerts to ping you when a run's cost exceeds a threshold (`>$5`).

## Cost estimate

### Per run (one report)
Observed real numbers (from the project's own `ENHANCEMENTS.md` measurements):

| Trip complexity | Sonnet input tokens | Sonnet output | Haiku tokens | Wall time | **$ per run** |
|---|---|---|---|---|---|
| Simple (2 destinations, ~5 options each) | ~1.0 M | ~40 k | ~150 k | ~16 min | **~$1.50–2.00** |
| Typical (Wave 2 baseline, 2 dest, family-mode) | ~1.5 M | ~63 k | ~355 k | ~20 min | **~$1.97** |
| Pre-tuning baseline | ~1.8 M | ~75 k | ~400 k | ~22 min | **~$2.44** |
| Complex (5+ destinations) | ~2.5–3.5 M | ~120 k | ~700 k | ~30+ min | **~$3–5** |

There is no token-budget cap or hard cost gate — the orchestrator enforces "≤8 WebSearch, ≤3 WebFetch per run" via prose rules in agent prompts, not code limits. Watch Langfuse for runaway runs.

### Monthly totals at realistic use

| Use pattern | Reports / month | Monthly cost |
|---|---|---|
| **Personal — occasional trip planner** | 1–2 | **$2–5/mo** |
| **Personal — active planning across multiple trips** | 4–6 | **$8–25/mo** |
| **Family — comparing 10+ trip options for one big trip** | 10+ | **$20–50/mo** |
| **Hardware / software / web search** | n/a | **$0** |
| **Optional Langfuse hosting** | n/a | **$0** (self-hosted on existing infra) |

### What drives cost up vs down
- **Drives up:** more destinations, more candidate hotels/activities, deeper itinerary days, more children (triggers extra family-section work), repeated validation retries.
- **Drives down:** fewer destinations, simpler family composition, well-scoped travel folder (fewer files to categorize), `max_validation_retries: 1` if you trust the agents.

### Hardware
- Workstation only — no NAS / VPS required. Runs anywhere with Python + Claude Code CLI.
- If you want a dedicated automation host: **Hetzner CX22** (€4.51/mo) is overkill for this; the workload is bursty and short-lived. A laptop with Task Scheduler is enough.

## Verification plan

After Phase 6:
1. **End-to-end success:** one run produces a Markdown report and seven JSON files with non-empty bodies.
2. **Composite ranking sanity:** cheapest option is not always rank 1 (proves the weighted formula isn't degenerating).
3. **Cross-agent reconciliation:** no duplicate hotels under near-identical names in the final report.
4. **Family-mode trigger:** if the input mentions a child's age, the report has a Family section with hospitals / pharmacies / supermarkets.
5. **Validation retries:** force a known-bad input (e.g., empty travel folder, or a folder with one corrupt PDF) and confirm the orchestrator retries up to 3 times then fails gracefully with a clear log line.
6. **Cost reconciliation:** Langfuse-recorded spend matches Anthropic console within 5% over a 7-day window.
7. **Reproducibility:** running the same folder twice produces semantically equivalent reports (cost may vary ±15% due to non-determinism, but no missing sections).

## Risks and open questions

- **No tests.** The audit found no unit, integration, or regression tests. A schema change in any agent's JSON output silently breaks the orchestrator. Adding a smoke-test fixture run is the highest-leverage next step beyond MVP.
- **Cost runaway potential.** No hard token budgets — a pathological input (huge travel folder, many destinations) could 5–10× the per-run cost. Watch Langfuse for the first few runs and consider adding a cost cap before automating.
- **Web sourcing fragility.** Built-in WebSearch/WebFetch covers Booking.com, Kayak, TripAdvisor, etc. by URL fetching. If those sites tighten anti-bot rules, agents may return thinner data — visible as more validation retries.
- **Windows-first.** `run_research.py` hardcodes a Windows Git Bash path; macOS / Linux works only with `CLAUDE_CODE_GIT_BASH_PATH` overridden (or path detection patched).
- **Claude Code CLI dependency.** Unlike pure-SDK projects, this needs `claude -p` available — that's a runtime dependency that's easy to miss when porting to a server / container.
- **Shortlist gaps remain unresolved.** UI, accounts, and payments are deliberately out of MVP. They are the work that turns this into a product — see the shortlist for context.
