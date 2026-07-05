# Multi‑role self‑review — synthesis

*Synthesizes seven role‑focused reviews of `docs/superpowers/specs/2026-06-30-dadctl-concept-design.md`, dispatched in parallel on 2026‑06‑30.*

The seven reviews are stored alongside this file:

| # | Role | File |
|---|---|---|
| 1 | Security engineer | [`01-security.md`](01-security.md) |
| 2 | Privacy & child‑data legal | [`02-privacy-legal.md`](02-privacy-legal.md) |
| 3 | Systems & deployability engineer | [`03-systems-deployability.md`](03-systems-deployability.md) |
| 4 | AI/ML engineer | [`04-ai-ml.md`](04-ai-ml.md) |
| 5 | Family product / UX designer | [`05-family-ux.md`](05-family-ux.md) |
| 6 | Open‑source community & sustainability | [`06-oss-community.md`](06-oss-community.md) |
| 7 | Child development & family therapy | [`07-child-development.md`](07-child-development.md) |

## Headline

The concept is intellectually coherent and philosophically differentiated, but the spec is **not yet ready for an implementation plan**. Multiple reviewers independently surfaced the same gaps — high‑confidence signal that real design work remains.

## Convergent findings (flagged by ≥2 reviewers)

| # | Issue | Reviewers |
|---|---|---|
| 1 | **§2 principle 6 ("warn, don't block") contradicts §9 / §12.2 ("enforce last‑known contract")** | Security, Systems |
| 2 | **Cryptographic signing scheme is named but never defined** — `parent_sig`, key custody, canonical encoding, offline verification | Security, Systems |
| 3 | **Contract monotonic versioning / anti‑rollback missing** — without it, "enforce last‑known offline" is exploitable | Security, Systems |
| 4 | **Hub admin auth and pairing‑code security parameters undefined** | Security |
| 5 | **No backup / restore** — Hub disk loss = total loss of audit + signing chain | Systems |
| 6 | **MVP runtime LLM role is muddled** — J3/§5.2 implies LLM on the hot path, contradicting §6/§8 | AI |
| 7 | **No compile verification / no LLM eval harness** | AI |
| 8 | **Onboarding is homelab‑grade, not family‑grade** | UX, Systems, Community |
| 9 | **Parent PWA + WebPush is unreliable for the decision loop (esp. iOS)** | UX, Systems |
| 10 | **Empty "type your contract" box with no templates** | UX |
| 11 | **Append‑only history conflicts with GDPR Art. 17 right to erasure** | Privacy |
| 12 | **No operator‑as‑controller / lawful basis / verifiable parental consent model** | Privacy |
| 13 | **Kid `Accept` is being modeled like consent — it isn't legal consent at any age** | Privacy |
| 14 | **v0.3 Expectations on chat content = surveillance of a minor's private communications** | Privacy, Child Development |
| 15 | **Cross‑border / third‑party LLM trap** — parent in DE on US VPS calling OpenAI = DPA + transfer assessment | Privacy |
| 16 | **Personas too flat; "kid 8–17" is not one user** | UX, Child Development |
| 17 | **No graduation / autonomy off‑ramp** | Child Development |
| 18 | **Appeal grace default 0 + no crisis safe‑harbor** | Child Development |
| 19 | **One‑tap "ground 30 min" trains reactive policing** | Child Development |
| 20 | **No governance / release signing / security disclosure / sustainability story** | Community, Security |
| 21 | **No security‑update discovery path (consequence of zero telemetry)** | Security, Systems, Community |
| 22 | **Cross‑platform daemon cost underestimated (macOS TCC, X11/Wayland, DNS bypass)** | Systems |
| 23 | **Session lock + dismissible warning is thin enforcement for a teen** | Systems |
| 24 | **Brand "dadctl" + slogan "Parenting as a Code" — exclusionary to non‑dads, developer‑coded** | Community, Child Development implied |

## Issues that touch resolved §12 decisions

- **Q2 (Hub‑unreachable = enforce last‑known)** — preserved, but **monotonic contract versioning + anti‑rollback** must be added.
- **Q6 (no age‑based defaults)** — preserved in spirit, but reframed: **age‑banded templates and developmentally informed copy** (the parent still authors the contract; we just don't ship an empty box).
- **Q7 (no telemetry)** — preserved, with explicit carve‑out that **"no telemetry" ≠ "no security advisories"**; a parent‑initiated update check is allowed.

## Cross‑cutting recommendation

**Remove Expectations from the v0.3 roadmap entirely.** Two independent reviewers (Privacy, Child Development) flagged it as BLOCKER for different reasons that nevertheless point the same way: surveillance of a minor's private communications is the wrong default product surface, even framed as advisory. Demote to "design exploration, no committed version" until a dedicated communications‑privacy + developmental design pass exists.

## Revision plan (agreed with user 2026‑06‑30)

### Step A — mechanical spec corrections (this revision)

1. Fix the §2 / §9 internal contradiction (principle 6).
2. Add contract monotonic versioning + anti‑rollback to §7 and §9.
3. Rename `kid_acceptances` → `kid_acknowledgments` and clarify it is **not** legal consent.
4. Clarify §12.7: "no telemetry" ≠ no security advisories; parent‑initiated update check is allowed.
5. Remove Expectations from the v0.3 roadmap; downgrade to "design exploration, no committed version" in §6, §10, §11.
6. Acknowledge MVP positioning explicitly as **developer‑parent beachhead on Linux/macOS**, not "general parental‑control market," in §10.
7. Add §14 *Review history* pointing at this directory.
8. Update §13 *What this document is not* to reflect Step B in progress (no longer "implementation plan is the next artifact").

### Step B — new design sections, brainstormed one at a time

Each round goes through the brainstorming workflow with the user — propose options, get a decision, fold it into the spec.

1. **Cryptography & identity model** — key types, custody, rotation, `parent_sig` envelope, daemon‑side offline verification.
2. **Hub parent authentication & pairing security parameters** — auth model, OTC entropy/TTL/rate‑limit, mTLS PKI lifecycle (CA, rotation, revocation, broker ACLs).
3. **Retention, erasure & operator legal roles** — GDPR/COPPA‑compatible deletion model + operator‑as‑controller framing + pre‑collection consent flow.
4. **Developmental tiers & graduation** — banded UX (8–10 / 11–13 / 14–17), age‑banded templates, autonomy off‑ramp, crisis safe‑harbor categories, appeal grace defaults, daily‑acceptance triggers.
5. **Parent‑absent fallback & inbox UX** — what happens when parent doesn't respond; batching, parent quiet hours, "conversation beat" before punitive one‑taps, day‑in‑the‑life flows for kid and parent.
6. **Install & onboarding redesign** — 15‑minute first‑run, bundled templates, rules‑only default, reachability model (LAN vs. overlay vs. cloud), parent notification surface beyond PWA.
7. **Governance, security disclosure, sustainability** — `SECURITY.md`, `CONTRIBUTING.md`, code of conduct, release signing, who runs the project, year‑two plan.
8. **Brand check** — keep `dadctl` as dev codename / choose a parent‑facing name / rename now.

Step B may surface further open questions; the brainstorming workflow is the right place to decide them.
