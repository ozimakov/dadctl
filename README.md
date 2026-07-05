# dadctl

> **Parenting as a Code.**
>
> AI‑augmented, open‑source parental software focused on keeping kids **safe**,
> not on building walls around them.

`dadctl` is an early‑stage project. We're designing in the open.

## The idea in 30 seconds

1. The **parent** writes a device‑usage contract in plain language
   ("2 h on weekdays, none after 9 pm, YouTube only after homework…").
2. The **kid** reviews and accepts the contract the first time they touch the
   device each day.
3. A small **local daemon** reports lightweight usage events to a self‑hosted
   **Hub**.
4. The Hub uses AI to analyze usage against the contract and raises issues in
   near real time.
5. The Hub (within pre‑agreed limits) or the parent makes a decision:
   *nothing*, *warn*, *ground*.
6. The daemon enforces the decision — a warning card or a session lock — and
   logs every step.
7. Parent **and kid** see the same append‑only, signed history. No hidden
   surveillance, no opaque rules.

## Principles

* **Safety, not walls.** Help a family honor a contract they wrote together.
* **Transparent to the kid.** Same data, same reasoning, same history.
* **Open source, auditable, self‑hostable.** No mandatory cloud component.
* **Privacy first.** Collect the minimum. Process locally where possible.
* **Human in the loop.** AI proposes, parent decides.

## Status

Concept phase. The full vision, architecture sketch, MVP scope, roadmap, and
the in‑progress revision plan live in:

* [`docs/superpowers/specs/2026-06-30-dadctl-concept-design.md`](docs/superpowers/specs/2026-06-30-dadctl-concept-design.md)

The seven foundational design decisions are recorded in §12; a seven‑role
self‑review surfaced 24 follow‑on items, queued as **Step B** in §14 of that
document and brainstormed one at a time. The implementation plan starts once
Step B lands. The reviews themselves are in
[`docs/superpowers/specs/reviews/2026-06-30-multi-role-self-review/`](docs/superpowers/specs/reviews/2026-06-30-multi-role-self-review/).

Read it, push back, open issues.

The MVP is explicitly a **developer‑parent beachhead on Linux/macOS** — not a
general‑market parental‑control product. If your kid's primary device is a
Windows laptop or a phone, dadctl isn't for you yet; see the §11 roadmap.

## License

**Apache‑2.0** for the entire project (Hub, daemon, SDKs, schemas). See §12.3
of the concept doc for the rationale.
