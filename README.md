# TradeFlow Sales Coach

An Android-first, AI-powered live-call sales assistant for TradeFlow's founder: it hears
the prospect, keeps a live transcript, and puts **one** line on screen to say next,
informed by sales knowledge and verified TradeFlow facts.

Stack: mobile-first PWA → FastAPI backend → Deepgram Nova-3 streaming STT →
NVIDIA Nemotron 3 Ultra via OpenRouter. Knowledge lives in two Markdown files under
`knowledge/`. No vector database in v0.

## Start here

**[`docs/BUILD_PLAN.md`](docs/BUILD_PLAN.md)** — the build procedure: verified findings,
target architecture, the capture decision, milestones M0–M5 with definitions of done,
latency/quality/rate-limit budgets, and the open decisions blocking each stage.

## Status

Pre-build. Repository contains the plan and configuration templates; no product code yet.

## Hard constraints this project takes seriously

- An ordinary Android app **cannot** capture a cellular call's audio. The prototype must
  prove a capture arrangement (second device, uploaded recording, or an app-controlled
  softphone line) before claiming live assistance.
- The free OpenRouter route logs usage and is rate-limited. It is for synthetic test
  conversations only — never a real prospect call.
- The permanent Deepgram key never reaches the browser; the client gets a short-lived token.
- The coach must never invent customers, results, guarantees, integrations, or revenue figures.
