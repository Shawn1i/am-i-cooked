# Am I Cooked — Systems Design Document

> **Status:** Final consolidated architecture, post-adversarial-review. Every "blocker" and "major" from the review is resolved inline; genuinely-open items are tracked in §8.
> **Audience:** CTO / principal eng. Read §1 and §2 for the bet, §5 for the contracts you build against, §8 for what we deliberately decided.

---

## 1. Executive Summary

**Am I Cooked** (working title VibeCheck AI) is a no-login web app that tells a user their romantic "Cooked %" with a person of interest, by analyzing uploaded text screenshots and/or full chat-log exports through the lens of MBTI cognitive functions, and drafts their next message in the target's linguistic dialect.

### The core architectural bet

**The browser is the system of record; the server is a stateless analysis function.** Raw chat content never crosses to our infrastructure in a form we persist. The client parses, scrubs, computes deterministic metrics, embeds, and selects; it sends up only a bounded, scrubbed, minimized payload; the server fans those out to Claude with model tiering and caching and returns derived JSON. The durable "portfolio" that powers the dashboard lives in `sessionStorage`, not in any server table. This inversion — *client owns data, server owns compute* — is the only way the locked "zero server-side persistence + durable dashboard" pair can coexist.

This bet is honest, not absolute. There are **two deliberate, scoped exceptions** to "the server sees nothing," both documented openly because the legal posture is "minimized + transient + un-persisted," not the indefensible "we can't see anything":

1. **Screenshot pixels are sent to Claude un-scrubbed** (regex cannot redact a face rendered as pixels). This is consent-gated and, for two-party/GDPR jurisdictions, the default-stronger posture (§7).
2. **Long chat-export analysis runs client-orchestrated** (browser fires N short server calls) so that **nothing derived-from-content is written to any server store, not even short-TTL KV.** We rejected the background-job-with-KV-result-store design precisely because it breached the invariant (§2.4, §6.1).

### The three hardest problems and how we solve them

**Problem 1 — Zero-persistence vs. a durable dashboard.** The dashboard implies history and a portfolio; the server stores nothing. **Resolution:** the portfolio is a derived, PII-free JSON object in `sessionStorage`, keyed by an ephemeral `sessionId`; "history" is *intra-session* evolution (drop a screenshot → drop the full export → watch Cooked% and Data-Weight move). Cross-tab continuity uses `BroadcastChannel` only — the durable-`localStorage` "courier" the original Frontend spec proposed is **removed** (review blocker: it could leave romantic evidence on a shared laptop if cleanup failed). See §2 and §4.

**Problem 2 — 50k-line exports can't go in one prompt; and they can't touch a server store.** **Resolution:** a two-stage pipeline. Stage 1 is 100% client-side deterministic metrics (zero tokens, dashboard renders instantly). Stage 2 embeds chunks **locally in-browser** (MiniLM via transformers.js) to rank emotional turning points, then uploads only the ~12 highest-signal scrubbed excerpts. The server-side "S2 Embedding Service" from an earlier draft is **deleted** — server embeddings would push the entire candidate set across the boundary, breaking both the 98% cost-reduction thesis and the zero-egress-of-unselected-content invariant. The long-running-job problem is solved by **client orchestration of short calls**, not a Vercel background function writing KV.

**Problem 3 — Cost control on an anonymous, viral, slot-machine-shaped product whose growth loop rewards the most expensive behaviors.** **Resolution:** (a) one model per tier pinned in a shared config, with **screenshot vibe extraction locked to Sonnet** (resolving a three-way Opus/Sonnet/Haiku contradiction); (b) **fixed, bounded thinking** on scored calls (not adaptive) so reasoning tokens are accounted and the score is reproducible; (c) a **single egress chokepoint — the Claude Broker (S6) — that enforces a per-session token cap before every Anthropic call**, set near the documented power-session ceiling (≈250k, not 1.5M); (d) a **visible Analysis-Credits meter** so the gamified evidence-drop loop is bounded by design, not by a cookie-clearable IP limit. See §6.

A fourth, non-negotiable theme runs through the whole document: **defense-in-depth must be deterministic enforcement, not a model instruction.** Where the original sections claimed a guarantee from "the model is told not to," we replaced it with code (deterministic share-card scrub, fixed tripwire regexes, injection-tainted-field discard).

---

## 2. High-Level Architecture

### 2.1 Mental model

A **stateless analysis function** wrapped in a **stateful browser**. The browser holds raw content, scrubs it, derives a portfolio, persists it in `sessionStorage`. The server is a pure-ish compute tier: it accepts already-scrubbed, bounded payloads, fans them to Claude through one broker, returns derived JSON, and forgets everything when the response flushes.

### 2.2 Component inventory

| # | Component | Tier | Responsibility | Stateful? |
|---|-----------|------|----------------|-----------|
| C1 | Shell / Portfolio UI | Client (React) | Dashboard, predictor, card; orchestrates the client pipeline **and the chat-export multi-call loop** | Reads sessionStorage |
| C2 | Scrubber | Client (Web Worker) | Regex PII redaction (text fields only) before egress | No |
| C3 | Metadata Cruncher | Client (Web Worker) | Deterministic `Stage1Metrics` over a full export (0 tokens) | No |
| C4 | Semantic Chunker | Client (Web Worker) | Turn-aware chunking + local prerank → candidate cap | No |
| C5 | **Local Embedder** | Client (Web Worker, transformers.js MiniLM) | Embeds candidate chunks **in-browser**; turning-point selection | No |
| C6 | Session Store | Client (sessionStorage + in-mem) | The only store; tab tether via BroadcastChannel | **Yes (the only store)** |
| C7 | OG/Result-Card | Client + Edge | Shareable card; OG image via Edge `@vercel/og` over a **server-scrubbed, HMAC-signed** payload | No |
| E1 | Edge Gateway | Edge Middleware | Turnstile gate, rate limit, payload-size cap, Zod schema gate, **text-field** PII re-scan | No (KV counters only) |
| S1 | Multimodal Adapter | Server (Node route) | Sends screenshot to Claude vision; extracts bubbles + vibe | No |
| S3 | MBTI Engine | Server (Node route) | Maps evidence → cognitive-function read | No |
| S4 | **Heuristic Scorer** (pure) + **Judge/Blend** (server-only) | Client+Server / Server | Split: pure heuristic runs both sides; LLM judge+blend is server-only | No |
| S5 | Response Predictor | Server (Node route) | Rizz Translator: intent → 3 drafts; success% computed in code | No |
| S6 | **Claude Broker** | Server (lib) | **The single Anthropic egress.** Model tiering, prompt cache, retry, token accounting, **and per-session cap enforcement** | KV counters only |

> **Deleted vs. earlier drafts:** the server-side **S2 Embedding Service** (embeddings now local-only, C5) and the **background-export job + KV result store** (replaced by C1 client orchestration). Both deletions are load-bearing for the privacy invariant — see §2.4.

### 2.3 ASCII topology

