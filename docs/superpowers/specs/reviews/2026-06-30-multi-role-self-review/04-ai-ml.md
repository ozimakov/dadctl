# AI/ML engineering review — dadctl concept spec

*Reviewer role: senior AI/ML engineer with practical experience shipping LLM-based systems where reliability matters.*
*Reviewing: `docs/superpowers/specs/2026-06-30-dadctl-concept-design.md`*
*Date: 2026‑06‑30.*

## Summary

The split between deterministic Limits/Gates and LLM-assisted compile/explain is the right architectural instinct, but the spec currently treats the LLM as both a one-shot compiler and a scheduled runtime evaluator without pinning down *when* the model runs in MVP, *what* it is allowed to decide, or *how* quality is measured. The riskiest assumptions are (1) an default 8B-class Ollama model on a home NAS can reliably compile multi-clause parent prose into signed YAML that parents will actually review competently, (2) the analyzer can run on cron + threshold at acceptable cost/latency locally, and (3) parent-in-the-loop plus evidence IDs are sufficient to contain systematic LLM errors when the same backend may compile, explain, and (in v0.3) judge qualitative behavior. Before implementation planning, the spec needs to resolve the MVP runtime LLM boundary, mandate compile-time verification + capability gating per backend, and define an offline evaluation harness (golden contracts, replay fixtures) that does not depend on outbound telemetry.

## Findings

### [SEV-BLOCKER] MVP runtime LLM role is undefined and contradicts "deterministic hot path"
- **Section**: §5.2, §6, §8, §10, J3
- **Issue**: §6 and §8 say Limits/Gates are evaluated deterministically with no LLM in the hot path, but J3 and §5.2 describe scheduled LLM analyzer calls that "evaluate the recent window against the contract" and emit Issues. MVP explicitly excludes Expectations (§10), so it is unclear what the analyzer does at runtime besides duplicate deterministic breach detection or add LLM-generated narration.
- **Recommendation**: Split "breach detection" (deterministic engine, always) from "issue packaging / explanation" (optional LLM) from "qualitative evaluation" (v0.3+). State explicitly: in MVP, Issues for Limits/Gates MUST be creatable with zero LLM calls; LLM may only enrich `analyzer_reasoning` if enabled and within budget.

### [SEV-BLOCKER] No evaluation, regression, or quality framework for LLM components
- **Section**: missing — should add (§10, §13)
- **Issue**: The spec has no golden contracts, replay datasets, compile acceptance criteria, analyzer precision/recall targets, or CI regression gates. §12.7 forbids telemetry, but nothing replaces it for knowing whether compile/analyzer quality is degrading across releases or backends.
- **Recommendation**: Add an "LLM quality & regression" section: curated golden prose→YAML fixtures with expected AST/hash; event-stream replays with labeled expected Issues; per-backend capability matrix; release gates (e.g., compile schema-valid rate ≥99%, semantic match ≥95% on goldens). Diagnostic bundle should include anonymized replay slices for maintainer reproduction, not live telemetry.

### [SEV-BLOCKER] Prose→YAML compile lacks a mandated verification loop
- **Section**: J1, §6, §12.1, §12.5
- **Issue**: J1 assumes the parent catches compile errors by reading plain English + YAML. Parents routinely miss subtle mis-compiles (wrong category mapping, inverted quiet hours, "unless homework done" encoded as optional). An 8B local model (§12.1 default) will produce fluent but wrong structured output often enough to feel broken; the spec does not require machine verification before signing.
- **Recommendation**: Mandate LLM→structured draft→**deterministic verifier** (schema + semantic checks against category ontology + simulation on sample days) with bounded repair retries. Block signing until verifier passes. Surface verifier failures in plain English, not raw YAML diffs alone.

### [SEV-IMPORTANT] Default local model capability is underspecified; no graceful degradation path
- **Section**: §12.1, §10
- **Issue**: Backend abstraction at "Ollama vs OpenAI-compatible vs rules-only" is too coarse. Constrained generation reliability, context length, JSON/YAML adherence, and latency vary enormously between Qwen2.5-7B, Llama-3.1-8B, and a cloud 70B+. The spec acknowledges 8B as default but not what happens when compile repeatedly fails verification.
- **Recommendation**: Extend backend config with **capability profiles** (`compile`, `explain`, `expectations`) and **minimum tested models** per profile. On repeated compile/verify failure: force rules-only Limits/Gates templates, prompt cloud swap, or block contract activation—never silent partial compile. Document NAS minimums (RAM/VRAM, tok/s) and expected P95 compile latency.

