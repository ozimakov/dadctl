# Open‑source community & sustainability review — dadctl concept spec

*Reviewer role: open-source community lead / project maintainer.*
*Reviewing: `docs/superpowers/specs/2026-06-30-dadctl-concept-design.md`*
*Date: 2026‑06‑30.*

## Summary

The concept has a genuinely differentiated philosophical wedge ("safety, not walls," kid-visible history, self-hosted) that can attract a small, values-aligned cohort of developer-parents and privacy-minded contributors—the same rough audience that grows projects like Home Assistant early on. What the spec does not yet set up is the *infrastructure of trust and continuity* that turns a compelling design doc into a project strangers will install on their kids' machines and still maintain in year two: governance, release signing, sustainability/funding, user-facing documentation, and community guardrails are all absent. The riskiest gap is the combination of **no sustainability story + no governance/release-trust model + documentation treated as an afterthought**—without those, even a well-built v0 risks becoming a impressive demo that fifty families try once and no one reliably maintains.

## Findings

### [SEV-BLOCKER] No governance, release signing, or malicious-contribution response model
- **Section**: missing — should add (§12 or new §14)
- **Issue**: Governance is completely unmentioned. For software that enforces rules on children's devices, "who do I trust?" is a community question, not just a crypto question. There is no stated model (BDFL, core team, foundation), no owner of release-artifact signing keys, and no process for the first malicious PR, supply-chain incident, or compromised maintainer scenario—which *will* happen if the project gets any attention.
- **Recommendation**: Add a short "Governance & trust" section: name the initial maintainer(s)/BDFL, how core commit access is granted/revoked, who holds release signing keys (and rotation plan), and a lightweight security response path (triage owner, embargo contact, patch SLA target). State explicitly that Hub/Docker images will be signed and how users verify them.

### [SEV-BLOCKER] No funding or long-term sustainability model
- **Section**: missing — should add
- **Issue**: Family-focused self-hosted OSS rarely sustains on stars and goodwill alone. The spec describes a multi-component system (Rust daemon, Hub, PWA, LLM integration, future mobile daemons) with no revenue model, sponsorship plan, grant strategy, or "who works on this in year 2?" answer. Historical pattern: parental-control OSS projects stall after the founder's kid ages out or burnout hits.
- **Recommendation**: Add a "Sustainability" section with at least one primary path (e.g., optional managed Hub hosting from the project, GitHub Sponsors/Open Collective, NLnet/SSI-style grants) and an honest statement of maintainer capacity (volunteer nights/weekends vs. funded part-time). Even "undecided, seeking input" is better than silence.

### [SEV-BLOCKER] Documentation strategy treats docs as internal artifacts, not the product
- **Section**: §13; missing — should add
- **Issue**: The spec assumes a README plus design docs under `docs/superpowers/specs/`. For self-hosted OSS, documentation *is* the product for the primary user (parents) and for growth (search, forums, comparisons). There is no split between user docs (install, pairing, "will this work for my family?", troubleshooting) and contributor docs (architecture, dev setup, protocol). A parent landing on the repo today cannot determine hardware requirements, supported OS versions, or what "success" looks like after install.
- **Recommendation**: Add a "Documentation strategy" section: target docs site (MkDocs/Docusaurus/etc.), minimum v0 doc set (quickstart, architecture-for-users-not-devs, FAQ, comparison to Pi-hole/router-DNS/OpenSnitch), and a rule that every MVP feature ships with user-facing docs, not only spec updates.

### [SEV-IMPORTANT] Positioning wedge is sharp for values, blunt for attention and contribution
- **Section**: §1–§2
- **Issue**: "Safety, not walls" and kid transparency are a real differentiator versus adversarial blockers and cloud nannies—but in the crowded self-hosted lane (Pi-hole, router DNS, OpenSnitch, Safe Eyes, timekpr, etc.), the *mechanism* looks heavier and slower to a first-time contributor than "block a domain" or "alert on outbound connection." The wedge recruits philosophically aligned people; it does not yet give a crisp "10-minute win" hook comparable to Pi-hole's instant blocklist gratification.
- **Recommendation**: Sharpen §1–§2 with an explicit comparison table (dadctl vs. DNS blockers vs. network firewalls vs. commercial suites) and one concrete "day-one story" (e.g., "contract + budget + appeal loop on a Linux laptop in one evening"). Name the initial community beachhead: "self-hosted families already running Home Assistant / a NAS."

