# Family product / UX review — dadctl concept spec

*Reviewer role: senior product designer specializing in family software.*
*Reviewing: `docs/superpowers/specs/2026-06-30-dadctl-concept-design.md`*
*Date: 2026‑06‑30.*

## Summary

The concept is intellectually coherent — shared transparency, contract-as-agreement, and "safety not walls" are differentiated and align with how many thoughtful parents *want* to parent. But the spec describes a **governance product** (negotiate, sign, daily re-affirm, review issues, decide, appeal) while most successful family software wins by being **invisible after day three**. The hardest UX bet is that a stressed parent will reliably show up as the decision engine via notifications, after surviving a multi-step technical onboarding and writing policy prose from scratch. Before pitching investors or running a user test, the founder should shrink first-run to "one kid, one device, one template contract, working limits in 15 minutes," fix the parent notification channel (PWA alone is not enough), and write explicit positioning for families who want collaboration vs. families who want a wall.

## Findings

### [SEV-BLOCKER] Onboarding is longer and harder than any comparable family product
- **Section**: §10 MVP scope, §4 J1/J2, §12.1 (Ollama first boot)
- **Issue**: The minimum path is: self-host Hub (`docker compose`), install daemon on kid device, pair via OTC code, parent drafts contract in prose, LLM compiles, parent signs, kid reviews/comments, kid accepts — plus optional Ollama model download on first Hub boot. That is multiple evenings of work for a technical parent and is unrealistic for the "non-technical" parent in §3. Most families abandon setup before the kid ever sees a "Today" card.
- **Recommendation**: Define an explicit **MVP first-run** (target: ≤15 minutes to first enforced limit): pre-provisioned Hub image with rules-only mode default, skip kid negotiation on day one ("provisional contract, review together this weekend"), defer LLM download. Spec should state what is *required* vs. *recommended* in onboarding order.

### [SEV-BLOCKER] Empty "type your contract" box with no templates at MVP
- **Section**: §4 J1, §12.6, missing — should add
- **Issue**: The spec defers contract templates to v1.0 marketplace while making prose authoring the primary J1 entry point. Real parents at 9pm do not write policy; they say "block bad stuff," "normal screen time," or paste a rule they heard on a podcast. An empty textarea plus YAML feedback is a usability cliff, not a feature.
- **Recommendation**: Ship **3–5 bundled starter contracts** at MVP (e.g., "Elementary shared laptop," "Teen weekday/weekend," "Homework-first household") as prose the parent edits, not blank-slate authoring. Add a short wizard ("How old? Shared or personal device? Homework rules?") that *prefills* prose before the LLM compiles.

### [SEV-BLOCKER] Parent PWA as sole client breaks the decision loop for typical families
- **Section**: §5.3, §4 J4
- **Issue**: The parent client is a Hub-served PWA with WebPush. Non-technical parents do not reliably "install" PWAs, and iOS PWA push remains fragile (Add to Home Screen, OS-level permission quirks, background limitations). The core loop — parent gets Issue, one-taps a decision — assumes a notification channel that works on the phone parents actually carry. If notifications fail, J4 collapses into "kid gets warned/grounded with no timely parent input."
- **Recommendation**: Spec should name a **primary parent notification surface for MVP** (email/SMS fallback, or native mobile companion on roadmap with MVP-critical priority). Treat "open Hub in browser when you remember" as fallback, not the main path. Document iOS limitations explicitly in parent-facing UX requirements.

### [SEV-BLOCKER] "Human in the loop" without a parent-absent fallback
- **Section**: §4 J3–J4, §2 principle 5, §9
- **Issue**: Issues generate parent notifications with three tap options; auto-action is limited to contract-pre-authorized cases. There is no specified behavior when the parent is in a meeting, asleep, or habituated to ignoring alerts after week two. Default appeal grace is 0, so the kid bears enforcement while the parent is unreachable. This inverts the promise of "not playing cop every evening" into "must play cop on demand, via phone."
- **Recommendation**: Define **tiered defaults**: (1) deterministic Limits/Gates enforce silently with kid-visible reason; (2) only ambiguous/escalation Issues ping parent; (3) contract-level "while I'm unavailable, default to warn-only / last decision / defer until morning." Spec should state expected notification frequency bands (e.g., compliant kid vs. boundary-testing teen) so designers can prototype inbox load.

### [SEV-BLOCKER] Product positioning for "wall" parents is underspecified
- **Section**: §8, §1–§2
- **Issue**: "We don't fight a kid with root" is the right engineering stance but the wrong unspoken promise for a large buyer segment. Parents shopping parental controls often want containment when trust is broken. If they discover post-install that tamper is visible but not prevented, they will feel misled — and the kid may learn the lesson "stop the daemon, have a conversation" is the actual bypass path.
- **Recommendation**: Add a **"Who this is for / not for"** section in product-facing docs: for families building shared rules and willing to treat bypass as a relationship signal; not for families needing MDM-grade lockdown or covert monitoring. Onboarding should include a plain-language **honest capability card** the parent reads before the kid sees anything.