```
                              TRUST BOUNDARY
        CLIENT TIER (browser)        │        EDGE / SERVER TIER (Vercel)        EXTERNAL
 ───────────────────────────────────┼──────────────────────────────────────────────────────
  ┌──────────────────────────────┐  │   ┌───────────────────┐
  │ C1 Shell / Portfolio UI      │  │   │ E1 Edge Gateway   │   KV: rate/abuse + token
  │  orchestrates chat-export    │◄─┼──►│ Turnstile + RL +  │◄──────────────┐  counters ONLY
  │  multi-call loop             │  │   │ schema + TEXT     │               │  (never content)
  └─┬────┬────┬────┬────┬────────┘  │   │ PII re-scan       │          ┌────▼─────┐
    │    │    │    │    │            │   └────────┬──────────┘          │ Vercel KV│
    ▼    ▼    ▼    ▼    ▼            │            │                     │(eph. TTL,│
  ┌────┐┌────┐┌────┐┌────┐┌──────┐  │   ┌────────┼──────────────┐      │ counters)│
  │C2  ││C3  ││C4  ││C5  ││C6    │  │   ▼        ▼              ▼      └──────────┘
  │Scrb││Meta││Chnk││Embd││Store │  │ ┌─────┐ ┌───────┐   ┌─────────┐
  └─┬──┘└─┬──┘└─┬──┘└─┬──┘└──────┘  │ │S1   │ │S3 MBTI│   │S5 Predict│
    │     │     │     │             │ │Vibe │ │Engine │   │ (Rizz)  │
    └─────┴─────┴──┐  │ (embeds &   │ │(vis)│ └───┬───┘   └────┬────┘
                   ▼  │  selects     │ └──┬──┘     │            │
            ┌──────────────┐ locally)│    │   ┌────▼─────────┐  │
            │ only ~12     │         │    │   │S4 Heuristic  │  │
            │ scrubbed     │         │    │   │Scorer (pure) │  │
            │ chunks leave │         │    │   │ + Judge/Blend│  │
            └──────────────┘         │    │   │ (server-only)│  │
  ┌──────────────────┐               │    └───┴──────┬───────┘  │
  │ C7 Result Card   │◄───OG─────────┼──────────────►│          │
  │ (server-scrubbed │   (Edge)      │     ┌─────────▼──────────▼─────┐    ┌──────────────┐
  │  + HMAC-signed)  │               │     │ S6 Claude Broker         │───►│ Anthropic API│
  └──────────────────┘               │     │ tier+cache+retry+CAP     │    │ (Claude)     │
                                      │     └──────────────────────────┘    └──────────────┘
   RAW + scrubbed content lives here  │  ONLY scrubbed/bounded JSON + screenshot pixels cross ─►
   raw text never leaves un-scrubbed  │  nothing content-derived is written to disk/db
```

### 2.4 Trust boundary — what is true, stated honestly

There is **one trust boundary**, the Client↔Edge hop. The rules, corrected per review:

1. **Text-bearing payloads must be scrubbed before egress**, and E1 re-scans **text fields only** (`textLayer` is removed — see §4.2; the scanned fields are chunk text, predictor `intent`, `styleExemplars`, `vibeNote`). E1 rejects `422` on a residual email/phone hit in those fields.
2. **Image bytes are an explicit, consent-gated exception.** A regex cannot see a face rendered as pixels, so we do **not** claim the screenshot body is scrubbed. E1 exempts image bytes from the PII re-scan; the screenshot path is gated by `consentAck` and the stronger-by-default consent posture in §7. The §2-era claim that "everything crossing it must already be scrubbed" is corrected to: *everything crossing it is either (a) scrubbed text or (b) a consent-gated image.*
3. **KV holds counters only** — rate-limit windows and the per-session token tally. **No content, no derived metrics, no content↔IP linkage, ever.** This is now literally true because the background-job KV result store is deleted. The "clean hands" claim is re-scoped accordingly in §7.
4. **No content-derived data in viral artifacts** — the OG card payload is server-scrubbed by a deterministic filter *and* HMAC-signed (§4 / §7).
5. **Degradation:** with the network/LLM tier down, the client still renders the full deterministic dashboard (C3) and the tether still works; only LLM-derived layers degrade to a clearly-labeled "local-only estimate" (which, per §4 Scorer split, is the **heuristic-only** component, never the blended number).

---

## 3. End-to-End Data Flows

### 3.1 Path A — Screenshot (multimodal, low data weight)

```
1.  User completes 20-sec onboarding seed (rel state, both MBTIs, vibe, who-is-who hint).
2.  C1 reads image → ImageBitmap. NO client OCR. (Review blocker resolved: tesseract-wasm pass removed.)
3.  C2/canvas: re-encode to JPEG (strips EXIF/GPS), downscale ≤1568px, optional default-ON header crop.
4.  Consent gate: user must ack "this image may contain the other person's name/face" (§7).
        ===================== TRUST BOUNDARY =====================
5.  E1: Turnstile, rate-limit by sessionTok, size cap (≤8MB), Zod validate ScreenshotReq.
        E1 PII re-scan applies to seed.vibe (text) only; image bytes exempt by design.
6.  S6 Claude Broker: CHECK per-session token cap → reject 429 if over.
7.  S1: Sonnet vision (NOT Opus, NOT Haiku — locked, §6) emits VibeSignals + mapping/extraction confidence.
        If contains_injection_attempt=true → S1's other fields are DISCARDED/down-weighted, not just flagged (§7).
8.  S3 MBTI: seed + observed evidence → MbtiRead (low confidence; capped data_weight ≤ "moderate").
9.  S4 Heuristic Scorer (pure) + Judge/Blend (server) → ScoreResult, shrunk toward 50 by low dataWeight.
10. Response = AnalysisResult. NOTHING persisted server-side; request memory GC'd.
11. C6 appends Episode, writes sessionStorage; dashboard renders with "VIBES ONLY" Data-Weight badge.
```

### 3.2 Path B — Full chat-export (.txt, linguistic forensics, high data weight)

```
1.  User uploads export.txt (may be 50k+ lines).
2.  C1 stream-parses → Message[] in a Web Worker (never one giant string).
3.  C2 scrubs every Message.text in-place (span-map single pass, §4); raw discarded after scrub.
4.  C3 computes Stage1Metrics (0 tokens) → C1 renders the ENTIRE deterministic dashboard NOW.
5.  C4 windows into sessions, preranks, applies HARD candidate cap (≤150, §6) BEFORE any model call.
6.  C5 embeds candidates IN-BROWSER (MiniLM); scores turning points; selects top-M (M≈12, capped 8–20).
        ===================== TRUST BOUNDARY (only ~12 scrubbed chunks ever leave) ====
7.  C1 ORCHESTRATES the loop: for each batch of selected chunks, POST /api/forensics/map.
        (Client-orchestrated short calls — NOT a server background job. No KV result store.)
8.  E1 gates each call (Turnstile token, RL, size cap, Zod, text PII re-scan).
9.  S6 Broker: per-call cap check → Haiku map (cheap extraction) → MicroSummary[] returned to client.
10. C1 POSTs /api/forensics/reduce with Stage1Metrics + MicroSummary[] (tiny, already scrubbed).
11. S6 Broker: cap check → Sonnet reduce → RelationshipSummary (2–4 KB).
12. C1 runs S4 Heuristic Scorer locally for instant re-score; server returned the authoritative blended score.
13. C6 appends Episode, writes sessionStorage; rich dashboard (decay graph, landmine map, ghosting EWS).
```

The asymmetry: **Path A is LLM-heavy/signal-poor; Path B is local-heavy/LLM-targeted.** We never send 50k lines to a model and we never write a transcript chunk to a server store.

---

## 4. Subsystem Designs

### 4.1 Ingestion & Scrubbing (client, Web Worker)

