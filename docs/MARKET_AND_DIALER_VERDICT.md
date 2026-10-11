# Verdict — Rebtel as the dialer, and the trade mix

**Date:** 2026-10-11 · **Status:** recommendation for decision · **Verdict: Rebtel is the wrong
dialer, and plumbing may be the wrong beachhead.**

Everything below is separated into **verified fact**, **inference**, and **judgement** so you
can disagree with the parts you should disagree with.

---

## 1. Verdict on Rebtel: do not build on it

Rebtel is a well-established (since 2006) consumer international calling app. It is good at
what it is for: cheap calls to family abroad. It is not a business dialing platform, and it
fails this project on **five independent counts**. Any one of them alone would be
survivable. Together they are disqualifying.

### 1.1 You cannot choose the number your prospect sees — and it will be Indian

**Fact, from Rebtel's own help centre:**

> "When you make a call through Rebtel, the number that will appear to the person you're
> calling (the B-side) is the one registered with your Rebtel account. At this time, it is
> **not possible to choose or change the number that shows** when calling other people."

Your account is registered to an Indian number. **A US plumber will see an Indian number.**

This is the most damaging single fact in this document. Cold-calling a US local service
business is hard at the best of times; doing it from a foreign number that the owner does not
recognise converts a difficult call into a reflex hang-up. Rebtel does have a "Local Numbers"
feature, but it works the other way round — it gives *you* a local number to dial *in order to
reach* your contact. It does not let you present a US number to them.

Note it also fails the transparency requirement in your own brief in an odd way: you cannot
even choose an honest, consistent business identity for outbound.

### 1.2 US carriers will label your calls "Spam Likely"

**Fact — verified via multiple sources including an FCC-background briefing:** US and Canadian
networks authenticate calls with **STIR/SHAKEN**, which assigns one of three attestation levels:

| Level | Meaning | Outcome |
| --- | --- | --- |
| **A – Full** | Carrier verified the caller's identity **and** their right to use that caller ID | Passes normally; may show "Verified" |
| **B – Partial** | Identity verified, number ownership not | Usually passes |
| **C – Gateway** | Carrier only confirms the call entered their network | **High risk of "Spam Likely" label or outright blocking** |

Calls originating outside North America receive **C-level attestation**. And per an April 2026
FCC rule change, **US gateway providers are now required to apply at least C-level attestation
to inbound international traffic** — which, as one analysis puts it, "often results in the call
being labelled 'Spam Likely'."

So: an India-originated Rebtel call to a US plumber is likely to arrive labelled as spam, or
routed straight to voicemail without ringing. Rebtel cannot fix this — it is an international
consumer calling app, not a US-registered carrier with SHAKEN/STIR credentials.

### 1.3 There is no path for the coach to hear the call — on one device or any device

**Fact (technical):** Rebtel runs on the phone app only. Per Rebtel's help centre:

> "Rebtel does **not** currently support making calls from a computer. To place calls, you'll
> need to use the Rebtel **mobile app** on your smartphone."

**Inference:** this closes off the cleanest possible architecture. If Rebtel had a web app,
Chrome's tab-audio capture (`getDisplayMedia` with "share tab audio") could feed the coach a
**pure digital, both-sides, zero-loss audio stream** with no speakerphone, no second device and
no acoustic degradation. That would have solved the audio-capture problem in §7 of your brief
outright.

With Rebtel on Android, you are back to the worst case:

- Rebtel routes calls via **intermediary access numbers**, and "toggles between data and GSM"
  — so some calls are ordinary cellular calls, during which **Android blocks third-party apps
  from the microphone entirely** (see `docs/BUILD_PLAN.md` §1.1).
- When it runs over data as VoIP, a second app grabbing the mic hits Android's concurrent-capture
  priority rules, and you are reduced to capturing **acoustic leakage through the phone's
  speaker** — which means speakerphone, which means the prospect hears it, which means they
  know they're on a speaker, which changes the conversation.
- Either way the coach hears a degraded, one-microphone, echo-laden version of a two-party call.

Your brief already says the prototype must not assume automatic cellular listening. **Rebtel
guarantees the worst case rather than avoiding it.**

### 1.4 It has none of the tooling a sales operation needs

No CRM, no lead management, no call recording, no call disposition, no power/sequence dialing,
no DNC screening, no analytics, no webhooks, no API. It is a consumer app with a dial pad.
Every one of these is something a founder doing outbound at volume will end up rebuilding by
hand — badly — or bolting on later.

For this project specifically: **no recording and no API means no audio data to transcribe,
test against, or improve the coach with.** The prototype's own evaluation strategy
(BUILD_PLAN M3/M5) depends on being able to replay conversations.

### 1.5 Your entire GTM sits inside a live US regulatory question

**Fact:** the FCC's 9th Further Notice of Proposed Rulemaking proposes to require
**foreign-originated calls to be labelled as international**, and may **limit the use of
US-based numbers by offshore call centres.** It is not final — but per compliance counsel
reviewing it, businesses "should watch for Federal Register publication and be ready to
participate in the comment period, **especially if they rely on offshore call centres**."

