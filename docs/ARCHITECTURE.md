# TradeFlow Sales Coach — Architecture

**Status:** authoritative design · supersedes `BUILD_PLAN.md` §2 · **Date:** 2026-10-11

This document describes how the coach is built, why each seam exists, and what we are
deliberately *not* building yet. It assumes `knowledge/TradeFlowContext.md` and
`knowledge/SaadKhaja.md` are populated (they are) and that the product being sold is an
Android-only, local-first SMS app at $599 one-time.

---

## 1. The decision that shapes everything

**Audio never touches the browser. The browser is a display surface, not a capture device.**

In the original plan (BUILD_PLAN §2), the browser captured the microphone and streamed to
Deepgram directly via an ephemeral token. That design existed to avoid needing inbound
WebSockets on our host, because we assumed the only available audio source was a phone
microphone.

Choosing a US-numbered carrier with **call-media streaming** overthrows that assumption. The
carrier can deliver both parties' audio directly to our backend over a WebSocket — digitally,
already separated into inbound and outbound tracks. So:

| | Browser-capture design (old) | Server-side media stream (chosen) |
| --- | --- | --- |
| Audio quality | Acoustic, through a speaker and a microphone | Digital, lossless |
| Speaker separation | Unknown — requires diarization or guesswork | **Free** — the carrier labels the tracks |
| Android capture problem | Central blocker | **Disappears entirely** |
| Browser needs mic permission | Yes | No |
| Device required during a call | Android phone, speakerphone on | **Any device with a browser** |
| Host needs inbound WebSocket | No | Yes (acceptable — see §10) |
| Latency | capture → encode → network | network only |

Four consequences worth stating plainly:

1. **The Android audio-capture problem stops being a problem.** We never need the phone's
   microphone. This was the single riskiest item in the original plan (BUILD_PLAN §3) and the
   carrier decision removes it rather than mitigating it.
2. **We always know who is speaking.** The prospect is on one track, the founder on the other.
   That eliminates diarization from the roadmap and, more importantly, means we can transcribe
   the founder too — so the coach knows what he has already said and promised, and cannot
   contradict him.
3. **The founder can run the call from a laptop.** No phone balancing, no speakerphone, no
   second device.
4. **The cost of this decision is hosting.** We now need inbound WebSocket support. That is a
   real but small constraint (§10).

**Fallback layers are still built**, because the carrier path depends on an unresolved question
(§15.1). The architecture treats the audio source as a swappable seam, not a fixed assumption.

---

## 2. Component architecture

```
                     ┌──────────────────────────────────────────┐
                     │  US carrier (Twilio / Telnyx class)      │
                     │  · US local number, area-code matched    │
                     │  · A-level STIR/SHAKEN attestation       │
                     │  · SHAKEN/STIR + Voice Integrity reg.    │
                     └───────────────┬──────────────────────────┘
              outbound call placed  │  media stream (WSS)
              via REST API          │  + call events (WebSocket)
                     ┌───────────────▼──────────────────────────┐
                     │  FastAPI backend                          │
                     │                                           │
                     │  ┌─────────────────────────────────────┐  │
                     │  │ Call controller                     │  │
                     │  │ dial · hang up · mute · consent ·   │  │
                     │  │ recording · per-call metadata       │  │
                     │  └──────────────┬──────────────────────┘  │
                     │                 │                         │
                     │  ┌──────────────▼──────────────────────┐  │
                     │  │ Audio relay                         │  │
                     │  │ dual-track split · PCM16 16k ·      │  │
                     │  │ backpressure · no disk by default   │  │
                     │  └───┬─────────────────────┬───────────┘  │
                     │      │ prospect track      │ founder track│
                     │  ┌───▼─────────────────────▼───────────┐  │
                     │  │ STT provider (Deepgram Nova-3)      │  │
                     │  │ persistent WS · interim + final ·   │  │
                     │  │ UtteranceEnd · per-track            │  │
                     │  └───┬─────────────────────────────────┘  │
                     │      │ final segments, speaker-labelled    │
                     │  ┌───▼─────────────────────────────────┐  │
                     │  │ Turn controller                     │  │
                     │  │ who spoke · debounce · cooldown ·   │  │
                     │  │ min-words · budget guard · trigger  │  │
                     │  └───┬─────────────────────────────────┘  │
                     │      │                                     │
                     │  ┌───▼─────────────────────────────────┐  │
                     │  │ Reasoning orchestrator              │  │
                     │  │ session state · context assembly ·  │  │
                     │  │ LLM provider · claim validator ·    │  │
                     │  │ latency stopwatch                   │  │
                     │  └───┬─────────────────────────────────┘  │
                     └──────┼────────────────────────────────────┘
                            │  SSE (server → client, one-way)
        ┌───────────────────▼────────────────────────────────────┐
        │  PWA — laptop, phone, tablet. Any browser.             │
        │  dial controls · live transcript (2 lanes) ·           │
        │  SAY card + TONE chip · objection badge ·              │
        │  stage · latency badge · budget meter · kill switch    │
        └────────────────────────────────────────────────────────┘
                            │
                    OpenRouter (Nemotron) → fallbacks
```