**Contract:** files in → one `ScrubbedExportReq` or `ScreenshotReq` out, validated by Zod at the egress chokepoint, raw text destroyed.

**Parsing.** MIME sniffed from magic bytes (never `File.type`). Four formats (WhatsApp `.txt`, Telegram JSON, Discord JSON, iMessage `.txt`) + generic fallback, autodetected from a 64KB head sample with a confidence score. Everything normalizes to `Message` (§5). Streaming reads with a carry-buffer; JSON formats use a streaming tokenizer to keep peak memory flat. Limits: images 8MB, text 25MB.

**Scrubbing — order is load-bearing.** Stage A structured PII (email → url → phone → handle → address → long-number) applied via a **span map, single splice** (not sequential `String.replace`). Stage B identity-anchored names: the two participant labels are known from onboarding and scrubbed literally; third-party names use a capitalization heuristic + a 6KB Bloom filter of ~5,000 given names, **biased hard toward over-redaction**.

**Resolved review findings:**

- **Names are best-effort, stated in-product (blocker).** We add a prominent in-UI line: *"We remove emails, phones, and the names we can detect — name removal is best-effort, not guaranteed."* The body-scrub regex is built case-insensitively from participant labels **plus a small static diminutive map** (Alex/Alexander, etc.) — over-redaction is the stated safe direction, so adding diminutives is correct, not "too risky." `dataWeight` is **down-weighted when `nameScrubConfidence` is low**. For non-Latin scripts, the export feature is **hard-gated off** rather than shipping a known-leaking path behind a one-line notice.
- **Tripwire correctness (major).** The §C tripwire (a) uses **non-global regex clones (or resets `lastIndex`)** — the original `.test()` on `/g` regexes alternated true/false and silently missed every other call; this was a real bug, now fixed; (b) on a hit, **re-scrubs the offending span and re-checks**, hard-failing egress only if the re-scrub still trips (avoids a self-inflicted DoS when a legit 7+ digit order number matches); (c) the doc no longer claims the tripwire *guarantees* PII removal — it covers email/phone only and is a backstop, not a proof.

**Images (blocker — "image path is not scrubbed").** We stop calling the image path "scrubbed." Ingestion does: EXIF strip via **unconditional** canvas re-encode on every image egress path (independent of the optional header crop), downscale ≤1568px, **default-ON header crop for all flows** (not just ex/closure), and a consent interstitial that **explicitly names third-party PII transmission**. The honest limit is stated in-product: a face or name *in the conversation body* cannot be removed client-side; what we guarantee is metadata-gone, used-once, not-persisted-by-us, and (for two-party/GDPR) the stronger consent posture or feature degrade in §7.

### 4.2 Multimodal Screenshot Analysis (S1, server, Node)

**One multimodal call, no separate OCR stage.** Sender attribution and the vibe metrics are *visual-layout physics* (bubble side/color/area, timestamp gaps, double-texts, read receipts) that OCR destroys. Claude vision reads them natively and degrades gracefully (lower confidence) instead of emitting garbage.

**Model: `claude-sonnet-4-6` (locked).** This resolves the three-way Opus/Haiku/Sonnet contradiction. Sonnet is the defensible choice for a viral-scale, anonymous, cold-start endpoint: strong vision + spatial reasoning on bounded input, at ~1/2 the input and ~3/5 the output price of Opus. **Adaptive thinking is replaced with no/low thinking** here to bound output tokens — adaptive thinking bills unbounded reasoning at output rates and would silently blow the §6 budget on the highest-volume path. The Infra cost estimate (§6) is computed against Sonnet, so it is now self-consistent.

**Structured output:** forced `tool_choice: {type:"tool", name:"emit_vibe_extraction"}` with `strict:true`. Per the API reference this is GA and composes with a small request; we keep a parse-failure fallback (one repair retry, then a valid degraded `VibeExtraction` with `flags.likely_not_a_chat` or `unreadable_regions`) rather than treating `strict:true` as infallible.

**Seed delivery & caching.** The frozen extraction system prompt carries a `cache_control` breakpoint and is byte-identical across all users. The per-session seed goes in the **user turn after the breakpoint** (not a mid-conversation system message). Rationale: the API reference confirms `tool_choice:{type:"tool"}` is incompatible with `max_tokens:0` pre-warm, and mid-conversation-system caching benefit is model-gated; putting the seed in the user turn is the guaranteed-safe placement that preserves the shared prefix. We assert `cache_read_input_tokens > 0` on the 2nd+ call as a **runtime alert on the screenshot path** (not just an audit note).

**Injection containment (major — overstated as a guarantee).** The system prompt frames image text as data, and the forced single tool limits the output channel — but a successful injection can still **poison the extracted fields** (inflate enthusiasm, plant a fake `one_line_read`, suppress a name-scrub). Therefore: when `flags.contains_injection_attempt === true`, the orchestration layer **discards or heavily down-weights the entire extraction** before it reaches the scorer/predictor — it is not merely flagged. The in-model name-scrub backstop is documented as itself injection-defeatable and is never the only line of defense.

### 4.3 Linguistic Forensics Pipeline (C3/C4/C5 client + S-tier map/reduce)

**Stage 1 (client, 0 tokens).** Pure functions over `Message[]` produce `Stage1Metrics` (§5): turn segmentation, per-party median reply latency (median, not mean; awake-window filtered), text-volume ratio, double-text frequency, cold-start initiation %, punctuation/emoji decay slope, ghosting z-score. Renders the whole deterministic dashboard before any network call.

**Stage 2 (the only LLM part).** Turn-aware chunking on session boundaries, packed to ~1500 tokens. **Embeddings are local-only (C5, MiniLM via transformers.js)** — this is the linchpin that makes the privacy story and the cost story the same decision, and it resolves the embedding-location contradiction: the earlier server-side S2 is deleted. Turning-point score fuses local embedding drift + centroid distance + Stage-1 behavioral anomalies; force-include first chunk, last 2 chunks, and ghosting-border chunks; NMS to avoid adjacent duplicates.

**Hard pre-LLM caps (major).** Before any model call: **candidate windows capped at ≤150** (selected by Stage-1 signal magnitude), embedding calls are local so free but bounded by the candidate cap, and final selection K ∈ [8,20]. The single-job budget is a **pre-flight estimate** (`count_tokens` on the planned fan-out — see §6 for why only here) that degrades to fewer chunks *before* issuing calls, not a running tally discovered mid-loop.

**Map/reduce.** Map = `claude-haiku-4-5`, one short call per selected chunk, parallel, returns `MicroSummary` (no verbatim quote > 6 words = a second privacy layer). Reduce = `claude-sonnet-4-6`, one call, `Stage1Metrics` + `MicroSummary[]` → `RelationshipSummary`. **Both run through S6 and count against the cap.** Map/reduce calls use a **fixed small thinking budget**, not adaptive, so reasoning tokens are bounded and counted.

**Group chats / wrong self-label / tiny exports:** refuse export path for >2 senders (every asymmetry assumes a dyad); one-tap "swap me/them" re-runs Stage 1 cheaply and relabels Stage 2 with no re-call; <40 msgs skips embeddings and sends one chunk to Sonnet at `dataWeight: "light"`.

### 4.4 MBTI Engine & Cooked% Scorer (S3 + S4)

**Cognitive functions, not 16 type labels.** We score the target as a calibrated vector over the 8 Jungian functions (each in `[0,1]`, not a simplex). A `LEXICON_V3` constant (signal → function weights) lives in code, not the prompt, so the deterministic half never drifts with model updates.

