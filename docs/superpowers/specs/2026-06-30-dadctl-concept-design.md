# dadctl — Concept & Design

> **Slogan:** *dadctl — Parenting as a Code.*
>
> AI‑augmented, open‑source parental software focused on keeping kids **safe**, not on
> building walls around them.

This is the initial concept document for the project. It captures the product vision,
guiding principles, primary user journeys, a first‑pass system architecture, an MVP
scope, and the open questions that need parent‑in‑the‑loop input before we move to a
detailed implementation plan.

---

## 1. Why this exists

Existing parental control software falls into two unhappy camps:

1. **Walls.** Heavy blockers, opaque rules, adversarial relationship with the kid.
   They treat the kid as an attacker and the parent as a warden. They break the
   moment a kid is motivated enough to climb over them, and they teach kids that
   safety = surveillance.
2. **Cloud nannies.** SaaS suites that hoover up usage data to a vendor and gate
   features behind subscriptions. Privacy posture is take‑it‑or‑leave‑it.

`dadctl` aims at a third option: **a transparent, auditable, AI‑augmented assistant
that helps a family live up to a contract they wrote together.** The kid sees the
same rules, the same data, and the same reasoning the parent sees. The software is
open source so a technically inclined parent (or kid) can read exactly what it does.

## 2. Guiding principles

These are load‑bearing — every design choice should be checkable against them.

1. **Safety, not walls.** The product helps a family honor a contract. Bypass is a
   conversation, not an arms race. We don't ship root‑kits, kernel rootkits, or
   stealth mode.
2. **Transparent to the kid.** Whatever the parent sees about the kid, the kid can
   also see about themselves: the contract, the data collected, the reasoning behind
   every alert, every decision, every enforcement.
3. **Open source, auditable, self‑hostable.** A family can run the entire stack on
   hardware they own. No mandatory cloud component.
4. **Privacy first.** Collect the minimum needed to evaluate the contract. Process
   locally where possible. Never sell, never share with third parties.
5. **Human in the loop.** AI proposes; the parent decides. Fully automatic actions
   are limited to soft warnings and to enforcement steps the contract explicitly
   pre‑authorized.
6. **Reliable, in both directions.** The daemon must be hard to crash and hard to
   make over‑enforce. When the Hub is unreachable, the daemon keeps enforcing the
   contract the family last agreed to (the highest signed `version` it has seen),
   never invents new escalations, and logs the offline gap so it is visible later.
   See §9 and §12.2.
7. **Contract as code, written in human.** Parents describe rules in natural
   language. The system compiles those rules into a structured policy that *both
   sides can read* and discuss.
8. **Append‑only history.** Events, decisions, and overrides are signed and
   never silently rewritten. A grounded kid can prove the grounding ended.
9. **Standard proven pieces over new code.** Where a mature protocol, library,
   or component exists that fits the requirement, we use it — even at the cost
   of a slightly imperfect fit. Every line of code we write is a line we have
   to secure, debug, and maintain; every dependency we pull in has a community
   that shares that cost with us. New code is the exception, not the default.
   The concrete stack that follows from this principle is in §16.

## 3. Personas

* **Parent (primary user).** Range from non‑technical to highly technical. Wants the
  kid safe online and a healthy screen‑time balance, but does not want to play cop
  every evening. Pain today: setting up controls takes hours and breaks weekly.
* **Kid (8–17, secondary user).** Increasing autonomy with age. Wants fairness and
  explanations. Will work with the system if it feels fair; will work *against* it
  if it feels arbitrary.
* **Co‑parent / guardian (tertiary).** Shared visibility, optionally shared decision
  rights. Should not double‑enforce.

## 4. Core user journeys

### J1. Drafting the contract

1. Parent opens the Hub UI and types: *"2 h screen time on weekdays, none after 9 pm,
   no YouTube unless homework's done, social apps are fine but no DMs with people
   she hasn't met, gaming OK on weekends up to 3 h."*
2. The Hub's LLM compiles that into a structured policy and shows it back in plain
   English plus the underlying YAML. Examples of compiled rules:
   * `weekday_budget: 2h, applies_to: [device:*], categories: [interactive]`
   * `quiet_hours: 21:00–07:00`
   * `gating: youtube.com requires task:"homework" marked done that day`
