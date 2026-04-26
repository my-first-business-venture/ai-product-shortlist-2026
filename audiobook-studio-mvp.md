# Audiobook Studio for Kids' Stories — MVP Implementation Plan

## Context

`little-mind-stories` already produces an illustrated children's-comic PDF from a brief. Audiobook Studio extends that pipeline by adding a narrated MP3 (optionally in a cloned family-member voice), bundled with the PDF as a $20–40 keepsake. Family use first; service tier comes later.

**Goal of the MVP:** From an existing `little-mind-stories` output, produce a chapterized MP3 narrated by a chosen voice profile, packaged with the source PDF and a cover image into a single deliverable folder/zip.

## Scope

### In scope (MVP)
- Trigger from a completed `little-mind-stories` job (scene-by-scene text + page metadata).
- Single narrator voice per book; voice profile selected at job submission.
- Voice cloning of a family member from a ~30 s sample (ElevenLabs Instant Voice Clone).
- MP3 output with chapter markers per scene/page.
- Cover image derived from existing book illustration (first page).
- Bundle: `book/` folder containing `audiobook.mp3`, `book.pdf`, `cover.jpg`, `metadata.json`.
- Optional Cloudflare R2 upload + signed URL for sharing with grandparents.

### Out of scope (deferred)
- Multi-character voice switching (single narrator only in MVP).
- Sound effects, ambient audio, background music (the **AI Soundtrack** project — Table 2 #9 — covers this as an upsell on top of MVP).
- Web UI for end-user upload (CLI / FastAPI endpoint reusing little-mind-stories patterns is enough).
- Multi-tenancy, accounts, billing, print-on-demand.
- Service-tier features: account isolation, per-tenant Langfuse, Stripe.

## Architecture

```
little-mind-stories job done
        │
        ▼
┌─────────────────────────┐
│ 1. Script extractor     │  Parses PDF/JSON output → list of (scene_idx, text, hints)
├─────────────────────────┤
│ 2. Prosody enhancer     │  Claude Sonnet adds SSML pauses / emphasis (optional)
├─────────────────────────┤
│ 3. Voice synthesizer    │  ElevenLabs API per-scene → mp3 chunks
├─────────────────────────┤
│ 4. Audio assembler      │  FFmpeg concatenates + writes ID3 chapter metadata
├─────────────────────────┤
│ 5. Bundler              │  Builds book/ folder; optional R2 upload
└─────────────────────────┘
        │
        ▼
   Deliverable (local NAS path + optional shareable URL)
```

## Required components and choices

| Component | Choice | Reasoning | Cost |
|---|---|---|---|
| Source pipeline | Existing `little-mind-stories` | Already produces structured story output; reuse | $0 |
| Voice synthesis | **ElevenLabs Creator tier** | Best quality, includes Instant Voice Clone, 100k chars/mo | **$22/mo** |
| Alt voice synthesis | Cartesia Sonic (deferred) | Cheaper pay-per-char, lower quality, no IVC | ~$0.30/100k chars |
| Audio processing | **FFmpeg** | Industry standard, local, handles concat + chapter markers | Free |
| Prosody refinement | Claude Sonnet | Inserts SSML pauses / emphasis tags before synthesis | ~$0.05/book |
| Job orchestration | Python + Pydantic + SQLite | Matches existing little-mind-stories stack | Free |
| Container | Docker on existing NAS | Reuses NAS infrastructure | $0 marginal |
| Distribution storage | Cloudflare R2 (optional) | Free egress; only needed for sharing | $0.015/GB stored |
| Email delivery (optional) | Resend free tier (3k/mo) | For sending download links | Free at family scale |

## Implementation steps

### Phase 1 — Plumbing (1–2 days)
1. Add an `audiobook-studio/` directory inside the existing `little-mind-stories` repo (or a sibling repo if you prefer separation).
2. Define a `JobInput` Pydantic model: `{book_id, source_pdf_path, scene_json_path, voice_profile_id}`.
3. Wire a CLI entry point: `audiobook-studio render <book_id>`.
4. SQLite schema for jobs: `id, book_id, voice_id, status, started_at, completed_at, output_path, error`.

### Phase 2 — Voice profile management (1 day)
5. ElevenLabs API integration (`elevenlabs` Python SDK).
6. CLI command `audiobook-studio voice clone <name> <sample.mp3>` → uploads to ElevenLabs IVC, stores returned `voice_id` locally.
7. CLI command `audiobook-studio voice list` → shows cloned voices with sample-text playback.

### Phase 3 — Synthesis pipeline (2 days)
8. Script extractor: read `little-mind-stories` scene JSON, produce per-scene narration text. Strip illustration-only scenes.
9. Optional prosody enhancer: prompt Claude Sonnet with the scene text + a system prompt asking for SSML `<break>` / `<emphasis>` tags. Skip if disabled.
10. Synthesizer: per-scene call to ElevenLabs API; save MP3 chunk to `tmp/<job_id>/scene_NN.mp3`. Retry on 5xx with exponential backoff.

### Phase 4 — Audio assembly (1 day)
11. FFmpeg concat: build `concat.txt` listing chunks, run `ffmpeg -f concat -safe 0 -i concat.txt -c copy out.mp3`.
12. Chapter metadata: build an ffmetadata file with `[CHAPTER]` blocks per scene (start/end timestamps), then `ffmpeg -i out.mp3 -i ffmeta.txt -map_metadata 1 -codec copy final.mp3`.
13. Validate output with `ffprobe` (duration > 0, chapter count == scene count).

### Phase 5 — Bundling and delivery (0.5 day)
14. Cover: extract page 1 of source PDF as JPG via `pdftoppm` or `pymupdf`.
15. Bundler: assemble `book/` folder with `audiobook.mp3`, `book.pdf`, `cover.jpg`, `metadata.json` (book title, author, voice profile name, generation timestamp).
16. Optional R2 upload via `boto3` with R2 endpoint; return a 7-day signed URL.
17. Optional Resend email with the signed URL.

### Phase 6 — Observability and polish (0.5 day)
18. Wire Langfuse traces around synthesis steps (already self-hosted).
19. Cost log: per-job record of total characters synthesized + estimated cost.
20. Smoke-test script that runs an end-to-end render against a known fixture book.

**Total build effort: ~5–6 days of focused work.**

## Cost estimate

### Fixed monthly
- **ElevenLabs Creator: $22/mo** — includes 100,000 characters and Instant Voice Clone.

### Per book (5-minute story, ~750 words, ~4,500 characters)
- Voice synthesis: included in Creator quota; effectively **~$1/book** amortized over ~22 books/month before exceeding quota (then $0.18 per 1k extra chars).
- Claude Sonnet prosody enhancement: ~$0.05/book.
- Storage: ~$0.0001/book on R2 (a 5-min MP3 ≈ 5 MB).
- Compute: $0 on existing NAS.

### Total MVP recurring
- **Family scale (1 household, ~5 books/month): $22/mo** flat — all costs absorbed by ElevenLabs base tier.
- **Heavy use (>22 books/month):** $22/mo + ~$0.18 per 1,000 extra characters synthesized.

### Service-tier projection (not MVP, for context)
- At $25 per bundle and $1.05 marginal cost, gross margin per sale ≈ **~96%** — voice synthesis is a small fraction of the price.
- Break-even on the $22/mo subscription: **1 sale/month**.

## Verification plan

1. **Smoke test:** synthesize a single paragraph in your own voice; listen for artifacts.
2. **End-to-end:** pick an existing `little-mind-stories` PDF, run `audiobook-studio render <id>`, listen to the full output on a phone with chapters showing in the player UI.
3. **Voice clone:** record a 30 s sample of a family member, clone, regenerate the same book, and verify recognition by a second listener.
4. **Edge cases:** test on (a) a short story (<3 scenes), (b) a long one (>10 scenes), (c) a dialogue-heavy one to spot prosody issues.
5. **Cost audit:** run 5 generations and confirm Langfuse + ElevenLabs dashboard agree on character counts.

## Risks and open questions

- **Voice cloning consent.** Any cloned voice must come from someone who has consented; document this in the CLI (warning prompt) and in any future UI.
- **ElevenLabs ToS.** Creator tier permits commercial use; double-check before charging for any bundle.
- **Quality ceiling for non-English.** ElevenLabs handles Spanish well, but verify on your actual stories before assuming.
- **PDF parser fragility.** If `little-mind-stories` changes its output schema, this pipeline breaks — keep the script extractor decoupled and unit-tested.
