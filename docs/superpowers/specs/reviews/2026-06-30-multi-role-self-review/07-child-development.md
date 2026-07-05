# Child development & family therapy review — dadctl concept spec

*Reviewer role: child development specialist and family therapist (20 years clinical experience).*
*Reviewing: `docs/superpowers/specs/2026-06-30-dadctl-concept-design.md`*
*Date: 2026‑06‑30.*

## Summary

Clinically, dadctl is a meaningful step away from stealth surveillance and adversarial lockdowns: transparency, co-drafted rules, human-in-the-loop decisions, and an explicit refusal to treat the child as an attacker are all aligned with healthy parental mediation research (active mediation, autonomy support, and repair over punishment). Those strengths are real, but they do not neutralize the product's core risks. The spec treats an 8-year-old and a 16-year-old as the same kind of user, leans on a quasi-legal "contract" frame that fits older adolescents better than younger children, and builds a parent workflow optimized for near-real-time intervention rather than conversation. The most concerning patterns it could entrench are: surveillance normalized as daily family life; discipline delivered through a low-friction UI that trains parents to react instead of relate; performative "consent" rituals that look collaborative but do not shift power; and (in v0.3) LLM-mediated moral judgment of a child's private peer communication, which directly conflicts with adolescents' need for backstage space. Before installing this, a therapist would want parents to think about developmental fit, what happens in genuine distress, whether the system has an exit ramp as the child matures, co-parenting and custody dynamics, and whether "transparent" monitoring still erodes trust and psychological safety in friendships—even when the kid can see the same dashboard the parent sees.

## Findings

### [SEV-BLOCKER] Qualitative "Expectations" rules threaten backstage privacy and peer psychological safety
- **Section**: §6, §10 (v0.3 roadmap), §4 J3–J4
- **Issue**: Surfacing issues like "kid was disrespectful in chats" to a parent—even as advisory-only—requires inspecting and interpreting private peer communication. For adolescents, friendships and identity work happen in spaces they need to keep from adult audiences (boyd's "social steganography," Goffman's frontstage/backstage). Parent-visible LLM judgments on tone, respect, or content create a chilling effect, increase shame, and can expose LGBTQ+ youth, mental-health disclosures, or conflict with peers to adults who may not be safe recipients.
- **Recommendation**: Keep Expectations permanently out of chat/DM content analysis, or require narrow, explicit, revocable scopes with kid-initiated sharing only. Add explicit "backstage carve-outs" (peer DMs, group chats, journaling apps) and document clinical rationale. If qualitative rules ever ship, default off with strong warnings—not a roadmap checkbox.

### [SEV-BLOCKER] No developmental scaffolding across ages 8–17
- **Section**: §3, §12.6, §4 J1–J2
- **Issue**: The spec acknowledges "increasing autonomy with age" in one persona line but otherwise treats all kids identically: same contract metaphor, daily Accept ritual, appeals workflow, and enforcement grammar. An 8-year-old needs co-regulation, concrete routines, and limited abstract negotiation; a 16-year-old needs negotiated autonomy, privacy boundaries, and identity-safe space. One UX for both is clinically unrealistic and will feel infantilizing to teens or overwhelming/unfair to younger children.
- **Recommendation**: Add a developmental tier model (e.g., co-managed / guided / negotiated autonomy) that changes default workflows, language, monitoring granularity, and kid agency—not just contract prose. Document what changes at each stage and how families transition between tiers.

### [SEV-BLOCKER] Appeal grace default 0 risks locking out kids in genuine distress
- **Section**: §9, §4 J5
- **Issue**: Default "no auto-pause on appeal" means a child can be session-locked or grounded while trying to reach a parent about cyberbullying, safety fears, or emotional crisis. The enforcement chain is transparent, but transparency does not help if the device is the lifeline and the appeal does not pause punishment.
- **Recommendation**: Default a non-zero grace window for hard enforcement (session lock, grounding), plus mandatory emergency bypasses (parent contact, crisis hotlines, school counselor, trusted adult list) that cannot be blocked by contract rules. Severe/crisis-tagged issues should never trigger auto-lock without human confirmation.

### [SEV-BLOCKER] No graduation or autonomy off-ramp
- **Section**: missing — should add
- **Issue**: Healthy tech mediation includes progressive release of control as competence and trust grow. The spec describes ongoing contract, monitoring, and enforcement with no pathway to phase out dadctl, reduce monitoring, or "graduate" to family norms without the daemon. A child who starts at 8 and is 14 four versions later may still be living inside the same structural relationship to the product.
- **Recommendation**: Add an explicit "autonomy ladder" and graduation mode: scheduled reduction in monitoring, sunset dates, kid-initiated renegotiation triggers, and a documented end state where the family runs without enforcement. Make this a first-class journey, not an implicit hope that parents delete the app.

