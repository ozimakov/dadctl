# Systems & deployability review — dadctl concept spec

*Reviewer role: senior distributed-systems engineer with home/SRE deployability experience.*
*Reviewing: `docs/superpowers/specs/2026-06-30-dadctl-concept-design.md`*
*Date: 2026‑06‑30.*

## Summary

The Hub–daemon split, deterministic Limits/Gates hot path, and local SQLite buffer are sound structural choices for a self-hosted family product, but the spec assumes a always-reachable home LAN in ways that break the stated journeys (parent decides from phone, kid at school, week-long offline). MQTT + embedded broker + mTLS on a non-443 port, `docker compose up`, first-run Ollama model pull, and one-time pairing codes together describe a credible homelab stack for a technical parent—not the "non-technical parent" persona in §3—while several reliability decisions (24h buffer, clock resync, offline contract enforcement) are underspecified enough to fail in real travel/sleep/off-network conditions. The riskiest bets are: (1) connectivity without a defined remote-access and firewall-traversal story, (2) conflating "self-hosted" with "zero-ops install," and (3) offline contract freshness with no backup/restore or monotonic version rules.

## Findings

### [SEV-BLOCKER] No remote connectivity model for daemon ↔ Hub or parent ↔ Hub
- **Section**: §5.2, §5.3, §5.4, §9
- **Issue**: The architecture assumes the kid's device can reach the Hub over the home network, but school/coffee-shop networks routinely block outbound MQTT (8883) and non-standard TLS. A parent PWA with WebPush also needs a browser-trusted HTTPS endpoint reachable when the parent is away—self-signed LAN certs and "Hub on parent PC" do not satisfy that without Tailscale, a tunnel, or a small cloud VM, none of which are in the install story.
- **Recommendation**: Add an explicit "reachability tier" to the spec: LAN-only MVP vs. recommended overlay (Tailscale/WireGuard) vs. optional small cloud Hub. Prefer daemon↔Hub transport on 443/WSS (or MQTT-over-443/WebSocket bridge) for school traversal; document hairpin NAT, dynamic DNS, and "Hub asleep on parent laptop" as known failure modes.

