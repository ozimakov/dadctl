# Privacy & child‑data legal review — dadctl concept spec

*Reviewer role: privacy counsel with expertise in child-data regulation (COPPA, GDPR/GDPR-K, UK AADC, LGPD, CCPA/CPRA).*
*Reviewing: `docs/superpowers/specs/2026-06-30-dadctl-concept-design.md`*
*Date: 2026‑06‑30.*

## Summary

The spec's technical posture—local-first collection, no vendor telemetry, kid-readable history, deterministic limits—is directionally aligned with privacy-by-design, but it is **not yet defensible as a shippable product for real families** under COPPA, GDPR/GDPR-K (including Article 8), UK AADC, LGPD, or CCPA/CPRA. Those regimes all apply because the system processes **identifiable usage and behavioral data about minors** (often under 13, sometimes under the EU digital-consent age of 13–16), and v0.3 would extend that to **qualitative inference over communications and content**. The spec never identifies who is **data controller** vs. **processor**, what **lawful basis** applies, or how **parental consent** is obtained and evidenced before collection begins. Append-only signed history directly conflicts with **erasure** obligations unless the spec defines a compliant retention and deletion model. "Transparent to the kid" and "privacy first" are stated principles but are undercut by hidden LLM-backend choice, optional window-title inspection, and planned chat-content evaluation without a communications-confidentiality analysis. Before families deploy this, the spec needs: operator legal-role assignment, pre-collection consent and disclosure flows, a data-subject-rights architecture (including erasure), third-party LLM/DPA and transfer warnings, a v0.3 Expectations/chat-surveillance gate, and cross-border transfer guidance for self-hosters.

Where the spec is already in reasonable shape: no project telemetry, local Ollama default, rules-only mode, evidence-linked Issues with parent-in-the-loop enforcement, and honest acknowledgment that Hub compromise is a full breach. Those are necessary but not sufficient.

---

## Findings

### [SEV-BLOCKER] No identified controller, processor, or lawful basis
- **Section**: missing — should add (§2, §7, §12)
- **Regime(s)**: GDPR-K / COPPA / LGPD / UK AADC / CCPA-CPRA / general
- **Issue**: The spec describes data flows but never states who is legally responsible. In a self-hosted deployment, the **parent (or school/NGO operator) is almost certainly the controller** for the child's data; the dadctl project is at most a **provider of means** (and possibly processor only if it ever operated a shared service—which it does not in MVP). Without this mapping, there is no place to hang lawful basis, consent, DPIA obligation, breach notification duty, or DPA requirements.
- **Recommendation**: Add a § "Legal roles & lawful basis" stating: operator = controller; child = data subject; dadctl project = software vendor, not controller in self-hosted mode. Require operators to choose and document lawful basis (typically **parental consent** under GDPR Art. 8 / LGPD Art. 14 / COPPA; **legitimate interests** is risky for systematic child monitoring). Mandate privacy notice and consent **before** the daemon's first collector runs.

### [SEV-BLOCKER] Append-only history conflicts with erasure and rectification rights
- **Section**: §2 (principle 8), §7, §10
- **Regime(s)**: GDPR-K / LGPD / CCPA-CPRA / UK AADC
- **Issue**: "Append-only and signed… never silently rewritten" is incompatible with GDPR Art. 17 (right to erasure) and Art. 16 (rectification) unless the design defines what gets deleted, what gets retained for legal/audit purposes, and how. A child (or parent on their behalf) can request deletion of usage history, chat-derived Issues, and identifiers; regulators will not accept "immutable audit log" as a blanket exemption.
- **Recommendation**: Specify a **retention schedule**, a **crypto-shredding or pseudonymization** path for erasure (delete/re-key PII while optionally retaining anonymized aggregates), and an operator-facing **DSR workflow** (access, rectification, erasure, restriction). Distinguish "tamper-evident enforcement log" from "indefinite personal-data archive."