**The Scorer split (blocker — pure-fn vs live LLM judge).** This is the single most important contract correction:

```ts
// Pure, runs IDENTICALLY client and server. No IO, no Claude. Unit-testable.
function heuristicScore(s: Signals): HeuristicScore;   // the 0.6-weight component, renormalized

// Server-ONLY. Folds in the Opus/Sonnet judge call. Cannot run client-side (no API key).
function judgeAndBlend(h: HeuristicScore, s: Signals): ScoreResult;  // authoritative blended number
```

- The client's offline/degraded score is **explicitly labeled as the heuristic-only component**, never presented as the blended Cooked%.
- The authoritative `ScoreResult.cookedPct` is **server-only**. §2/§4's earlier "S4 is one shared pure fn that reproduces `current` offline" claim is corrected: only the heuristic half is shared.
- Blend: `Cooked_raw = 0.6·heuristic + 0.4·judge`, then `shrink(raw, dataWeight)` pulls thin-data results toward 50.

**Reproducibility (blocker side-effect).** Because the headline number includes a live model judge, "same input → same number" is engineered as: (1) **fixed bounded thinking** on the judge call (adaptive thinking is non-deterministic *and* unbudgeted — both reasons to drop it here); (2) judge output **quantized to steps of 5**; (3) `inputHash = sha256(canonicalize(input))`; (4) **client memoization keyed by `inputHash`** so a reload/duplicate-tab re-render returns the *identical* cached `ScoreResult` rather than re-calling; (5) version pins (`schemaVersion`, `lexiconVersion`, model ID). Net: the blended number is reproducible *via memoization* for the same input, and the heuristic component is reproducible *by computation* anywhere.

**Judge call shape (model + thinking corrections).** `claude-opus-4-8` for the one high-stakes synthesis (the only place depth converts), `output_config:{effort:"medium", format:{json_schema}}`, **`thinking` omitted/disabled with a fixed budget rather than `thinking:{type:"adaptive"}`** (cost + determinism), `max_tokens` lowered to the actual schema size (≈2,000, not 16,000 — `JUDGE_SCHEMA` is small) and streamed only if needed. The model is given pre-computed numbers and told **not to recompute them** — it judges only what counters can't (reciprocity of interest, sarcasm, subtext).

**Third-party profiling / defamation (blocker).** All third-party psychological characterization (`landmines`, `ghostingRisk`, `blueprintProse`, function tendencies) stays **client-side and off every shareable surface**; the dashboard carries an explicit *"private, not for redistribution; entertainment, not an assessment of a real person"* banner on the analysis surface itself (not just the privacy policy). `derivedTypeLabel` (e.g. "ENFJ-leaning") is **dropped** — high defamation risk, marginal value. For GDPR jurisdictions, third-party profiling is **region-gated/stripped** (default-stronger; see §7). Data-Weight + shrinkage mitigate over-claiming but are explicitly **not** treated as a substitute for the missing legal basis to profile a non-consenting person.

### 4.5 Response Predictor / "Rizz Translator" (S5)

**The credibility moat: success% is code, not the model.** The model returns a self-assessed `pressureLevel` per draft and the drafts themselves; **server code** computes the percentage from the responsiveness signals via a logistic base receptivity × bounded MBTI/stage/pressure factors. The forced-tool `input_schema` **omits any success field** so the model structurally cannot emit a fake number.

**Reframed away from manipulation (blocker).** The original "overcome ghosting" optimization is **removed**: we do **not** tune pressure against a detected disengagement signal — that was the manipulative core. The predictor expresses the **user's own stated intent** in a register that meshes with the target's dialect; it does not optimize wording to defeat a specific person's withdrawal. Concretely:

- The **`chill` track no longer keys on `ghostingRisk`**; it exists as a low-pressure phrasing option the user may choose, not as a re-engagement weapon.
- **Per-message P(success) against a real person is relabeled** as a generic, clearly-hedged "vibe estimate," not an effectiveness score targeting an individual.
- When `relationshipState ∈ {post_ghost, rekindling}` **and** the intent is re-contact, the predictor **refuses** (a real detection mechanism for "contacting someone who may want to be left alone," which the original only listed without enforcing).
- The attestation (§7) is strengthened to cover *"the target has not asked me to stop contacting them."*

**Dialect cloning consumes fields that actually exist (major).** The forensics subsystem's `RelationshipSummary.theirDialect` is the **canonical `DialectProfile`** (§5) and is **enriched** to emit exactly what `buildDialectBlock()` reads: `casingStyle`, `topEmojis`, `avgWordsPerMsg`, `abbreviations`, `signaturePhrases`, `punctuationDensity`, `emojiRatePerMsg`, `vocabularyDensity`. There is one definition, produced by forensics, consumed by the predictor — no more references to fields that were never computed.