3. Parent edits if needed and **signs** the contract (parent identity, timestamp).
4. The kid sees the proposed contract on their device, can comment ("can I have
   30 more minutes on Friday?"), and the parent accepts, edits, or declines. The
   final agreed version becomes the contract of record for that day forward.

### J2. Daily acceptance

The first time the kid uses the device on a given day, the daemon shows a small
"Today" card:

* Today's contract (diff vs. yesterday, highlighted).
* Yesterday's usage summary (budgets used, breaches if any).
* Any pending decisions (e.g., "still grounded from PUBG until 18:00").
* An **Accept** button.

The kid can skip; skipping is recorded as a non‑acceptance and is visible to the
parent. Non‑acceptance never silently widens enforcement — it just gets logged.

### J3. Live monitoring

* Daemon emits low‑volume `UsageEvent` messages over MQTT (active window category,
  duration window, network category bucket, idle/active).
* Hub aggregates and, at fixed intervals plus on threshold triggers, asks the LLM
  analyzer to evaluate the recent window against the contract.
* Analyzer emits `Issue` records: severity, evidence, plain‑English reasoning.

### J4. Decision loop

* Each `Issue` produces an `IssueNotification` to the parent client, with three
  pre‑computed suggested decisions (e.g., *do nothing*, *warn*, *ground 30 min*).
* The parent taps one, or writes a custom decision. The Hub broadcasts a `Decision`
  to the relevant daemon, which produces an `EnforcementRecord` when applied.
* For decisions the contract pre‑authorized as automatic (e.g., "auto‑warn on first
  quiet‑hours breach"), the Hub can act without waiting for the parent, but the
  parent still receives the notification with an "undo" affordance for a window.

### J5. Review & appeal

* Both parent and kid have a History view: a timeline of usage summaries, issues,
  decisions, enforcements, and appeals.
* The kid can file an `Appeal` on any decision. Appeals are a first‑class object,
  not an email. The parent answers; the answer is part of the history.

## 5. System architecture

```
            ┌────────────────────────────────────┐
            │ Parent client (web / PWA)          │
            │  - notifications, decisions, audit │
            └──────────────┬─────────────────────┘
                           │ HTTPS + WebSocket
                           ▼
   ┌──────────────────────────────────────────────────┐
   │ Hub  (self‑hosted: home server, NAS, parent PC,  │
   │       or small cloud VM)                         │
   │                                                  │
   │  contract store · analyzer (LLM) · decision      │
   │  engine · MQTT broker · REST/WS API · audit log  │
   └──────────────┬───────────────────────────────────┘
                  │ MQTT over TLS (mTLS, per‑device cert)
                  ▼
   ┌──────────────────────────────────────────────────┐
   │ Local daemon (dadctld) on the kid's device       │
   │                                                  │
   │  collectors · enforcer · kid UI · local buffer · │
   │  signed history mirror                           │
   └──────────────────────────────────────────────────┘
```

### 5.1 Local daemon (`dadctld`)

* **Language:** **.NET 10 (C# 14)**, published AOT‑compiled and self‑contained
  per target OS (Linux x64/arm64 + macOS arm64/x64 for MVP; Windows x64
  post‑MVP). Chosen over Rust and Go for two decisive reasons: `.NET for
  macOS` gives production‑quality AppKit / IOKit / Accessibility bindings so
  the client‑side pain surface (foreground‑app collection, idle detection,
  TCC prompts, session lock, launchd integration) does not have to be
  hand‑rolled, and one‑language reuse with the Hub (§5.2) lets the wire
  protocol, canonical serialization, signing, and MQTT topic conventions live
  in a single shared class library (`Dadctl.Protocol`). Full stack in §16.
  Runs as a system service via `Microsoft.Extensions.Hosting` (systemd on
  Linux, launchd on macOS, Windows Service post‑MVP).
* **Collectors** — pluggable, each one a separate process or module:
  * Foreground app + window title category (privacy‑preserving: title hashed or
    redacted unless the contract requires text inspection).
  * Active vs idle time.
  * DNS category bucket (via local resolver hook or a small filtering DNS).
  * Optional browser extension for URL category (more accurate than DNS).
* **Enforcer** — implements the actions the Decision Engine can issue:
  * Soft full‑screen warning (the kid can dismiss with one click; the click is
    logged).
  * Hard warning (requires acknowledgment + short cool‑down).
  * Session lock (lockscreen with explanation card).
  * App‑level suspend (SIGSTOP / platform API), opt‑in per category.
* **Local buffer:** SQLite WAL. The daemon must function offline for at least 24 h
  with no quality loss; events sync when the Hub returns.
* **Kid UI:** small tray app + a "Today" page reachable from the tray,
  implemented with **Avalonia 11+** (native tray on macOS/Windows/Linux,
  native windows, MVVM via CommunityToolkit.Mvvm). Always shows: budget
  remaining, what's currently active in the contract, last alert and why,
  how to file an appeal. Kid‑UX daily flows and copy are a Step B item.

### 5.2 Hub server (`dadctl-hub`)

* **Language:** **.NET 10 (C# 14)** on **ASP.NET Core** (minimal APIs). AOT
  single‑file publish → ~50 MB container image on
  `mcr.microsoft.com/dotnet/runtime-deps:10.0-chiseled`. Shared class library
  (`Dadctl.Protocol`) with the daemon.
* **MQTT broker:** **Mosquitto** as an external service in the docker‑compose
  file — not embedded. Standard, packaged in every distro, minimal ops
  surface, decades of hardening.
* **Contract store** — versioned, signed, immutable history (SQLite via EF Core
  for MVP, migration path to Postgres).
* **Analyzer** — schedules evaluations via `Quartz.NET`, calls the LLM backend
  through **Semantic Kernel** for structured‑output + verifier loops, produces
  `Issue` records with structured evidence + reasoning text.
* **Decision engine** — turns `Issue` into one or more candidate `Decision`
  objects. Owns the escalation ladder: first breach → soft warning, repeated →
  hard warning or grounding, severe → grounding + parent notify.
* **API** — REST + WebSockets (native ASP.NET Core), OpenAPI at `/openapi`, no
  MQTT bridge on the API surface (daemons connect to Mosquitto directly).
* **LLM backend abstraction** — pluggable. MVP supports two backends plus a
  zero‑LLM mode: **Ollama (local, the default)** via `OllamaSharp`, an
  **OpenAI‑compatible endpoint** (via the official `OpenAI` SDK) the parent
  can switch to in settings, and a strict **"rules‑only, no LLM"** mode for
  parents who don't want any model in the loop. Only one backend is active at
  a time. Switching backends is a parent‑only setting and is *not* surfaced
  in the kid's History.

### 5.3 Parent client

* Progressive web app served by the Hub — **Blazor WebAssembly** with the
  `Dadctl.Protocol` types shared with the Hub (no DTO duplication). Bundle
  cached via Service Worker after first load.
* Push notifications via WebPush.
* Critical surfaces: **Inbox** (pending issues + suggested decisions),
  **Contract** editor, **History** timeline, **Devices**.
* No app‑store‑distributed native app in MVP; a native mobile companion is
  post‑MVP (§11).

### 5.4 Messaging

* **MQTT 3.1.1 over TLS** (WebSocket transport on port 443 is the default so
  the protocol traverses school and coffee‑shop firewalls; raw MQTT on 8883
  is available for LAN‑only deployments). Broker is **Mosquitto**. Client is
  **MQTTnet** (both daemon and Hub — the same library).
* **Topics:** retained `device/<id>/status` for liveness (with MQTT Last Will
  and Testament for free daemon‑down detection), `device/<id>/events` for
  `UsageEvent` batches, `device/<id>/decisions` for `Decision` push. Per‑device
  ACLs at the broker so device A cannot subscribe or publish to device B's
  topics.
* **Authenticity is not carried by MQTT or TLS.** Every payload is a signed
  envelope per §15 — the parent key signs Contracts and Decisions, the Hub
  key signs Issues and hub‑actor Decisions, the device key signs
  EnforcementRecords and UsageEvent checkpoints. MQTT provides delivery, TLS
  provides confidentiality on the wire, signatures provide authority. Any of
  those layers can be swapped without touching the others.
* **QoS strategy:** `QoS 1 at‑least‑once` for `events` and `decisions` (dedup
  by sequence number in the signed envelope); `QoS 0` for high‑volume liveness
  pings; `QoS 2` reserved and unused in MVP.
* **Why MQTT** — mature standard with battle‑tested broker (Mosquitto), rich
  client libraries in every language, semantics (retained messages, LWT, QoS
  levels) that map cleanly to our shape without us writing them, existing
  operator tooling (`mqtt-explorer`, `mosquitto_sub` for debug). This is a
  direct application of principle 9.

## 6. The contract DSL

The parent never has to *write* the DSL — they write prose. The DSL is what the
LLM compiles to and what both parent and kid review. Three rule kinds:

1. **Limits** — quantitative, deterministic. Time budgets, schedules, category
   caps. Evaluated by a small deterministic engine, no LLM in the hot path.
2. **Gates** — conditional access. "YouTube requires homework done today",
   "social apps allowed only after 16:00". Deterministic.
3. **Expectations** — qualitative. "Be respectful in chats", "no violent video".
   The DSL reserves a slot for them so contracts written today don't have to be
   rewritten later, but Expectations are **not evaluated at runtime in MVP** and
   are intentionally **not on any committed roadmap version**. Two independent
   reviewer roles flagged that LLM‑judged Expectations on a minor's private
   communications is a fundamentally different product surface from screen‑time
   metering — see [`reviews/2026-06-30-multi-role-self-review/`](reviews/2026-06-30-multi-role-self-review/).
   Expectations are *design exploration only* until a dedicated communications‑
   privacy and developmental design pass exists; if they ever ship, they will be
   advisory‑only, opt‑in per expectation, and never gated on content inspection
   by default.

Each compiled rule carries the natural‑language sentence it came from so the kid
can see *why* a rule exists, not just *what* it does.

## 7. Data model (sketch)

| Object | Key fields |
|---|---|
| `Contract` | `id`, `version` (monotonic per Hub), `effective_at`, `prose`, `compiled_policy`, `parent_sig`, `kid_acknowledgments[]` |
| `UsageEvent` | `device_id`, `ts`, `category`, `app_or_domain`, `seconds`, `metadata` |
| `Issue` | `id`, `contract_version`, `severity`, `evidence_event_ids`, `analyzer_reasoning` |
| `Decision` | `id`, `issue_id`, `actor` (`parent`/`hub`), `action`, `rationale`, `expires_at` |
| `EnforcementRecord` | `decision_id`, `daemon_id`, `applied_at`, `result` |
| `Appeal` | `decision_id`, `kid_message`, `parent_response`, `resolved_at` |

All objects are append‑only and signed. The kid has read access to every row
about their own device. The parent has read access to all rows.

**Contract monotonic versioning.** `Contract.version` is strictly increasing per
Hub and is part of the signed payload. Each daemon stores the highest version it
has successfully applied (`max_applied_contract_version`). The daemon refuses
any contract presented to it whose version is lower than that watermark, even if
otherwise validly signed. This closes a rollback attack where a kid blocks Hub
connectivity, then feeds the daemon an older permissive contract restored from
a backup or another device; a rollback attempt produces a signed
`contract_rollback_attempt` event that surfaces in the parent's audit view on
reconnect.

**Note on `kid_acknowledgments`.** This field records that the kid saw the
contract on a given day. It is **not** legal consent to data processing under
COPPA, GDPR Art. 8, or any analogous regime — only the parent (or guardian
holding parental responsibility) can give that. The field exists for procedural
fairness and dispute resolution, not as a consent receipt. The retention,
erasure, and lawful‑basis model for the data underneath is a Step B design
section (see §14).

## 8. Threat model & trust assumptions

* **Not adversarial.** We assume the kid has a normal user account, not root /
  Administrator. We do not try to defeat a kid with root. We make tamper *visible*
  (the daemon notifies the Hub when it stops, when it's blocked at network, etc.),
  but we do not try to make tamper *impossible*. This is a deliberate choice and
  the principal differentiator from existing tools.
* **Network attacker** between daemon and Hub is mitigated by mTLS.
* **Hub compromise** is treated as a complete breach (it holds the contract and the
  history). Hub is self‑hosted so the attack surface is small.
* **LLM hallucination** is mitigated by: (a) deterministic engine for Limits/Gates,
  (b) requiring evidence event IDs on every Issue, (c) parent in the loop for
  anything beyond a soft warning, (d) full reasoning visible to both sides.

## 9. Reliability & failure modes

| Failure | Behavior |
|---|---|
| Hub unreachable | Daemon enforces the **highest** signed contract version it has applied (`max_applied_contract_version`), queues events, raises no new escalations, banner to kid: "offline, contract still active." |
| Contract rollback attempt | Daemon refuses any contract whose `version` is below `max_applied_contract_version` and emits a signed `contract_rollback_attempt` event surfaced to parent on reconnect. |
| Daemon crash | Service auto‑restarts; gap logged as `daemon_gap` event so it can't hide. |
| LLM unavailable | Deterministic Limits/Gates still apply; LLM‑augmented explanation paused with a banner in the parent UI. (Expectations are not evaluated at runtime in MVP regardless.) |
| Clock skew | Hub authoritative, daemon resyncs on each connect; large skew flagged. |
| Disputed enforcement | Kid can one‑click appeal; the grace window applied while the appeal is pending is a Step B design item (see §14, item 4 — *Developmental tiers & graduation*). Until that design lands, the spec does not commit a default. |

## 10. MVP scope (the cut)

**Positioning.** The MVP is explicitly a **developer‑parent beachhead on
Linux/macOS**, not a general‑market parental‑control product. The target first
user is a parent who already self‑hosts something (Home Assistant, a NAS,
Pi‑hole) and has a kid using a Linux or macOS laptop at home. This excludes the
majority of real‑world kid devices (Windows desktops, iOS/Android phones and
tablets) on purpose: those are post‑MVP, and we'd rather ship a coherent
beachhead than a flaky multi‑platform v0. Several Step B sections (install
redesign, parent‑absent fallback, governance) are precondition work for opening
the audience beyond this beachhead.

In:

* Linux + macOS daemon (Rust).
* Single Hub, single parent, single kid, single device.
* Contract prose → policy compile, with curated category list.
* Limits + Gates rule kinds (deterministic).
* Enforcement actions: soft warning, hard warning, session lock.
* Web parent client (PWA), kid tray app + Today page.
* Self‑hosted Hub via `docker compose up`.
* LLM backend abstraction with two adapters (Ollama default, OpenAI‑compatible
  optional) plus a rules‑only mode.
* Append‑only signed history with kid‑readable view.
* Pairing flow: daemon prints a one‑time code, parent enters it in the Hub UI,
  Hub issues the per‑device mTLS certificate.
* Apache‑2.0 license headers and `LICENSE` / `NOTICE` files in the repo.
* In‑Hub "copy diagnostic bundle" action (scrubbed archive for manual bug
  reports; offsets the lack of telemetry).

Explicitly out:

* Windows daemon (post‑MVP).
* Mobile (Android post‑MVP; iOS is a separate research project — Screen Time API +
  MDM, harder).
* Expectations (qualitative rules) — DSL slot only in MVP, not evaluated at
  runtime, **not on any committed roadmap version** until a dedicated
  communications‑privacy and developmental design pass exists. See §6 and
  [`reviews/2026-06-30-multi-role-self-review/`](reviews/2026-06-30-multi-role-self-review/).
* App‑level process suspension as an enforcement action.
* Multi‑parent quorum decisions.
* Federated viewers (grandparents, etc.).
* Anomaly detection (unusual locations, new contacts).
* Plugin SDK for third‑party collectors.

## 11. Roadmap after MVP

The roadmap is **capacity‑shaped, not calendar‑shaped**: each version ships when
its preconditions are met, not on a fixed schedule.

1. **v0.2 — Windows daemon, app‑level suspend, mobile companion notifications.**
2. **v0.3 — Multi‑kid, multi‑device, multi‑parent (decision quorum optional).**
3. **v0.4 — Android daemon.**
4. **v0.5 — Plugin SDK for collectors and enforcement adapters.**
5. **v1.0 — Federated viewers, anomaly detection, contract templates marketplace.**

**Expectations (qualitative LLM‑judged rules) are deliberately not scheduled.**
The reviewer feedback in [`reviews/2026-06-30-multi-role-self-review/`](reviews/2026-06-30-multi-role-self-review/)
(Privacy and Child Development independently) makes a case that the design
question — "should we ever surface LLM judgments of a minor's private
communications to a parent, and under what gates?" — is large enough to need its
own brainstorm + spec round before it sits on a version line. We will revisit
after the v0 Step B design sections settle.

## 12. Design decisions

The seven questions originally raised in this section have been resolved. They
are recorded here with their final answer and a short rationale, so the
implementation plan and any future contributor can trace why the system behaves
the way it does.

1. **LLM default location — Local (Ollama) by default, one‑click switch to an
   OpenAI‑compatible endpoint.** A brand‑new install never sends data off the
   home network until the parent explicitly opts in. Cost: heavier first‑run
   experience (default small model, e.g. 8B class, downloaded on first boot); a
   "rules‑only, no LLM" escape hatch is available for parents who don't want any
   model at all. Hub never runs both backends silently. **The backend identity
   (which model, local vs. cloud) is parent‑facing only and is not surfaced in
   the kid's History view** — kid transparency is about contracts, decisions,
   and reasoning, not infrastructure.

2. **Hub‑unreachable failure stance — Enforce last‑known contract, no new
   escalations.** The deterministic engine keeps running against the most
   recently synced contract. Existing decisions persist. New Issues are not
   raised while offline. The kid sees an "offline" banner. Events queue locally
   and flush on reconnect. The offline gap is itself a logged event so it can be
   reviewed later.

3. **License — Apache‑2.0 for the entire project** (Hub, daemon, SDKs, protocol
   schemas). Permissive license maximizes adoption and community contribution.
   We accept the trade‑off that a vendor could build a closed‑source "dadctl
   Pro"; the trust signal of "you can audit the upstream we ship" plus the
   self‑hostable design is our defense rather than copyleft. Apache‑2.0 is
   preferred over MIT for its explicit patent grant.

4. **Identity model — Pairing model.** Each device pairs to the Hub at install
   time via a one‑time code displayed by the daemon. The Hub stores a
   `(device_id, kid_label)` mapping. Kids do not sign in — the device *is* the
   identity. Multi‑device per kid is supported by pairing each device and
   grouping `device_id`s under the same kid label. The compiled policy decides
   whether budgets aggregate across grouped devices.

5. **Contract ground truth — Compiled policy is canonical.** The deterministic
   engine evaluates the compiled YAML, which is signed and versioned. The prose
   is stored alongside as the human source. Re‑editing the prose produces a
   *proposal* — a diff against the current compiled policy — which the parent
   must sign before it takes effect. This guarantees determinism in the hot
   path: the kid being warned or grounded can always point at one specific
   signed rule.

6. **Age‑based defaults — None.** The engine treats every kid identically.
   Age‑appropriateness is whatever the parent encodes in the contract prose.
   This avoids embedding our own editorial judgment about what a "12‑year‑old"
   should be allowed. Contract templates may appear in a later release once
   we have signal from real families about what defaults *they* settled on.

7. **Project telemetry — None.** No crash reports, no version pings, no install
   beacons. The only outbound calls a running install makes are (a) the LLM
   endpoint the parent configured, if any, and (b) Hub‑to‑daemon traffic on the
   home network.

   *Carve‑out: "no telemetry" is not "no security communications."* The project
   ships a `SECURITY.md` and uses GitHub Security Advisories for coordinated
   disclosure, and the Hub UI offers a **parent‑initiated** "check for updates"
   action (queries the project's GitHub releases on demand only — no install
   beacon, no automatic phone‑home, no payload identifying the install). The
   bar is: the family decides when to talk to the network; the project never
   decides on their behalf.

   Compensating UX requirement: the Hub UI offers a one‑click "copy diagnostic
   bundle" action that produces a scrubbed, parent‑reviewable archive the
   parent can paste into a GitHub issue. The scrubbing rules (allowlist, parent
   preview before export, automated tests on bundle contents) are a Step B
   design item; the diagnostic bundle itself is in the §10 MVP cut.

## 13. What this document is not

* It is not an implementation plan. The implementation plan is produced **after
  Step B of the revision plan in §14** lands — not directly after this
  document.
* It is not a marketing site for parents. The README is the contributor‑facing
  pitch; a parent‑facing docs site is a Step B deliverable.
* It is not a final architecture. Specifics (broker choice, daemon language,
  exact category taxonomy, signing scheme, auth model, retention rules) are
  first‑pass and explicitly open to revision in Step B.

## 14. Review history & revision plan

This spec was self‑reviewed in parallel by seven role‑focused subagents on
2026‑06‑30. The full reviews and a synthesis live in
[`reviews/2026-06-30-multi-role-self-review/`](reviews/2026-06-30-multi-role-self-review/).
Seven distinct lenses (security, privacy & child‑data legal, systems &
deployability, AI/ML, family product/UX, open‑source community & sustainability,
child development & family therapy) flagged 24 issues independently; roughly a
third were flagged by two or more reviewers, which is the high‑confidence
signal we acted on.

### Step A — applied in this revision

The current document already reflects:

1. Resolved the §2 ⇄ §9 ⇄ §12.2 internal contradiction (principle 6 updated).
2. Added contract monotonic versioning + anti‑rollback to §7 and §9.
3. Renamed `kid_acceptances` → `kid_acknowledgments` and clarified they are
   **not** legal consent (see note under §7).
4. Clarified §12.7: "no telemetry" does not preclude `SECURITY.md`, GitHub
   Security Advisories, or a parent‑initiated update check.
5. Removed Expectations from the v0.3 roadmap; downgraded to *design
   exploration only*, no committed version (§6, §10, §11).
6. Made the MVP positioning explicit: **developer‑parent beachhead on
   Linux/macOS**, not a general parental‑control product (§10).
7. Added this §14 with the review pointer and the Step B queue.
8. Updated §13 *What this document is not* to remove the now‑false claim that
   the next artifact is the implementation plan.

### Step B — open design sections, brainstormed one at a time

Each item below is its own brainstorming round (multiple choice → user decision
→ spec section). The order matters: earlier items are preconditions for later
ones. The implementation plan only starts once these have landed.

1. ~~**Cryptography & identity model**~~ — **landed as §15.**
2. **Hub parent authentication & pairing security parameters** — auth model
   details beyond password (§15.5 already covers password hygiene; this round
   covers session management, CSRF, kid‑vs‑parent endpoint separation),
   one‑time pairing code entropy/TTL/rate‑limit, mTLS PKI lifecycle (CA
   protection, rotation, revocation, broker ACLs). Most of the auth
   plumbing here is delegated to first‑party ASP.NET Core middleware per
   §16, so this round is smaller than originally scoped.
3. **Retention, erasure & operator legal roles** — GDPR/COPPA‑compatible
   deletion model on top of an append‑only event store, operator‑as‑controller
   framing, pre‑collection consent flow, kid‑facing privacy notice, third‑party
   LLM warning + opt‑in.
4. **Developmental tiers & graduation** — banded UX (8–10 / 11–13 / 14–17),
   age‑banded contract templates, autonomy off‑ramp, crisis safe‑harbor
   categories that cannot be gated off, appeal grace‑window defaults,
   daily‑acceptance triggers (change‑gated vs. daily).
5. **Parent‑absent fallback & inbox UX** — what happens when the parent
   doesn't respond; batching, parent quiet hours, "conversation beat" before
   punitive one‑taps, kid and parent day‑in‑the‑life flows.
6. **Install & onboarding redesign** — 15‑minute first‑run, bundled prose
   templates, rules‑only default, reachability tier (LAN‑only vs. overlay /
   tunnel vs. small cloud Hub), parent notification surface beyond PWA/WebPush
   (esp. iOS).
7. **Governance, security disclosure, sustainability** — initial maintainer
   model, release signing key custody, `SECURITY.md`, `CONTRIBUTING.md`, code
   of conduct, year‑two sustainability path (donations / managed hosting /
   grants), bus‑factor stance.
8. **Brand check** — keep `dadctl` as the dev codename and choose a separate
   parent‑facing brand, or rename now while the cost is zero.

Step B may surface its own follow‑on questions; we revisit this list at the end
of each round.

## 15. Cryptography & identity

*Step B item 1. Landed 2026‑07‑03.*

### 15.1 What gets signed and by whom

| Object | Signed by | Notes |
|---|---|---|
| `Contract` | Parent | Covers `id`, `version`, `effective_at`, `prose`, `compiled_policy` |
| `Decision` (`actor: parent`) | Parent | The routine case |
| `Decision` (`actor: hub`) | Hub | Only when the contract pre‑authorized this auto‑action |
| `Issue` | Hub | Record of what the analyzer found; not an authority statement |
| `EnforcementRecord` | Device | The daemon proves it applied the decision |
| `Appeal` | Kid (device‑witnessed) + Parent (response) | Kid signature is a record, not authority |
| `UsageEvent` | Device, in checkpointed batches | Per‑event signing at monitoring cadence would be wasteful |
| `KeyRotation` envelope | Old key + new key (co‑signed) | The audit‑visible way authority moves |

### 15.2 Three key identities

- **Parent key** — Ed25519 keypair. Pubkey pinned at pairing on every daemon.
  Private key held by whichever authenticator is currently bound to the parent
  identity (see §15.4).
- **Hub key** — Ed25519 keypair generated at Hub install. Pubkey distributed to
  daemons at pairing. Private key held by the Hub process.
- **Device key** — Ed25519 keypair generated on each daemon at pairing. Pubkey
  registered with Hub. Private key never leaves the device.

Ed25519 is chosen for all three: small, fast, deterministic, universally
supported. No hardware security module is required.

### 15.3 Signature envelope

Canonical serialization is **JSON Canonical Form (RFC 8785)** — human‑readable
so the audit view can literally show the bytes that were signed. Deterministic
(sorted keys, canonical numbers) so two daemon implementations cannot disagree
about what a `Contract` hashed to.

```json
{
  "payload": { ... canonical object fields ... },
  "signatures": [
    {
      "signer": "parent",
      "key_id": "parent-2026-07",
      "alg": "Ed25519",
      "sig": "base64url(...)"
    }
  ]
}
```

Multiple signatures are allowed on a single envelope — this is the mechanism
for Hub co‑signing an auto‑Decision, and later for a co‑parent's second
signature.

### 15.4 The authenticator abstraction — "leave room for later"

The Hub records the parent identity like this:

```
parent_identity {
  id: "parent-1",
  authenticators: [
    { kind: "password", enrolled_at: ..., active: true }
  ],
  active_signing_key: "parent-2026-07"  // pubkey ref
}
```

`parent_sig` is *not* "a password‑signed thing"; it is "a signature by whichever
authenticator is currently bound to the parent identity." Adding a WebAuthn
passkey later — or a hardware key, or a phone‑push authenticator — is a **new
enrollment**, not a schema change:

1. Parent authenticates with the currently active authenticator (password).
2. Enrolls a new authenticator (e.g., WebAuthn passkey), generating a new
   Ed25519 signing key wrapped by that authenticator.
3. Hub writes a `KeyRotation` event to the signed history:
   `{ from: parent-2026-07, to: parent-2026-11, authorized_by: password,
   authorized_at: ... }`. The event is co‑signed by the old key.
4. Daemons accept both keys during a 30‑day overlap window, then only the new
   one.
5. The kid's audit view shows the transition as a first‑class event ("parent
   authority moved from password to passkey on 2026‑07‑15").

Same abstraction later covers hardware key, phone‑push, and a co‑parent second
signature. The MVP ships **only the password authenticator**, but the schema,
the `KeyRotation` primitive, and the daemon's multi‑key acceptance window are
present from day one so no future migration is a breaking change.

### 15.5 Password hygiene (the only thing between an attacker and forgery in MVP)

- Minimum 12 characters, zxcvbn strength estimator shown at set time (warns
  but does not block).
- **Argon2id** at rest, parameters tuned to ≥250 ms on target Hub hardware.
- Login rate‑limit: 5 attempts per 15 minutes per source IP; exponential
  lockout on the parent account.
- The Hub is bound to `localhost` by default. Any remote access requires the
  parent to explicitly stand up a reverse proxy or an overlay network — that
  conversation is Step B.6 (install & onboarding).
- **No password recovery via email or security questions.** The parent writes
  down a one‑shot recovery code at first‑run (16 characters, base32, decrypts
  the Hub keystore exactly once). Losing both password and recovery code
  means starting fresh as a new parent identity — past records still verify
  against the old pubkey, new records sign under the new key, and the kid
  sees the transition in the audit log.

Email recovery is deliberately absent: it would create a silent alternate
authority (the parent's email provider) with the power to reset the family's
Hub. That is worse than "you lose two things, you start fresh."

### 15.6 Rotation and revocation

- **Planned rotation** — parent authenticates, Hub generates a new keypair,
  writes a `KeyRotation` event co‑signed old+new. Daemons accept both keys for
  a 30‑day overlap window.
- **Suspected compromise** — parent triggers `KeyRevoke`. Old key is
  immediately invalid for *new* signatures; existing records still verify.
  Daemons refuse any newly‑arrived record signed by the revoked key.
- **Hub key rotation** — same pattern; daemons discover the new Hub pubkey
  over the mutually‑authenticated Hub connection (mTLS details are Step B.2).

### 15.7 Trust chain

```
Parent (pubkey pinned at pairing)
    ├── signs → Contract, Decision(parent-actor)
Hub (pubkey pinned at pairing)
    ├── signs → Issue, Decision(hub-actor), KeyRotation envelopes
Device (pubkey registered at pairing)
    └── signs → EnforcementRecord, UsageEvent checkpoints
```

Every object in §7 traces back to exactly one of these three authority roots.

### 15.8 What §15 deliberately does not do (yet)

- No hardware‑security‑module requirement.
- No Merkle tree over the audit log at the object level (per‑object signing is
  enough for family scale; a small Merkle root per `UsageEvent` batch is the
  one exception).
- No transparency log / external witnesses (a v1.0+ topic if we ever want the
  audit log to be verifiable outside the family).
- No cross‑signing between families or federated hubs (v1.0 item).
- No PKI / mTLS specifics for transport — that is Step B.2.

## 16. Stack & dependencies

*Landed 2026‑07‑04 during Step B.2 discussion, following the addition of
principle 9 (§2). This section is the concrete instantiation of that principle.*

### 16.1 Language and runtime

**.NET 10 (C# 14)**, one language across the whole project. Chosen over
alternatives (Python, Go, Rust) after explicit trade‑off analysis captured in
this branch's PR history. The decisive reasons:

- `.NET for macOS` (the mature Xamarin.Mac lineage folded into .NET)
  eliminates the client‑side pain surface on macOS that any of the
  alternatives would have made us hand‑roll: foreground‑app collection, idle
  detection, TCC permission prompts, session lock, launchd integration.
- **Semantic Kernel** covers the LLM compile‑verify‑simulate loop the AI
  review made mandatory, closing the ecosystem gap that would have made Go
  or Rust more work at the analyzer layer.
- One shared class library (`Dadctl.Protocol`) between the Hub and the
  daemon absorbs all the wire‑type, canonical‑serialization, signing, and
  MQTT‑topic boilerplate.
- Windows daemon at v0.2 (§11) becomes essentially the same code path.

Deployment cost is a somewhat larger container image than a Go static binary
(~50 MB vs ~20 MB via AOT single‑file publish onto a chiseled runtime image)
and a slightly slower cold start. Both are acceptable for a self‑hosted Hub
that runs continuously.

### 16.2 Solution layout

One `.sln`:

```
src/
  Dadctl.Protocol/            # shared: types, canonical JSON, signature envelopes, MQTT topics
  Dadctl.Crypto/              # shared: Ed25519, Argon2id, key rotation primitives
  Dadctl.Hub/                 # ASP.NET Core minimal APIs, Semantic Kernel analyzer
  Dadctl.Daemon/              # cross-platform daemon core, MQTT client, enforcer
  Dadctl.Client.Mac/          # AppKit / IOKit / Accessibility bindings
  Dadctl.Client.Linux/        # DBus + X11 P/Invoke bindings
  Dadctl.Client.Windows/      # (post-MVP)
  Dadctl.Ui.Avalonia/         # tray + Today page views
  Dadctl.Pwa/                 # Blazor WebAssembly parent client
tests/
  Dadctl.Tests.Protocol/
  Dadctl.Tests.Hub/
  Dadctl.Tests.Daemon/
```

### 16.3 Component picks

**Shared (both binaries and the PWA where relevant):**

| Concern | Pick | Notes |
|---|---|---|
| Ed25519 signing | **`NSec.Cryptography`** | libsodium binding, actively maintained |
| Argon2id | **`Konscious.Security.Cryptography.Argon2`** | de‑facto .NET Argon2 |
| Canonical JSON (RFC 8785) | Small internal impl in `Dadctl.Protocol` (~150 lines over `System.Text.Json`) | No first‑class .NET library exists; JCS is small enough to own |
| MQTT client | **`MQTTnet`** | Supports 3.1.1 + 5, WebSocket transport, TLS, all QoS levels |
| SQLite | **`Microsoft.Data.Sqlite`** | First‑party ADO.NET provider |
| ORM + migrations | **Entity Framework Core** | First‑party |
| Logging | `Microsoft.Extensions.Logging` + **Serilog** sink | Structured JSON to stdout |
| Config, DI, hosting | `Microsoft.Extensions.{Configuration,DependencyInjection,Hosting}` | First‑party |
| Tests | **xUnit** + **FluentAssertions** + **NSubstitute** | Standard |

**Hub:**

| Concern | Pick |
|---|---|
| Web framework | **ASP.NET Core** minimal APIs |
| WebSocket | Built into ASP.NET Core; **SignalR** as optional convenience layer |
| MQTT broker | **Mosquitto** as external service in docker‑compose |
| Cookie auth + CSRF | `Microsoft.AspNetCore.Authentication.Cookies` + `Microsoft.AspNetCore.Antiforgery` |
| Rate limiting | `Microsoft.AspNetCore.RateLimiting` |
| WebAuthn (for §15.4 later authenticators) | **`Fido2NetLib`** |
| LLM tooling | **Semantic Kernel** + official **`OpenAI`** SDK + **`OllamaSharp`** |
| Scheduler | **`Quartz.NET`** |
| HTTP resilience | `Microsoft.Extensions.Http.Resilience` (Polly) |
| Metrics | **OpenTelemetry** for .NET |
| API docs | `Microsoft.AspNetCore.OpenApi` |

**Daemon:**

| Concern | Pick |
|---|---|
| UI (tray + Today page) | **Avalonia 11+** with **CommunityToolkit.Mvvm** |
| Service lifecycle | `Microsoft.Extensions.Hosting.Systemd` (Linux), custom launchd wrapper for macOS (~50 lines), `Microsoft.Extensions.Hosting.WindowsServices` (post‑MVP) |
| macOS native APIs | **.NET for macOS** bindings (AppKit / IOKit / Accessibility / Foundation) |
| Linux native APIs | **`Tmds.DBus`** + P/Invoke for X11 primitives |

**Parent PWA:**

| Concern | Pick |
|---|---|
| Framework | **Blazor WebAssembly** |
| UI kit | (Deferred to Step B item on parent UX — likely **MudBlazor** or similar) |
| WebPush | Standard browser `PushManager` API + a Hub‑side WebPush library (TBD in Step B.5) |
| State | Blazor's built‑in DI + `Dadctl.Protocol` DTOs (no duplicate models) |

**Cross‑cutting:**

| Concern | Pick |
|---|---|
| Container image (Hub) | AOT single‑file on `mcr.microsoft.com/dotnet/runtime-deps:10.0-chiseled` — ~50 MB |
| Native packaging (daemon) | `dotnet publish --self-contained -r <rid>`, wrapped in `.deb`/`.rpm`/signed macOS `.pkg` |
| Release signing | **`cosign`** (Sigstore) |
| Local‑dev orchestration | **.NET Aspire** — first‑party, spins up Hub + Mosquitto + Ollama with one command |
| CI | GitHub Actions |
| Docs site | MkDocs Material |

### 16.4 Honest gaps in the reuse story

- **JCS (RFC 8785) has no first‑class .NET library.** We own ~150 lines in
  `Dadctl.Protocol`. Cheaper than depending on an unmaintained community
  port; the spec is small and stable.
- **macOS launchd hosting** has no first‑party equivalent to the systemd /
  Windows Service integration packages. We ship a `.plist` template plus a
  small `LaunchdLifetime` service class wiring signals. ~50 lines.
- **Contract compile‑verify loop** is ~200 lines of application code on top
  of Semantic Kernel. Semantic Kernel does most of the work (structured
  output, retries, validation); the domain‑specific verifier (schema check +
  ontology check + scenario simulator) is ours to write once.

Everything else is a first‑party Microsoft package or the de‑facto community
standard, chosen deliberately to keep the amount of code we own small.

### 16.5 What §16 deliberately does not do (yet)

- Does not fix the parent UI kit (Blazor supports many — MudBlazor, Radzen,
  Fluent UI Blazor — that's a Step B.5 decision, part of the parent inbox &
  day‑in‑the‑life design).
- Does not name a specific WebPush library on the Hub side (Step B.5).
- Does not commit to a specific Ollama model or model size (that's part of
  the analyzer / compile‑pipeline design, still open).
- Does not commit to a specific structured‑output pattern (JSON‑schema mode
  vs grammar‑constrained) — Semantic Kernel abstracts both; we pick per
  backend at runtime.

---

*Status: Step B items 1 and stack from item 2 landed as §15 and §16. Next
action: complete Step B item 2 — the remaining auth details (session TTL,
kid‑vs‑parent endpoint separation, pairing OTC parameters, mTLS PKI
lifecycle), most of which are now smaller conversations because §16 delegates
the plumbing.*