### [SEV-IMPORTANT] Live monitoring + one-tap discipline trains reactive policing over relationship
- **Section**: §4 J3–J4, §3 (parent persona)
- **Issue**: The parent persona wants to avoid "playing cop every evening," but J3–J4 push real-time alerts and pre-computed actions ("ground 30 min") to the parent's phone. Clinically, frictionless punishment channels displace the hard work of curiosity, regulation, and repair. Over time, parents learn to monitor first and talk second; kids learn that the parent's attention is triggered by the system, not by trust.
- **Recommendation**: Require a "conversation beat" before punitive actions (cool-down prompt, suggested openers, optional deferral). Weight suggested decisions toward connection and problem-solving; make punitive one-taps secondary, logged, and contract-gated. Add weekly relationship check-ins, not only breach summaries.

### [SEV-IMPORTANT] Daily "Accept" ritual risks performative consent, not meaningful assent
- **Section**: §4 J2
- **Issue**: Daily acceptance is logged but does not change enforcement if skipped—so the ritual is symbolic. Children often click through to avoid friction (learned compliance), not because they agree. For teens, daily ToS-style assent can breed cynicism; for younger kids, it may be meaningless without comprehension supports. Neither builds genuine buy-in.
- **Recommendation**: Differentiate by developmental tier: younger kids might use a simple daily preview with caregiver co-sign; teens might use weekly renegotiation or change-triggered assent only. If Accept is retained, tie it to something substantive (e.g., triggers review of pending appeals or same-day rule questions), and never treat logged acceptance as proxy for informed consent.

### [SEV-IMPORTANT] Contract metaphor is structurally hierarchical despite collaborative language
- **Section**: §4 J1, §2 (principles 1, 7)
- **Issue**: Joint drafting sounds healthy, but the parent signs, the kid comments, and the parent "accepts, edits, or declines." That is negotiation with veto power, framed as quasi-legal agreement. For older teens, contracts can support autonomy when power is genuinely shared; when power is asymmetric, they teach compliance, external regulation, and "what can I get away with" thinking rather than internalized values.
- **Recommendation**: Rename/reframe for developmentally appropriate tiers (e.g., "family agreements," "our plan," "shared goals"). Require kid co-signature above a certain age, document power imbalance explicitly in the UI, and distinguish values-based agreements from punitive rule sets. Flag when contracts become long, punitive, or frequently revised—a clinical warning sign of escalating control.

### [SEV-IMPORTANT] No safe-harbor for mental health, crisis, or identity-support use
- **Section**: missing — should add (also §6 Expectations, §4 enforcement actions)
- **Issue**: Devices are often where kids access mental-health resources, peer support, or LGBTQ+ community—especially when home is not affirming. Contract gates ("no social until homework done," category blocks, future Expectations flags) can cut off the very uses that support regulation and safety. The spec does not address carve-outs, clinician-approved exceptions, or what happens when parental values conflict with a child's need for support.
- **Recommendation**: Add immutable safe-harbor categories (crisis lines, known mental-health resources, school portals) and a kid-triggered "I need help" flow that loosens enforcement and alerts a designated trusted adult. Document guidance for parents on not using the tool to block identity exploration or help-seeking.

### [SEV-IMPORTANT] Algorithmic punishment has distinct emotional harms transparency alone does not fix
- **Section**: §4 J3–J4, §7 (Issue → Decision → EnforcementRecord), missing — should add
- **Issue**: The chain is auditable, but being grounded because an LLM flagged an Issue and a parent one-tapped a suggestion still feels algorithmic and depersonalizing. Kids often experience shame, reactance, and an external locus of control ("the app got me grounded"). Transparency can even amplify humiliation ("here's the reasoning for why you're locked out").
- **Recommendation**: Add explicit repair workflows after enforcement: kid-facing explanation in relational language, parent prompt for offline conversation, and limits on repeated automated escalations. Research-informed copy on how to talk after a lockout—not only what rule was violated.