### [SEV-BLOCKER] Collectors lack consent, disclosure, and special-category analysis
- **Section**: §5.1, §7 (`UsageEvent`), J3
- **Regime(s)**: COPPA / GDPR-K / UK AADC / LGPD / CCPA-CPRA
- **Issue**: Each collector processes **personal data** about a child:

  | Collector | Data categories | Consent/disclosure burden |
  |---|---|---|
  | Foreground app + window title (even hashed/redacted) | Online behavior, possibly **special-category inferences** (health, religion, sexuality, political views from titles) | COPPA: notice + **verifiable parental consent** before collection. GDPR: likely Art. 8 parental consent; **DPIA required** for systematic monitoring. UK AADC Std 2 (detrimental use), Std 8 (high privacy defaults). |
  | Active/idle time | Behavioral/biometric-adjacent usage patterns | Same parental-consent and transparency duties; lower sensitivity but still child data. |
  | DNS category bucket | Web browsing interests; can infer health, sexuality, religion | Same; category bucketing reduces but does not eliminate sensitivity. |
  | Browser extension URL category | More granular browsing behavior; higher re-identification and inference risk | Same, plus higher minimization scrutiny under UK AADC and GDPR data-minimization principle. |

  The escape hatch "title hashed or redacted **unless the contract requires text inspection**" creates a path to **content-level surveillance** without a separate, heightened consent and legal analysis.

- **Recommendation**: Add a **pre-pairing disclosure screen** (parent signs; age-appropriate notice to child) listing each enabled collector, data categories, retention, and who can see it. Default to **most privacy-preserving** collector set. Treat window-title text inspection and URL-level collection as **opt-in with explicit warning** about special-category inference. Document recommended DPIA scope for operators.

### [SEV-BLOCKER] Third-party LLM: controller/processor chain and DPA gap
- **Section**: §5.2, §12.1, §12.7
- **Regime(s)**: GDPR-K / COPPA / LGPD / CCPA-CPRA
- **Issue**: When a parent switches to an OpenAI-compatible endpoint, the Hub sends **contract prose, usage summaries, evidence event metadata, and LLM reasoning context**—all potentially identifying a child—to a third party. Under GDPR, the **parent/operator is controller**; the LLM provider is **processor** (Art. 28 DPA required). Under COPPA, disclosing a child's personal information to a third party requires **verifiable parental consent** and typically a **written agreement** placing COPPA obligations on the vendor. The spec hides backend choice from the kid (§12.1) and does not warn the parent about DPAs, sub-processors, or international transfers.
- **Recommendation**: Surface a **mandatory interstitial** before enabling any non-local LLM: "You are sending your child's usage data to [provider] under your account; you need a DPA / COPPA-compliant vendor terms; transfers outside [jurisdiction] require SCCs or equivalent." Default remains local/rules-only. Log the parent's explicit opt-in with timestamp. Provide operator documentation naming typical processor relationships (OpenAI, Anthropic, AWS Bedrock, etc.).

### [SEV-BLOCKER] v0.3 Expectations rules imply private-communications surveillance
- **Section**: §6, §11 (v0.3), J1 (example: "no DMs with people she hasn't met"), §12.1
- **Regime(s)**: GDPR-K / UK AADC / COPPA / LGPD / general (ePrivacy / communications confidentiality)
- **Issue**: Qualitative rules like "be respectful in chats" and gating on DMs require **inspecting chat content or metadata** that EU law treats as **electronic communications** (ePrivacy confidentiality; GDPR correspondence data). LLM-judged "advisory issues" on chat behavior is still **processing of private communications of a minor**, not merely usage metering. This is materially different from screen-time limits and triggers heightened scrutiny: UK AADC Std 11 (policy standards), Std 2 (detrimental use—chilling effect on expression), and likely a **DPIA**. A parent's house rules do not automatically override a child's privacy rights in communications, especially in the EU.
- **Recommendation**: Keep Expectations **out of MVP and v0.3** until a dedicated legal section exists. If enabled later: require **separate explicit opt-in** per expectation type; ban content inspection by default; limit to **metadata-only** rules where legally viable; never auto-enforce; document that some rules may be **unlawful to implement** in certain jurisdictions; add kid-facing notice when any communication-adjacent collector is active.

### [SEV-BLOCKER] Kid "Accept" is not legal consent—and the spec blurs the line
- **Section**: §4 J2, §7 (`kid_acceptances[]`), §12.6
- **Regime(s)**: COPPA / GDPR Art. 8 / UK AADC / LGPD
- **Issue**: A 9-year-old tapping "Accept" on a daily card has **no legal effect** as consent to data processing. Under COPPA, only the parent can consent. Under GDPR Art. 8, only the holder of parental responsibility can consent for children below the member-state digital-consent age (13–16). The kid's tap is, at best, **acknowledgment of house rules** or procedural fairness—not a waiver of privacy rights. Storing `kid_acceptances[]` as if it were consent-like evidence is misleading and could create false compliance comfort.
- **Recommendation**: Rename and reframe: `kid_acknowledgments[]` or `daily_check_in[]`. Document clearly: **parental authority + parental consent** governs processing; kid acknowledgment is for transparency and dispute resolution only. Provide age-appropriate **privacy information** to the child (UK AADC Std 3), separate from any "accept" ritual.