**Model & runtime:** `claude-sonnet-4-6` (persona-cloning/wit need it; Haiku falls flat, Opus is overkill), on **Node** (resolving the Edge-vs-Node contradiction — the Anthropic SDK's Node deps are the stated blocker; Node + streaming gives acceptable TTFB), streamed, with aggressive caching of the system prompt + per-session dialect prefix. A rare Haiku creep-classifier fires **only on regex-flagged intents** (Gate A), and **all retries/repairs/classifier calls route through S6 and count against the cap** (closing the "guard calls aren't budgeted" gap).

**Bounded by design, not just rate limits (major — slot-machine cost).** Regenerations are charged against a **visible Analysis-Credits meter** enforced at S6; the per-session generation cap is lowered to **~40**, and once the token cap is hit the meter is exhausted regardless of the IP rate limit (which a cookie-clear defeats). Output tokens (3 drafts × ~200 tokens each per regen) are explicitly in the §6 math.

### 4.6 Frontend / Session (C1/C6)

**Browser-as-database.** One versioned `sessionStorage` key (`vibecheck.portfolio.v1`); all mutation through a single `commit()` that serializes, enforces caps (FIFO-evict oldest snapshots under a 4MB soft ceiling, never the latest snapshot of any POI), bumps `updatedAt`, and broadcasts. `sessionStorage` (not `localStorage`) is the privacy feature: close the tab, it's gone — stated honestly in onboarding.

**Tether — BroadcastChannel only (blocker).** The `localStorage` "courier" is **removed**. A duplicate/new tab requests sync over `BroadcastChannel("vibecheck")`; a live tab replies with the portfolio; the new tab hydrates into its own `sessionStorage`. For the rare no-BroadcastChannel browser we **accept an empty new tab** rather than ever writing the portfolio to durable storage. (If a fallback is ever truly required it would be a one-time nonce behind `try/finally` + `pagehide` cleanup, never the portfolio body — but the default ships without it.)

**Recompute boundary.** Derived data (metrics, summaries, scores) survives reload and re-renders instantly with zero network. Raw evidence is discarded after analysis, so a reloaded evidence chip is **read-only ✓ analyzed**; new analysis requires new drops. **Paid-call caps:** beyond `MAX_POIS=12` / `MAX_SNAPSHOTS=25`, additional screenshot drops per POI switch to **local-only Stage-1 deltas** rather than firing new multimodal calls, and the Data-Weight meter **visibly caps** instead of inviting drops once the credits are spent (closing the gamification-drives-cost gap).

**Streaming protocol:** typed NDJSON-over-SSE (`token` | `metric` | `landmine` | `weight` | `done`); transient stream state updates gauges live, `done` commits the canonical snapshot (single source of truth).

### 4.7 Infra / Cost / Privacy (E1/S6 + platform)

Covered in depth in §6 (cost/tiering) and §7 (privacy/legal). Platform decisions:

- **Authoritative route→runtime table** (resolves all runtime contradictions):

| Route | Runtime | Why |
|-------|---------|-----|
| `/`, dashboard, predictor UI | Edge (static/RSC) | Cacheable, no secrets |
| `GET /api/og/[token]` | Edge (`@vercel/og`) | Image gen; opaque pre-scrubbed signed scalars only |
| `POST /api/analyze/screenshot` | **Node**, streamed | Anthropic SDK + base64 image |
| `POST /api/predict` | **Node**, streamed | SDK Node deps; Edge can't host them |
| `POST /api/forensics/map` and `/reduce` | **Node**, streamed | SDK; short calls, client-orchestrated |
| `POST /api/turnstile/verify`, `/api/session` | Edge | Sub-second gates |
| `GET /api/card/sign` | Node | HMAC + deterministic verdict scrub |

- **No background functions, no job store.** Long exports are client-orchestrated short Node calls. This keeps the §6 long-call paths inside per-request limits and removes the only path that wrote content-derived data to a server store.
- **APM/crash-dump hardening (major):** request-body capture and heap/crash-dump capture are **disabled on all analysis Node functions**, so the brief in-memory window of chunk text on a request cannot leak via a trace. Loggers are content-blind by construction (allowlisted typed events; a runtime guard rejects any field outside the schema).

---

## 5. Key Data Contracts

These live in one shared TypeScript package (`@vibecheck/contracts`) imported by name everywhere. **This package is the single source of truth** — the review's dominant systemic risk was five incompatible definitions of the Stage-1 metrics object; there is now exactly one, with one party enum (`self | target`), one time-series shape, and one dialect type.

```ts
// ---------- Shared primitives ----------
export type ISO = string;
export type UnitPct = number;            // 0..100
export type Unit01 = number;             // 0..1
export type MBTI = `${'I'|'E'}${'N'|'S'}${'T'|'F'}${'J'|'P'}`;
export type Party = 'self' | 'target';   // THE party enum. Not me/them, not USER/TARGET.
export type CognitiveFunction = 'Ni'|'Ne'|'Si'|'Se'|'Ti'|'Te'|'Fi'|'Fe';

export const SCHEMA_VERSION = 2 as const;
export const LEXICON_VERSION = 'LEXICON_V3' as const;

// ---------- Canonical message ----------
export interface Message {
  i: number;
  party: Party;
  ts: number | null;        // epoch ms, null if export lacked timestamps
  text: string;             // POST-scrub only
  kind: 'text' | 'media_omitted' | 'call' | 'system' | 'reaction';
}

// ---------- THE Stage-1 metrics type (one definition, period) ----------
export interface Stage1Metrics {
  msgCount: number;
  spanDays: number;
  perParty: Record<Party, {
    msgCount: number;
    charCount: number;
    initiations: number;          // first msg after a >6h gap
    doubleTexts: number;
    coldStartInitiations: number; // first msg after a >12h gap
  }>;
  textVolumeRatio: number;        // self_chars / target_chars
  initiationPctSelf: UnitPct;
  medianReplyLatencyMs: Record<Party, number>;   // median, awake-window filtered
  punctuationDecay: Array<{ party: Party; bucket: ISO; score: Unit01 }>;  // ONE time-series shape
  enthusiasmDecaySlope: Record<Party, number>;   // OLS slope; negative = cooling
  ghostingRisk: Unit01;           // z-score → sigmoid, relative to this pair's own baseline
  tsConfidence: 'high' | 'low';
  nameScrubConfidence: 'high' | 'low';
  candidateGapMsgIndices: number[];
}

// ---------- Signals: the universal currency into the Scorer ----------
export interface Signals {
  source: 'screenshot' | 'export';
  stage1?: Stage1Metrics;          // export path
  vibe?: VibeSignals;              // screenshot path
  mbti: { self: MbtiRead; target: MbtiRead };
  evidence: Array<{ chunkId?: string; quote: string; label: string; weight: Unit01 }>;
  dataWeight: Unit01;              // down-weighted when nameScrubConfidence === 'low'
  injectionTainted?: boolean;      // if true, screenshot fields were discarded/down-weighted
}
export interface VibeSignals {
  bubbleRatioSelf: Unit01; tsDeltasMin: number[]; initiator: Party | 'unknown';
  emojiDensity: number; avgLenSelf: number; avgLenTarget: number;
  readReceiptState: 'read_no_reply'|'delivered'|'none'|'unknown';
}
export interface MbtiRead {
  type: MBTI | null;               // shown only above a confidence floor; never on share surfaces
  functions: Record<CognitiveFunction, Unit01>;
  confidence: Unit01;
}

// ---------- THE dialect type (enriched so the predictor's fields all exist) ----------
export interface DialectProfile {
  register: string;
  casingStyle: 'all_lower' | 'sentence' | 'ALL_CAPS' | 'mixed';
  avgWordsPerMsg: number;
  emojiRatePerMsg: number;
  topEmojis: string[];
  abbreviations: string[];
  signaturePhrases: string[];
  punctuationDensity: number;
  vocabularyDensity: number;       // type-token ratio 0..1
  tells: string[];
}

// ---------- Scorer: SPLIT into pure + server-only ----------
export interface HeuristicScore {       // the shared, pure, client+server result
  component: UnitPct;                    // 0..100, the 0.6-weight piece, renormalized for solo display
  contributions: Record<'latency'|'enthusiasm'|'volume'|'initiation'|'punctuation', number>;
}
export interface ScoreResult {          // authoritative, SERVER-ONLY (folds in the LLM judge)
  cookedPct: UnitPct;                    // post-blend, post-shrink, quantized
  chanceOfSuccess: UnitPct;
  confidence: Unit01;                    // derived from dataWeight, ORTHOGONAL to cookedPct
  subMetrics: { enthusiasm: UnitPct; reciprocity: UnitPct; momentum: UnitPct; landmines: number };
  heuristic: HeuristicScore;
  judgeComponent: UnitPct;
  inputHash: string;                     // sha256(canonicalize(Signals)) — drives client memoization
}
export type HeuristicScorer = (s: Signals) => HeuristicScore;   // pure, IO-free, runs both sides

// ---------- Forensics outputs ----------
export interface MicroSummary {
  chunkId: string; span: [number, number];
  topics: string[];
  emotionalTone: Record<Party, 'warm'|'flirty'|'neutral'|'tense'|'withdrawn'>;
  initiator: Party | 'mutual';
  landmines: string[];
  dialectNotes: Partial<DialectProfile>;   // contributes to the canonical DialectProfile
  highlight?: string;                       // ≤6-word paraphrase, never verbatim PII
}
export interface RelationshipSummary {
  cookedRationale: string[];
  trajectory: 'warming' | 'stable' | 'cooling' | 'ghosting';
  arc: Array<{ ts: number; label: string }>;
  landmineMap: Array<{ topic: string; severity: 1|2|3; lastSeen: number }>;
  theirDialect: DialectProfile;            // THE dialect the predictor consumes
  dataWeight: 'light' | 'medium' | 'heavy';
}

// ---------- The durable client object ----------
export interface Portfolio {
  schemaVersion: typeof SCHEMA_VERSION;
  sessionId: string;
  createdAt: ISO; updatedAt: ISO;
  pois: Record<string, Poi>;
  activePoiId: string | null;
}
export interface Poi {
  id: string; label: string; createdAt: ISO;
  seed: OnboardingSeed;
  episodes: Episode[];
  current: ScoreResult;
}
export type Episode =
  | { kind: 'screenshot'; at: ISO; signals: Signals; score: ScoreResult }
  | { kind: 'export';     at: ISO; signals: Signals; score: ScoreResult; summary: RelationshipSummary };

export interface OnboardingSeed {
  relationshipState: 'talking'|'first_dates'|'situationship'|'dating'|'rekindling'|'post_ghost'|'unknown';
  userMbti: MBTI | null;
  targetMbti: MBTI | null;
  targetMbtiGuessed: boolean;
  vibeNote: string | null;          // scrubbed before storage/egress
  whoIsWhoHint: 'user_is_right' | 'user_is_left' | 'unsure';
}

// ---------- The wire shapes (the ONLY shapes that cross the boundary) ----------
export interface ScreenshotReq {     // POST /api/analyze/screenshot
  schemaVersion: typeof SCHEMA_VERSION; sessionTok: string;
  image: { b64: string; mime: 'image/jpeg' };   // EXIF-stripped, header-cropped, consent-gated
  consentAck: true;                              // egress refused if not literally true
  seed: OnboardingSeed;                          // NO textLayer field — client OCR removed
}
export interface ForensicsMapReq {   // POST /api/forensics/map  (client-orchestrated, batched)
  schemaVersion: typeof SCHEMA_VERSION; sessionTok: string;
  chunks: Array<{ id: string; text: string; localShiftScore: Unit01 }>;  // ≤ batch size, scrubbed
}
export interface ForensicsReduceReq {// POST /api/forensics/reduce
  schemaVersion: typeof SCHEMA_VERSION; sessionTok: string;
  stage1: Stage1Metrics;             // computed client-side; server never recomputes (never has the log)
  microSummaries: MicroSummary[];
  seed: OnboardingSeed;
}
export interface PredictReq {        // POST /api/predict — ONE canonical shape
  schemaVersion: typeof SCHEMA_VERSION; sessionTok: string;
  intent: string;                    // scrubbed, ≤280 chars
  dialect: DialectProfile;           // canonical, produced by forensics
  signals: { responsiveness: ResponsivenessSignals; relationship: OnboardingSeed['relationshipState']; cookedPct: UnitPct };
  recentTurns: Message[];            // last ≤10, scrubbed
  regenerate: boolean;
}
export interface ResponsivenessSignals {
  enthusiasmRatio: number; initiationPctSelf: UnitPct; questionReciprocity: Unit01;
  trend7d: 'warming'|'flat'|'cooling'; lastTargetLatencyMin: number; medianTargetLatencyMin: number;
}
export interface PredictResp {
  tracks: Array<{ id:'direct'|'playful'|'chill'; text:string; rationale:string; pressureLevel:1|2|3|4|5 }>;
  scores: Record<'direct'|'playful'|'chill', { vibeEstimatePct: UnitPct; band:string; confidence:'low'|'medium'|'high' }>;
  refusal?: { reason: string };      // fires on re-contact-while-disengaged, etc.
  guardrailFlags: string[];
}

// Common server response
export interface AnalysisResult { signals: Signals; score: ScoreResult; usage: { tier: string; tokens: number }; }

// ---------- OG card (server-scrubbed + signed) ----------
export interface CardPayload { cooked: UnitPct; band: string; verdict: string; weight: Unit01; v: 1; }
// token = base64url(JSON) + "." + base64url(HMAC_SHA256(JSON, CARD_SECRET))
// `verdict` passes a DETERMINISTIC server-side identifiability filter before signing (§7).
```

---

## 6. Cost Model & LLM Tiering

### 6.1 The single egress: Claude Broker (S6) enforces the cap

**Every** Anthropic call — screenshot (S1), MBTI/judge (S3/S4), forensics map+reduce, predictor, every retry/repair, the Haiku creep-classifier — routes through S6. **S6 checks the per-session token cap in KV before calling Anthropic and rejects with 429 if over.** A cap documented but not wired to the chokepoint is not a cap; this closes that blocker. The cap is set to **≈250k tokens/session** (near the documented power-session ceiling, ~10× lower than the original 1.5M), and a fresh Turnstile challenge is required to extend it.

```ts
// Inside S6, before every Anthropic call:
const estimate = estimateTokens(req);                 // local char/4 heuristic on hot paths
const used = await kv.incrby(`tok:${sessionTok}`, estimate);
if (used > SESSION_TOKEN_CAP) { await kv.incrby(`tok:${sessionTok}`, -estimate); return reject429(); }
const resp = await anthropic.messages.create(...);     // model per pinned tier config
await kv.incrby(`tok:${sessionTok}`, actualTokens(resp) - estimate);  // reconcile incl. thinking tokens
```

**Token estimation (minor):** the hot paths (screenshot, predictor) use a **local char/4 heuristic** for the pre-flight reservation and reconcile from `usage` post-call; `count_tokens` (a billed, rate-limited round-trip) is used **only on the export path** pre-flight fan-out estimate, which is large and latency-insensitive.

### 6.2 Pinned model tiers (one shared config constant)

```ts
export const MODELS = {
  screenshotVibe: 'claude-sonnet-4-6',   // LOCKED — resolves the 3-way contradiction; no adaptive thinking
  forensicsMap:   'claude-haiku-4-5',    // cheap extraction, fixed small thinking
  forensicsReduce:'claude-sonnet-4-6',   // synthesis, fixed thinking
  judge:          'claude-opus-4-8',      // one high-stakes call/export, fixed thinking, effort:medium
  predictor:      'claude-sonnet-4-6',   // persona/wit, temp ~0.8 for track spread
  creepClassifier:'claude-haiku-4-5',    // rare, regex-flagged intents only
} as const;
```

**Thinking is fixed/bounded everywhere a score or a high-volume path is involved** — never `adaptive`. Adaptive thinking bills unbounded reasoning at output rates and is non-deterministic; both are disqualifying for our cost ceiling and our reproducibility requirement. Effort: `low` on Haiku map, `medium` on Sonnet/Opus synthesis.

### 6.3 Cost per task (Sonnet-based, thinking tokens included)

| Task | Model | Rough input | Rough output (incl. bounded thinking) | ~Cost |
|------|-------|-------------|----------------------------------------|-------|
| Screenshot vibe | Sonnet 4.6 | ~4k (img+seed, cached prefix) | ~1.5k | **~$0.035** |
| Predictor (1 generation) | Sonnet 4.6 | ~2.5k (cached dialect prefix) | ~0.6k (3 drafts) + bounded thinking | **~$0.024** |
| Forensics map (×~12, Haiku) | Haiku 4.5 | ~22k total | ~5k | **~$0.05** |
| Forensics reduce | Sonnet 4.6 | ~6k | ~1.5k + thinking | **~$0.04** |
| MBTI judge (Opus, once/export) | Opus 4.8 | ~8k | ~2k + fixed thinking | **~$0.10** |
| Embeddings | **local MiniLM** | 1.5M tokens | — | **$0 (never leaves browser)** |

**Per-session envelopes (with retry/guard calls assumed at ~10% and counted at S6):**
- Casual session (a couple screenshots + a few predictor gens): **~$0.12–0.18**.
- Power session (one full export end-to-end + predictor use): **~$0.30–0.40** (Opus judge once; map/reduce on Sonnet/Haiku; embeddings free).
- **Absolute ceiling: the ~250k-token S6 cap**, which a single anonymous visitor cannot exceed without a fresh Turnstile.

**Pre-warming (major).** `max_tokens:0` cache pre-warm is **dropped on the hot paths**: the API rejects `max_tokens:0` together with forced `tool_choice:{type:"tool"}`, `output_config.format`, and `stream:true` — i.e. it is invalid for exactly the screenshot and judge calls — and on Vercel's non-sticky instances a per-cold-start warm call is a paid cache *write* that often never gets a second read. **We let the first real request warm the cache.** Caching pays off *within* a warm burst/session, which is where our gains actually come from. We keep the `cache_read_input_tokens > 0` assertion as a **runtime alert on both the screenshot and export paths**, so a silent prefix-invalidation (e.g. a per-session ID leaking into the cached prefix) is caught immediately.

**Engagement-loop guardrail.** The Data-Weight meter and predictor regenerations are charged against the **visible Analysis-Credits meter enforced at S6**. Once credits/token-cap are spent, the meter visibly caps and additional screenshot drops degrade to local-only Stage-1 deltas. Growth incentives are thus bounded by a budget-aware ceiling, not by a cookie-clearable IP limit.

---

## 7. Privacy, Legal & Abuse-Prevention (Resolved Stance)

### 7.1 The honest posture (rewritten — "clean hands" was structurally false)

We do **not** claim "we can't see anything." We claim, and the architecture enforces:

> **Minimized, pseudonymized text content is processed transiently and never persisted by us. Screenshot images are processed as-is (we cannot redact pixels), used once, and not persisted. Embeddings and turning-point selection happen in your browser, so only ~12 scrubbed excerpts ever leave it. No content or content-derived data is written to any server store — KV holds counters only.**

Concrete actions, replacing the indefensible claim:
- **Anthropic is a sub-processor.** We sign a DPA, enable no-train + (where eligible) zero-data-retention, and list Anthropic in the privacy policy.
- **For GDPR, we are a controller for the non-consenting target's data.** "The user attests they have the right" is **not a lawful basis** to process a third party's personal data or face. So: third-party profiling and the screenshot path are **region-gated to the stronger posture by default** (see 7.3), and the consent/attestation flow is **legally reviewed before launch**.
- Product is labeled **entertainment / self-reflection, not psychological assessment**, on the analysis surface itself.

### 7.2 Client-side scrubbing (text) + the screenshot exception

Text egress is scrubbed before the boundary (span-map, participant + diminutive + Bloom name redaction, fixed-regex tripwire — §4.1). **The screenshot path is the named exception:** it transmits a third party's identifiable image. We do not pretend otherwise. The consent interstitial **explicitly states third-party PII transmission**; the header crop is **default-ON**; EXIF strip is unconditional. In two-party-consent and GDPR regions the screenshot path is **degraded/off** (7.3).

### 7.3 One-party vs two-party consent — default to the stronger posture (major: geo is bypassable)

`request.geo` is a hint only and is trivially VPN-bypassed, so it **cannot be the gate that decides whether to collect consent**. Therefore:

- **Default globally to the stronger posture**: an explicit attestation interstitial before the first analysis, and the screenshot/third-party-profiling restrictions applied. Geo is used only to *relax* toward the lighter flow where we are confident, never to *withhold* protection.
- The attestation (unchecked by default) reads: *"This is my own conversation. I have the right to analyze it. The other person has not asked me to stop contacting them. I won't use VibeCheck to harass, surveil, or deceive anyone."*
- ToS indemnification stays, with the explicit acknowledgment that **it does not transfer GDPR controller obligations** — it is a backstop, not the basis.

### 7.4 Abuse — cost *and* human-harm (major: threat model was cost-only)

We name the human-harm class explicitly: a harasser/stalker uploading a target's export or screenshots to profile them or generate re-contact messages; analyzing an ex who blocked them. Zero-persistence is a privacy feature for the user **and** an impunity feature for an abuser (no audit trail). Mitigations where they count:
- **Predictor refuses re-contact-while-disengaged** (`post_ghost`/`rekindling` + re-contact intent → `refusal`), a real detection mechanism, not just a list item.
- **Attestation covers "no-contact" status.**
- The "perfect zero-persistence forecloses abuse investigation" tradeoff is a **deliberate, documented decision**, not a blind spot.

Cost/scraping controls (stacked, none trusted alone): invisible **Turnstile** to mint a session token (HMAC, httpOnly), session-token primary throttle, IP backstop, and the **S6 token cap** as the hard floor. Tight per-action limits (export 3/hr/session, predictor ~40/session). KV stores only counters + the token tally.

### 7.5 Share artifact — deterministic scrub, not a prompt instruction (major)

The OG `verdict` line is **passed through a deterministic server-side identifiability filter before HMAC signing** — regex for names/roles/quotes/relationship-identifying detail; on a hit it is rejected and regenerated. We do **not** trust the prompt's "no names" instruction as the guarantee. The HMAC prevents forged-number cards; the deterministic scrub prevents content leakage in the prose. The card payload carries only a number + band + scrubbed verdict + theme — never message text, never the target's identity.

### 7.6 App-store / payment-processor reality (major)

We do **not** bank the launch on "web app = no review." Payment processors (Stripe/PayPal) and ad platforms restrict relationship-surveillance / analyze-a-specific-person-without-consent tooling — our core loop is stalkerware-adjacent, and ex/post-ghost flows + dialect cloning amplify it. Pre-emptive measures: lead with self-reflection framing, **remove the most stalkerware-adjacent affordances** (re-contact optimization, `derivedTypeLabel`, deep third-party profiling on shareable surfaces), and get the consent/attestation flow legally reviewed **before** launch.

### 7.7 Secrets

`ANTHROPIC_API_KEY`, `SESSION_SECRET` (HMAC), `TURNSTILE_SECRET`, `CARD_SECRET`, `UPSTASH_*` are Node-only Vercel env vars, never `NEXT_PUBLIC_`, distinct per environment (**preview deployments never share the prod Anthropic key** — a public preview URL with a prod key is an open spend faucet). The content-blind logger drops any value matching key/token patterns as a final net.

---

## 8. Resolved Risks & Open Questions

### 8.1 Blockers — resolved

| Review blocker | Resolution |
|---|---|
| KV result store vs zero-persistence invariant | **KV result store deleted.** Long exports are client-orchestrated short calls (§2.4, §3.2, §6.1). KV holds counters only; the invariant is now literally true. |
| Screenshot client-OCR contradiction (Ingestion vs OCR section) | **No client OCR.** `textLayer` removed from `ScreenshotReq`; canonical shape is `image + mime + seed + consentAck` (§4.1/§4.2/§5). |
| E1 PII re-scan can't see pixels | E1 re-scan **scoped to text fields**; image bytes are an explicit consent-gated exception; the "all crossing payloads are scrubbed" claim is corrected (§2.4). |
| Pure Scorer vs live LLM judge | **Scorer split**: pure `heuristicScore` (client+server) + server-only `judgeAndBlend`; offline/degraded score is labeled heuristic-only (§4.4/§5). |
| Screenshot model tiered 3 ways (Opus/Haiku/Sonnet) | **Locked to Sonnet** in one shared config; adaptive thinking dropped on it; §6 cost re-derived against Sonnet (§4.2/§6.2). |
| Per-session token cap not enforced | **Enforced at S6 before every call**, cap lowered to ~250k, Turnstile to extend (§6.1). |
| Embedding location contradiction / cost-thesis breach | **Server S2 deleted; embeddings local-only (MiniLM).** Only ~12 chunks leave (§4.3). |
| Image path leaks third-party PII; "consent" doesn't cover the other party | Stop calling it "scrubbed"; consent names third-party transmission; header crop default-ON; region-gated/degraded for two-party/GDPR (§4.1/§7.2/§7.3). |
| "Clean hands" legal claim structurally false | Rewritten to "minimized, transient, un-persisted; images as-is"; DPA + sub-processor disclosure; controller obligations acknowledged (§7.1). |
| Name-stripping unreliable but shipped silently | In-product "best-effort, not guaranteed"; diminutive map added; dataWeight down-weighted on low confidence; non-Latin export hard-gated (§4.1). |
| Predictor = manipulation tooling | Re-contact optimization removed; `chill` decoupled from ghosting; success% relabeled vibe estimate; refusal on re-contact-while-disengaged; attestation covers no-contact (§4.5/§7.4). |
| Third-party profiling/defamation on shareable surfaces | Profiling stays client-side/off shares; `derivedTypeLabel` dropped; GDPR strip; entertainment banner on the analysis surface (§4.4). |

### 8.2 Majors — resolved

Schema fragmentation → one `@vibecheck/contracts` package, one `Stage1Metrics`, one `Party` enum, one `DialectProfile` (§5). PredictReq mismatch + dialect fields that didn't exist → one `PredictReq`, `DialectProfile` enriched to what `buildDialectBlock` reads (§4.5/§5). Model IDs / runtime placement → pinned model config + authoritative route→runtime table; Sonnet is `claude-sonnet-4-6` (not `4-5`) (§4.7/§6.2). localStorage courier → removed, BroadcastChannel-only tether (§4.6). Thinking-token undercount → fixed bounded thinking everywhere scored/high-volume, included in §6. Pre-warm `max_tokens:0` → dropped on hot paths (invalid with forced-tool/format/stream; cold-start waste) (§6.3). Candidate-window fan-out unbounded → hard ≤150 candidate cap pre-LLM + pre-flight `count_tokens` estimate (§4.3). Slot-machine cost → S6-enforced credits meter, ~40-gen cap, output tokens in the math (§4.5/§6.3). Per-screenshot re-analysis cost incentive → paid-call caps, local-only deltas, meter visibly caps (§4.6). Retry/guard calls unbudgeted → all route through S6 and count (§4.5/§6.1). KV in-memory/APM leak → APM body + crash-dump capture disabled on analysis functions (§4.7). Geo-bypass consent → default-stronger globally (§7.3). Share-verdict prose leak → deterministic server scrub before signing (§7.5). Injection poisons extracted fields → tainted extraction discarded, not just flagged (§4.2). App-store/processor → pre-emptive de-risking + legal review (§7.6).

### 8.3 Minors — resolved

Tripwire global-regex statefulness bug → non-global clones / reset `lastIndex` + re-scrub-span-then-fail (§4.1). EXIF strip unconditional on every egress path; rhetorical weight moved from GPS to faces/names-in-pixels (§4.1). Mid-conversation-system caching unverified → seed placed in the user turn after the breakpoint (guaranteed-safe placement per API ref); cache-read assertion on the screenshot path (§4.2). `count_tokens` per hot call → local heuristic on hot paths, `count_tokens` export-only (§6.1). API-param composition (forced tool + adaptive thinking + `max_tokens:0`) → verified against the API reference: forced `tool_choice` is GA, adaptive thinking dropped on these calls anyway, and `max_tokens:0` pre-warm is invalid with forced-tool/format/stream so it's not used there (§4.2/§6.3).

### 8.4 Genuinely open questions

1. **Legal sign-off on the consent/attestation flow and the GDPR controller posture** — flagged for counsel before launch; the architecture is built to support either "EU third-party profiling fully stripped" or "EU export feature off," and the final choice is a legal decision, not an engineering one.
2. **MiniLM cold-start on low-end mobile** — ~25MB weights lazy-load behind the instant Stage-1 dashboard; we need real-device telemetry to confirm the hide-the-latency UX holds on budget Android, and a fallback (refuse export → screenshot mode) if WebGPU/WASM is too slow. We will **not** add a server embedding fallback that embeds the full candidate set — any future server embedding path must embed only the already-selected top-K and carry its own budget line.
3. **Payment-processor classification risk** — even de-risked, the loop may still trip stalkerware-adjacent policies; needs a processor conversation pre-launch and possibly a narrower public framing.
4. **Determinism ceiling** — quantization + memoization make the *displayed* number stable per input, but a genuinely new input near a quantization boundary can still flip ±5; acceptable, monitored, surfaced as "re-scored" only on real input change.
5. **Abuse-investigation tradeoff** — with zero persistence we cannot investigate a reported abuser; this is documented and deliberate, but if law-enforcement obligations ever require otherwise, it forces a posture change we have intentionally made expensive to reach.

---

## 9. Recommended Build Phases

**Phase 0 — Viral MVP (screenshot → Cooked% → shareable card).**
Delivers: onboarding seed; screenshot ingestion (EXIF strip, header-crop default-ON, consent interstitial naming third-party transmission); S1 Sonnet vibe extraction with forced tool + injection-tainted-discard; heuristic-only + Opus-judge blended `ScoreResult` with Data-Weight shrinkage; `sessionStorage` portfolio + BroadcastChannel tether; deterministic-scrubbed + HMAC-signed OG card; Turnstile + S6 token cap. This is the growth engine and the smallest slice that exercises the trust boundary, the Scorer split, and the cost chokepoint end-to-end.

**Phase 1 — Full chat-export forensics.**
Delivers: streaming parser + span-map scrubber + fixed tripwire; Stage-1 deterministic metrics (instant dashboard, 0 tokens); local MiniLM embeddings + turning-point selection with the ≤150 candidate cap; **client-orchestrated** Haiku-map / Sonnet-reduce producing `RelationshipSummary` (incl. the canonical `DialectProfile`); non-Latin export hard-gate; "names are best-effort" in-product copy. Establishes the "only ~12 chunks leave, nothing content-derived is stored" path.

**Phase 2 — Relationship dashboard.**
Delivers: the full instrument over the existing contracts — enthusiasm ratio, punctuation-decay tracker, landmine map, ghosting early-warning, intra-session history graphs from `Portfolio.episodes`; read-only reloaded-evidence chips; paid-call caps + visibly-capping Data-Weight meter; private/not-for-redistribution + entertainment banner. Mostly client work on data already produced in Phases 0–1.

**Phase 3 — Response Predictor ("Rizz Translator").**
Delivers: Sonnet dialect-cloning over the canonical `DialectProfile`; code-computed vibe-estimate (no model-emitted success number); three tracks with `chill` decoupled from ghosting; layered guardrails (regex pre-gate, model self-refusal, deterministic + rare-Haiku post-gate) all routed through S6; re-contact-while-disengaged refusal; Analysis-Credits meter enforcing the budget. Ships last because it is the highest legal-risk surface and depends on the dialect profile from Phase 1.
