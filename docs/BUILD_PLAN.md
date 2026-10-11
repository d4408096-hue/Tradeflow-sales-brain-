# TradeFlow Sales Coach — Build Procedure (v0 plan)

Status: **draft for review** · Owner: founder · Last updated: 2026-10-11

This document is the build *procedure*: what we build, in what order, what must be
proven before moving on, and what we deliberately refuse to assume. It is written to
be revised — anything marked **[DECIDE]** needs a call from the founder before code
lands on it.

---

## 0. What we are actually building first

A phone-first screen the founder opens before dialing that:

1. hears the prospect (through a capture arrangement we have *proven* works),
2. shows a live transcript,
3. shows **one** line to say, plus **one** short next action,
4. shows how long that suggestion took to appear.

Not in scope for the first prototype: automatic cellular-call listening, vector
retrieval, multi-user auth, CRM sync, analytics dashboards.

---

## 1. Findings that change the plan

These were verified before writing any code, because each one invalidates an
assumption in the original stack document.

### 1.1 Automatic cellular-call capture is not possible for an ordinary Android app

`MediaRecorder.AudioSource.VOICE_CALL` requires `android.permission.CAPTURE_AUDIO_OUTPUT`,
which is **system-only** and denied to third-party apps. On Android 10+ the platform
additionally locks the microphone while a cellular call is active, so an ordinary app
gets *no* audio at all during a call — not even the near-end side. Bluetooth routing
does not change this; the whole call is a protected audio session.

**Consequence:** the "app listens to both sides of the cellular call" milestone is not a
work item we can schedule against the current approach. It requires either a second
device or a different call path (see §3). The project brief already suspected this; it
is now confirmed, and the plan treats it as settled rather than a risk.

### 1.2 The backend does not need WebSockets — which removes the hosting risk

Deepgram can mint a **short-lived token** (`POST https://api.deepgram.com/v1/auth/grant`)
from a Member-role key held server-side. The browser then opens the Deepgram WebSocket
**directly** with that token. Consequences:

- The permanent Deepgram key never leaves the backend. ✅ (your rule)
- The audio stream never touches our server → no bandwidth cost, no audio buffering, lower latency.
- The backend only serves **short REST requests and static files** → it runs on any free
  tier, including ones without long-lived connection support. The "host must support
  persistent WebSockets" constraint from the stack doc **disappears**.
- A token that expires after 30s still holds an *open* socket; the token gates connection
  setup only. So we mint a fresh token at each "Start listening" tap, TTL 60–300s for slow
  mobile networks, and reconnect by re-minting.
- Verify on first live call: the grant body parameter is documented inconsistently across
  sources (`ttl_seconds` vs `time_to_live_in_seconds`). Test both, keep the working one.

**Cost note:** minting tokens via `/v1/auth/grant` is not subject to the ~250 key-creation-
per-day cap that applies to the Management API key-creation endpoint. Good — a session may
reconnect several times.

### 1.3 The free Nemotron route is an evaluation route, not a sales route

`nvidia/nemotron-3-ultra-550b-a55b:free` is real, free, 1M context, text-only.
But per the listing and third-party tracking:

| Property | Value | Why it matters |
| --- | --- | --- |
| Rate limit | ~20 req/min, ~200 req/day (default tier often quoted as 50/day) | A 30-min call can want 40–80 suggestions. One or two calls exhaust a day. |
| Data handling | Usage is **logged for security and may be used to improve NVIDIA products** | Directly conflicts with using it on a real prospect call. |
| Availability | Fluctuates; third-party snapshots show ~84% success over 3 days | Tail failures are expected, not exceptional. |
| Reasoning | Enabled by default | Reasoning tokens are the #1 threat to a 2–3s latency target. |

**Consequence, stated plainly:** the free tier is fine for synthetic conversations and
quality evaluation, and **must not carry a real prospect call**. The prototype therefore
ships from day one with (a) a provider-swappable LLM layer, (b) a "no confidential data on
free routes" banner in the UI, and (c) a documented upgrade path to a paid route before
the first real call. The founder's own rule #2 is thus enforced in code, not just in a doc.