### [SEV-BLOCKER] Cross-border processing is an undocumented trap
- **Section**: §5 (Hub self-hosted "or small cloud VM"), §12.1, §12.7 — missing transfer analysis
- **Regime(s)**: GDPR-K / LGPD / UK GDPR / COPPA
- **Issue**: A German parent running a Hub on a **US VPS**, with a child's laptop in **Spain** on a school trip, calling an LLM on **AWS us-east-1**, creates multiple jurisdictional hooks: GDPR applies to the child's EU presence; the controller's establishment may be DE; the processor may be US. Without **transfer tools** (SCCs, UK IDTA, LGPD authorization mechanisms) and a **transfer impact assessment**, this configuration is a common compliance failure. Self-hosting does not eliminate transfer law—it often **creates** it.
- **Recommendation**: Add operator guidance: "**Data location checklist**"—where Hub runs, where LLM runs, where backups live, where the child physically is. Warn against default US cloud LLM/VPS for EU families. Document that the operator, not the project, must ensure lawful transfers. Consider geo-aware install warnings based on operator-declared jurisdiction.

### [SEV-IMPORTANT] "Privacy first" and "transparent to the kid" are not fully delivered in regulatory terms
- **Section**: §2 (principles 2 & 4), §12.1
- **Regime(s)**: UK AADC / GDPR-K / COPPA
- **Issue**: Transparency requires telling the child **what is collected, why, who sees it, and where it goes**—including whether a **cloud LLM** processes their data. §12.1 explicitly **withholds backend identity from the kid's History**, which undercuts principle 2 for any install using a remote model. "Privacy first" is weakened by optional URL/title inspection and future content rules without privacy-preserving defaults mandated in the spec.
- **Recommendation**: Kid-facing transparency must include: active collectors, retention period, who receives alerts, and **whether data leaves the home network** (yes/no, not vendor name if necessary, but "processed locally" vs. "sent to external AI service"). Align principle 2 with UK AADC Std 3 and GDPR transparency (Arts. 12–14).

### [SEV-IMPORTANT] No age-based defaults do not satisfy Article 8, COPPA, or UK AADC
- **Section**: §12.6, §3 (personas 8–17)
- **Regime(s)**: GDPR Art. 8 / COPPA / UK AADC
- **Issue**: "The parent encodes age-appropriateness" conflates **parental house rules** with **legal thresholds**. COPPA imposes obligations for under-13 regardless of contract text. GDPR Art. 8 requires parental consent for children below the MS age (13–16) for information-society services. UK AADC applies to **all users under 18** and expects **high privacy defaults** and best-interests assessment—not identical treatment of 8- and 17-year-olds. Treating every kid identically avoids editorial judgment but **does not discharge** regulatory duties tied to age bands.
- **Recommendation**: Distinguish **contract flexibility** from **legal compliance hooks**: prompt operator for child's age/band at setup; apply **jurisdiction-specific compliance checklist** (COPPA under-13 path, GDPR Art. 8 consent path, AADC under-18 defaults); do not claim the parent's prose substitutes for statutory age rules.

### [SEV-IMPORTANT] Hub compromise: breach notification duties fall on the operator, undocumented
- **Section**: §8
- **Regime(s)**: GDPR-K / LGPD / CCPA-CPRA / UK GDPR
- **Issue**: "Hub compromise is a complete breach" is accurate. For self-hosted installs, the **parent/operator is the controller** and bears GDPR Art. 33–34 notification (72 hours to authority if risk; communication to data subjects if high risk), LGPD ANPD notification, and CPRA breach duties. Most parents will not know this. The project's "no telemetry" stance does not remove the operator's obligation—and the project provides no breach playbook.
- **Recommendation**: Add operator documentation: breach definition for dadctl (Hub + backup theft + LLM log exposure), notification timelines, suggested authority contacts by region, and "copy diagnostic bundle" scrubbing rules so parents don't accidentally exfiltrate **more** child data during incident response.