**Judgement:** you are an India-based founder cold-calling US businesses. That is not illegal,
and B2B cold calling to businesses is broadly permitted. But the regulatory current is running
against foreign-originated outbound calls, and the proposed rule would make an India-originated
number *worse*, not better, over the next year.

There is also a **two-party consent** issue for any recording (see §4).

### 1.6 The counter-argument, stated fairly

Rebtel is cheap, works from India, requires no setup, and gets you dialing today. If the goal
were to prove that you *can* hold a conversation with a US plumber and that the coach's
suggestions land, Rebtel is a zero-cost way to do the first 20 test calls — **provided you go
in knowing the answer rate will be artificially awful** and do not conclude the offer doesn't
work from those calls. Use it as a free trial, not as the foundation.

---

## 2. What to use instead

The replacement has to satisfy four requirements, and it is worth being explicit about them
because they are what eliminate Rebtel:

| # | Requirement | Why |
| --- | --- | --- |
| **R1** | **US caller ID you own** | Otherwise §1.1 and §1.2 kill your answer rate |
| **R2** | **STIR/SHAKEN A-attestation** as a US-registered originator | Otherwise "Spam Likely" |
| **R3** | **Access to the call audio** — ideally as a digital stream | Otherwise the coach is deaf or reduced to speakerphone acoustics |
| **R4** | **Recording + a way to replay it** | Otherwise you cannot test, evaluate or improve the coach |

### 2.1 The strong option: a programmable carrier with media streaming

**What it gives you:** a US local number in the prospect's own area code, US-originated with
A-level attestation, plus SHAKEN/STIR registration, Voice Integrity (anti-spam-labelling),
Branded Calling and CNAM registration. Twilio explicitly documents all of these as
answer-rate tools for US outbound, and states that **regulatory compliance registration is
not required for US phone numbers**.

**And the important part — R3:** programmable voice platforms can stream live call audio
(call media streams / media forking) to a WebSocket endpoint. That means **the coach can be fed
a clean, digital, dual-channel stream of both parties** — no speakerphone, no second device, no
acoustic loss, and perfect speaker separation for the transcript.

**This is the single biggest architectural win available to this project**, and it makes
BUILD_PLAN §3 Path C into the *primary* path rather than a fallback. It also means:

- The transcription is dramatically more accurate (the prospect's voice arrives digitally, not
  through the air).
- Diarization stops being a problem — the channels are already separated, so you know exactly
  who said what. That removes the biggest risk in the v0 design (figuring out whether the
  prospect or the founder is speaking).
- Latency improves — no acoustic capture delay.
- The Android capture problem **disappears entirely**, because the audio never needs to come
  from the phone's microphone at all. You dial from a laptop, or a softphone, or the carrier's
  own web dialer.

**Trade-offs, honestly:** it costs per minute (roughly $0.014/min to make a US local call and
about $1.15/month for a US local number on the larger provider's published rates; the smaller
competitor publishes $0.002/min), it requires real build work (an outbound call flow, webhooks,
a media-stream consumer, plus DNC and consent handling), and it means the founder no longer
just taps "call" in a consumer app. **Verify pricing and current policy on the provider's own
site** — these rates move. Also note that trial accounts are restricted to your sign-up country;
you need a paid account to dial internationally.

**One thing to verify before committing:** whether the provider will issue and permit US
outbound numbers on an India-registered account, and any KYC that applies. This is a genuinely
open question and it decides whether this path is available to you at all.

### 2.2 The pragmatic middle: a business VoIP platform with a US number

A US-facing business phone system (the Dialpad / JustCall / CloudTalk / OpenPhone class) will
give you R1, R2 and R4 today, on an Android app, with much less build work than a raw carrier
integration. Some of them expose APIs and recording downloads, which covers most of R3 after
the fact — even if not live.

| | **Consumer app (Rebtel)** | **Business VoIP** | **Programmable carrier** |
| --- | --- | --- | --- |
| US number you own | ✗ | ✓ | ✓ |
| A-attestation | ✗ (C) | Usually ✓ | ✓ |
| Live audio feed to the coach | ✗ | Usually ✗ | **✓** |
| Recording for evaluation | ✗ | ✓ | ✓ |
| Build effort | none | low | **high** |
| Cost | ~free | ~$15–30/user/mo | per-minute |
| Solves Android capture | ✗ | ✗ | **✓** |

### 2.3 Recommendation

1. **Immediately:** decide that Rebtel is a throwaway trial tool, not the platform.
2. **This week:** test one business VoIP or programmable-carrier account, confirm a US number
   plus SHAKEN/STIR registration, and confirm that it will issue numbers to an India-registered
   business.
3. **Then:** decide between §2.1 and §2.2 based on whether you want to solve the audio-capture
   problem permanently. My recommendation is to **choose a provider with media streaming even if
   you don't use it on day one** — it is the difference between a prototype that works on a
   laptop with a second device and a product that works on one phone.
4. **Update `docs/BUILD_PLAN.md` §3** accordingly — the capture-path decision is no longer
   "second device vs. softphone vs. recording", it is "which carrier gets us both sides digitally".