The browser talks to the backend over **HTTPS + SSE**. The only inbound WebSockets the server
accepts are from the carrier (media + events). This keeps the client simple and means a dropped
browser tab does not drop the call.

---

## 3. The four seams

Everything provider- or source-specific sits behind an interface. Swapping any of them must be
an env change plus a registry entry — never a rewrite. This is the brief's provider-independence
requirement made concrete.

### 3.1 `AudioSource`

```python
class AudioSource(Protocol):
    """Where the two audio tracks come from."""
    async def start(self, session: Session) -> None: ...
    async def stop(self) -> None: ...
    def tracks(self) -> TrackPair:  # {"prospect": ..., "self": ... | None}
        ...
```

| Implementation | Tracks | Status | Use |
| --- | --- | --- | --- |
| `carrier_media_stream` | prospect + self, separated | **Primary** | Live calls |
| `browser_mic` | one mixed track | Fallback | If the carrier path fails; speakerphone in the room |
| `file_upload` | one or two tracks | Testing | Recorded mock calls, STT accuracy measurement |
| `mock` | canned transcript | Dev | Sandbox (no egress), UI work, deterministic tests |

### 3.2 `STTProvider`

```python
class STTProvider(Protocol):
    async def open_stream(self, track: str, on_result: Callable) -> StreamHandle: ...
    async def transcribe_file(self, audio: bytes, mime: str) -> Transcript: ...
```

| Implementation | Use |
| --- | --- |
| `deepgram` | Streaming (primary) + prerecorded file |
| `groq_whisper` | Prerecorded alternative — evaluated in M3, not a requirement |
| `mock` | Tests |

> Note: with the carrier path, Deepgram is fed from our server, so we use the **permanent key
> server-side** and do not need ephemeral tokens. The `/v1/auth/grant` flow from BUILD_PLAN §1.2
> is retained only for the `browser_mic` fallback, where the client streams directly.

### 3.3 `LLMProvider`

```python
class LLMProvider(Protocol):
    async def suggest(self, ctx: CoachContext) -> Suggestion: ...
    @property
    def name(self) -> str: ...
```

| Implementation | Use |
| --- | --- |
| `openrouter_nemotron` | Primary reasoning |
| `openai_compatible` | Any fallback (paid OpenRouter route, NVIDIA NIM, local vLLM, Groq) |
| `mock` | Sandbox, tests, demo mode |

### 3.4 `TelephonyProvider`

```python
class TelephonyProvider(Protocol):
    async def dial(self, to: str, from_: str) -> CallHandle: ...
    async def hangup(self, call_id: str) -> None: ...
    async def send_sms(self, to: str, body: str) -> None: ...   # for the self-demo motion
```

| Implementation | Use |
| --- | --- |
| `twilio` / `telnyx` | Live calls |
| `mock` | Dev — simulates a call, feeds a scripted conversation through the real pipeline |

**`mock` telephony is deliberately powerful.** Because it pushes a scripted two-track
conversation through the *entire real pipeline* — STT mock → turn controller → reasoning →
validator → SSE — we can develop and test the whole system in this sandbox and in CI without
touching a provider. It is the difference between "we can't test anything offline" and "we can
test everything except the network."

---

## 4. The audio path in detail