### [SEV-IMPORTANT] Apache-2.0 is legally fine—but "the project" is not the responsible entity
- **Section**: §12.3, §10
- **Regime(s)**: GDPR-K / COPPA / general
- **Issue**: Permissive licensing does not create controller liability for the upstream project when each family self-hosts. **Each operator becomes controller** (parent, school district, NGO). A vendor shipping "dadctl Pro" would be controller or joint controller for their cloud offering. The spec never states this, which risks contributors assuming open-source status confers a compliance shield.
- **Recommendation**: Add a **"Operator responsibilities"** section and README-level disclaimer: software is not legal advice; operator must provide privacy notice, obtain consent, honor DSRs, and execute DPAs. Optionally ship **template** privacy notices and consent records (not legal advice)—marked for operator customization.

### [SEV-IMPORTANT] Data model lacks retention, minimization, and purpose limitation
- **Section**: §7, §5.1 (local buffer 24h offline)
- **Regime(s)**: GDPR-K / UK AADC / LGPD / CCPA-CPRA
- **Issue**: `UsageEvent` includes `app_or_domain` and `metadata` with no spec'd retention cap, aggregation strategy, or deletion trigger. Indefinite behavioral history on minors violates storage limitation (GDPR Art. 5(1)(e)) and UK AADC Std 12. The 24h offline buffer is operational, not a privacy retention policy.
- **Recommendation**: Define default retention (e.g., raw events 30–90 days, aggregated summaries longer with erasure path), automatic purge jobs, and minimization rules for `metadata` fields.

### [SEV-NICE-TO-HAVE] No telemetry is good; still need security advisory distribution
- **Section**: §12.7
- **Regime(s)**: general / GDPR (security of processing, Art. 32)
- **Issue**: Zero telemetry is a compliance positive. It does not eliminate the need to inform operators of **critical vulnerabilities** (e.g., Hub RCE, mTLS bypass). That is not "personal data telemetry" but is a **security notification channel**—distinct from product analytics.
- **Recommendation**: Document a **non-telemetry advisory path**: GitHub Security Advisories, mailing list, in-app "update available" check (optional, no child data). Clarify in spec that "no telemetry" ≠ "no security communications."

### [SEV-NICE-TO-HAVE] Multi-parent, school, and NGO deployments need separate analysis
- **Section**: §3, §11 (v0.4+), missing
- **Regime(s)**: GDPR-K / COPPA / FERPA-adjacent (US schools) / UK AADC
- **Issue**: Roadmap items (multi-parent, federated viewers, school-like deployment) introduce **joint controllership**, **school official vs. commercial provider** distinctions, and potentially **FERPA** if US schools process education records. Not MVP-blocking but the architecture should not paint into a corner.
- **Recommendation**: Footnote in roadmap that school/institutional mode requires a separate compliance profile (likely operator = school as controller; parental role differs).

### [SEV-NOTE] Diagnostic bundle could itself become a data-exposure vector
- **Section**: §10, §12.7
- **Regime(s)**: GDPR-K / general
- **Issue**: "Copy diagnostic bundle" for bug reports is sensible given no telemetry, but if scrubbing is incomplete, parents may paste **child usage logs** into public GitHub issues—secondary disclosure.
- **Recommendation**: Spec should require **aggressive scrubbing defaults**, parent review step, and warning not to post raw child data publicly.

### [SEV-NOTE] Co-parent visibility and LGPD/GDPR access rights
- **Section**: §3, §7
- **Regime(s)**: GDPR-K / LGPD
- **Issue**: Parent has read access to all rows; co-parent sharing is fine if authorized, but any person with Hub access must be accounted for in access-control and privacy notice (processors/sub-processors in family context).
- **Recommendation**: When multi-parent ships, require explicit invitation and document in operator privacy notice.

---

## Top 3 must-fix before implementation plan

1. **Add "Legal roles, lawful basis & consent"** — Identify operator as controller, require verifiable parental consent and pre-collection disclosure for all collectors, rename `kid_acceptances` to non-consent acknowledgment, and block daemon collection until the parent completes setup consent.
2. **Resolve append-only vs. data-subject rights** — Define retention periods, erasure/pseudonymization mechanics, and an operator DSR workflow; do not ship immutable personal-data stores without a documented deletion path.
3. **Third-party LLM and cross-border transfer guardrails** — Mandatory parent-facing warning + opt-in before any non-local LLM; operator DPA/transfer checklist; kid-visible "data leaves home network: yes/no"; keep Expectations/chat inspection out of scope until a separate communications-privacy legal design exists.