---

## 3. Verdict on the trade mix: plumbing may be your worst choice, not your best

You said plumbers first, then electricians, and "mostly high-ticket contractors". There is
third-party data suggesting **the plumbing-first sequencing is backwards for a missed-call
product.**

### 3.1 The data

**Source:** Invoca 2026 home-services benchmarks, as reported by a third party
(`skygnosis.com`). **This is third-party secondary reporting, not first-party data, and I have
not verified it against Invoca's original report. Treat it as a hypothesis to test, not as
fact.**

| Trade | Answer rate |
| --- | --- |
| **Plumbing** | **74%** |
| Pest control | 73% |
| *All home services* | *52%* |
| **HVAC** | **34%** |
| **Construction** | **32%** |

Also reported: businesses with **fewer than 5 technicians book only 24% of calls**, versus 59%
for 25+ technicians.

### 3.2 What this means

If these numbers are directionally right, **plumbers already answer three out of four calls.**
They are among the *best* in home services at picking up — which makes them the **weakest**
possible beachhead for a product whose entire value proposition is "you're losing calls you
don't answer."

Meanwhile **HVAC misses about two out of every three calls, and construction misses about
two in three.** Those are the trades where the pain is acute enough that a contractor will
recognise it instantly when you describe it — which is exactly the condition SaadKhaja.md §2.2
requires for the Value Delta to be large, and §9.3 Gate 1 requires for the need to be specific
and felt rather than hypothetical.

**Judgement:** "high-ticket contractors" is the right instinct and the wrong ordering. The
combination you want is **high ticket value × low answer rate**:

| | High answer rate | Low answer rate |
| --- | --- | --- |
| **High ticket value** | Plumbing, pest control — pain is latent, harder sell | **HVAC, construction, roofing, remodeling — ideal** |
| **Low ticket value** | Weak | Volume play |

HVAC also has a seasonal emergency dynamic (no-heat / no-cool) that makes a missed call
viscerally expensive to the owner in a way a routine plumbing call is not.

### 3.3 Caveats, and how to resolve it

- Single source, secondary reporting, no published methodology visible to me.
- "Answer rate" is not the same as "missed revenue" — a missed call at 2am for a $90 job is
  different from a missed call for a $12,000 system replacement.
- Trade mix varies enormously by market and by firm size.
- **The plumbing answer rate may be high precisely *because* plumbing demand is urgent enough
  that owners prioritise answering** — which would mean plumbers have already solved the
  problem with a phone, and don't need you.

**How to resolve it in a week, for free:** in your first 30 test calls, ask the sequencing
question directly and record the answers. *"When your phone rings and you're under a sink —
what happens to that call?"* Track the answer by trade. Thirty calls across three trades will
tell you more than any benchmark, and you need to make those calls anyway.

**Do not change your ICP on the strength of this document.** Change it on the strength of your
own first 30 calls.

---

## 4. Two compliance items that need a real answer before real calls

I am not a lawyer and this is not legal advice. Both of these need a qualified opinion before
you record a real prospect.

### 4.1 Recording consent

Your brief already flags this. It gets sharper with US prospects: **several US states require
all-party consent** to record a call. If you are recording or transmitting prospect audio to
Deepgram and OpenRouter, you need to know the rule in the prospect's state, and you need a
consent line — which itself has to be delivered without poisoning the call.
`docs/BUILD_PLAN.md` M0 requires a `docs/PRIVACY.md` with a consent script. That is now a
**blocker for the first real call**, not a nice-to-have.

Note the design consequence: if all-party consent is required in a target state, the coach
needs a one-line consent ask that the founder can deliver naturally inside the first 30 seconds
— which interacts directly with the opening decision in SaadKhaja.md §11.3.

### 4.2 Data flowing to free providers

The free OpenRouter route logs usage and may use it to improve NVIDIA products, and it is
rate-limited. That is incompatible with real prospect calls. **The first real call requires a
paid route.** This was already flagged in `BUILD_PLAN.md` §1.3; choosing a proper dialer now
means the paid-model switch should happen at the same time, not later.

---

## 5. What changes in the build plan

| Finding | Change |
| --- | --- |
| Rebtel can't carry this | `BUILD_PLAN.md` §3 capture decision is superseded — replace Path A/B/C/D with "which carrier gives us digital dual-channel audio" |
| Programmable carrier can stream call audio | v0 client no longer needs Android mic capture at all; M3 shrinks substantially and M4's on-device feasibility test may become unnecessary |
| Speaker separation becomes free | Removes the "who is speaking" risk from the v0 design and the diarization work from v1 |
| Spam labelling | Add an answer-rate measurement to the eval harness — a good suggestion on a call that never connects is worth nothing |
| Trade mix | Add a trade comparison to the first 30-call experiment |
| Recording consent + paid model route | Both become M0 blockers rather than M5 concerns |

**Net effect: this is good news for the build.** The hardest unsolved problem in the original
plan — how the coach hears the prospect through an Android phone — is solvable by choosing the
right dialer infrastructure instead of the right audio-capture hack. That was always the
better end of the trade.
