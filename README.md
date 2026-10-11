# TradeFlow Sales Coach

An Android-first, AI-powered live-call sales assistant for TradeFlow's founder: it hears
the prospect, keeps a live transcript, and puts **one** line on screen to say next,
informed by sales knowledge and verified TradeFlow product facts.

Stack: mobile-first PWA → FastAPI backend → Deepgram Nova-3 streaming STT →
NVIDIA Nemotron 3 Ultra via OpenRouter. Knowledge lives in two Markdown files under
`knowledge/`. No vector database in v0.

## Start here

**[`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)** — the authoritative system design. Why
audio never touches the browser, the four provider seams, the reasoning pipeline and claim
validator, deployment topology, cost shape, and the failure/degradation ladder.

**[`docs/BUILD_PLAN.md`](docs/BUILD_PLAN.md)** — the build procedure. Verified findings,
milestones M0–M5 with definitions of done, and the latency/quality/rate-limit budgets.

**[`docs/SALES_PLAYBOOK.md`](docs/SALES_PLAYBOOK.md)** — the call-time layer. Call
architecture, the discovery drill, the 14-objection map, and the tonality cheat-sheet.

**[`docs/MARKET_AND_DIALER_VERDICT.md`](docs/MARKET_AND_DIALER_VERDICT.md)** — why Rebtel
was rejected, what to use instead, and the corrected ICP.

**`knowledge/`** — `SaadKhaja.md` (sales knowledge, tag-indexed for retrieval) and
`TradeFlowContext.md` (product truth; every claim tagged sayable, testable, or blocked).

## Status

Pre-build. Repository contains the plan, the knowledge base, and configuration templates;
no application code yet.

## Hard constraints this project takes seriously

- The coach's audio never comes from the Android phone's microphone. A US carrier with
  call-media streaming delivers both tracks digitally, which removes the capture problem
  entirely and gives us speaker separation for free.
- An ordinary Android app **cannot** capture a cellular call's audio — so the phone
  microphone is a fallback for degraded modes only, never the primary design.
- The free OpenRouter route logs usage and is rate-limited. It is for synthetic test
  conversations only — never a real prospect call.
- No audio is written to disk. Transcripts are in-memory unless explicitly enabled for
  development, and a consent gate blocks transcription on real calls.
- The coach must never invent customers, results, guarantees, integrations, or revenue
  figures. TradeFlow has **no integrations** and is **not** AI-powered; the claim validator
  hard-blocks both.
