# ADR-0005 · PHI minimization in outbound messages

**Status:** Proposed
**Date:** 2026-07-16

> **This ADR is Proposed, not Accepted.** It requires ratification by Jay and compliance
> counsel. It is [backlog #3](../project-status.md#prioritized-backlog) because it blocks
> the data model and Phase 1 — `form_fields.mergeable` and the guard-5 contract both
> depend on the outcome.

## Context

The form engine can capture the most sensitive data a behavioral health agency holds.
Verified from `SEED_FORMS` and `FIELD_TYPES`:

- Field **types** include `ssn`, `signature`, `file`, `address`
- Fields include `ssn`, `dob`, `photo_id`, `home_address`, `tax_id`, `income_doc`,
  `fin_signature`, and a whole `clinical` form (`primary_dx`, `phq9_severity`,
  `risk_level`, `presenting_concern`)

**Every captured field is addressable as a merge tag** — `{{clinical.primary_dx}}` in an
email body resolves per recipient. The seeded templates already do this with financial
data: `{{financial.balance_due}}`, `{{financial.account_holder}}`.

Today's control is a scanner plus an acknowledgment. `scanPhiText()` finds PHI two ways
(5 regex patterns, 13 merge-tag keys + the `clinical` slug), shows a warning, and offers
a checkbox: *"I understand this content may include PHI and want to send anyway."*
`phiAckValid()` gates the send on it.

Two problems.

**The denylist is incomplete, and denylists always are.** `home_address` isn't in
`PHI_TAG_KEYS`. Neither is `photo_id`, `income_doc`, or `fin_signature` — all real seeded
fields, all identifying. Every new field an agency adds is safe-by-default until someone
remembers to add it to a hardcoded `Set` in a 4,598-line file. That's a control that
degrades silently as the product succeeds.

**The override is the actual failure mode.** The most likely PHI breach in this system is
not an attacker. It's a coordinator, at 4:45pm, who needs to get appointment reminders
out, who has seen the warning fifty times, and who has learned that the checkbox makes it
go away. Every acknowledgment is a decision made by a tired person under time pressure —
and the design currently routes the highest-stakes decision in the product to exactly
that person, at exactly that moment.

Meanwhile, the prototype already contains the alternative. `state.cluster.portalNotice`
and `{{demographics.portal_url}}` implement the **portal notice pattern**: send "you have
a secure message," link to the authenticated portal, put the clinical content there. The
right answer is already half-built.

## Decision (proposed)

**Three rules.**

**1 · PHI may not appear in an outbound email or SMS body. Hard block, no override.**

Guard 5 becomes a 422 with no acknowledgment path. `phiAcknowledged` is removed from the
send flow for both channels. If a template contains PHI, it cannot be sent — it must be
edited, or converted to a portal notice.

**2 · The allowlist replaces the denylist.**

`form_fields` gains two columns:

- `is_phi boolean NOT NULL DEFAULT true` — a field is PHI **until proven otherwise**
- `mergeable boolean NOT NULL DEFAULT false` — a field may appear in an outbound body
  only if explicitly marked

A new field is non-mergeable and PHI by default. Making it mergeable is a deliberate act
with a reviewer. The failure mode inverts: forgetting to classify a field makes it
*unusable*, not *leaked*.

**3 · The portal notice pattern is the sanctioned way to deliver clinical content.**

"You have a secure message from [Agency]" + authenticated portal link. The infrastructure
is already modeled.

Retained: the regex scanner (`PHI_PATTERNS`) for **literal** PHI typed directly into a
body. Nothing structural catches someone pasting "SSN 123-45-6789" into free text, so the
pattern scan stays — but it too becomes a block rather than a warning.

## Consequences

**Positive**

- The most likely breach path closes. You cannot click through a control that doesn't
  exist.
- New fields are safe by default. The control stops degrading as the product grows.
- The system's compliance story becomes simple enough to state in one sentence to an
  auditor: *"PHI cannot leave in a message body — structurally, not by policy."* That is
  a far stronger position than "we warn users and log their acknowledgment."
- Guard 5's contract simplifies: no ack state, no signature, no override branch.
- Aligns with HIPAA's minimum-necessary principle by construction.

**Negative**

- **It removes a capability someone may be relying on.** `{{financial.balance_due}}` is
  in a *seeded template* — balance reminders with real numbers are plainly an intended
  use. If billing reminders must carry a balance, this rule breaks a real workflow, and
  the honest answer is a portal link ("view your statement") rather than the amount. That
  is a product decision with revenue implications, and it's exactly why this is Proposed
  rather than Accepted.
- Portal notices have worse engagement than direct content. Fewer people click through.
  For appointment reminders, that may mean more no-shows — a real clinical cost, not just
  a metric.
- Requires the patient portal to exist and be reliable. `portal.credifyfast.com` is
  currently a placeholder string in seed data, not a system.
- Some templates need rewriting.
- Field-level classification is ongoing work for agencies adding custom fields.

**Open question for counsel**

Is `{{financial.balance_due}}` PHI? A balance owed to a behavioral health provider
arguably implies a treatment relationship, which is the same reasoning that makes the
contact list itself sensitive. But it's also ordinary billing communication that every
provider sends. **This ADR does not answer that — counsel must.** The answer determines
whether the billing workflow survives rule 1 intact.

## Alternatives considered

**Status quo — scan and acknowledge.** Rejected: routes the highest-stakes decision in
the product to a tired person at 4:45pm, and relies on a denylist that is already
incomplete and will fall further behind with every custom field. Its one virtue is
flexibility, which is the same thing as "it can be bypassed."

**Block by default, allow with a compliance-officer approval workflow.** A second person
approves any PHI-containing template. Genuinely better than self-acknowledgment — it adds
a reviewer who isn't under the sender's deadline pressure. Rejected for now on cost:
requires a role, a workflow, a notification path, and an SLA, all before the pilot. Worth
revisiting if rule 1 proves too strict in practice — it's the natural escape hatch and it
preserves the "not the person under pressure" property.

**Encrypt the email instead.** Encrypted-email gateways exist and are used in healthcare.
Rejected: recipients hate them, they need key exchange or a vendor portal, and if you're
sending someone to a portal to decrypt a message, the portal notice pattern is the same
thing with less machinery.

**Rely on the ESP's BAA and send PHI freely.** A BAA does make ESP transmission
*permissible*. Rejected anyway: a BAA is a liability arrangement, not a security control.
Email lands in inboxes on shared computers, forwards, gets synced to personal phones, and
sits in mail servers you have no relationship with. Minimum necessary applies regardless
of who signed what.

## Implementation

If accepted:

- [data-model.md](../data-model.md) — `form_fields.is_phi` + `mergeable` (already drafted
  in the proposed schema)
- [api-design.md](../api-design.md#post-sends--the-guard-chain) — guard 5 becomes a hard
  422; remove the ack path
- Client — remove the acknowledgment checkbox; surface the portal-notice conversion at
  the point of failure, so the safe path is the obvious next click
- Templates — audit and rewrite the seeded financial templates
- [roadmap.md](../roadmap.md#phase-1--backend-skeleton) — blocks Phase 1's data model

**Decide before Phase 1 starts.** The schema depends on it, and retrofitting a
`mergeable` allowlist after templates exist means auditing every template in the field.
