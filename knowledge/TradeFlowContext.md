# TradeFlowContext.md — Verified Product & Offer Reference

> **Population status: COMPLETE — sourced from founder-supplied `ProductContext.md`.**
> **Last updated:** 2026-10-11
> **Source basis:** public website (tradeflowapps.netlify.app) + static inspection of the
> supplied Android APK. **Not** a source-code audit, penetration test, legal review, or
> runtime QA. The app was never installed or exercised.
>
> **Retrieval contract for the coach:** every fact carries an evidence tag. Only facts tagged
> **[SITE]** or **[APK]** may be *stated* to a prospect, and **[APK]** facts must be phrased as
> design intent ("the app is built to…"), never as verified behaviour. **[TEST]** facts may be
> raised only as something to test together. **[NEVER]** items are hard-blocked by the claim
> validator — a suggestion containing one is regenerated, then degraded to a clarifying
> question.

---

## 1. What TradeFlow is — the one-sentence version

**Approved sales description** (verbatim from the product review, §10 — this is the safest
baseline the coach may build from):

> TradeFlow is a one-time-purchase Android app for solo tradespeople who miss calls while
> working. It can text missed callers from the business phone's own SIM, guide interested
> customers through service and appointment options, and let the owner approve a proposed job.
> It is designed around local-first data storage and has no advertised monthly TradeFlow
> subscription. Phone compatibility, permissions, carrier SMS charges, and message delivery
> still matter.

**Value proposition form** (SaadKhaja.md §3.0 requires "I do X through Y to get Z" — the call
has nowhere to land without it):

- **X** — recover missed-call opportunities
- **through Y** — automated SMS follow-up from the owner's own SIM and number
- **to get Z** — owner-approved job bookings, without the owner stopping work

The reviewed document's own framing: *"Help recover missed-call opportunities and turn suitable
text conversations into owner-approved job bookings, using the Android phone and SIM the
business already has."* **[SITE]**

---

## 2. Mechanism — exactly how it works

The intended flow **[SITE]**, with APK corroboration where noted:

1. The owner switches on **Busy Mode**. **[APK]** — strings include `Busy Mode`, `Busy Mode ON`,
   and state indicating auto-replies do not run while Busy Mode is off.
2. A call is missed. **[APK]** — the build contains a `MissedCallHandler` and text describing
   call-log (`READ_CALL_LOG`, `READ_PHONE_STATE`) checks to confirm the call was actually missed
   before texting, so it should not reply to a call that was answered.
3. TradeFlow sends an **initial SMS from the owner's own SIM/number**. **[SITE]**
4. The customer replies with a service choice (e.g. a numbered option — leak, drain, toilet,
   water heater, Other) or other information. **[SITE]** *(the specific service list is an
   illustration, not proof every install starts with those options)*
5. TradeFlow asks follow-up questions and offers available appointment slots. **[SITE]**
6. The owner reviews the proposed booking and **explicitly approves**, calls the customer, or
   rejects it. **[SITE]** — the website states TradeFlow does *not* finalise a job without
   owner approval.
7. Approved work appears in the app's job records. **[SITE]**

**This is rule-driven SMS orchestration.** It is not a voice product and not an AI product.

### 2.1 Human handoff and opt-out

- **[SITE]** Replies containing "human", "call me", or "help" pause automation and alert the owner.
  "STOP" is treated as an opt-out and the number should not be messaged again by automation.
- **[APK]** A handoff pattern matches `call`, `call me`, `human`, `help`, `person`, `agent`,
  `stop`. Conversation records carry a `needsHuman` state. Opt-out and blocked-number data
  models are present.
- **[TEST]** Exact matching and case handling, opt-out persistence, what happens to pending
  queued messages, and whether opt-outs survive backup/restore and app updates. **A STOP
  handler existing is not proof of legal compliance** — do not claim compliance on this basis.

---

## 3. Commercial offer

| Item | Value | Tag |
| --- | --- | --- |
| Price | **US$599, one time** | **[SITE]** — state as the *advertised* price; checkout and licence terms unconfirmed |
| Licence framing | "Lifetime access" | **[SITE]** — scope is undefined (see §9 unknowns) |
| Ongoing TradeFlow fee | **$0/month** advertised | **[SITE]** |
| Messaging route | Owner's own SIM and mobile plan | **[SITE]** |
| Platform | **Android only** | **[SITE]** |
| Distribution | Direct APK download, **not** a Play Store listing | **[SITE]** |
| Web dashboard / iPhone app | Explicitly **none** | **[SITE]** |

### 3.1 What "no monthly fees" actually means — say it precisely

- **Sayable:** "TradeFlow itself charges no recurring software fee and no per-message gateway
  fee, because it uses your own SIM."
