# ADR-0007 · BAA-gated vendor selection

**Status:** Accepted
**Date:** 2026-07-16

## Context

This system will hold PHI ([security-compliance.md](../security-compliance.md#scope-phi-is-in-scope)).
Under HIPAA, any third party that creates, receives, maintains, or transmits PHI on our
behalf is a Business Associate and requires a signed **Business Associate Agreement**
before it touches that data.

The obvious vendors get remembered: the ESP, the SMS provider, the hosting company. The
ones that don't get remembered are the ones that quietly see everything — the log
aggregator, the error tracker, the APM. Nobody thinks of Sentry as a place patient data
lives, right up until an exception serializes a request body containing
`{{clinical.primary_dx}}` and ships it to a vendor with no BAA, 90-day retention, and a
different access-control model than your app.

There's a live template for that failure in the repo already:

```js
// index.html:1898
console.debug("[api stub]", method, path, body || "");
```

Harmless in a browser with fake data. Ported to Node as-is, that line writes PHI to
stdout, which the log shipper forwards to a third party. Nothing errors. Nothing alerts.
It's a reportable breach caused by a debug statement nobody thought about.

The scheduling problem: **BAA execution is not an engineering task on an engineering
timeline.** It's a legal review at a vendor, and it takes 4–10 weeks of somebody else's
calendar. It cannot be compressed by working harder, and it is the longest pole in this
project.

## Decision

**No vendor may receive PHI — including incidentally, via logs or error payloads —
without a signed BAA. BAA procurement starts in week 1, before the code that needs it.**

Four rules.

**1 · The BAA gates the vendor, not the other way around.**
Do not select an ESP on features and then request a BAA. Filter to vendors that will sign
one, *then* choose among those. A vendor that won't sign is not a candidate regardless of
how good it is.

**2 · "Sees a request body" means it needs a BAA.**
Applies to log aggregators, error trackers, APM, session replay, analytics, uptime
monitors that hit authenticated endpoints, and support tooling. If PHI *could* reach it,
it needs a BAA or it must be architecturally incapable of receiving PHI.

**3 · Procurement starts week 1.**
[Backlog #1](../project-status.md#prioritized-backlog), ahead of every engineering task —
including tasks that are trivially fast. Four hours of forms, then weeks of waiting. Every
day it's delayed moves the launch by a day, and no amount of engineering recovers it.

**4 · Vendors sit behind interfaces.**
The ESP is called through an interface defined in `src/domain/`, not imported into route
handlers. Vendors get replaced — over pricing, over reliability, over a refused BAA
renewal — and that must be a service-layer change, not a rewrite.

## Consequences

**Positive**

- Legally sound before PHI exists, rather than retrofitted after.
- The forgettable vendors get caught by rule 2 at design time instead of during an
  incident.
- Starting week 1 means the legal track runs in parallel with Phases 0–3 and is done when
  Phase 4 needs it. This is the difference between a 16-week and a 24-week schedule.
- The interface rule keeps vendor switching cheap, which matters because BAA terms change
  at renewal and you need the option to walk.

**Negative**

- **The vendor pool shrinks and the price goes up.** HIPAA-eligible tiers cost
  meaningfully more, and some excellent tools are simply unavailable. That's the cost of
  the domain.
- Some tooling has to be given up or self-hosted. A session-replay tool that would make
  debugging easy is probably not happening.
- The 4–10 week wait is on the critical path and cannot be worked around.
- BAAs must be tracked and renewed — an ongoing administrative burden with no owner today.
- Vendor interfaces add a layer of indirection that looks like over-engineering right up
  until you need to switch.

## Alternatives considered

**Pick vendors on merit, request BAAs later.** The default path, and the one that
produces the worst outcome: you build against a vendor's SDK, discover in month four that
they won't sign, and either rebuild the integration or ship non-compliant. Rejected.

**Architect so no vendor ever sees PHI; skip BAAs.** Genuinely appealing — scrub
everything at the boundary, and vendors receive only ids and codes. Rejected as
*insufficient alone*: the ESP transmits the message body, which is PHI by definition if
anything clinical is in it, so at minimum the ESP needs a BAA. **Adopted as defense in
depth for everything else** — logs and error trackers should be architecturally incapable
of receiving PHI ([logging rules](../security-compliance.md#logging)) *and* covered by a
BAA where feasible. Belt and braces, because the log scrubber will have a bug someday.

**Self-host everything.** Own SMTP, own log stack, own monitoring. No BAAs needed because
no third parties. Rejected: a self-hosted mail server run by a one-person team will have
worse deliverability *and* worse security than a HIPAA-eligible ESP. You'd trade a
paperwork problem for an expertise problem, and lose. Reconsider for log aggregation
specifically, where self-hosting is tractable and the PHI risk is highest.

**Use a HIPAA-compliant PaaS that bundles BAAs.** One BAA covering hosting, DB, and
logging. Genuinely attractive for a small team — fewer relationships to track. Not
rejected; **deferred**. It's a live option once hosting is decided, and it may collapse
several rows of the checklist into one. Worth pricing during Phase 1.

## Implementation

- [Backlog #1](../project-status.md#prioritized-backlog) — starts week 1, owner Jay
- [security-compliance.md](../security-compliance.md#go-live-checklist) — the BAA
  checklist items
- [roadmap.md](../roadmap.md#phase-4--deliverability) — Phase 4 is gated on a signed ESP
  BAA
- [deployment.md](../deployment.md#configuration-reference) — every vendor-credential env
  var maps to a BAA that must exist

**The test for whether this ADR is being followed:** for every vendor credential in
`.env.example`, someone can name the signed BAA that covers it. If they can't, that
vendor must not see production traffic.