### 4.1 Primary — carrier media stream

1. Founder hits **Dial** in the coach UI. Backend calls the carrier API.
2. Carrier connects the call and opens a WebSocket to our media endpoint, streaming both tracks
   as base64-framed PCM16 (carrier-specific framing; normalise on ingest).
3. The relay decodes and forwards each track to its own Deepgram stream — **prospect track and
   founder track transcribed separately**.
4. No audio is written to disk by default. `STORE_TRANSCRIPTS` gates the transcript only.
5. On hangup, the carrier event closes the session; both Deepgram sockets are closed and state
   is flushed.

**Media endpoint auth:** the carrier cannot send our headers, so the endpoint is authenticated by
a **short-lived signed token in the URL path**, minted when the call is placed, single-use, and
bound to the call SID. Never a static secret in a URL.

### 4.2 Fallback — browser microphone

Used if the carrier path is unavailable, or when the founder is running a business VoIP app on
a phone with speakerphone on and a second device running the PWA nearby.

- Browser captures via `getDisplayMedia` (tab audio — preferred, digital) or `getUserMedia`
  (microphone — acoustic fallback).
- Streams to Deepgram with an ephemeral token minted by the backend (BUILD_PLAN §1.2).
- Forwards **final transcripts only** to the backend over SSE/HTTP; interim stay client-side.
- Single mixed track, so `speaker` is `unknown`. The UI gains a tap-to-toggle ("my turn /
  prospect") and the turn controller suppresses triggers while the founder is marked speaking.

**Reality check on quality:** this path hears the prospect through a speaker and a microphone.
Expect materially worse transcription than §4.1. It exists to keep the product *usable*, not to
be equivalent — and the difference should be visible in the UI so the founder knows which mode
he is in.

### 4.3 Testing — file upload

`POST /api/transcribe-file` → batch transcription → feed the resulting segments through the
reasoning pipeline as if live. This is how we measure STT accuracy against a known script and
how we build the eval corpus.

---

## 5. Turn detection

The single most important control in the system. Reason on every fragment and we burn the free
tier in four minutes and flood the founder with noise. Reason too rarely and the suggestion
arrives after the moment has passed.

**Trigger a reasoning request when all of these hold:**

| Condition | Value | Why |
| --- | --- | --- |
| Segment is **final** (not interim) | required | Never reason on partial words |
| Speaker is the **prospect** | required | Never suggest a response to the founder's own words |
| Utterance end signalled | `speech_final` or `UtteranceEnd` | Don't fire mid-sentence |
| Silence since last word | ≥ 700 ms | Prospect may just be pausing |
| New words since last trigger | ≥ 3 | "Yeah" is not a turn |
| Time since last trigger | ≥ 8 s | Cooldown |
| Budget available | see §7.3 | Protect the rate limit |

Plus a manual **"Suggest now"** button — because the founder is the best turn detector in the
room, and the system should defer to him instantly.

**Transcribing the founder's track is not optional.** It is what lets the coach know:

- which questions have already been asked (anti-repetition — the #1 failure mode in
  SaadKhaja.md §13.21);
- which claims were already made, so a later suggestion cannot contradict an earlier one;
- whether he has already delivered the price, the demo, or the close.

---

## 6. Reasoning pipeline

### 6.1 Context assembly

The prompt is assembled per request, in a fixed order, from four sources. It is **compact and
bounded** — latency is the budget, and every token of context costs response time.

```
┌─ SYSTEM ──────────────────────────────────────────────────────────┐
│  Role: live sales coach for TradeFlow's founder.                  │
│  Output contract (§6.2) — strict JSON.                            │
│  HARD RULES: §8.2 NEVER-list from TradeFlowContext, verbatim.     │
│  Behaviour: prefer a question when unclear; never invent product  │
│  facts; defer to playbook stage architecture.                     │
├─ PRODUCT TRUTH ───────────────────────────────────────────────────┤
│  TradeFlowContext.md §1–§4 + §8  (always included — it is the     │
│  truth layer and it is short. Cheaper than getting it wrong.)     │
├─ SALES KNOWLEDGE ─────────────────────────────────────────────────┤
│  SaadKhaja.md — section-routed, top 2–3 by tag overlap with the   │
│  latest prospect utterance + detected stage. Never the whole file. │
├─ PLAYBOOK ────────────────────────────────────────────────────────┤
│  Sales_Playbook.md §3 stage table + §4 the objection entry that    │
│  matches the detected class.                                      │
├─ SESSION STATE ───────────────────────────────────────────────────┤
│  stage · objections already raised · questions already asked ·    │
│  facts already established · product claims already made ·        │
│  compact rolling summary of older turns                           │
├─ CONVERSATION ────────────────────────────────────────────────────┤
│  last ~10 turns, speaker-labelled, verbatim                       │
└───────────────────────────────────────────────────────────────────┘
```

The section router is keyword/tag overlap, not embeddings. `SaadKhaja.md` is tag-indexed
precisely so this works without a vector database. If the knowledge library grows past ~5 files,
add retrieval then — not now.

### 6.2 Output contract

```json
{
  "say":            "One sentence, out-loud ready, ≤ 30 words",
  "tone":           "confident_and_assertive",
  "do":             "One next action or question, ≤ 12 words",
  "objection_class":"buying | allocation | access | need_gap | trust_sideload | timing | fit | none",
  "stage":          "open | permission | discovery | delta | objection | proof | method | price | close | nurture",
  "playbook_ref":   "4.5",
  "confidence":     "high | medium | low",
  "needs_clarification": false
}
```

Design notes on the non-obvious fields:

- **`tone`** is not decoration. SaadKhaja.md §6 claims tonality carries the majority of the
  outcome, and the playbook maps specific tones to specific call moments (calm-and-reassuring
  for pain extraction, confident-and-assertive for price). A coach that outputs only words is
  solving a fraction of the problem. Six enum values, from §6.2.
- **`objection_class`** drives a UI badge and forces the model to commit to a diagnosis. The
  playbook's §4 makes clear these need *different moves* — mistaking an allocation problem for
  an access problem loses the sale, and mistaking a buying objection for a real objection
  means conceding price for no reason. Classifying explicitly makes the error visible.
- **`playbook_ref`** exists for evaluation. It lets a human see which rule fired, which is how
  we find out whether the router is selecting the right knowledge.
- **`stage`** makes repetition detectable: if stage hasn't advanced in six turns, something is
  wrong with the call.

### 6.3 The claim validator

Runs **after** generation, **before** display. This is the mechanism that turns "the model was
told not to lie" into "the product does not lie."

**Deterministic, not model-based** — a model asked to police itself will not do so reliably, and
a false negative here is the one failure the project cannot recover from.

| Check | Action on failure |
| --- | --- |
| `say` matches any **[NEVER]** pattern (TradeFlowContext §8.2) — "AI-powered", "encrypted backup", "never double-books", "integration", "guaranteed", "100% secure", "free texting", any testimonial shape | Regenerate once with a corrective instruction naming the violation |
| `say` contains an **integration name** (Jobber, Housecall Pro, ServiceTitan, any calendar/CRM) | Block |
| `say` contains a **dollar figure** not present in TradeFlowContext | Block |
| `say` ends on a statement when `needs_clarification` is false | Flag as advisory (SaadKhaja §3.5) — not a hard block |
| `say` > 30 words, or contains multiple distinct asks | Regenerate for length |
| `say` re-asks a question already in session state | Regenerate with the answered question named |
| JSON invalid or schema-violating | One repair attempt → then degraded card |

**Degradation is a first-class outcome, not an error.** If the second attempt also fails:
render a neutral card — *"Not sure what to say here — ask them to clarify what they mean."* —
with a visible "couldn't generate" state. **It never spins, never hangs, never shows the
founder something unverified.**

---

## 7. State model

### 7.1 Session

```python
@dataclass
class Session:
    id: str
    created_at: datetime
    mode: str                      # "live" | "mock" | "replay"
    audio_source: str              # carrier_media_stream | browser_mic | file_upload | mock
    stage: str
    turns: list[Turn]              # speaker-labelled
    summary: str                   # compact rolling summary of older turns
    concerns: dict[str, str]       # objection_class -> status
    questions_asked: list[str]     # normalised, for anti-repetition
    claims_made: list[str]         # what the founder has asserted
    budget: BudgetState
    latency: list[LatencyRecord]
    consent: ConsentRecord | None
```

### 7.2 Storage

**In-memory by default.** SQLite only if it earns its place — the honest test being whether we
need sessions to survive a server restart. For a live-call tool where the session ends when the
call ends, in-memory is correct.

**No audio stored. No transcripts stored by default.** `STORE_TRANSCRIPTS=1` is a development
flag, and enabling it on a real prospect call is a privacy decision, not a config toggle. The
backup/consent implications are in §11.

### 7.3 Budget guard

Free-tier reality (BUILD_PLAN §5.3): ~20 req/min, ~200 req/day, and a normal call wants 40–80
requests. The guard enforces debounce, cooldown, min-words, and a per-session cap, and it
exposes a **visible meter** in the UI. When the budget is exhausted the coach says so rather
than going silent — the founder needs to know he is alone, not wonder why the card is empty.

---

## 8. Client

A PWA, ~5 screens' worth of functionality. Design constraint: **readable at arm's length while
talking to a stranger.**

```
┌───────────────────────────────────────────────┐
│ ● LIVE  ·  04:12  ·  187 ms   ·  ▓▓▓▓░░ 34/120│  status · duration · latency · budget
├───────────────────────────────────────────────┤
│                                               │
│  SAY                                          │
│  ┌─────────────────────────────────────────┐  │
│  │ "When your phone rings and you're under  │  │  ← the one thing that matters
│  │  a sink — what happens to that call?"    │  │
│  └─────────────────────────────────────────┘  │
│  TONE: calm & reassuring      [🔄 again]      │
│                                               │
│  DO: Get him to say a number out loud         │
│                                               │
├───────────────────────────────────────────────┤
│ NEED GAP                          DISCOVERY   │  ← objection badge · stage
├───────────────────────────────────────────────┤
│ PROSPECT                                      │
│ ...it just rings out, I call em back later    │
│ ME                                            │
│ no worries, how long's later usually?         │
├───────────────────────────────────────────────┤
│   [ DIAL ]  [ MUTE ]  [ SUGGEST NOW ]  [ END ]│
└───────────────────────────────────────────────┘
```

Rules that shape the UI:

- **The SAY card is the largest element on screen.** Everything else is secondary.
- **The client never blocks on the model.** It renders the last suggestion and shows a pending
  state; a late suggestion that arrives at the wrong moment is worse than none.
- **Two transcript lanes, always labelled.** With dual-channel audio the labels are free and
  they stop the founder misreading who said what.
- **Tone is visible next to the words**, because it is half the instruction.
- **Errors are always visible.** Listening, reconnecting, budget-exhausted, provider-down,
  degraded-card — each has a distinct state. Silence is never a state.
- **Call controls live in the app** so the coach can act: later it can dim or mute itself while
  the founder is speaking, and detect a hangup. Two browser tabs (dialer + coach) is a bad
  experience and is not the target design.
- **A single kill switch** stops transcription immediately — required by the brief's privacy
  section.

---

## 9. Latency instrumentation

Every request carries its own stopwatch; the segments are recorded per turn and surfaced on the
card. Targets are what we aim for, to be measured and reported honestly — not claims.

| Segment | Target | Measured where |
| --- | --- | --- |
| prospect stops speaking → carrier WS delivers last frame | ≤ 150 ms | relay |
| → Deepgram final | ≤ 600 ms | STT wrapper |
| → trigger decision | ≤ 50 ms | turn controller |
| → backend → model first byte | ≤ 1200 ms | LLM wrapper |
| → validated + rendered in browser | ≤ 100 ms | client |
| **end of speech → SAY card visible** | **≤ 2500 ms** | client stopwatch |

`docs/LATENCY.md` accumulates p50/p95 per segment per run. **We publish the number we measure,
including the bad one.**

**Known threat:** Nemotron Ultra is a reasoning model, and reasoning tokens can dominate the
response time. Mitigations to test in order: `reasoning.effort` low/none → hard `max_tokens` →
the smaller sibling model on the latency path → and only if p95 still misses, token streaming
with an incremental parser. We do not build streaming first; we measure first.

---

## 10. Deployment topology

| Environment | Where | Why |
| --- | --- | --- |
| **Dev / sandbox** | This repo + Arena sandbox | Egress is restricted to GitHub/npm/PyPI, so **providers are unreachable**. `mock` LLM + `mock` STT + `mock` telephony let the full pipeline, UI and tests run here. |
| **Live testing** | Fly.io / Railway | Needs **inbound WebSockets** for carrier media, plus egress to OpenRouter and Deepgram. Codespaces is fine for development but a poor fit for carrier webhooks — public port URLs are auth-gated and unstable. |
| **Production** | Same host, `min_machines_running = 1` | See below. |

### The free-tier trap

A host that sleeps on idle is a **false economy here**. The failure mode is a dead coach
mid-sales-call, which costs a $599 sale to save a few dollars a month. Run one always-on
instance — a small paid instance is the correct choice for a live revenue tool. "Free tier
where compatible" from the brief still applies to *development*, and it has already paid off
(everything runs in mock mode).

**Requirements for the host:** inbound WSS accepted, no aggressive idle timeouts (media streams
are long-lived and quiet between sentences — a 60-second idle timeout would kill a call),
secrets in environment variables, and outbound access to `openrouter.ai` and `api.deepgram.com`.

---

## 11. Privacy, consent and recording

Non-negotiable, and it constrains the architecture rather than being bolted onto it.

- **Consent gate before transcription.** The session will not start streaming until consent is
  recorded. Several US states require all-party consent, so this is a per-call gate, not a
  setting.
- **No audio at rest.** Audio is relayed in memory and discarded. Recording is off by default
  and, when used for its legitimate purpose (building the eval corpus from *consented mock
  calls*), is a separate explicit session mode.
- **Transcripts in memory only** unless `STORE_TRANSCRIPTS=1`, which is a development flag and
  must never be on for real prospect calls.
- **Real prospect calls require a paid model route.** The free OpenRouter listing logs usage and
  may feed NVIDIA products. Sending real prospect conversations through it is not acceptable,
  and the model provider is swappable precisely so this switch is one env var.
- **The kill switch** stops transcription immediately, mid-call.

---

## 12. Failure and degradation ladder

Every layer fails to a *visible* state. The founder must always know what the coach is doing.

| Failure | Detection | Behaviour |
| --- | --- | --- |
| Deepgram socket drops | WS close | Auto-reconnect with bounded backoff; UI shows "reconnecting"; the call continues unassisted |
| Deepgram unavailable | Connect fails ×3 | Banner: transcription down, manual mode — founder types or carries on alone |
| LLM 429 / rate limit | HTTP 429 | Bounded backoff → then degrade to the last suggestion staying on screen, greyed |
| LLM timeout | > `LLM_TIMEOUT_S` | Cancel, render degraded card, do not block the next trigger |
| LLM returns invalid JSON | Parser | One repair attempt → degraded card |
| Validator blocks twice | Validator | Degraded card: "ask them to clarify" |
| Carrier media stream drops | WS close | Attempt re-establish; if it fails, fall back to `browser_mic` and tell the founder |
| Browser disconnects | SSE close | **The call continues.** Media stream and transcription are server-side; the browser reconnects and catches up |
| Budget exhausted | Guard | Meter turns red; coach says so explicitly |
| Backend restart mid-call | — | Session lost (in-memory). Accepted: the call continues by phone while the coach is down. Revisit only if it happens in practice. |

Notice the pattern: **the phone call is never dependent on the coach.** The coach is an
assistant, not a switchboard. Nothing in this system should be able to drop a live call with a
$599 prospect on the line.

---

## 13. What this changes in `BUILD_PLAN.md`

| BUILD_PLAN item | Change |
| --- | --- |
| §1.2 — ephemeral Deepgram tokens so the backend avoids WebSockets | **Partly superseded.** We now accept inbound WS for carrier media. Tokens are retained only for the `browser_mic` fallback. |
| §2 — browser streams to Deepgram directly | **Superseded.** Server-side relay; browser is a display surface. |
| §3 — capture path decision (second device / softphone / upload) | **Superseded** by the carrier choice; `file_upload` and `browser_mic` survive as fallback and testing sources. |
| M3 — build Android audio capture | **Shrinks substantially.** No Android microphone work on the primary path. |
| M4 — on-device capture feasibility | **May become unnecessary.** Replace with a carrier-integration and latency test. |
| M0 — privacy doc | Gains a **consent gate** as an architectural requirement, not just a document. |
| Hosting | Free tier is no longer sufficient for production (inbound WS + no idle sleep). |

---

## 14. Cost shape

Per-minute, so it scales with usage rather than being fixed. Verify all published rates before
committing — they change.

| Component | Structure |
| --- | --- |
| Carrier — outbound US local | ~$0.002–0.014 / min depending on provider; number rental ~$1–2 / month |
| Carrier — media streaming | may be metered separately; confirm |
| Deepgram streaming | per-minute of audio; **two streams per call** (both tracks) — budget for double |
| OpenRouter Nemotron | free tier for testing; **paid route required for real calls** |
| Host | one always-on small instance |

**The two-stream point matters:** transcribing both tracks doubles the STT cost per call. It is
worth it — knowing who spoke and what the founder already said is the difference between a
coach and a transcription service — but it should be a conscious line item, not a surprise on
the invoice.

---

## 15. Open risks

1. **Can an India-registered account obtain a US number and place US outbound calls?** This
   gates the entire primary architecture. Resolve first, before building §4.1. If the answer is
   no, the project falls back to §4.2 and the capture problem comes back in reduced form.
2. **Carrier routing latency from India.** A media stream from a US carrier to a server in Asia
   adds a round trip to every segment of the latency budget. If it is material, place the
   backend in a US region and accept the founder's UI latency instead — the UI is one small SSE
   stream, the audio is not. **Decision rule: optimise for the audio path, not the UI path.**
3. **Whether the carrier's `send_sms` and the product's SMS share a number.** If the coach dials
   from a US number but TradeFlow sends from the founder's Indian SIM, the self-demo motion in
   `SALES_PLAYBOOK.md` §2 breaks. Needs resolution before that motion is used at volume.
4. **A2P filtering on automated SMS** (`TradeFlowContext.md` §10.11) — unresolved, and it is
   the one open question that can harm a customer rather than just cost us money.
5. **Reasoning-model latency** may make the 2.5 s target unreachable. Measure in M1; the
   fallback ladder is defined in §9 and a non-reasoning model on the live path is an acceptable
   answer if the quality holds.
6. **Nobody has heard the coach on a real call yet.** Every latency number and every playbook
   assumption is currently theoretical. That is what M1–M4 exist to fix.

---

## 16. Explicitly deferred

Not in v0, and the architecture should not be contorted to accommodate them: vector database ·
multi-user accounts · CRM integration (the *product* has none, so the *coach* should not pretend
otherwise) · automatic cellular-call capture on the phone · call recording storage ·
diarization (unnecessary with dual-channel audio) · fine-tuning · analytics dashboards ·
a native Android app for the coach itself (the PWA covers it) · token streaming (until measured
as necessary) · auto-detected consent.

---

## 17. Build order implied by this architecture

Mapped onto the milestones in `BUILD_PLAN.md`:

1. **M1 —** `LLMProvider` (mock + OpenRouter), context assembly over the two knowledge files,
   output contract, **claim validator**, `POST /api/suggest`. *No audio at all.* Runs entirely
   in this sandbox against `mock`.
2. **M2 —** session state, rolling summary, anti-repetition, `eval.py` over scripted
   conversations. Still no audio.
3. **M3 —** `AudioSource` + `STTProvider` interfaces with `file_upload` and `mock`. Measure STT
   accuracy and end-to-end latency on recorded conversations.
4. **M3.5 —** *new, gated on risk §15.1:* `TelephonyProvider` + `carrier_media_stream`. This is
   the step that removes the Android capture problem, and it should be attempted as soon as the
   carrier question is answered — not last.
5. **M4 —** first consented live call on the real pipeline. Measure everything.
6. **M5 —** failure drills against the §12 ladder, fallback provider tested by *forcing*
   failures, eval against held-out scenarios.

The important sequencing change from the original plan: **the carrier integration moves earlier**,
because it is the decision that determines whether the hard problem exists at all.