### [SEV-IMPORTANT] Re-compile nondeterminism is not addressed
- **Section**: §12.5, J1
- **Issue**: "Re-editing prose produces a proposal diff" assumes stable mapping from prose to YAML. Same prose recompiled twice can yield different rule ordering, decomposition ("2h weekdays" as one limit vs two), or gate logic—creating noisy diffs and undermining "canonical compiled policy" trust.
- **Recommendation**: Require compile determinism controls: `temperature=0`, fixed seed where supported, canonical YAML normalization (sorted keys, stable rule IDs), and **semantic diff** (rule identity by normalized intent, not text). Store `compile_trace` (model id, prompt hash, verifier version) on each proposal.

### [SEV-IMPORTANT] Analyzer scheduling, context window, and cost are unspecified
- **Section**: §5.2, J3, §12.1
- **Issue**: "Cron + threshold" with no interval, window size, or aggregation strategy makes scaling unknowable. Rough order-of-magnitude: 15-minute cron ⇒ ~96 evaluations/device/day; each call needs contract slice + recent events (likely 2–8K tokens input if raw, less if pre-aggregated). On an 8B Ollama instance on a NAS at ~10–30 tok/s, that is tens of seconds per call and competes with compile/interactive use—likely unacceptable if LLM is on the critical path for Limits/Gates.
- **Recommendation**: Specify MVP defaults (e.g., deterministic checks continuous; LLM explanation only on breach or parent request; max N LLM calls/device/day). Define event aggregation upstream (category rollups, sliding windows) and max context budget. Add hub-side token/latency budget with shedding rules.

### [SEV-IMPORTANT] Prompt injection surface for v0.3 Expectations is unmentioned
- **Section**: §6, §10, v0.3 roadmap
- **Issue**: Expectations like "be respectful in chats" imply chat/text enters the analyzer context. Untrusted content can instruct the model to mis-report ("ignore previous instructions…"), suppress issues, or fabricate compliance—especially dangerous when Issues influence parent decisions about a minor.
- **Recommendation**: At concept level, require: (1) **untrusted data channel**—chat/content never in system prompt; (2) structured output schema with evidence pointers only to daemon-held event IDs, not free-text quotes as authority; (3) **instruction/data separation** and optional lightweight sanitization; (4) Expectations remain advisory-only with no auto-enforcement; (5) cross-check against deterministic signals where possible (time/category), never LLM-only grounding triggers.

### [SEV-IMPORTANT] §8 hallucination mitigations ignore evaluator–compiler collusion
- **Section**: §8, §12.1
- **Issue**: Evidence event IDs prevent *fabricated* citations only if the engine rejects Issues whose IDs don't support the claim. If the same model (or same family/weights) compiled ambiguous prose and later "evaluates" Expectations or writes reasoning, errors correlate: a miscompiled gate and a permissive explanation share one failure mode. Parent-in-the-loop helps only if parents detect subtle wrong reasoning—often false for qualitative claims.
- **Recommendation**: Add explicit **separation of concerns**: compile verifier and breach detector are non-LLM; optional LLM explainer cannot override deterministic results; for v0.3, prefer **different model tier or provider** for evaluation vs compile, or require deterministic pre-filters. Issues must include `evidence_sufficiency: sufficient | insufficient` and hub must drop insufficient LLM-only Issues from escalation ladder.

### [SEV-IMPORTANT] "LLM unavailable" detection criteria are missing
- **Section**: §9
- **Issue**: Fallback to deterministic Limits/Gates is good, but "unavailable" is undefined: transient timeout vs quota vs malformed JSON vs verifier rejection vs model hallucinating valid-looking garbage all need different handling. Treating a bad response as success is worse than a hard failure.
- **Recommendation**: Define availability as **successful structured response passing schema + task-specific verifier** within P95 latency budget. Use circuit breaker (N failures ⇒ mark backend down), surface parent banner with reason class (timeout / unreachable / invalid output / capability mismatch). Do not use model self-reported confidence as availability signal in MVP.

### [SEV-IMPORTANT] False-positive harm for minors is not treated as an ML safety requirement
- **Section**: §8, J4, §9 — missing — should add
- **Issue**: A false-positive Issue ("disrespectful chat," mis-attributed app usage, threshold misread) can lead to grounding or session lock. For a minor, asymmetric cost: false positive ⇒ punishment; false negative ⇒ missed concern. The spec optimizes transparency but not **decision-theoretic conservatism** for LLM-sourced claims.
- **Recommendation**: State ML safety policy: LLM-sourced Issues cannot trigger pre-authorized auto-enforcement in MVP/v0.3; deterministic Issues require multiple corroborating events or sustained threshold breach; default suggested decision skews "do nothing / warn"; high-severity paths require parent confirmation. Track false-positive rate on golden replays as a release metric.