- **Also must be said when relevant:** this does **not** mean the phone plan, carrier SMS
  charges, or mobile service are free. Message delivery depends on carrier, phone, Android
  version, permissions and local conditions. **[SITE]**
- **Blocked:** any phrasing implying texting is free or unlimited. It is not — it is billed by
  the owner's carrier.

### 3.2 The competitor comparison — handle with tongs

The website compares $599 one-time against two **illustrative examples**: cloud CRM + texting
at $175/mo + $36/mo texting ($211/mo), and a virtual answering service at $350/mo. The page
labels these as illustrative.

- **[SITE]** May be referenced *as the website's own illustrative comparison*.
- **[NEVER]** Must not be described as a universal industry price, a quote, or a verified
  competitor cost.

---

## 4. Target customer

**[SITE]** A **solo operator or small field-service business whose owner personally answers the
business phone** and frequently cannot pick up while working.

Segments named as *potential* (explicitly "reasonable target segments, not evidence the app has
been tested or configured for every trade"): plumbers, electricians, HVAC technicians, appliance
repair technicians, handymen, other appointment-based local trades.

### 4.1 The architectural ICP constraint — read this before any targeting decision

The product's design imposes hard constraints on who can actually use it:

| Requirement | Why it excludes some prospects |
| --- | --- |
| One Android phone, owner's own SIM | Multi-truck shops with a dispatcher, office manager, or phone system are not the user |
| Owner personally answers the phone | If someone else answers, the product has no missed call to recover |
| Can sideload an APK and grant call-log + SMS permissions | Excludes low-willingness, heavily locked-down, or managed devices |
| $599 one-time | Requires real business revenue and enough job value for payback |

**Conclusion carried into `docs/SALES_PLAYBOOK.md`:** the ICP is an **owner-operator of a
solo-to-small trade business with meaningful job values, who answers his own phone.** That is a
narrower and more specific person than "high-ticket contractors" — and it is a *correction* to
the ICP advice given in `docs/MARKET_AND_DIALER_VERDICT.md` §3.

---

## 5. Feature inventory

### 5.1 Missed-call detection and initial reply
- **[SITE]** Uses call-log / phone-state information to determine a call was missed, then initiates a text.
- **[APK]** `MissedCallHandler` present; call-log and phone-state checks described.
- **[TEST]** Missed / answered / declined / cancelled / blocked / dual-SIM / voicemail
  scenarios; battery optimisation, background restrictions, reboot, and manufacturer-specific
  power management.

### 5.2 Busy Mode
- **[SITE]** The owner switches Busy Mode on while working.
- **[APK]** `Busy Mode`, `Busy Mode ON`, state showing auto-replies suppressed when off.
- **[TEST]** Whether it must be manual or can be scheduled, behaviour after reboot, and exactly
  which events it suppresses.

### 5.3 SMS conversation flow
- **[SITE]** Initial message explains the owner is working and asks what service is needed →
  numbered service choice → time options → customer picks a slot and gives name/address →
  proposed job presented to the owner for approval.
- **[APK]** App-specific SMS sender/receiver classes and message/conversation data models exist.
- **[TEST]** Exact wording, service list, language behaviour and conversation rules are
  **build-dependent** — do not promise the sample conversation is what a given install sends.

### 5.4 Availability and booking controls
- **[SITE]** Offered slots are based on working hours and existing appointments; the chosen slot
  is checked again before confirmation.
- **[APK]** Calendar, day-off, job, availability and slot classes/strings; a string indicates
  confirmation can be blocked due to a slot clash.
- Intended flow: **suggest slot → customer selects → owner approves → job created.**
- **[TEST]** Overlapping jobs, time zones, daylight-saving changes, day-off handling, edits and
  cancellations, stale slot offers, simultaneous conversations.
- **[NEVER]** **"Never double-books."** Treat slot-clash handling as an intended safeguard, not
  a proven guarantee, until adversarial testing demonstrates it.

### 5.5 Jobs, customers, messages, calendar
- **[APK]** Modules/components for customers, jobs, messages and threads, calendar, day-off
  records, blocked numbers, settings, diagnostics, a local data repository, and a Room database.
- **[SITE]** Job records contain customer name, number, address and selected slot. Also a
  service list, working hours, slot length, local backup/restore, and a local
  revenue/recovery counter.
- **[TEST]** Static inspection supports the presence of these modules; it does **not** confirm
  every dashboard metric, revenue counter or screen is complete, accurate or reliable.

### 5.6 Backup and restore — a real vulnerability, handle honestly
- **[SITE]** Local backup and restore so the owner can move data to a replacement phone.
- **[APK] ⚠️** A string explicitly states the backup writes a **plain-text, unencrypted file**
  containing customers, jobs, message history, days off and blocked numbers.
- **[NEVER]** **Do not describe exported backup files as encrypted.** The website's "encrypted
  local database" wording and the plain-text export are different things: even if the live
  database is encrypted, an exported backup may be readable by anyone who obtains the file.
- **[TEST]** Actual database encryption implementation and key management; exactly which fields
  are exported; whether export files are auto-deleted; restore validation; what happens if a
  backup is shared or synced to a cloud drive.

---

## 6. Privacy, permissions, installation friction

### 6.1 Intended privacy model — sayable, but carefully
- **[SITE]** Local-first: customer records and conversations are intended to remain on the phone.
- **[SITE/APK]** SMS is sent via the phone's SIM. The APK contains a message stating the app has
  no internet permission.
- **Sayable:** "designed to keep customer data on the device."
- **[NEVER]** Unqualified security guarantees, "100% secure", or implying that "no internet
  permission" eliminates all privacy or device-security risk.

### 6.2 Permissions requested
**[SITE/APK]** Call log / phone state (to verify a missed call), SMS receiving, SMS sending
(with send-status reporting), notifications, and boot handling. Android 14+ may require the
user to enable **Allow restricted settings** for the relevant background/SMS permissions.

### 6.3 Installation friction — the prospect will raise this, so the coach must own it
**[SITE]** Distributed as a direct APK because Google Play restricts broad use of call-log and
SMS permissions. Consequences, all sayable: Android may warn the APK is from outside Google
Play; the user may need to allow installation from their browser or file manager; restricted
settings and permission grants may need manual steps; some devices restrict background work or
auto-start.

**Rules the coach must follow on this topic [NEVER]:**
- Never tell a customer to ignore security warnings.
- Never claim every Android phone supports all background triggers identically.
- Never promise perfect SMS delivery or uninterrupted background execution.
- Instead: honest installation guide, name the publisher/build, explain why each permission is
  required, provide a safe support channel.

---

## 7. Product boundaries — what TradeFlow is NOT

**[SITE]** Not currently presented as: an iPhone app · a web dashboard or multi-user cloud CRM ·
a third-party SMS-gateway service · an AI voice receptionist · a system that auto-approves every
job · a guarantee of recovering a particular number or dollar amount of customers.

- **Main mechanism:** rule-driven SMS follow-up and appointment coordination on an Android phone.
- **[NEVER]** **Do not describe TradeFlow as AI-powered.** No released AI feature is established.
  *This applies to the coach itself: the reasoning model is a founder-side internal tool. It must
  never be described to a prospect as a TradeFlow feature, and it must never blur into the pitch.*

---

## 8. Claims register — the sayable / never-say list

### 8.1 Sayable (with the tag's own hedging)

| Claim | Tag | How to say it |
| --- | --- | --- |
| $599 one-time | **[SITE]** | "The advertised price is $599, one time." Confirm checkout/licence separately |
| No TradeFlow monthly subscription | **[SITE]** | "TradeFlow advertises no recurring software fee." |
| Texts from the owner's own SIM/number | **[SITE]** | Explain the intended mechanism |
| Owner approves every booking | **[SITE/APK]** | Core intended control — safe to emphasise |
| Missed-call verification before texting | **[APK]** | "It's built to check the call was actually missed first." |
| Android-only, direct APK install | **[SITE]** | State plainly; explain the Play Store restriction honestly |
| Local-first storage, no internet permission | **[SITE/APK]** | "Designed to keep data on the phone." Never an absolute guarantee |
| Human handoff + STOP opt-out supported | **[SITE/APK]** | State the feature; never claim legal compliance from it |
| Working hours, slot length, day-offs respected | **[SITE]** | Describe as configuration, not as a guarantee |

### 8.2 Hard-blocked **[NEVER]**

| Blocked claim | Why |
| --- | --- |
| **"AI-powered" / "AI receptionist" / "smart AI"** | No released AI feature established. Directly contradicted. |
| **"Encrypted backups"** | Contradicted by APK string describing plain-text export |
| **"Never double-books"** | Intended safeguard, unproven absolute |
| **"Guaranteed revenue" / "guaranteed payback" / "X jobs recovered"** | No evidence whatsoever |
| **"Works on every Android phone"** | No evidence; tested-device list does not exist |
| **Any integration** — Housecall Pro, Jobber, ServiceTitan, Google Calendar, anything | The product is local-first with no internet permission. **There are no integrations.** |
| **"100% secure" / "fully private"** | Un-audited |
| **"Free texting" / unlimited messages** | Carrier SMS charges apply |
| **Any testimonial, case study, or named customer** | None supplied |
| **"Fully automatic" / "hands-off"** | Owner approval is required by design |
| **"HIPAA / TCPA / GDPR compliant"** | No compliance review exists |
| **Competitor prices as industry fact** | Website marks them illustrative |

---

## 9. Assets the founder can actually use on a call

| Asset | Status | How the coach should use it |
| --- | --- | --- |
| **The live product on the founder's own phone** | **[SITE]** exists | **Strongest available proof.** Screen-share or describe the real SMS flow live. SaadKhaja.md §9.5 needs proof after a belief-break; this is the substitute for testimonials. |
| **The missed call itself** | created live on each dial | See `docs/SALES_PLAYBOOK.md` §2 — the ICP's unavailability is the demo. |
| **Website ROI calculator** | **[SITE]** editable: average job value, missed calls per week, estimated conversion rate | **Use it *with* the prospect on the call** so the value delta is computed by them, not asserted at them (SaadKhaja.md §11.1). Must be labelled an estimate. |
| **Cost comparison over 1/3/5-year horizons** | **[SITE]** illustrative | Only as "the website's illustrative comparison". |
| **Six-step "How It Works" section** | **[SITE]** | Structure for the method walkthrough. |
| **Testimonials / case studies / before-after numbers** | **DO NOT EXIST** | Coach must route to live demo or the ROI calculator instead — never fabricate proof. |

### 9.1 ROI calculator — mandatory framing
**[SITE]** It computes: weekly jobs = missed calls × booking conversion rate; monthly recovery
estimate = weekly jobs × job value × 4.33; payback relative to $599.

- **Sayable:** "an estimate of potential gross job revenue."
- **[NEVER]** Present it as guaranteed incremental revenue, profit, or cash collected. It does
  not account for costs, margins, customer quality, cancellations, seasonality, or whether those
  customers would have booked anyway. Label it an estimate **every time it is used**.

---

## 10. Unknowns the coach must ask about rather than guess

These are the reviewed document's own open questions (§11), plus commercial gaps. Until each is
resolved, the coach must route to an honest clarification.

**Product / technical**
1. Exact supported Android version and tested-device list. **[UNKNOWN]**
2. Reliability of missed-call detection across dual-SIM, answered/declined calls, and major manufacturers. **[UNKNOWN]**
3. What exactly is encrypted at rest, and how keys are managed. **[UNKNOWN]**
4. Whether backup export can be password-encrypted; and whether the app warns users it is plain text. **[UNKNOWN]**
5. Precise post-STOP behaviour, including queued messages and restored backups. **[UNKNOWN]**
6. Calendar and slot-clash correctness across time zones and concurrent conversations. **[UNKNOWN]**
7. Supported languages, service categories, message templates in the shipped build. **[UNKNOWN]**

**Commercial / legal**
8. What "lifetime access" covers: purchaser, one device, multiple devices, updates, support, future major versions. **[UNKNOWN — sell this carefully]**
9. **Update path.** Direct APK distribution with no Play Store listing means there is no obvious mechanism to push updates to installed customers. This is an unresolved support and longevity question. **[UNKNOWN]**
10. **Refund policy.** A $599 one-time purchase combined with meaningful installation friction is a chargeback risk. No policy is documented. **[UNKNOWN]**
11. **Carrier filtering risk.** Automated SMS from a personal number at volume can attract A2P filtering or number flagging in the US. Not addressed in the review. **[UNKNOWN — must be investigated before promising anything]**
12. Tested countries/carriers and applicable local messaging, privacy and marketing rules. **[UNKNOWN]**
13. Which payment methods, checkout flow, licence delivery and booking link are actually live. **[UNKNOWN]**

---

## 11. Website issues to fix before scaling sales

Carried forward from the product review, because each one becomes a live objection or a refund:

1. ROI figures update correctly for all slider values and rounding.
2. Calculator edge cases: low conversion rates, fractional weekly bookings.
3. Cost-comparison assumptions visible beside the figures, labelled illustrative.
4. Checkout, payment methods, licence delivery and booking link all work on Android **and** desktop.
5. The APK download is the current signed release, with a clear version number and release date.
6. **Privacy page wording matches actual database encryption and plain-text backup behaviour.**
7. Replace "never double-books" and similar absolutes with qualified wording.
8. Direct-download installation steps match the Android versions and devices actually tested.

---

## 12. Fact-check log

| Date | Item | Result |
| --- | --- | --- |
| 2026-10-11 | File created as template | Attachment did not reach workspace |
| 2026-10-11 | **Populated from founder-supplied `ProductContext.md`** | Source: public website + static APK inspection. Runtime behaviour, security and legal compliance **remain unverified**. |

**Standing limitation:** this file is product context, not proof. It is not a source-code audit,
penetration test, legal review, or runtime QA. Any claim affecting security, reliability,
compatibility or customer revenue must be tested before being advertised as guaranteed.