### [SEV-IMPORTANT] Personas are too flat to drive age- and household-specific design
- **Section**: §3
- **Issue**: "Parent (non-technical to technical)" and "Kid (8–17)" are demographics, not personas. An 8-year-old on a shared iMac, a 16-year-old with a personal laptop, a neurodivergent kid with time blindness, a single parent, separated parents with alternating custody, and blended families with conflicting house rules have incompatible UX needs. Co-parent is "tertiary" but custody schedules affect when rules apply — a core contract concept.
- **Recommendation**: Split kid personas into at least **three bands** (8–10, 11–13, 14–17) with different UI literacy, appeal behavior, and autonomy expectations. Add household variants: single parent, co-parent (synced rules), co-parent (conflicting houses — even if out of MVP scope, flag as future constraint). Add one **accessibility/neurodivergence** note: predictability, literal rules, and change sensitivity affect daily diff cards and warnings.

### [SEV-IMPORTANT] J1 first-run will fail at the "review compiled policy" step
- **Section**: §4 J1, §6
- **Issue**: At 9pm a stressed parent pastes messy prose, the LLM compiles, and the UI shows "plain English plus underlying YAML." Non-technical parents will not validate YAML; they will skim English, miss edge cases, and sign. Kid comment loop ("30 more minutes on Friday?") turns setup into async negotiation when the parent wanted sleep. Mismatch between parent mental model ("I said no YouTube until homework") and compiled Gates creates day-one "unfair" enforcement.
- **Recommendation**: Parent review UI should show **only kid-readable plain English + concrete examples** ("If homework not marked done → YouTube blocked. Example: 4pm Tuesday."). Hide YAML behind "advanced." Cap first-run negotiation: kid can comment, but **provisional contract** activates with a scheduled family review — don't block all enforcement on kid signature night one.

### [SEV-IMPORTANT] Daily acceptance will become Cookie-Banner Fatigue 2: Kid Edition
- **Section**: §4 J2
- **Issue**: A every-morning "Today" card with diff, yesterday summary, pending decisions, and Accept — when skip is logged but doesn't change enforcement — becomes ritual theater. By day 30 kids tap Accept without reading; by day 365 it's resentment fuel, especially when the contract is stable. Neurodivergent kids may find daily diffs anxiety-inducing even when changes are trivial.
- **Recommendation**: Differentiate **acceptance triggers**: full re-accept only when contract version changes; otherwise a lightweight "Today's budget" glance with no mandatory tap. If daily acceptance stays, spec should state **what parent-facing action** non-acceptance enables (digest? conversation prompt?) so it's not purely surveillance of ritual compliance.

### [SEV-IMPORTANT] J3/J4 notification volume and fatigue risk is unmodeled
- **Section**: §4 J3–J4, §5.2
- **Issue**: Low-volume events + interval/threshold analyzer can still produce a steady Issue stream for a normal teen (gaming, chat, idle/active edge cases, DNS category ambiguity). Three suggested decisions per Issue trains parents to tap "do nothing" to clear inbox — or disable notifications. The spec doesn't batch Issues, dedupe, or quiet hours for *parents*.
- **Recommendation**: Add **parent inbox UX requirements**: daily digest vs. immediate alert, severity thresholds, quiet hours for parent notifications, batching ("3 minor issues tonight"), and snooze. Prototype with realistic event volumes before committing to LLM-in-the-loop for MVP.

### [SEV-IMPORTANT] Appeals are realistic for older teens, not for most 8–12 year-olds
- **Section**: §4 J5, §9
- **Issue**: Formal Appeal objects with kid message and parent response assume literacy, motivation, and trust in the system. An 8-year-old will cry or ask aloud; a 12-year-old might appeal once if grounded from a game; a 16-year-old may appeal performatively or bypass the UI entirely. Default no auto-pause on appeal means the UI adds friction without relief during the wait.
- **Recommendation**: Age-band appeal UX: **"Ask parent"** one-tap with optional voice/text for younger kids; full Appeal history for teens. Consider brief automatic pause for soft enforcements when appeal filed (configurable). Appeals should link from the warning surface, not only from History.

### [SEV-IMPORTANT] Limits / Gates / Expectations framing is engineer-clean, parent-opaque
- **Section**: §6, §10
- **Issue**: Parents think in "screen time," "bedtime," "not until homework is done," and "don't be a jerk online." They will not internalize Gates vs. Limits. Expectations are in the DSL and parent prose but **not enforced in MVP** — parents will write qualitative rules expecting them to work and won't understand why "be respectful in chats" does nothing.
- **Recommendation**: Parent-facing vocabulary: **"Schedules & budgets"** (Limits), **"Before you can…"** (Gates), **"Family values — advisory only until enabled"** (Expectations). Compiler output and kid UI use the same words. If Expectations are design-only in MVP, **disable or clearly watermark** them at authoring time so parents don't think they're live.

