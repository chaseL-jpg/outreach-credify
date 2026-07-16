# ADR-0006 · localStorage persistence is prototype-only

**Status:** Accepted
**Date:** 2026-07-16
*Retroactive — documents an implicit decision and sets its expiry.*

## Context

All application state persists to a single browser key:

```js
// index.html:1879
const PERSIST_KEY = "credify_cluster_email_v1";
```

`persistState()` serializes the entire `state` object into it. `api()` calls it on every
"request." A reset handler does `localStorage.removeItem(PERSIST_KEY)` and reloads.

This was never a decision so much as the obvious way to make a prototype survive a page
refresh. It works well for that. Recording it now because it has a property that is easy
to miss: **`state.auditLog` lives in there too.**

An audit log that any user can destroy by clearing their browser data, that exists only
on one machine, that no administrator can read, and that has no integrity protection, is
not an audit log. It's a UI feature that resembles one. Under HIPAA's audit-controls
requirement, it fails outright — and it fails *silently*, because the tab renders
beautifully and nobody notices the log isn't real until an auditor asks a question it
can't answer.

The same applies to `suppressions`. A suppression list in `localStorage` means "we never
contact this person again" is scoped to one browser profile. Clear site data and the
consent record is gone — while the obligation isn't.

## Decision

**`localStorage` is the prototype's persistence layer and must not survive into any
environment holding real data.**

Concretely:

1. `localStorage` persistence is correct **today**, while the data is seeded and fake.
2. **It has an expiry**: Phase 1, when `api()` starts speaking HTTP.
3. **Two datasets may never live only in `localStorage` once real:** the **audit log**
   and the **suppression list**. Both are compliance records with legal retention
   obligations.
4. After Phase 1, `localStorage` may hold **UI preferences only** — active tab, open
   panels, column widths. Never contacts, never audit entries, never suppression, never
   PHI.
5. The key stays versioned (`_v1`), and a real migration/backfill gate is required if the
   shape changes while the prototype is still in use.

## Consequences

**Positive**

- The prototype survives refreshes with zero infrastructure — which is exactly what made
  the twelve-tab domain exploration possible.
- Reset-to-clean is one line, which makes demos and testing easy.
- The versioned key is genuinely forward-thinking: a persisted `_v1` snapshot from an
  older build can be detected and migrated rather than silently shadowing new seed data.
  (That trap — **stale persisted state overriding fresh defaults** — has bitten this
  project before and is worth watching for on every schema change.)

**Negative**

- **The audit log is not durable, not shared, not tamper-evident, and user-clearable.**
  This is the disqualifying one. See [R-4](../project-status.md#risks).
- **The suppression list is per-browser.** Two staff on two machines have two different
  ideas of who unsubscribed. Consent is not a local concern.
- No concurrency. Two tabs will clobber each other's writes, last-write-wins, silently.
- ~5–10MB quota. A few thousand contacts with 5 forms of field data will approach it, and
  the failure mode is a thrown exception mid-write — the code catches it and returns,
  meaning **the write silently doesn't happen**.
- Unencrypted, and readable by any script on the origin. Fine for fake data; an
  exfiltration path for real data.
- No backup. Clearing site data is unrecoverable and takes two clicks.

**The point:** every one of these is acceptable for a prototype and disqualifying for
production, and none of them announce themselves. The system looks like it's working
right up until an auditor asks "show me who was emailed on March 3rd" and the answer is
"whatever's in that person's browser, if they haven't cleared it."

## Alternatives considered

**IndexedDB.** More storage, transactions, no synchronous-write cliff. Rejected: solves
capacity and concurrency, solves none of the durability, sharing, or tamper-evidence
problems that actually matter. It would make the wrong architecture more comfortable to
live in, which is worse than leaving it uncomfortable.

**No persistence at all.** Reload gives a clean seeded state. Rejected: makes demos
painful and hides real bugs — specifically the stale-persisted-state class, which you
want to encounter in development rather than in the field.

**Go straight to a backend, skip local persistence.** In hindsight the "right" answer,
and rejected at the time for the reason in [ADR-0003](0003-single-file-html-prototype.md):
the domain was the unknown, and building a server to store a data model nobody had
designed yet would have inverted the order of discovery. The prototype earned the schema
that Phase 1 will build.

**Encrypt the localStorage payload.** Rejected as theater: the key would have to live in
the same JavaScript that reads it. It obscures data from a casual look and stops no
adversary, while adding the false comfort that something is protected.

## Supersession

Superseded in practice by [ADR-0002](0002-node-express-postgres-backend.md) at Phase 1,
when Postgres becomes the store of record. This ADR remains as the record of *why*
`localStorage` was acceptable, and — more usefully — as the explicit statement that it
**was never acceptable for the audit log**, so nobody in Phase 3 has to relitigate it
under deadline pressure.

Tracked: [backlog #9](../project-status.md#prioritized-backlog) (durable audit log),
[roadmap Phase 2](../roadmap.md#phase-2--guards-server-side).
