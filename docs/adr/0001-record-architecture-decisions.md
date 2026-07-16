# ADR-0001 · Record architecture decisions

**Status:** Accepted
**Date:** 2026-07-16

## Context

This project has one part-time developer and no written decision history. The code
carries real, deliberate design thinking — the `api()` stub is a designed seam, the
`dischargeAckSig()` hash is a thoughtful answer to a subtle problem, the Triggers
dispatcher is commented out rather than deleted so it can be restored. None of that
reasoning is written down anywhere.

The only history that exists is the FILE LINEAGE comment block at the top of
`index.html`, and it's capped at 10 entries — by design, to stop it growing without
bound. So the reasoning behind decisions older than ten changes is already gone.

That's a bus-factor problem now and an onboarding problem later. When a new engineer asks
"why is the PHI check client-side?", the answer "it was a prototype and the backend never
happened" is very different from "we decided client-side was sufficient" — and without a
record, they cannot tell which, so they'll assume the second and preserve a mistake.

## Decision

Record significant decisions as ADRs in `docs/adr/`, numbered sequentially:
`NNNN-short-title.md`.

**Format:** Status · Date · Context · Decision · Consequences (both signs) ·
Alternatives considered.

**Significant** means: it constrains future work, it would be expensive to reverse, or a
reasonable engineer would ask "why?" — including decisions to *not* do something, and
decisions inherited from the prototype that were never explicitly made.

**ADRs are immutable once accepted.** To change a decision, write a new ADR that
supersedes the old one, and mark the old one `Superseded by ADR-NNNN`. Never rewrite an
accepted ADR — the wrong reasoning is exactly as valuable as the right reasoning, because
it tells you what the author knew at the time.

Statuses: `Proposed` · `Accepted` · `Superseded by ADR-NNNN` · `Deprecated`.

## Consequences

**Positive**

- The "why" survives the person. This project has one developer; the docs are the bus
  factor mitigation.
- Debates get resolved once. The XAMPP-vs-Node question ([ADR-0002](0002-node-express-postgres-backend.md))
  has a written answer instead of being relitigated every quarter.
- New engineers can distinguish deliberate design from accident — which determines whether
  they preserve it or fix it.
- Compliance auditors ask "why did you build it this way." Written answers with dated
  alternatives are much better than recollection.

**Negative**

- Writing them takes time, and the pressure to skip one is highest exactly when the
  decision is most consequential.
- They go stale. A superseded ADR that isn't marked superseded actively misleads.
- Over-ADR-ing turns a useful record into noise nobody reads. Not every choice is
  architectural — most are just code.

## Alternatives considered

**Nothing (status quo).** Rejected: it's what produced this situation. The reasoning
behind the current design exists only in one person's head, and the capped FILE LINEAGE
block guarantees the older parts are already lost.

**A wiki.** Rejected: drifts from the code, requires separate auth, and nobody updates it
in the same change as the code. ADRs live in the repo and move through the same review.

**Commit messages only.** Rejected: right granularity for "what changed," wrong for
"what we decided and what we rejected." Nobody archaeologizes git log to find out why
Postgres won.

**Extend the FILE LINEAGE block.** Rejected: it's capped at 10 entries on purpose, it's
per-file, and it's a changelog — it records what changed, not what was considered and
discarded.