### [SEV-BLOCKER] Install story is homelab-grade, not family-grade
- **Section**: §5.2, §10, §12.1
- **Issue**: `docker compose up`, embedded MQTT broker, Ollama, reverse-proxy/TLS, and mTLS CA issuance is multiple failure domains before the first contract is signed. A non-technical parent cannot realistically stand this up on a NAS or old laptop without a single guided installer, health checks, and a default "rules-only, no LLM, no model download" path that works on first boot.
- **Recommendation**: Specify a tiered install: (A) one-command native installer or NAS package with bundled broker and auto-generated internal CA, (B) Docker as advanced, (C) optional cloud Hub image. First-run wizard must cover TLS (internal CA + optional Let's Encrypt), Hub URL the daemon will use (including overlay hostname), and resource preflight (RAM/disk for Ollama).

### [SEV-BLOCKER] Backup/restore entirely missing
- **Section**: missing — should add (§7, §8, §12)
- **Issue**: Append-only signed history, parent signing keys, device CA, mTLS certs, and contract versions live on Hub disk. Parent loses the NAS volume or repaves the Hub PC and the audit trail, signing chain, and device trust are gone; recovery path (re-pair all devices? accept history loss?) is undefined.
- **Recommendation**: Add a "Hub durability" subsection: encrypted export of CA + parent key + contract/event archive, scheduled backup target (local USB / NAS share), restore procedure, and explicit post-restore behavior (devices re-pair vs. cert re-issue, history continuity guarantees).

### [SEV-BLOCKER] Offline contract authority and stale-contract acceptance undefined
- **Section**: §9, §12.2, §7
- **Issue**: "Enforce last-known contract" does not define how the daemon rejects an older signed contract restored from backup, copied from another device, or replayed while Hub traffic is blocked. Without Hub-monotonic `contract_version` (or signed `effective_at` + anti-rollback), offline enforcement is not a well-defined state machine.
- **Recommendation**: Specify: daemon stores `max_applied_contract_version` and Hub-issued freshness token; on reconnect Hub pushes authoritative head and daemon rolls forward only; rollback attempts are signed `tamper_visible` events. Reconcile with §2 principle 6 ("warn, don't block") vs. §12.2 ("enforce last-known")—pick one offline stance or define thresholds (e.g., enforce Limits for ≤N hours offline, then warn-only).

### [SEV-BLOCKER] Internal contradiction on Hub-unreachable behavior
- **Section**: §2 (principle 6), §9, §12.2
- **Issue**: Principle 6 says offline daemons should err toward *warn, don't block*; §9/§12.2 say enforce last-known contract with no new escalations. Those differ materially for Limits/Gates (budget exhausted, quiet hours)—a week offline under one reading blocks, under the other does not.
- **Recommendation**: Resolve in spec with a single table: per rule kind and offline duration, exactly which enforcement actions remain active, which downgrade to warn/log-only, and what happens to in-flight Decisions past `expires_at`.

### [SEV-IMPORTANT] MQTT is workable but wrong default for firewall traversal and ops surface
- **Section**: §5.3, §5.4
- **Issue**: Traffic shape (low-volume events, retained status, control push) fits MQTT semantically, but running an embedded broker adds TLS termination, ACLs, cert rotation, and a second protocol stack alongside REST/WS for the parent client. NATS or gRPC streaming don't solve NAT/firewall; HTTPS long-poll + WebSocket on 443 is often simpler to operate and traverse. Retained MQTT status/decisions can serve stale payloads after reinstall unless tombstone/version rules exist.
- **Recommendation**: Either (1) standardize on Hub REST/WS + WSS for daemons on 443 with explicit reconnect/backfill API, or (2) keep MQTT but require TLS on 443 (WebSocket transport) and document retained-message lifecycle. Defer embedded broker to "zero-config mode"; allow external Mosquitto for advanced users.

### [SEV-IMPORTANT] Cross-platform daemon cost is underestimated (especially macOS)
- **Section**: §5.1, §10
- **Issue**: Rust + systemd/launchd is realistic for Linux/macOS MVP, but foreground-app collection, session lock, and DNS hooks differ sharply: macOS requires TCC (Screen Recording / Accessibility) with user-granted prompts; Linux splits X11 vs Wayland with no portable foreground-window API; DNS categorization on macOS needs Network Extension or a local resolver install, not a static binary drop-in. "Collectors as separate processes" multiplies packaging and autostart wiring per OS.
- **Recommendation**: Add a platform capability matrix to the spec (MVP vs post-MVP per primitive). Scope Linux MVP to one display stack first (likely X11 or explicit Wayland portal). Document macOS install as "guided permission wizard + signed/notarized pkg," not bare binary. Treat DNS collector as optional and non-gating for MVP Limits.

### [SEV-IMPORTANT] DNS category bucket is weak signal for contract enforcement
- **Section**: §5.1, §6
- **Issue**: DoH/DoT, browser secure DNS, and apps using hardcoded resolvers bypass system DNS; domain→category cannot support Gates like "YouTube requires homework" reliably. Foreground-app categorization covers screen-time Limits better than DNS for MVP.
- **Recommendation**: Classify DNS bucket as advisory/best-effort in MVP; enforce Limits/Gates on foreground app + curated allow/deny list. Defer browser extension to post-MVP unless URL-level Gates are in scope; if Gates stay in MVP, spec must name the minimum viable signal (app bundle ID / domain from a single supported browser path).

### [SEV-IMPORTANT] Session lock + dismissible warnings alone are a thin enforcement surface for teens
- **Section**: §5.1, §10
- **Issue**: For a motivated 14-year-old, session lock is a speed bump (unlock password, switch session, use phone/other device—explicitly out of MVP). Without app suspend or network-level gate, "grounding" reduces to UX friction that doesn't stop the active app. Credible for younger kids and contract-honor families; weak as the only technical lever for teen screen-time caps.
- **Recommendation**: Acknowledge enforcement ceiling in spec (aligned with "not adversarial"). For MVP, add one low-arm-race primitive if feasible per platform: e.g., full-screen modal overlay tied to active app category, or optional local DNS sinkhole for named categories (with documented bypass visibility). Keep SIGSTOP post-MVP but don't pretend the three current actions satisfy teen laptop scenarios.

### [SEV-IMPORTANT] 24h offline buffer is insufficient for stated travel use
- **Section**: §5.1, §9
- **Issue**: SQLite WAL can queue events far longer than 24h on disk; the "24h with no quality loss" reads as a retention/policy limit. A week-long school trip implies no contract updates, no parent Decisions, no appeal responses, and Hub-side aggregation gaps—not just delayed UsageEvents.
- **Recommendation**: Separate event buffer retention (target ≥7–14 days or size-capped) from enforcement offline policy. Define sync behavior on long disconnect: backfill order, Hub catch-up analysis window, and kid UI messaging for "Hub hasn't seen you in N days."

### [SEV-IMPORTANT] Clock skew handling is too thin to trust deterministic Limits/Gates
- **Section**: §9
- **Issue**: "Hub authoritative, resync on each connect" does not cover offline quiet-hours or daily budget boundaries when the daemon clock is wrong (deliberate or drift). Large skew "flagged" but still enforced locally can mis-apply or under-apply rules until reconnect.
- **Recommendation**: Require OS NTP in install preflight; daemon uses monotonic clock for durations, Hub-signed day boundaries for calendar rules; if skew > threshold while offline, downgrade to warn-only for time-window rules until resync.

### [SEV-IMPORTANT] Default Ollama + 8B first-run download is heavy for typical Hub hardware
- **Section**: §5.2, §12.1
- **Issue**: An 8B GGUF model is multi-GB download and often 6–8+ GB RAM at inference; many NAS units and old laptops fail silently or swap-thrash. "Download on first boot" on a metered or slow link can block setup for hours.
- **Recommendation**: Default new installs to rules-only mode; offer Ollama as opt-in with explicit RAM/disk gates and model size choices (small quantized vs. none). Hub preflight should refuse/enwarn sub-8GB RAM hosts. LLM is not on the hot path for Limits/Gates—treat it as optional analyzer capacity.

### [SEV-IMPORTANT] No security update path without telemetry
- **Section**: §12.7, §10
- **Issue**: Zero telemetry means no nudge to patch Hub, broker, or daemon CVEs in mTLS/TLS stacks. Self-hosted families historically run stale containers for years.
- **Recommendation**: Distinguish telemetry from update discovery: parent-initiated "check for updates" in Hub UI (GitHub releases/API, no install beacon), in-app release notes, and documented `docker pull` / package upgrade cadence. Specify daemon–Hub compatibility semver and minimum supported version for security fixes.

### [SEV-IMPORTANT] Kid read access to all rows is unspecified off-LAN
- **Section**: §7, §5.1
- **Issue**: "Kid has read access to every row about their own device" implies queryability, but the kid UI is local tray/Today page while authoritative store is on Hub. When Hub is unreachable, does the daemon mirror full signed history or only summaries? Who verifies signatures on the kid device?
- **Recommendation**: Define kid transparency as "daemon-served, Hub-synced signed mirror" with local verify against Hub CA/public key at sync time; specify which objects are replicated (Contracts, Decisions, Issues, Appeals) and max mirror retention.

### [SEV-IMPORTANT] Append-only signed model lacks compaction, rotation, and trust root
- **Section**: §7, §8
- **Issue**: Per-event append-only UsageEvents at monitoring cadence will grow without bound; signing scheme, batch vs. per-object signing, parent vs. Hub vs. device keys, and rotation/revocation are unstated. Kid read access and parent audit both depend on these choices.
- **Recommendation**: Add storage policy (monthly signed checkpoints, event rollup after N days). Specify trust root: Hub-operated CA for devices, parent key for Contracts/Decisions, Hub key for Issues; Ed25519 is fine if canonical serialization is defined. Document key rotation and device re-enrollment.

### [SEV-IMPORTANT] Pairing UX gaps for headless and async install
- **Section**: §10, §12.4
- **Issue**: "Daemon prints one-time code, parent enters in Hub UI" assumes co-present parent and kid machine with visible output; fails for SSH/headless Linux, parental install-before-handoff, or kid device setup while parent is at work.
- **Recommendation**: Specify code surfaces: tray, local HTTP pairing page, QR encoding `hub_url + code`, and optional deferred pairing (daemon runs local-only until claimed). Define code TTL, rate limits, and what the daemon does before pairing completes (collect only vs. enforce nothing).

### [SEV-IMPORTANT] Single-device MVP vs. identity model and real family shape
- **Section**: §10, §12.4, roadmap §11
- **Issue**: §10 cuts to single kid/device, but §12.4 already describes multi-device grouping; many families need phone + laptop or two kids to get value. Single-device MVP is viable for a controlled pilot, not as "a real family's daily driver" if the parent client must work mobile and the only kid device leaves the house.
- **Recommendation**: Either narrow MVP narrative to "alpha: one laptop, always home, technical parent operator" or add minimal multi-device read-only (second device paired, shared budget optional) and parent mobile reachability as MVP acceptance criteria.

### [SEV-NICE-TO-HAVE] Parent PWA HTTPS and WebPush need explicit TLS story
- **Section**: §5.3, §5.4
- **Issue**: WebPush and secure context require trusted HTTPS; LAN IP + self-signed cert breaks notifications and mobile install. Spec does not say how non-technical parents get a trust anchor on phones.
- **Recommendation**: Document supported patterns: overlay network with automatic TLS, reverse proxy + public domain, or "LAN-only PWA, no push away from home" as degraded mode with clear UX label.

### [SEV-NICE-TO-HAVE] Daemon packaging and service lifecycle underspecified
- **Section**: §5.1, §10, §11
- **Issue**: "Single static binary" still needs systemd unit, launchd plist, user vs. system service choice (session lock from user service is limited), uninstall, and upgrade without breaking mTLS identity.
- **Recommendation**: Add packaging deliverables per OS (.deb/.rpm, signed macOS pkg, Homebrew cask), service account model, and upgrade path that preserves device cert and local buffer.

### [SEV-NICE-TO-HAVE] Embedded MQTT broker operational boundaries
- **Section**: §5.2, §5.3
- **Issue**: Embedded broker simplifies day zero but complicates debugging, persistence tuning, and upgrade coupling; "point at existing broker" is mentioned but not when to prefer it.
- **Recommendation**: State defaults (embedded for single-node compose; external for split deployments) and persistence requirements (broker must survive Hub process restart without losing retained decisions).

### [SEV-NOTE] Rust cross-platform daemon is realistic with scoped MVP
- **Section**: §5.1, §10
- **Issue**: Deferring Windows is correct; Linux + macOS in one codebase is standard but permission and packaging work dominates feature work.
- **Recommendation**: No change to language choice; align MVP scope with platform matrix above.

### [SEV-NOTE] Deterministic Limits/Gates on Hub is the strong reliability choice
- **Section**: §6, §8
- **Issue**: Keeping LLM off the enforcement hot path avoids a class of deployability failures when Ollama is down or slow.
- **Recommendation**: Preserve; ensure offline and analyzer-down paths are identical for Limits/Gates.

## Top 3 must-fix before implementation plan

1. **Define reachability and install for real homes** — How daemon, Hub, and parent PWA talk over LAN, school Wi‑Fi, and parent-away scenarios; single supported path for TLS and remote access (overlay, tunnel, or cloud Hub), not only `docker compose` on a LAN IP.
2. **Specify offline authority, contract monotonicity, and backup/restore** — One coherent offline enforcement table, anti-rollback rules for signed contracts, Hub backup/export/restore, and device re-trust after disaster.
3. **Publish platform capability and signal honesty matrix** — What collectors and enforcers actually work on Linux vs macOS (X11/Wayland, TCC, DNS bypass), default to rules-only Hub, and demote DNS to advisory unless URL-level Gates are deferred.