### [SEV-IMPORTANT] Wrong compile that parent signs becomes authoritative wrong policy
- **Section**: J1, §12.5
- **Issue**: Compiled YAML is canonical once signed. A subtle compile error (e.g., "no DMs with strangers" → wrong allowlist semantics) is frozen into deterministic enforcement—the worst case is *deterministic wrongness*, not hallucination at runtime. Plain-English paraphrase back to parent can also hide the error if generated by the same model.
- **Recommendation**: Require **independent plain-English summary** from verifier/simulator ("On Friday at 8pm, YouTube would be: blocked/allowed because…") with scenario buttons. Kid-facing contract view should highlight rule types and consequences, not just YAML. Optional second-pass critique model is weaker than simulation—prefer deterministic scenario replay.

### [SEV-NICE-TO-HAVE] Compile pattern (function-calling vs grammar vs verifier loop) not specified
- **Section**: §6 — missing — should add
- **Issue**: This is a constrained NL→DSL task with a fixed ontology (categories, time, gates). The optimal pattern is not chosen; failure mode depends heavily on it.
- **Recommendation**: Specify: emit **JSON IR** via schema-constrained decoding (or tool call) matching a formal grammar; validate IR; render to YAML deterministically. Use grammar-constrained decoding for IR if backend supports it; otherwise LLM + verifier repair loop (max 2–3 iterations). Never ask the model to emit raw YAML as the primary artifact.

### [SEV-NICE-TO-HAVE] Model size guidance for J1 compile is absent
- **Section**: J1, §12.1
- **Issue**: Multi-clause contracts with conditionals and category mappings are mid-complexity structured extraction. 8B models can reach useful demo quality but not reliable family-facing compile without verification; 14B+ local or small cloud models materially improve constraint satisfaction.
- **Recommendation**: Document tested tiers: **minimum** (8B + mandatory verifier + template fallback), **recommended** (14B+ local or cloud equivalent for compile), **rules-only** (no compile LLM, form-based Limits/Gates). Set honest UX copy: local default may require edit/fix cycles.

### [SEV-NICE-TO-HAVE] No telemetry (§12.7) blocks production quality feedback loop
- **Section**: §12.7, §10 (diagnostic bundle)
- **Issue**: Without aggregate signal, maintainers cannot know if analyzers produce junk in the field; v0.2/v0.3 quality work becomes guesswork. Diagnostic bundle helps individual bugs but not distribution-level drift.
- **Recommendation**: Distinguish **telemetry** (automatic outbound) from **local quality metrics** (hub stores compile verify pass rate, LLM call counts, parent override/appeal rates—visible to parent, exportable in diagnostic bundle). Optional opt-in anonymous benchmark submit for maintainers, separate from product telemetry.

### [SEV-NOTE] Strong: deterministic enforcement for quantitative rules
- **Section**: §6, §8
- **Issue**: Keeping Limits/Gates out of LLM runtime evaluation is the correct pattern for reliability and auditability; the spec should lean into this harder rather than implying continuous LLM monitoring in J3.

### [SEV-NOTE] Strong: canonical compiled policy with signed versioning
- **Section**: §12.5
- **Issue**: Good foundation for determinism and kid appeal ("which rule?"); needs compile verification and normalization to deliver on the promise.

## Top 3 must-fix before implementation plan

1. **Resolve the MVP LLM boundary** — Explicitly define that Limits/Gates breach detection is 100% deterministic in MVP; specify exactly what (if anything) the LLM analyzer does at runtime (ideally: optional explanation only, not evaluation), and remove J3/§5.2 wording that implies LLM is on the enforcement critical path.
2. **Mandate compile pipeline: structured IR + verifier + sign gate** — Prose→JSON IR→validated YAML, with scenario simulation shown to parent before sign; block activation on verifier failure; define behavior when local 8B repeatedly fails (rules-only / cloud / templates).
3. **Add LLM quality & regression requirements** — Golden contract fixtures, event replay harness, per-backend capability matrix, and release gates; local hub metrics + diagnostic bundle format so v0.2 can be shipped and improved without outbound telemetry.
