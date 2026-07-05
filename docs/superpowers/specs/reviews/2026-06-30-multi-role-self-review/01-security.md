# Security review — dadctl concept spec

*Reviewer role: senior application security engineer.*
*Reviewing: `docs/superpowers/specs/2026-06-30-dadctl-concept-design.md`*
*Date: 2026‑06‑30.*

## Summary

The spec has a coherent high-level security posture (self-hosted Hub, mTLS transport, deterministic hot path, parent-in-the-loop for escalations, deliberate non-adversarial stance toward a non-root kid), but it is **not security-defensible as written for implementation planning**. Several load-bearing mechanisms are named but not specified: cryptographic signing and key custody, Hub admin authentication, pairing-code parameters, MQTT authorization beyond client auth, certificate lifecycle, and daemon update/supply-chain controls. There is also an internal contradiction between §2 principle 6 ("warn, don't block" when Hub is unreachable) and §9/§12.2 ("enforce last-known contract"), which must be resolved before failure-mode security can be reasoned about. The non-adversarial threat stance is honest as a product choice, but the spec under-documents what parents actually give up and where "tamper visible" can mislead.

## Findings

### [SEV-BLOCKER] Cryptographic signing scheme is unspecified
- **Section**: §6, §7, §12.5; missing — should add
- **Issue**: The data model requires append-only signed objects and a `parent_sig` on contracts, but the spec never defines algorithm, canonical serialization, what fields are covered, who holds private keys (parent device vs Hub), whether the Hub co-signs, or how daemons verify signatures offline. Without this, contract integrity, decision authenticity, and "grounded kid can prove grounding ended" are unimplementable and unauditable.
- **Recommendation**: Add a § "Cryptography & identity" specifying: key types (e.g., Ed25519 for objects, X.509/mTLS for transport), key generation and storage (parent key in Hub keystore vs passkey/HSM), signature envelope format for `parent_sig`, canonical JSON/CBOR encoding, and verification rules on daemon (including offline).

### [SEV-BLOCKER] Hub admin / parent API authentication is unspecified
- **Section**: §5.2, §5.3; missing — should add
- **Issue**: The Hub exposes REST, WebSocket, PWA, contract CRUD, pairing, and decision issuance. There is no model for parent authentication, session management, CSRF protection, or authorization boundaries (kid-readable history vs parent-only settings per §12.1). On a home LAN, an unauthenticated Hub is trivially reachable by any household device, guest Wi‑Fi client, or compromised kid browser tab.
- **Recommendation**: Specify parent auth (minimum: strong password + session cookies with CSRF tokens; better: WebAuthn/passkey), bind sessions to origin, separate read-only kid endpoints from parent mutating endpoints, and require re-auth for pairing, contract signing, and LLM backend changes.

### [SEV-BLOCKER] Pairing one-time code lacks security parameters
- **Section**: §10, §12.4; missing — should add
- **Issue**: Pairing is the root of device trust and mTLS cert issuance, but code length, entropy, display lifetime, single-use semantics, rate limiting, and lockout after failed attempts are all absent. A short numeric code on a LAN-facing Hub is brute-forceable; a long-lived code displayed on the kid screen is shoulder-surfable and replayable.
- **Recommendation**: Require ≥128 bits entropy (or 8+ alphanumeric with CSPRNG), ≤5 minute TTL, single use, constant-time verification, exponential backoff / lockout on the pairing endpoint, and optional out-of-band confirmation (parent sees device fingerprint; daemon shows Hub fingerprint).

### [SEV-BLOCKER] Offline "last-known contract" enables downgrade and monitoring gaps
- **Section**: §9, §12.2
- **Issue**: If the daemon enforces whatever signed contract it last held, a kid who blocks Hub connectivity can prevent sync of a newer, stricter contract and continue under an older permissive one. The same offline window suppresses new Issues/escalations (§9), allowing rule violations without parent notification. Queued events flush on reconnect, so delaying reconnect hides activity during the offline window.
- **Recommendation**: Define monotonic contract state: daemon tracks highest `(contract_id, version, effective_at)` seen from Hub; reject locally presented contracts that regress; on prolonged Hub loss, transition to an explicitly documented degraded mode (e.g., limits-only with tightened defaults, or warn-only per resolved §2/§9 policy) and emit tamper-visible `sync_stale`/`hub_unreachable` events with parent alerting when connectivity returns.