### [SEV-IMPORTANT] Kid UI is specified as a shell, not a day-in-the-life experience
- **Section**: §5.1 Kid UI, §4 J2/J5
- **Issue**: "Tray app + Today page + budget remaining + appeal" doesn't answer what the kid does at 4pm when deciding to open YouTube, or when a soft warning fires mid-game. Missing: pre-check ("Can I?"), homework-done marking, countdown to budget reset, what happens at gate denial. Without this, the kid experience is reactive punishment surfaces, not the transparency the principles claim.
- **Recommendation**: Add a **kid day-in-the-life** section: morning glance (optional), pre-activity check for gated apps, inline explanation when blocked, warning dismissal vs. acknowledgment, end-of-day summary. Design goal: kid can answer "why was I blocked?" without opening History.

### [SEV-IMPORTANT] Device-as-identity mismatches shared-device reality
- **Section**: §12.4, §10
- **Issue**: Kid doesn't sign in; device is identity. Two OS accounts on a family Mac → two `device_id`s; parent must know to group under one `kid_label`. Shared iPad with one login used by two siblings → one identity, wrong budgets. Personal device handed to sibling → wrong attribution. These are common patterns, not edge cases.
- **Recommendation**: MVP should document **explicit pairing copy** ("Is this Emma's only account on this computer?"). For shared hardware, spec should prefer **OS-user-level pairing** or session picker ("Who's using this?") even if MVP stays single-kid — design the identity model now to avoid rework at v0.4.

### [SEV-IMPORTANT] No age-based defaults shifts editorial burden to unprepared parents
- **Section**: §12.6, §4 J1
- **Issue**: "Age-appropriateness in prose" avoids vendor paternalism but assumes parents can articulate category-level policy. Most will under-specify ("block bad websites") or over-specify inconsistently. Without defaults, first contracts will be either useless or accidentally harsh.
- **Recommendation**: Even without engine-level age defaults, ship **age-banded template prose** and inline guidance ("Parents of 12-year-olds often include…"). Optional kid age field on `kid_label` only to select template, not to auto-enforce.

### [SEV-IMPORTANT] Missing: how parents find the product and explain it to their kid
- **Section**: missing — should add
- **Issue**: Open-source self-hosted tools don't have app store discovery. Family adoption fails or succeeds at the kitchen table: "We're trying something new." The spec has no **conversation script**, no kid-facing framing ("you can see everything I see"), and no guidance for introducing vs. upgrading from Apple Screen Time / a wall product.
- **Recommendation**: Add **"Family launch kit"**: 2-minute parent script, kid FAQ ("Can you read my DMs?" → only what contract says), printable one-page contract summary for the fridge, and "first week" parent checklist. Treat this as a first-class deliverable alongside the PWA.

### [SEV-NICE-TO-HAVE] Co-parent and custody households need a stated MVP stance
- **Section**: §3, §10
- **Issue**: Tertiary co-parent persona with "no double-enforce" but no UX for split households, conflicting rules, or one parent undermining the system. MVP is single-parent — fine — but real testers will include divorced co-parents; ambiguous ownership causes support nightmares.
- **Recommendation**: MVP docs should say **"single custodial parent / one household"** explicitly and list co-parent sync as v0.4 with a one-paragraph "expected pain if you try earlier."

### [SEV-NICE-TO-HAVE] Kid negotiation in J1 may undermine parental authority in some cultures
- **Section**: §4 J1
- **Issue**: Kid comments on contract before acceptance models collaborative parenting but fails in households where rules are set unilaterally — still a valid user segment. Forcing comment loop may feel like the product " sides with the kid."
- **Recommendation**: Support **parent-authored mode** (kid acknowledges, comments optional) vs. **collaborative mode** (kid comments required before sign). Let parent choose tone at setup.

### [SEV-NOTE] Transparency and append-only history are strong differentiators
- **Section**: §2, §4 J5, §7
- **Issue**: (Positive) Kid-readable reasoning, signed history, and provable grounding end dates address real fairness complaints families have with opaque wall products. These principles can anchor trust if the daily UX doesn't bury them under ritual and notifications.
- **Recommendation**: Keep these central in kid/parent marketing and **surface in warnings** ("Rule 3 from contract signed June 12 — view full text") rather than only in History.

### [SEV-NOTE] "Warn don't block" when Hub unreachable is parent-friendly failure mode
- **Section**: §9, §12.2
- **Issue**: (Positive) Erring toward not over-enforcing when offline matches family trust positioning and avoids "kid locked out because NAS rebooted" support incidents.
- **Recommendation**: Ensure kid UI explains offline mode in one sentence so it doesn't look like bypass or broken enforcement.

## Top 3 must-fix before implementation plan

1. **Define a 15-minute MVP first-run** with bundled contract templates, rules-only default, provisional day-one enforcement, and a parent notification strategy that does not depend solely on PWA/WebPush (especially iOS).
2. **Write explicit product positioning and a family launch kit** — who this is for vs. not for, honest tamper/limitations card, and a parent→kid conversation script — so user tests measure adoption, not just feature comprehension.
3. **Specify kid and parent day-in-the-life UX** — gate checks, warnings, budget glance, appeal entry points, and parent-absent/default decision behavior — so J2–J5 are flows, not object diagrams; validate notification frequency and daily acceptance cadence before building the LLM analyzer loop.
