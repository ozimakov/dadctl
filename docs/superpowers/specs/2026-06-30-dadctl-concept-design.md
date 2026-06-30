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
   make over‑enforce. When the Hub is unreachable, the daemon errs on the side of
   *warn, don't block*, and logs the gap.
7. **Contract as code, written in human.** Parents describe rules in natural
   language. The system compiles those rules into a structured policy that *both
   sides can read* and discuss.
8. **Append‑only history.** Events, decisions, and overrides are signed and
   never silently rewritten. A grounded kid can prove the grounding ended.

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

* **Language:** Rust (preferred) or Go. Single static binary, runs as a system
  service (systemd / launchd / Windows service).
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
* **Kid UI:** small tray app + a "today" page reachable from the tray. Always shows:
  budget remaining, what's currently active in the contract, last alert and why,
  how to file an appeal.

### 5.2 Hub server (`dadctl-hub`)

* Single binary or Docker image. Brings its own MQTT broker (embedded) for
  zero‑config installs, can also point at an existing broker.
* **Contract store** — versioned, signed, immutable history.
* **Analyzer** — schedules evaluations (cron + threshold), calls the LLM backend,
  produces `Issue` records with structured evidence + reasoning text.
* **Decision engine** — turns `Issue` into one or more candidate `Decision` objects.
  Owns the escalation ladder: first breach → soft warning, repeated → hard warning
  or grounding, severe → grounding + parent notify.
* **API** — REST for CRUD, WebSocket for live events, MQTT bridge for daemons.
* **LLM backend abstraction** — pluggable. MVP supports two backends plus a
  zero‑LLM mode: **Ollama (local, the default)**, an **OpenAI‑compatible
  endpoint** the parent can switch to in settings, and a strict **"rules‑only,
  no LLM"** mode for parents who don't want any model in the loop. Only one
  backend is active at a time. Switching backends is a parent‑only setting and
  is *not* surfaced in the kid's History.

### 5.3 Parent client

* Progressive web app served by the Hub. No app store required for MVP.
* Push notifications via WebPush.
* Critical surfaces: **Inbox** (pending issues + suggested decisions), **Contract**
  editor, **History** timeline, **Devices**.

### 5.4 Messaging

* MQTT 3.1.1 over TLS with per‑device client certificates issued by the Hub on
  pairing. Retained `device/<id>/status` topic for liveness, `device/<id>/events`
  for `UsageEvent`, `device/<id>/decisions` for `Decision` push.
* Why MQTT for MVP: low overhead on flaky home networks, retained messages help
  with offline/online transitions, mature client libraries across languages. NATS
  is a reasonable alternative we can revisit if we outgrow MQTT.

## 6. The contract DSL

The parent never has to *write* the DSL — they write prose. The DSL is what the
LLM compiles to and what both parent and kid review. Three rule kinds:

1. **Limits** — quantitative, deterministic. Time budgets, schedules, category
   caps. Evaluated by a small deterministic engine, no LLM in the hot path.
2. **Gates** — conditional access. "YouTube requires homework done today",
   "social apps allowed only after 16:00". Deterministic.
3. **Expectations** — qualitative. "Be respectful in chats", "no violent video".
   Designed in the DSL from day one so contracts written today won't have to be
   rewritten later, but **not evaluated at runtime in MVP** (see §10). When we
   turn them on (v0.3), they will be evaluated by the LLM analyzer and surfaced
   as advisory issues only — never auto‑enforced.

Each compiled rule carries the natural‑language sentence it came from so the kid
can see *why* a rule exists, not just *what* it does.

## 7. Data model (sketch)

| Object | Key fields |
|---|---|
| `Contract` | `id`, `version`, `prose`, `compiled_policy`, `parent_sig`, `kid_acceptances[]` |
| `UsageEvent` | `device_id`, `ts`, `category`, `app_or_domain`, `seconds`, `metadata` |
| `Issue` | `id`, `contract_version`, `severity`, `evidence_event_ids`, `analyzer_reasoning` |
| `Decision` | `id`, `issue_id`, `actor` (`parent`/`hub`), `action`, `rationale`, `expires_at` |
| `EnforcementRecord` | `decision_id`, `daemon_id`, `applied_at`, `result` |
| `Appeal` | `decision_id`, `kid_message`, `parent_response`, `resolved_at` |

All objects are append‑only and signed. The kid has read access to every row
about their own device. The parent has read access to all rows.

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
| Hub unreachable | Daemon enforces last‑known contract, queues events, no new escalations, banner to kid: "offline, contract still active." |
| Daemon crash | Service auto‑restarts; gap logged as `daemon_gap` event so it can't hide. |
| LLM unavailable | Deterministic Limits/Gates still apply; Expectations are paused with a banner in the parent UI. |
| Clock skew | Hub authoritative, daemon resyncs on each connect; large skew flagged. |
| Disputed enforcement | Kid can one‑click appeal; appeal pauses the enforcement for a parent‑configurable grace window (default 0 — i.e., no auto‑pause). |

## 10. MVP scope (the cut)

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
* Expectations (qualitative rules) — design only in MVP, not enabled.
* App‑level process suspension as an enforcement action.
* Multi‑parent quorum decisions.
* Federated viewers (grandparents, etc.).
* Anomaly detection (unusual locations, new contacts).
* Plugin SDK for third‑party collectors.

## 11. Roadmap after MVP

1. **v0.2 — Windows daemon, app‑level suspend, mobile companion notifications.**
2. **v0.3 — Expectations rules enabled (with strong opt‑in and transparency).**
3. **v0.4 — Multi‑kid, multi‑device, multi‑parent.**
4. **v0.5 — Android daemon.**
5. **v0.6 — Plugin SDK for collectors and enforcement adapters.**
6. **v1.0 — Federated viewers, anomaly detection, contract templates marketplace.**

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
   home network. Compensating UX requirement: the Hub UI must offer a one‑click
   "copy diagnostic bundle" action that produces a scrubbed, parent‑reviewable
   archive the parent can paste into a GitHub issue. This is a roadmap item, not
   a telemetry item.

## 13. What this document is not

* It is not an implementation plan. The implementation plan is the next artifact,
  produced via the writing‑plans skill once §12 is resolved.
* It is not a marketing site. The README is the marketing surface.
* It is not a final architecture. Specifics (broker choice, daemon language,
  exact category taxonomy) are first‑pass and open to revision before code.

---

*Status: design decisions resolved. Next action: invoke the writing‑plans skill
to produce a detailed implementation plan for the MVP scope in §10.*
