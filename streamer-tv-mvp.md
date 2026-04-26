# streamer-tv — Install / Run MVP Plan

## Context

`streamer-tv` is an existing self-healing IPTV channel orchestrator (per the [shortlist](ai-product-shortlist-2026.md)). Six specialized agents probe channels every 15 minutes with `ffprobe`, find replacement URLs from free public M3U sources, match them against your channel list (fuzzy first, Claude only when ambiguous), validate the candidate, and rewrite an M3U file that Threadfin serves to Plex / Jellyfin / Emby. Production code is complete (Phases A–D in the project's own [implementation-plan.md](D:\Repositories\streamer-tv\implementation-plan.md)).

**Goal of the MVP:** Deploy the existing system end-to-end on a Synology DS220+ (or any Docker-capable NAS), wire it to a media server, and validate that channel self-healing works on real failures. No code changes — only deployment and configuration.

## Scope

### In scope (MVP)
- Stand up the four-container Docker Compose stack (orchestrator + searxng + threadfin + sqlite-web) on a Synology DS220+ NAS.
- Provide all required secrets (Anthropic API key, SearXNG secret) and any optional ones (GitHub token, Langfuse keys, notification webhook, Threadfin reload URL).
- Populate `config/channels.json` with your channel list and any BYO paid-provider failover URLs.
- Wire Threadfin to Plex / Jellyfin / Emby and confirm playback.
- Validate the self-healing loop end-to-end by deliberately breaking a channel URL.
- Optional observability: Langfuse cost tracing + webhook notifications for healing / quarantine events.

### Out of scope (deferred)
- Phase E items from the project roadmap: EPG (xmltv) merging, bandwidth-aware probing, "last watched" tracking, auto-alias expansion, URL blocklist learning, full pytest coverage, optional web UI.
- Commercialization (the shortlist's "legal/licensing risk" gap) — paid providers remain BYO.
- Code modifications, refactors, or new features.
- Multi-NAS or HA deployments.

## Architecture (recap)

Six single-responsibility agents under `D:/Repositories/streamer-tv/agents/`:

1. **StreamProbe** — `ffprobe` execution, OK/LAG/FAIL classification (≥500 kbps OK, 100–499 LAG, <100 or transport errors >5 FAIL).
2. **HealthMonitor** — concurrent bulk probing, tier-aware (`critical=15 min`, `standard=30 min`, `low=120 min` per `config/settings.json:10`).
3. **ProviderManager** — discovers M3U sources via SearXNG + GitHub, runs PENDING→ACTIVE→DEGRADED→BANNED→DEAD lifecycle, executes targeted re-discovery for repeatedly-failing channels (cooldown ladder `[0, 14, 28, 84, 168]` days regular, `[0, 3, 7, 21, 60]` premium).
4. **ChannelDiscovery** — searches inside active providers for candidates matching channel names.
5. **StreamValidator** — probes top candidates and returns the first OK URL.
6. **NameNormalizer** — only Claude-cost-driving agent. Tries exact + fuzzy match (≥80%) first; if ambiguous, one **batch** Haiku 4.5 call with all candidates; if Haiku confidence <80%, escalates to Sonnet 4.6.

Scheduler (`main.py`) runs the channel-health cycle every 15 min and the provider-update cycle every 6 h, plus a daily DB backup at 03:30. HTTP status endpoints live on port `8090` (`/health`, `/channels`, `/metrics`).

## Required components and choices

| Component | Required? | Version / spec | Cost |
|---|---|---|---|
| **Synology NAS** (or any Docker-capable Linux box) | Required | DSM 7.2+ recommended; ≥2 GB RAM, ≥10 GB free disk | Existing |
| **Container Manager** (Synology Package Center) | Required (NAS) | DSM bundled | Free |
| **Docker + docker-compose** | Required | Compose v2 | Free |
| **Anthropic Claude API key** | **Required** | `sk-ant-…` — used by NameNormalizer (Haiku 4.5 + Sonnet 4.6 escalation) | Pay-per-use, ~$1–15/mo (see below) |
| **SearXNG secret** | **Required** | Random 64-char string for `SEARXNG_SECRET` env var | Free |
| **VPN auto-connect on NAS boot** | Strongly recommended | Provider of your choice | $0–10/mo |
| **GitHub Personal Access Token** | Optional | `read-only public` scope; raises rate limit from 60 → 5,000 req/h | Free |
| **Langfuse (external instance)** | Optional | `LANGFUSE_PUBLIC_KEY` + `LANGFUSE_SECRET_KEY`; cost tracking + traces | Free tier sufficient |
| **Notification webhook** | Optional | Discord / Telegram bot / ntfy / Home Assistant URL via `NOTIFY_WEBHOOK_URL` | Free |
| **Threadfin reload URL** | Optional | Threadfin's update endpoint via `THREADFIN_RELOAD_URL`; auto-refresh after M3U writes | Free |
| **Paid IPTV provider** (BYO) | Optional | Per-channel `provider_urls[]` in `channels.json` for premium failover | $5–25/mo (your existing sub) |

Container layout (from [docker-compose.yml](D:\Repositories\streamer-tv\docker-compose.yml)):

| Container | Image | Port | Mem / CPU | Purpose |
|---|---|---|---|---|
| `iptv-orchestrator` | local Dockerfile | `8090` | 700 MB / 1.5 vCPU | All 6 agents + ffprobe + scheduler |
| `searxng` | `searxng/searxng:latest` | `8080` | 400 MB / 0.5 vCPU | Provider discovery (meta-search) |
| `threadfin` | `fyb3roptik/threadfin:latest` | `34400` | 300 MB / 0.5 vCPU | M3U / EPG proxy for Plex / Jellyfin |
| `sqlite-web` | `coleifer/sqlite-web:latest` | `8181` | 80 MB / 0.25 vCPU | Read-only DB inspector (optional) |

**Total runtime envelope: ~1.5 GB RAM, ~3 vCPU.** Comfortably within a DS220+ (2 GB RAM, 4-thread Realtek RTD1296).

## Install steps

### Phase 1 — NAS prerequisites
1. Update DSM to 7.2 or later.
2. Install **Container Manager** from Package Center.
3. Enable SSH (Control Panel → Terminal & SNMP).
4. Configure your VPN client and set it to auto-connect on boot.
5. Verify free disk: ≥10 GB on `volume1` (more if you want long log retention).

### Phase 2 — Secrets and accounts
6. Create an Anthropic API key at <https://console.anthropic.com> → store the `sk-ant-…` value.
7. (Optional) Create a GitHub PAT at <https://github.com/settings/tokens> with `public_repo` (or no scopes — public access is enough). Raises rate limit from 60 → 5 k req/h.
8. (Optional) Create a Langfuse project on your existing instance → copy `pk-lf-…` + `sk-lf-…`.
9. (Optional) Decide a notification channel (Discord webhook, Telegram bot, ntfy topic, Home Assistant webhook) → record its URL.
10. Generate a 64-char random string for `SEARXNG_SECRET` (`openssl rand -hex 32`).

### Phase 3 — Folders, files, configuration
11. SSH into the NAS and create the folder layout:
    ```
    /volume1/docker/iptv-stack/
      orchestrator/{app,config,data,output,logs}
      searxng/config
      threadfin/{conf,temp}
    ```
12. Copy the `streamer-tv` source tree into `/volume1/docker/iptv-stack/orchestrator/app/`.
13. Copy `config/settings.json` from the repo into `/volume1/docker/iptv-stack/orchestrator/config/` (the 14 seed providers are already populated).
14. Create `/volume1/docker/iptv-stack/orchestrator/config/channels.json` with your channel list — required fields per [README](D:\Repositories\streamer-tv\README.md#L342-L358):
    ```json
    {
      "version": "1.0",
      "channels": [
        {
          "id": "cnn-espanol",
          "display_name": "CNN en Español",
          "group": "News",
          "logo_url": "",
          "tvg_id": "CNNEspanol.us",
          "aliases": ["CNN Español", "CNN ES"],
          "provider_urls": []
        }
      ]
    }
    ```
    Add paid-provider URLs to `provider_urls[]` only if you have an IPTV subscription you want to use as primary failover.
15. Create `/volume1/docker/iptv-stack/.env` with:
    ```
    ANTHROPIC_API_KEY=sk-ant-...
    SEARXNG_SECRET=<your 64-char string>
    GITHUB_TOKEN=                 # optional
    LANGFUSE_PUBLIC_KEY=          # optional
    LANGFUSE_SECRET_KEY=          # optional
    NOTIFY_WEBHOOK_URL=           # optional
    THREADFIN_RELOAD_URL=         # optional
    ```

### Phase 4 — Bring up the stack
16. From `/volume1/docker/iptv-stack/`, run:
    ```bash
    docker-compose up -d --build
    ```
17. Verify all four containers are healthy:
    ```bash
    docker ps
    docker logs iptv-orchestrator --tail 50 -f
    ```
    Expect to see `Status endpoint listening on :8090` and an initial provider-update + first health cycle.

### Phase 5 — Wire Threadfin to your media server
18. Open `http://<NAS-IP>:34400` (Threadfin UI).
19. Add a new M3U source pointing to the file the orchestrator writes: `/home/threadfin/m3u/auto_premium.m3u` (mounted read-only inside the Threadfin container per [docker-compose.yml:30](D:\Repositories\streamer-tv\docker-compose.yml#L30)).
20. (Optional) Add an XMLTV EPG source if you have one (Phase E item — not provided by this stack today).
21. Map channels in Threadfin and copy its HDHR-tuner URL.
22. In **Plex** (or Jellyfin / Emby) → Live TV & DVR → Set up → **HDHomeRun** → enter Threadfin's tuner URL → finish discovery and channel mapping.

### Phase 6 — First-run validation
23. Watch a real channel from Plex / Jellyfin for ≥30 s — confirm playback.
24. Check the status endpoints from your laptop:
    ```bash
    curl http://<NAS-IP>:8090/health
    curl http://<NAS-IP>:8090/channels
    curl http://<NAS-IP>:8090/metrics
    ```
25. Open `http://<NAS-IP>:8181` (sqlite-web) and confirm the schema is in place: `providers`, `discovery_queue`, `channel_provider_attempts`, `channel_health_history`, `channel_cooldown` (with `quarantined_until`, `last_check_at`, `next_check_at`), `url_probe_cache`.
26. **Force a heal:** in `channels.json`, temporarily change a channel's `tvg_id` to a guaranteed-bad URL (or rename one of its aliases to a non-existent string) → restart the orchestrator → wait ≤15 min → confirm a `[QUARANTINE]` or successful re-heal log line and that the M3U was rewritten.

### Phase 7 — Optional observability
27. Enable Langfuse: in `settings.json` set `langfuse.enabled: true` and `langfuse.host` to your instance → restart container → confirm one trace per cycle in the Langfuse UI.
28. Wire `NOTIFY_WEBHOOK_URL` and trigger a test heal failure → confirm a Discord / Telegram / ntfy message lands.
29. Wire `THREADFIN_RELOAD_URL` (Threadfin's Update endpoint) → confirm Plex / Jellyfin sees fresh streams within 30 s of an M3U write.

## Cost estimate

### Fixed monthly
- **Hardware:** $0 (existing NAS).
- **Software stack:** $0 (all containers + SearXNG + Threadfin + sqlite-web are free).
- **Optional paid IPTV sub (BYO):** $5–25/mo external — only if you want premium-channel failover.
- **Optional VPN:** $0–10/mo — strongly recommended for any IPTV traffic.

### Per Claude call (NameNormalizer only)
The README's own cost summary example (line 416–420) shows real per-call cost:
- Haiku 4.5: ~**$0.0003 per channel-heal call** (one batch call covers all candidates for one channel).
- Sonnet 4.6 escalation: ~**$0.001–0.003 per call**, fires only when Haiku confidence <80%.

### Monthly Claude estimate (varies with fleet + failure rate)

| Scenario | Channels | Failure rate | Heal calls/day | Monthly Claude |
|---|---|---|---|---|
| **Family-light** | 25 | ~3% | ~70 | **~$0.50–1** |
| **Family-typical (recommended baseline)** | 50 | ~5% | ~240 | **~$2–5** |
| **Family-heavy / many premium channels** | 100 | ~8% | ~770 | **~$8–15** |

Add ~$1–3/mo for the 6 h provider-update cycle's premium-channel query generation when premium channels are queued.

### All-in monthly (family-typical baseline)
- Claude: **~$3–5/mo**
- Hardware / software / SearXNG / GitHub / Langfuse: **$0/mo**
- Optional VPN: **$0–10/mo**
- Optional paid IPTV sub: **$0–25/mo (BYO)**

→ **Realistic family-typical total: ~$3–15/mo** depending on optionals.

### Hardware alternative if you don't have a NAS
- **Hetzner CX22** (€4.51/mo, 2 vCPU / 4 GB / 40 GB EU) — fits the full 1.5 GB / 3 vCPU envelope with headroom; add Restic → Hetzner Storage Box (€3.81/mo) for offsite SQLite backups if it matters.

## Verification plan

Acceptance tests after Phase 6:
1. **Cycle latency:** a single `orchestrator.run_cycle()` completes in <2 min for ~50 channels (visible in `/metrics` `last_cycle_duration_s`).
2. **Forced heal:** breaking a channel URL results in a successful re-heal within one cycle (15 min) for a channel with healthy free-source coverage; `[QUARANTINE]` after `max_attempts=5` for one with no coverage.
3. **Threadfin freshness:** if `THREADFIN_RELOAD_URL` is set, Plex / Jellyfin see updated streams within 30 s of an M3U write.
4. **Playback continuity:** a healed channel plays in Plex without restarting the player.
5. **Cost reconciliation:** Langfuse-recorded spend matches Anthropic console within 5% over a 7-day window.
6. **Status endpoints:** `/health`, `/channels`, `/metrics` all return 200 with non-empty bodies; `last_cycle_at` is recent.
7. **DB backups:** `data/backups/` has a snapshot from the most recent 03:30 run.

## Risks and open questions

- **Legal posture.** Only the 14 public free seed providers and SearXNG-discovered free M3Us are touched by this stack. Paid providers are entirely **BYO** — you supply URLs in `channels.json`. The `manual_review.txt` file lists DRM/auth-gated channels that the system honestly refuses to scrape. Commercializing as-is requires either licensing deals or a strict "BYO-paid-provider" model.
- **Channel coverage varies by region.** Latin America and English-language coverage in the seed list is strong; niche regional channels may quarantine permanently. Add aliases in `channels.json` to help fuzzy matching.
- **Pluto TV JWT refresh.** The Pluto seed source uses time-bound URLs; the 6 h provider-update cycle handles refresh, but a long NAS downtime can leave stale URLs requiring a forced provider update.
- **SearXNG rate limits.** The self-hosted instance is fine, but the upstream search engines it queries may rate-limit during a long provider-discovery storm. Default `max_new_providers_per_run=10` keeps this in check.
- **Re-probe storms after NAS reboot.** First post-boot cycle probes everything at once; the `channel_initial_delay_s=60` and `max_concurrent_probes=2` defaults absorb this, but watch CPU on the first cycle.
- **VPN dependency.** If your VPN drops, providers may geo-block or rate-limit. Use a VPN client with kill-switch behavior on the NAS side.
- **Phase E gaps.** EPG is not auto-merged today — you provide your own XMLTV source in Threadfin if you want guide data. End-to-end pytest coverage is thin (`test_name_normalizer.py` skipped per README).