> Cheap partial fix worth checking in the OpenRouter dashboard: adding ≥$10 of credit has
> been reported to raise the free-model daily cap substantially. Verify on your account
> before relying on it; it does **not** fix the logging issue.

---

## 2. Target architecture

```
┌─ Android Chrome / PWA (the only thing on screen during a call) ───────────────┐
│  mic → AudioWorklet → PCM16 16 kHz mono → 80–250 ms chunks                    │
│  WS ──► wss://api.deepgram.com/v1/listen  (Nova-3, interim_results=true)      │
│         auth: short-lived token fetched from our backend                      │
│  interim text = grey · final text = solid                                     │
│  on speech_final / UtteranceEnd ──► POST /api/session/{id}/turn               │
│  renders: SAY (huge) · DO (small) · stage/concern chips · latency badge       │
└───────────────────────────────────────────────────────────────────────────────┘
                    │ HTTPS, same origin as the static app
┌─ FastAPI backend (free-tier hostable, stateless-ish) ─────────────────────────┐
│  POST /api/session                  → create session, return config           │
│  POST /api/session/{id}/dg-token    → mint Deepgram ephemeral token           │
│  POST /api/session/{id}/turn        → {segment} → suggestion JSON (+ timings)  │
│  GET  /api/session/{id}/state       → stage, concerns, recent turns           │
│  POST /api/session/{id}/end         → wipe in-memory state                    │
│  POST /api/transcribe-file          → fallback STT for uploaded recordings    │
│  GET  /                             → serves the PWA (no CORS, one origin)    │
│                                                                                │
│  context assembly:                                                             │
│    system prompt (role + output contract + guardrails)                         │
│    + TradeFlowContext.md  (always, it is short; [VERIFIED]/[UNVERIFIED] tags)   │
│    + SaadKhaja.md         (section-level keyword routing, top-K, no vector DB) │
│    + rolling conversation window + compact running summary + stage/concerns    │
└───────────────────────────────────────────────────────────────────────────────┘
                    │
        OpenRouter (Nemotron) · Groq Whisper (later) · anything OpenAI-compatible
```

### Provider layer (non-negotiable interface boundary)

```python
class LLMProvider(Protocol):
    async def suggest(self, ctx: CoachContext) -> Suggestion: ...
    @property
    def name(self) -> str: ...

class STTProvider(Protocol):
    async def token(self, ttl_s: int) -> EphemeralToken | None: ...
    async def transcribe_file(self, audio: bytes, mime: str) -> Transcript: ...
```

Implementations, selected by env var, no code change to swap:

| Interface | v0 | Used for |
| --- | --- | --- |
| `LLMProvider` | `openrouter_nemotron` | primary reasoning |
| | `openai_compatible` | any fallback (OpenRouter paid route, NVIDIA NIM, local vLLM, Groq) |
| | `mock` | offline UI/dev, deterministic tests, demo mode in restricted networks |
| `STTProvider` | `deepgram` | live streaming tokens + prerecorded file |
| | `groq_whisper` | prerecorded-file alternative, evaluated later |
| | `mock` | tests |

Adding a fallback must be an env change plus a registry entry — never a rewrite.

### Suggestion contract (the whole UI depends on this staying tiny)

```json
{
  "say":  "One sentence, out-loud ready, ≤ ~30 words.",
  "do":   "One next action or question, ≤ ~12 words. Optional.",
  "concern": "price | timing | existing_solution | trust | not_interested | none",
  "stage":   "opener | discovery | problem | value | objection | close | nurture",
  "confidence": "high | medium | low",
  "needs_clarification": false
}
```