### [SEV-IMPORTANT] Apache-2.0 signals adoption-friendly, not community-defensive
- **Section**: §12.3
- **Issue**: Apache-2.0 will attract corporate-friendly contributors, embedders, and permissive-license developers; it signals "build on us freely." It also signals to companies that a hosted "dadctl Pro" or white-label family SaaS is legally straightforward without contributing back—exactly the SaaS-fork concern the spec acknowledges but does not mitigate structurally. Copyleft-minded privacy advocates (a natural contributor pool for this product) may contribute elsewhere instead.
- **Recommendation**: Keep Apache-2.0 if adoption is the priority, but document the trade-off explicitly in §12.3: expected contributor profile, trademark strategy (`dadctl` name/mark), and a "official vs. fork" trust story (signed releases, docs site domain, optional CLA or DCO—pick one). If SaaS-fork risk is existential, reconsider AGPL for Hub/server components only, with a stated rationale.

### [SEV-IMPORTANT] No telemetry + diagnostic bundle deferred = maintainer blindness at v0
- **Section**: §12.7; §10 (diagnostic bundle listed but overlaps roadmap tension)
- **Issue**: Zero telemetry is consistent with privacy principles but leaves the maintainer without install counts, version mix, crash rates, or "is v0.2 healthier than v0.1?" signal. Self-hosters under-report bugs; issue filers skew power-user. The compensating "copy diagnostic bundle" is explicitly *not* v0 in §12.7's framing ("roadmap item"), so early releases depend almost entirely on motivated manual reports and forum anecdotes.
- **Recommendation**: Either (a) ship scrubbed diagnostic bundle in v0 MVP scope, or (b) add an opt-in, self-hosted anonymous stats module (even "count installs per version" ping to the family's *own* Hub only). Spec should state what metrics the maintainer will use for prioritization when telemetry is zero.

### [SEV-IMPORTANT] MVP platform scope excludes the majority of real-world kid devices
- **Section**: §10
- **Issue**: Linux + macOS daemons exclude most kid computing: Windows still dominates home desktops/laptops (~70%+ in general populations; lower but non-trivial in tech households), and **all** phone/tablet use (iOS/Android)—where teens spend most screen time. MVP realistically targets developer-parents with Linux/macOS kid machines: a valid beachhead, but perhaps **5–15%** of "parental control" device surface area, not a majority. That caps word-of-mouth among typical parents and limits the *family* community while still allowing a *developer* community.
- **Recommendation**: Reframe §10 MVP as "beachhead for contributor-parents, not general parental-control market," publish explicit "who this is for / not for yet" criteria, and tie Windows daemon to a concrete community milestone (e.g., "v0.2 when N active installs report via issues/discussions") rather than an optimistic version bump alone.

### [SEV-IMPORTANT] Post-MVP roadmap reads as multi-team product plan, not a volunteer trajectory
- **Section**: §11
- **Issue**: Windows daemon → expectations/LLM → multi-parent → Android → plugin SDK → federation → marketplace in sequential dot-releases implies a pace that funded products struggle to hit. A zero-contributor community project starting cold rarely ships Android daemons, plugin SDKs, and a marketplace within 12–18 months without paid labor or an existing platform company behind it.
- **Recommendation**: Replace version-number roadmap with a **capacity-based 12-month trajectory**: e.g., months 0–6 = v0 Linux/macOS + docs + diagnostic bundle; months 6–12 = Windows OR multi-device (pick one); defer federation/marketplace to "post-traction" with explicit gates (contributor count, sustained install interest). Credibility beats ambition for recruitment.

### [SEV-IMPORTANT] Community guardrails unmentioned (CoC, SECURITY.md, CONTRIBUTING)
- **Section**: missing — should add
- **Issue**: No Code of Conduct, security disclosure policy, or contribution guide. For a project touching minors' device usage, SECURITY.md is not polish—it is a trust signal. CoC is needed before Discord/Discussions/PRs from strangers. CONTRIBUTING sets the bar for "design in the open" actually working.
- **Recommendation**: Add to spec: SECURITY.md and CoC required **before first public call for contributors**; CONTRIBUTING.md required **before v0 tag**; link from README. Note that security reports involving child-safety software may need a dedicated contact and clear scope (Hub vs. daemon vs. supply chain).

### [SEV-IMPORTANT] Brand name "dadctl" narrows the family community before code exists
- **Section**: title; §3 personas (guardian/co-parent mentioned, name contradicts)
- **Issue**: "dad" embeds a gendered parent role into the CLI-style name (`dadctl` reads like `sysctl`). Personas include co-parent/guardian and implicitly diverse families; the brand does not. This is low-cost to change now and costly later if docs, domains, and Docker images ship.
- **Recommendation**: Add a "Naming & brand" open decision: retain `dadctl` as developer in-joke with neutral user-facing name ("Pact Hub," etc.), or rename before v0. At minimum, spec should acknowledge the exclusion risk and user-test with non-"dad" parents.

### [SEV-IMPORTANT] README is wrong surface and internally inconsistent for day-1 audiences
- **Section**: §13; current `README.md`
- **Issue**: §13 says README is the marketing surface; the README is a one-page concept pitch with no install path, screenshots, demo, contributor onboarding, or "is this for me?" checklist. It also tells readers to answer "open questions in §12," but §12 marks decisions **resolved**—undermining "design in the open" credibility. Fine for pre-code; inadequate for v0, HN/Reddit traffic, or parent discovery.
- **Recommendation**: Update §13 to define **audience-specific surfaces**: README = contributor + quick orientation; docs site = parent install/trust; SECURITY.md = researchers. Fix README/spec consistency on §12 status. Before v0, README needs "Who this is for (Linux/macOS kid laptop today)" and link to user quickstart.

### [SEV-NICE-TO-HAVE] Slogan "Parenting as a Code" optimizes for developers, not parents
- **Section**: header; README
- **Issue**: The Infrastructure-as-Code parallel recruits GitHub/HN contributors and signals technical seriousness. Primary users (parents drafting contracts) may find it alienating or gimmicky; it reinforces that the project's *public voice* is developer-coded even when the Hub UI should feel family-coded.
- **Recommendation**: Split messaging in spec: **repo/tagline** for contributors vs. **in-product voice** for parents/kids (plain language, no dev metaphors). Document where slogan appears (README, not kid tray UI).

### [SEV-NICE-TO-HAVE] Non-adversarial threat model is a double-edged community magnet
- **Section**: §8
- **Issue**: Rare honesty ("we don't defeat root; bypass is a conversation") will strongly attract r/selfhosted, privacy, and gentle-parenting-adjacent technical crowds—and repel audiences seeking "my kid can't circumvent this." Which subreddit picks it up largely determines early contributor vs. drive-by critic mix.
- **Recommendation**: Add one paragraph to §8 on **community positioning**: lean into honesty as a feature for the target beachhead; prepare FAQ/snark-resistant copy for "just use kernel keylogger" threads so moderators/maintainers aren't improvising under fire.

### [SEV-NOTE] "Designing in the open" without contribution mechanics is half a community strategy
- **Section**: README; §13
- **Issue**: "Read it, push back, open issues" is a start, but there is no issue template, discussion forum choice, or "how decisions get made" loop—so open design may read as broadcast, not collaboration.
- **Recommendation**: Spec mention of GitHub Discussions (or Matrix/Discord) + decision log (`docs/decisions/` or ADRs) so contributors see feedback land somewhere durable.

### [SEV-NOTE] Single maintainer bus factor implicit throughout
- **Section**: missing — should add
- **Issue**: No mention of backup maintainers, core team growth criteria, or what happens if the founder steps away—common failure mode for family-use OSS.
- **Recommendation**: One sentence on bus factor and "core team" invitation criteria (e.g., sustained PRs over N months)—pairs with governance section.

---

## Top 3 must-fix before implementation plan

1. **Governance, release signing, and security response** — Name who runs the project, who signs releases, and how malicious PRs/vulnerabilities are handled; without this, "install on my kid's laptop" is a non-starter for cautious self-hosters.
2. **Sustainability model** — State how the project survives year two (funding path, maintainer capacity, optional hosted offering or grants); without this, the implementation plan has no realistic executor.
3. **Documentation and audience strategy** — Define user-facing docs (not just spec/README), fix README/§12 inconsistency, and explicitly frame MVP as a Linux/macOS beachhead so contributors and parents know who the project serves on day one.