### [SEV-BLOCKER] Internal contradiction on Hub-unreachable enforcement stance
- **Section**: §2 principle 6 vs §9 vs §12.2
- **Issue**: §2 says the daemon should "err on the side of warn, don't block" when Hub is unreachable; §9 and §12.2 say "enforce last-known contract." These are incompatible failure policies with different security and safety trade-offs. Implementers will pick one arbitrarily.
- **Recommendation**: Resolve to a single normative behavior, document the rejected alternative and why, and tie it to explicit parent-configurable policy with safe defaults.

### [SEV-IMPORTANT] mTLS pairing model is incomplete beyond "per-device cert"
- **Section**: §5.4, §12.4
- **Issue**: Per-device mTLS at pairing is directionally sound, but the spec omits: Hub CA creation and protection, cert TTL and rotation, revocation (CRL/OCSP or allowlist), broker topic ACLs (device A must not subscribe/publish to device B's topics), TLS minimum version/ciphers, and secure storage of daemon private keys (OS keychain, permissions). Compromised or cloned certs grant durable impersonation.
- **Recommendation**: Add a PKI subsection: Hub-internal CA, cert lifetime (e.g., 90 days) with automated rotation, explicit revocation on unpair, broker ACLs per `device_id`, and private-key non-exportability where platforms allow.

### [SEV-IMPORTANT] MQTT messaging lacks application-layer integrity and replay controls
- **Section**: §5.4
- **Issue**: mTLS authenticates the channel, not message semantics. Retained topics (`device/<id>/status`, decisions) invite stale-message replay, cross-device confusion if ACLs fail, and broker state manipulation. Decisions and contract pushes need freshness guarantees independent of TLS session reuse.
- **Recommendation**: Require signed envelopes on `Decision` and contract payloads with `(object_id, seq, ts, hub_key_id)`; monotonic sequence per device; reject stale/regressive seq; define QoS, retain policy per topic, and LWT semantics for liveness.

### [SEV-IMPORTANT] Daemon privilege model and IPC attack surface unspecified
- **Section**: §5.1
- **Issue**: The daemon runs as a system service with collectors (DNS hook, optional browser extension), kid UI, enforcer (session lock, SIGSTOP), and SQLite. User ↔ daemon IPC, collector ↔ daemon IPC, and extension messaging are all trust boundaries. Running as root/system broadens compromise blast radius; running as user eases kid tampering—neither is analyzed.
- **Recommendation**: Specify least-privilege per platform (e.g., dedicated system user, capability-based Linux collectors), authenticated IPC (socket permissions + token/nonce), and threat notes for each collector's extra privileges.

### [SEV-IMPORTANT] No daemon or Hub update / supply-chain security path
- **Section**: §5.1, §5.2; missing — should add
- **Issue**: "Single static binary" and Docker install imply ongoing updates, but there is no signed release channel, update verification, rollback policy, or dependency pinning strategy. This is a primary real-world compromise vector for parental-control software.
- **Recommendation**: Mandate signed releases (cosign/minisign), reproducible builds where feasible, in-band update verification before install, and documented refusal to auto-update without parent approval.

### [SEV-IMPORTANT] Default Ollama install expands local attack surface and supply-chain risk
- **Section**: §5.2, §12.1
- **Issue**: Ollama commonly listens on the network without authentication; default first-run model pull trusts registry integrity weakly compared to OS packages. The analyzer sends usage-derived content to the LLM—prompt injection via `UsageEvent.metadata`, window titles, or domains can influence Issue generation. Switching to a cloud endpoint (§12.1) moves child activity data off-LAN with no mention of egress controls.
- **Recommendation**: Bind Ollama to localhost by default, require Hub-side auth to the LLM adapter, verify model checksums/signature against published manifests, sandbox/prompt-harden the analyzer, strip or allowlist metadata fields, and document cloud-mode data classification.

### [SEV-IMPORTANT] Non-adversarial stance leaks substantial risk to parents without explicit acceptance criteria
- **Section**: §8
- **Issue**: The spec assumes a non-root kid but still promises enforcement and tamper visibility. A motivated kid with a normal account can plausibly: stop/disable the service, block Hub egress, use an unpaired device, bypass DNS hooks or extensions, or exploit offline semantics—all while producing ambiguous or delayed signals. Parents may interpret "signed history" and MQTT liveness as stronger guarantees than the architecture provides.
- **Recommendation**: Add a "Parent-accepted residual risk" table: bypass methods, expected visibility, and what the product explicitly does *not* prevent. Require Hub UI copy that offline/tamper states reduce assurance, not just "banner to kid."

### [SEV-IMPORTANT] "Tamper visible but not impossible" can mislead in common scenarios
- **Section**: §8, §9
- **Issue**: Visibility depends on parent monitoring, timely reconnect, and interpreting `daemon_gap` / offline events. A kid can create gaps before violations, prevent escalation while offline, or use alternate hardware; visibility arrives after the fact or not at all if the parent doesn't act on alerts.
- **Recommendation**: Define SLAs for tamper signals (push to parent, persist in signed history, cannot be cleared locally without Hub ack) and distinguish "detected tamper" from "continued safe enforcement."

### [SEV-IMPORTANT] Hub compromise model is stated but Hub hardening is not
- **Section**: §5.2, §8
- **Issue**: The spec correctly treats Hub compromise as total breach, then relies on "small attack surface" without mandating bind addresses, TLS for PWA/API, secrets management, database encryption at rest, backup integrity, or Docker port exposure defaults.
- **Recommendation**: Add Hub hardening defaults: localhost-only or reverse-proxy TLS, no default credentials, encrypted contract store, secrets outside compose env files, and security headers / CSP for the PWA.

### [SEV-IMPORTANT] No coordinated vulnerability disclosure path despite no telemetry
- **Section**: §12.7; missing — should add
- **Issue**: Zero telemetry means no fleet-wide exploit awareness. Without `SECURITY.md`, a supported disclosure channel, and release tagging policy, vulnerabilities may stay local or go unreported; diagnostic bundles are a poor substitute for structured intake.
- **Recommendation**: Publish SECURITY.md (embargo policy, contact, expected response times), GitHub Security Advisories workflow, and explicit guidance on what diagnostic bundles must never contain (keys, certs, raw window titles).

### [SEV-NICE-TO-HAVE] Contract DSL and compiled policy need injection and validation bounds
- **Section**: §6, §12.5
- **Issue**: LLM-compiled YAML becomes canonical enforcement input. Without schema validation, size limits, and a restricted expression language, malicious or buggy compilation could produce pathological rules or parser differentials between Hub and daemon.
- **Recommendation**: Specify a strict schema, max policy size, deterministic parser, and human-review diff before signing; daemon rejects policies failing validation even if signed.

### [SEV-NICE-TO-HAVE] Key rotation and recovery are unaddressed
- **Section**: §7; missing — should add
- **Issue**: No story for parent key rotation, device re-pair after compromise, Hub restore from backup, or kid label/device grouping changes. Stale keys undermine append-only history trust.
- **Recommendation**: Define rotation (overlap period with dual-key verify), re-pair/revoke flows, and backup restore that preserves monotonic contract history.

### [SEV-NICE-TO-HAVE] Diagnostic bundle scrubbing is a secret-leak channel
- **Section**: §10, §12.7
- **Issue**: Manual bug reports via scrubbed archives are necessary without telemetry, but scrubbing is hard: mTLS keys, pairing artifacts, window titles, and LLM prompts can leak via logs.
- **Recommendation**: Specify scrub allowlist (not blocklist), parent preview before export, and automated tests on bundle contents.

### [SEV-NOTE] Apache-2.0 has security-relevant trade-offs, mostly acceptable if acknowledged
- **Section**: §12.3
- **Issue**: Permissive licensing allows fork-and-close products that inherit trust branding without upstreaming fixes; there is no obligation to publish security patches in derivative works. Patent grant is a modest plus.
- **Recommendation**: Document that security assurance comes from auditing **your** deployed artifact (hash/signature vs upstream releases), not license alone; encourage signed release tags and advisory publishing on the main project.

### [SEV-NOTE] MQTT 3.1.1 + embedded broker is workable for MVP with caveats
- **Section**: §5.4
- **Issue**: Choice is reasonable for home networks, but MQTT 3.1.1 lacks modern auth extensibility; embedded brokers vary widely in hardening quality.
- **Recommendation**: Name a specific broker (or minimum security profile), document why not MQTT 5, and require TLS 1.2+ with mTLS mandatory in production configs.

## Top 3 must-fix before implementation plan

1. **Define the full crypto and identity model** — `parent_sig` format, signing keys, verification on daemon (including offline), decision/contract anti-replay, and monotonic contract versioning to close downgrade while Hub-unreachable.
2. **Specify Hub parent authentication and pairing hardening** — session auth for all mutating APIs, pairing code entropy/TTL/rate limits, and mTLS PKI lifecycle (CA protection, rotation, revocation, broker ACLs).
3. **Resolve and security-analyze the Hub-unreachable policy** — eliminate the §2 vs §9/§12.2 contradiction, document exploitable offline windows (contract downgrade, suppressed escalations, delayed event flush), and choose degraded-mode behavior parents explicitly accept.