Enforced by: `response_format` json_schema where the route supports it → falling back to a
strict parser + one repair retry → falling back to a **degraded card** ("I need to ask a
clarifying question — say: …") plus a visible UI state. It never spins forever.

Guardrails in the system prompt and re-checked in code after the response:
- no invented features, integrations, results, guarantees, testimonials, or revenue figures;
- no price/claim that is not present in `TradeFlowContext.md`;
- prefer a question when the situation is unclear;
- no manipulative pressure, no false scarcity, no fake certainty.

A post-generation validator scans `say` for banned claim patterns (numbers, "guarantee",
"we integrate with X", testimonial shapes) that lack a matching `[VERIFIED]` fact; on
violation, regenerate once with a corrective instruction, then degrade. This is the
difference between "the model was told not to lie" and "the product does not lie".

---

## 3. The capture decision — choose the path before Stage 3

> **⚠️ SUPERSEDED 2026-10-11 — see [`docs/MARKET_AND_DIALER_VERDICT.md`](MARKET_AND_DIALER_VERDICT.md).**
> The chosen dialer (Rebtel) cannot carry this project: it shows an Indian caller ID that
> cannot be changed, its international calls attract C-level STIR/SHAKEN attestation and
> therefore "Spam Likely" labelling in the US, it has no web/desktop app (which closes off
> clean tab-audio capture), and it exposes no API or recording.
>
> The replacement is a US-numbered **business VoIP or programmable carrier**, and if it
> supports call-media streaming then the coach can receive a **digital, dual-channel feed of
> both parties** — which removes the Android capture problem entirely, removes the
> "who is speaking" risk from v0, and shrinks M3 considerably. The paths below are retained
> for reference only.

[DECIDE] This is the single highest-leverage open question.

| Path | Setup | Prospect audio | Real-time | Cost | Verdict |
| --- | --- | --- | --- | --- | --- |
| **A. Second device** | Call on phone, speakerphone on; PWA open on laptop/tablet nearby | Yes, acoustically via near device's mic | Yes | $0 | **Recommended for the first feasibility test.** Proves the full pipeline today. |
| **B. Post-call file** | Android's built-in recorder / after-call file → upload | Device-dependent | No | $0 | Best for STT accuracy measurement and quality evals. Zero risk. |
| **C. Softphone / VoIP** | Outbound calls placed through an app-controlled WebRTC line (Twilio/Telnyx-class) | Yes, digital, dual-channel, clean | Yes | per-minute + number rental | The **only** path to real one-device live assistance. Also improves transcript accuracy and adds call recording/CRM hooks. Gate behind evidence from A/B. |
| **D. On-phone mic during a cellular call** | — | — | — | — | **Not viable** (§1.1). Do not schedule. |

Note that Path A captures *both* voices into one stream. v0 design: treat the transcript as
a two-party conversation and instruct the model to coach the prospect's latest substantive
point, plus a one-tap "my turn / mute" control so the founder's own speech does not trigger
suggestions. Diarization or a floor-token scheme is a v1 refinement, not a v0 blocker.

**Deliverable that closes this:** a one-page `docs/AUDIO_CAPTURE.md` recording the tested
device, arrangement, measured transcript accuracy on a consented mock call, and the honest
limitation. That document is what stops us from over-claiming later.

---

## 4. Build sequence

Each milestone has a demo and a definition of done. Do not start milestone N+1 while N's
"proven by" line is unmet — that is the whole point of the brief's "smallest working coach".

### M0 — Decisions and guardrails (no product code)

- [ ] Resolve §3 capture path and §6 open decisions.
- [ ] Verify credentials exist and where they live (never in chat, never in the client):
      Deepgram key with **Member role or higher** (the grant endpoint rejects default-role
      keys), OpenRouter key. Stored as `.env` locally / host secrets in deployment.
- [ ] Write `docs/PRIVACY.md`: consent script, what leaves the device, what each provider
      sees, retention = none by default, how to stop.
- [ ] Repo hygiene: `.gitignore` for `.env`, audio, logs, transcripts; `.env.example`.
- **Proven by:** keys resolve, `.env.example` lists every secret, privacy doc written.

### M1 — Knowledge base and one-shot reasoning (your Stage 1)

- [ ] `knowledge/TradeFlowContext.md` — product facts, each line tagged
      `[VERIFIED]` / `[UNVERIFIED]` / `[UNKNOWN — do not guess]`. The model may only assert
      VERIFIED lines. **Blocked on founder input** (§6 Q1).
- [ ] `knowledge/SaadKhaja.md` — principles, situational rules, question types,
      objection handling, examples, failure modes, source refs, TradeFlow adaptations.
      **Blocked on source material** (§6 Q2) — we will not attribute invented method to a
      named expert.
- [ ] Section-routing selector (keyword/tag overlap, top-K) — no vector DB.
- [ ] `POST /api/suggest` (text in → one suggestion out) + `LLMProvider` + OpenRouter impl.
- [ ] **Model latency benchmark before committing to a model** (see §5).
- **Demo:** type a realistic prospect objection, get one line back, on a phone-shaped screen.
- **Proven by:** 10 hand-written probe statements produce on-contract, claim-safe output.

### M2 — Conversation awareness (your Stage 2)

- [ ] Session store (in-memory first; SQLite only if it earns its place), rolling window,
      periodic compact summary, stage + concern trackers.
- [ ] Anti-repetition: the suggestion must not re-ask a question already answered, and must
      not contradict an earlier factual statement.
- [ ] `scripts/eval_scenarios.yaml` — 15–20 scripted multi-turn prospects covering price,
      no-interest, existing solution, "I'll think about it", gatekeeper, wrong buyer.
- [ ] `scripts/eval.py` — runs scenarios through the pipeline, scores: JSON validity,
      banned-claim hits, length, repetition, contradiction, latency p50/p95.
- **Demo:** a 6-turn price objection handled without repeating itself.
- **Proven by:** eval score recorded in `docs/EVAL.md`, with failures listed honestly.

### M3 — Speech-to-text and the client (your Stage 3)

- [ ] PWA: mic permission flow, AudioWorklet → PCM16 16 kHz, level meter, start/stop,
      visible listening state, offline/error/reconnect states.
- [ ] Direct-to-Deepgram streaming with backend-minted ephemeral token; interim vs final
      rendering; `KeepAlive`; auto-reconnect with bounded attempts; hard stop that also
      drops local buffers.
- [ ] Turn gating: reason on `speech_final`/`UtteranceEnd` only, 400 ms debounce,
      ≥8 s cooldown, skip <3 new words, manual "Suggest now".
- [ ] Request budget meter (see §5.3) so a free-tier day is never silently burned.
- [ ] File-upload transcription path (Path B) for accuracy testing without a live call.
- **Demo:** speak a scripted conversation aloud, watch transcript → suggestion → latency.
- **Proven by:** measured STT accuracy on a known script + full end-to-end latency report.

### M4 — Live-call feasibility (your Stage 4)

- [ ] Execute the §3 chosen path on the real Android device, consented mock call.
- [ ] Fill in `docs/AUDIO_CAPTURE.md`: device, arrangement, who got captured, accuracy,
      end-to-end latency under real acoustics.
- [ ] Decide, with evidence, whether Path C (softphone) is justified.
- **Proven by:** the doc above exists, includes a failure/limitation section, and the
  full latency budget is either met or the miss is explained with numbers.

### M5 — Reliability and evaluation (your Stage 5)

- [ ] Timeouts everywhere (bounded), exponential backoff with jitter, retry caps, circuit
      breaker on the LLM route, visible fallback card instead of indefinite waiting.
- [ ] Fallback LLM route wired behind the same interface and *tested* by forcing failures —
      not merely implemented.
- [ ] Load/failure drills: kill network mid-session, expire a token mid-stream, 429 the LLM.
- [ ] Eval run against held-out scenarios; record pass/fail and latency percentile table.
- **Proven by:** a failure drill video/log where the UI degrades cleanly and recovers.

---

## 5. Measurement and budget

### 5.1 Latency budget (target, to be measured — not a claim)

| Segment | Target | Instrumented in |
| --- | --- | --- |
| capture + chunking | ≤ 150 ms | client |
| last audio byte → Deepgram `speech_final` | ≤ 600 ms | client |
| trigger → backend → model | ≤ 1500 ms | server |
| render suggestion | ≤ 50 ms | client |
| **end of speech → card visible** | **≤ 2500 ms** | client stopwatch, shown on screen |

Every suggestion carries its own latency badge; `docs/LATENCY.md` accumulates p50/p95 per
segment per run. We publish the number we measure, including the bad one.

**Known threat:** Nemotron Ultra is a reasoning model. Reasoning tokens can dominate
latency. Mitigations to test in M1, in order: `reasoning.effort` set low/none, hard
`max_tokens`, a smaller sibling model (`nvidia/nemotron-3-super-120b-a12b:free`) as the
latency fallback, and — only if p95 still misses — token streaming so the first words render
before the sentence completes.

### 5.2 Quality evaluation

Scored per scenario: on-contract JSON · no unsupported claims · says one thing (not three) ·
does not repeat an answered question · does not contradict itself · moved the conversation
toward a next step · used the knowledge files when relevant (spot-checked by eye, not
trusted to a judge model in v0).

### 5.3 Rate-limit budget (free tier)

| Control | Value |
| --- | --- |
| reason only on final segments | mandatory |
| debounce | 400 ms |
| min cooldown between model calls | 8 s |
| min new words to trigger | 3 |
| per-session soft cap | 120 requests (warn at 80) |
| daily counter + visible meter | persisted locally, resets 00:00 UTC |
| on 429 | bounded backoff → degrade to "no suggestion available" card, never silent hang |

At ~20 RPM / ~200 RPD, a normal call is 40–80 requests. Design for **two calls/day on the
free route**, and say so in the UI rather than discovering it mid-call.

---

## 6. Open decisions and missing inputs

**[DECIDE] Q1 — Verified TradeFlow facts.** `TradeFlowContext.md` cannot be written from
the brief alone. Needed: is $599 current? What are the actual capabilities (missed-call
recovery mechanism, booking, notifications, integrations — yes/no per item)? Any real
results or case numbers that may legally be claimed? What must the coach never promise?
Until supplied, the file ships as `[UNKNOWN — do not guess]` for every claim, which makes
the coach honest but close to useless.

**[DECIDE] Q2 — Saad Khaja source material.** Either episode links/transcripts, or the
NotebookLM extraction notes. Without them we can supply a *generic*, clearly-labelled
principles scaffold (SPIN-style questioning, gap framing, objection taxonomy) that is
**not** attributed to the named expert, and swap it once real extraction exists. We will not
invent an expert's methodology.

**[DECIDE] Q3 — Capture path** (§3): **Superseded** — the question is now "which US-numbered
carrier gives us caller ID, attestation and audio access", not "which audio-capture hack".
See `docs/MARKET_AND_DIALER_VERDICT.md` §2.

**[DECIDE] Q4 — Trade mix.** Third-party benchmarks suggest plumbing already answers ~74% of
calls while HVAC (~34%) and construction (~32%) miss roughly two in three. Plumbing-first may
be backwards for a missed-call product. Resolve with your own first 30 calls, not with a
benchmark. See `docs/MARKET_AND_DIALER_VERDICT.md` §3.

**[DECIDE] Q5 — Where live integration testing happens.** This development sandbox has
egress restricted to GitHub/npm/PyPI only: it *cannot* reach `api.deepgram.com` or
`openrouter.ai`. So:
- code, unit tests, UI, and mocked-provider runs happen here (preview works, mock mode works);
- **live** model/STT calls must run in GitHub Codespaces, on your own machine, or on the
  deployed host.
A `mock` provider and a visible "provider unreachable" state are therefore required for the
sandbox preview to be usable — and they double as the offline demo mode.

**[DECIDE] Q6 — Deploy target** (after M3): the ephemeral-token design means we only need
static file serving + small REST endpoints, so any free tier works; pick whichever you can
configure secrets on most easily. Runner-up consideration: the host's egress must reach
`openrouter.ai`.

---

## 7. Repository layout (target)

```
backend/app/
  main.py            config.py        models.py       sessions.py
  prompting/         (system prompt, context assembly, section router, validators)
  providers/llm/     (base, openrouter, openai_compatible, mock)
  providers/stt/     (base, deepgram, groq, mock)
backend/tests/       (unit + contract + failure drills)
frontend/            (PWA, served by FastAPI)
knowledge/           SaadKhaja.md · TradeFlowContext.md
scripts/             eval.py · eval_scenarios.yaml · latency_report.py
docs/                BUILD_PLAN.md · ARCHITECTURE.md · AUDIO_CAPTURE.md
                     PRIVACY.md · LATENCY.md · EVAL.md
.env.example         .gitignore
```

## 8. Explicit non-goals for v0

Automatic cellular-call capture · vector database / RAG · multi-user accounts · CRM or
calendar integration · call-recording storage · mobile app-store build · fine-tuning ·
analytics dashboards · anything that requires a paid tier before the pipeline is proven.
