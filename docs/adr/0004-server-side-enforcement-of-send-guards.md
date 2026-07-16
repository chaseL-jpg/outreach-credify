# ADR-0004 · Send guards enforced server-side

**Status:** Accepted
**Date:** 2026-07-16

## Context

`doQueueSend()` implements nine sequential guards between an operator and a send:

```
recipients > 0 · template selected · scheduled date set · not in the past
phiAckValid() · dischargeAckValid() · capAckValid()
willSend > 0 after suppression · sender identity selected
```

This chain is the product. It's the reason an agency would choose this over Mailchimp,
and it encodes real domain insight — `dischargeAckSig()` hashes the sorted discharged-
recipient IDs so that changing the audience invalidates a prior acknowledgment, which
closes the "acknowledge the warning, then add 200 more people" hole that a naive boolean
flag would leave open.

**All nine guards run in the browser.** Every one is bypassable:

```js
// devtools console, ~15 seconds
state.cluster.phiAcknowledged = true;
state.suppressions = [];
state.freqCap.enabled = false;
```

Today this is harmless — there's no server to accept a send and no real data. It stops
being harmless the instant a backend exists.

The failure this prevents is not exotic. It's an operator who is behind, who has been
told the warning is "just being cautious," and who has learned that clicking through it
works. The guard exists precisely because well-meaning people under deadline pressure
make this mistake, and the client-side version stops exactly nobody who has ever opened
devtools — or who uses a modified client, or a replayed request.

## Decision

**The server re-runs every guard and trusts nothing from the client. The client-side
chain is demoted to a UX affordance.**

Specifically:

1. **`POST /sends` accepts audience *criteria*, never a resolved recipient list.** The
   server re-runs the filter chain against the database. A `recipients` array in the
   request body is rejected as an unknown field by schema validation.
2. **All nine guards re-execute server-side**, in the same order, returning 422 with
   structured findings on failure.
3. **Acknowledgments are validated, not trusted.** `dischargeAckSig` is recomputed from
   the *server-resolved* audience and compared. A stale signature fails.
4. **Suppression and opt-out are applied inside the send transaction**, against the
   database, immediately before dispatch.
5. **The sender is the session**, never a client-supplied id.
6. The client keeps its copy for instant feedback. It is advisory. If the two ever
   disagree, the server wins and the client is the bug.

## Consequences

**Positive**

- The guards become real. A bypassed client changes nothing about what actually sends.
- Suppression becomes an invariant enforced by a `UNIQUE` constraint plus a transactional
  check, rather than a client-side filter that a modified request skips.
- The audit log can record what the *server* decided, which is what an auditor needs.
  Client-reported audit entries are worth nothing — a client that can skip a guard can
  skip the audit call.
- Multi-client safety: a future mobile app, an API integration, or a replayed request all
  hit the same enforcement.
- Race conditions close. Two operators sending concurrently can both pass a client-side
  frequency check and breach the cap; a server-side check in a transaction cannot.

**Negative**

- **The logic exists twice** — client (advisory) and server (authoritative). They will
  drift, and drift means a user sees "ready to send" and gets a 422. Mitigation: extract
  the guards to `src/domain/` and have **both** import the same module
  ([ADR-0003](0003-single-file-html-prototype.md)). Same language is exactly why
  [ADR-0002](0002-node-express-postgres-backend.md) chose Node.
- Server-side audience resolution costs a round trip on every audience change. The live
  match count in the UI can stay client-side and approximate; only the send is
  authoritative. Accept the small inconsistency, don't try to make the preview exact.
- More work in Phase 2. This is the phase that cannot be cut.
- The 422-with-findings contract must mirror `scanPhiText()`'s `{kind, id, label}` shape,
  or the existing client rendering breaks.

**Neutral but important**

- This ADR does **not** decide whether PHI findings should be acknowledgeable at all.
  [ADR-0005](0005-phi-minimization-in-outreach.md) proposes making them a hard block with
  no override. If accepted, guard 5 becomes a 422 with no acknowledgment path — a
  strictly simpler contract.

## Alternatives considered

**Trust the client; keep guards client-side only.** Rejected: this is the status quo, and
it means the product's central claim is false. A guard that a devtools one-liner disables
is not a control; it's a decoration. Under HIPAA it is also indefensible — "we showed a
warning" is not an access control.

**Server-side guards only; remove the client copy.** Genuinely tempting — one
implementation, no drift, no dual maintenance. Rejected on UX grounds: the PHI warning
must appear *while composing*, not after clicking Send. Discovering that your template is
unsendable only at submit time is bad enough to push people toward workarounds, which is
how you get PHI pasted into a plain-text field to dodge the merge-tag scanner. The client
copy earns its cost by making the safe path fast.

**Sign the client's guard results.** Have the client compute the guards and send a signed
attestation. Rejected as security theater: the client controls both the inputs and the
signing, so a modified client signs whatever it likes. Cryptography doesn't make an
untrusted computation trustworthy.

**Guard at the ESP layer.** Configure suppression lists in the vendor. Rejected as
insufficient alone — the ESP knows nothing about discharge stage, frequency caps, or
merge-tag PHI. Worth doing *additionally* as defense in depth for the suppression list
specifically, since it's a genuinely independent check.

## Implementation

- [roadmap.md Phase 2](../roadmap.md#phase-2--guards-server-side) — with exit criteria
- [api-design.md](../api-design.md#post-sends--the-guard-chain) — the per-guard contract
- [project-status.md](../project-status.md#prioritized-backlog) — backlog #8

**Exit criterion, restated because it's the one that matters:** every guard has a test
that *tries to bypass it and fails*. Happy-path tests on a security control prove nothing
— the whole point is behavior under attack.