### [SEV-IMPORTANT] Offline "contract still active" reinforces inescapable surveillance
- **Section**: §9, §12.2 (design decision 2), §2 principle 6
- **Issue**: Consistency helps younger children with predictability, but "still working even when nobody's watching" also models panopticon permanence—no temporal off, no relief when the Hub is down. Principle 6 says "warn, don't block" when unreachable, but the resolved design enforces last-known contract offline, which clinically reads as surveillance that never sleeps.
- **Recommendation**: Clarify the clinical intent: offline behavior should bias toward least restrictive safe default for older tiers, with optional strict mode for younger tiers chosen explicitly by parents. Give kids visible "offline status" that includes when enforcement will soften and how to reach a parent if locked.

### [SEV-IMPORTANT] No age-based defaults dumps developmental expertise onto anxious parents
- **Section**: §12.6
- **Issue**: Avoiding embedded editorial judgment is ethically understandable, but clinically most parents are not developmental specialists. Without scaffolding, anxious parents over-restrict and permissive parents under-protect; both harm trust. "Templates later" does not help MVP families.
- **Recommendation**: Ship developmentally informed templates and wizard copy (not hidden defaults) with clear rationale: "typical for age X, adjust together." Include therapist-reviewed guidance on monitoring intensity, privacy expectations, and when to loosen rules—not a single policy for all ages.

### [SEV-IMPORTANT] Co-parenting conflict and weaponization are under-specified
- **Section**: §3 (co-parent persona), §10 (multi-parent out of MVP)
- **Issue**: Shared visibility without shared decision governance is dangerous in separated/divorced families. One parent can tighten rules, override appeals, or use alerts as evidence in conflict. The kid becomes the battlefield. Multi-parent quorum is deferred to v0.4 with no clinical guardrails in the interim.
- **Recommendation**: Add conflict-aware design notes now: split-household contracts, appeal routing when parents disagree, anti-weaponization language in onboarding, and warnings about using enforcement logs in custody disputes. Do not ship single-parent MVP without documenting these risks.

### [SEV-NICE-TO-HAVE] Appeals are empowering and infantilizing at once
- **Section**: §4 J5, §9
- **Issue**: Structured appeals beat silent punishment and teach advocacy. But petitioning a parent through the same system that locked you out reinforces hierarchical authority—especially with default grace 0. Kids may stop appealing if outcomes rarely change.
- **Recommendation**: Pair appeals with measurable responsiveness metrics for parents, kid-facing acknowledgment SLAs, and automatic escalation (second adult, co-parent) if unresolved. Celebrate overturned decisions as system health, not kid "winning."

### [SEV-NICE-TO-HAVE] Non-adversarial threat model is healthy for trust but needs developmental nuance
- **Section**: §8
- **Issue**: Refusing to "fight a kid with root" respects adolescent autonomy and avoids destructive arms races—clinically sound for many families. It can backfire when parents believe they have safety they do not, when younger children need firmer scaffolding, or when bypass triggers rupture ("you lied to me / the system failed") instead of the intended conversation.
- **Recommendation**: Onboarding must set honest expectations by age and risk profile: what this does not protect against, how to respond to visible tamper events relationally, and when professional or platform-level help is still needed (grooming, CSAM, self-harm).

### [SEV-NOTE] "Safety, not walls" and "transparent to the kid" are the right orientation—with important caveats
- **Section**: §1–§2
- **Issue**: Active, transparent mediation is better than covert surveillance and pure restriction (supported in parental mediation literature). Nuance: transparency without shared power can still feel controlling; "safety" defined only as rule compliance can crowd out connection, sleep, exercise, and offline relationship—the actual buffers for adolescent wellbeing (Haidt/Turkle concerns are about displacement and constant partial attention, not only content).
- **Recommendation**: Add a principle on relationship-first mediation: the product should measure and prompt for offline conversation, sleep, and family connection—not only breaches. Distinguish "transparent monitoring" from "collaborative governance."

## Top 3 things to address before this product ships to families

1. **Developmental fit and an exit ramp** — Stop treating 8–17 as one persona. Add tiered workflows, age-informed templates (without hidden paternalism), and a explicit graduation/autonomy-reduction path so the product can grow with the child instead of locking them into perpetual contractee status.
2. **Distress-safe enforcement** — Change appeal-default and crisis behavior: non-zero grace on hard lockouts, mandatory emergency bypass contacts, and safe-harbor resources that cannot be gated off—so a kid in cyberbullying or mental-health crisis is never punished into silence.
3. **Protect backstage space; curb reactive discipline** — Do not ship LLM "Expectations" on private communication to parents; redesign the parent loop so real-time alerts and one-tap grounding are not the path of least resistance, and pair every punitive enforcement with guided repair conversation—not only an audit trail.
