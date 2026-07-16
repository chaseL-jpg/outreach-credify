# Engineering standards

> **Audience:** anyone writing code here. **Purpose:** how we write, test, review, and
> ship.

> **⚠ Aspirational in part.** There is no test runner, no linter, and no CI in this repo
> today — verified: no `package.json`, no `.github/`. Sections below marked ⬜ describe
> what Phase 0 establishes. The **Git flow** and **editing rules** sections apply right
> now.

## Style

**The prototype has conventions. Follow them rather than importing your own.** Consistency
with a coherent existing style beats your preferred style, every time.

| Rule | Why |
| --- | --- |
| No inline `on*` handlers | `addEventListener` + `data-action` dispatcher. The existing pattern survives re-render; inline handlers don't |
| Design tokens only | `var(--green-primary)`, never `#1a8a66`. Every color is already a token |
| Stable, semantic `id`/`name` on every interactive element | The backend keys off them |
| `data-*` for anything the backend reads | |
| No hidden state | One named `state` object. If state lives in the DOM, it's a bug |
| Pure logic stays pure | No `window`/`document`/`localStorage` in domain functions |
| Standard JSON types only | Strings, numbers, booleans, arrays, plain objects, ISO-8601 dates. Must survive `JSON.stringify`/`parse` |
| One `collect<Thing>()` per form | Returns the exact POST body |
| All transport through `api()` | One place to add auth, retries, error handling |

**Comments state constraints, not narration.** `// dischargeAckSig invalidates when the
audience changes — do not memoize` earns its line. `// loop over recipients` does not.

**Formatting:** the existing file is dense — multiple statements per line, minimal
whitespace. That's a deliberate response to a 4,598-line single file, and reformatting it
would produce a diff nobody can review. **Match the local style when editing; don't
reformat.** New `src/` files (Phase 1+) get Prettier defaults and 2-space indent.

## Editing rules

These are specific to this repo and have been learned the hard way.

| Rule | Why |
| --- | --- |
| **Never regenerate `index.html` wholesale** | A whole-file rewrite has already silently erased work in this project's history. Targeted replacements only |
| Map before reading | `grep -n` the target, read that range. Don't read 4,598 lines to change 3 |
| Re-read after editing | Line numbers shift; a stale view produces a wrong edit |
| Update the FILE LINEAGE header | Newest entry first, cap 10, drop the oldest |
| Delete dead code you touch | Don't leave commented-out blocks — except deliberate parks like the Triggers dispatcher case, which is commented **on purpose** |

## Testing

**Current state: ⬜ zero tests.** No runner, no files, no coverage. Every refactor today
is a leap of faith, which is exactly why [backlog #6](project-status.md#prioritized-backlog)
(extract domain logic) must not start before #5 (manifest + runner).

### Strategy

Weighted toward pure domain tests, because that's where the risk is.

```
        ╱╲          security/     bypass attempts. Small, non-negotiable.
       ╱  ╲         integration/  routes + real DB. Some.
      ╱────╲        domain/       pure logic. THE BULK.
     ╱──────╲
```

| Layer | What | Speed |
| --- | --- | --- |
| **domain/** | Guards, PHI scanner, audience filter, merge render, suppression | ms. No DB, no server |
| **integration/** | Routes through the gate against a real Postgres | seconds |
| **security/** | Bypass attempts. Forge recipient lists, skip acks, replay keys | seconds |
| Manual | Anything subjective: does the warning read clearly, is the layout right | — |

### Rules

- **Write the test first.** Failing assertion, then the minimum code to pass.
- **Test behavior, not implementation.** `phiAckValid()` returning false for an
  unacknowledged clinical tag is behavior. How it iterates is not.
- **Every guard gets a bypass test.** Not "does it pass when valid" — "does it *fail* when
  someone tries to route around it." A guard with only happy-path tests is untested.
- **No real data in fixtures. Ever.** Not anonymized real data either — that's a
  re-identification bet with no upside. The `SEED_*` constants exist so you never need it.
- **Timezone matrix for date logic.** Quiet hours, business hours, scheduled sends, and
  frequency caps are all date math, and all four are wrong in exactly one timezone if you
  only test in yours. Run `TZ=America/Los_Angeles`, `TZ=America/New_York`, `TZ=UTC`.
- **A test proves logs carry no PHI.** Feed a realistic payload, assert the log output
  contains none of it. This is the only defense against the
  [`console.debug` class of bug](security-compliance.md#logging).

### Verification ladder

For anything non-trivial, in order — cheapest gate first:

1. `node --check` on every extracted `<script>` block
2. CSS brace balance on every `<style>` block
3. Unit tests on touched pure functions (+ the TZ matrix for date logic)
4. A real browser interaction (Playwright headless): perform the actual click path,
   assert the DOM, capture the console, **zero new errors or warnings**, save
   before/after screenshots
5. Cite the screenshot path when reporting

If any step fails, fix and restart from step 1. Never hand back partially verified work.

**✅ means you ran something and watched it pass.** Not "the code looks right." A ✅ with
no command behind it is a lie with a checkmark on it.

## Git flow

| | |
| --- | --- |
| Default branch | `main` |
| Branches | `type/short-description` — `feat/`, `fix/`, `docs/`, `chore/`, `refactor/` |
| Commits | Imperative mood, explain **why** in the body |
| Merge | PR into `main`. No direct pushes |
| History | Prefer new commits over amends on pushed branches |
| Hooks | Never `--no-verify`. If a hook fails, fix the cause |

Commit message:

```
Add server-side PHI guard to POST /sends

The client-side check in doQueueSend() is advisory — devtools bypasses it
in seconds. This re-runs scanPhiText() against the server-rendered body and
422s on unacknowledged findings.

Refs ADR-0004.
```

The body answers "why now, why this way." The diff already says what changed.

## Review

Every PR needs one approval. Reviewers check, in order:

1. **Correctness** — does it do what it claims, and did the author *run* it?
2. **Security** — does it touch sending, PHI, auth, or audit?
3. **Layering** — does `domain/` stay pure? ([dependency rules](project-structure.md#dependency-rules))
4. **Tests** — do they cover the failure mode, not just the happy path?
5. **Docs** — updated in *this* PR?
6. **Simplicity** — can any line be deleted?

### Needs extra review

Two approvals, one from someone who has read
[security-compliance.md](security-compliance.md):

- Anything in the send path or the guard chain
- Anything touching PHI detection, storage, or transmission
- Anything touching the audit log
- Anything touching authn/authz
- Any new logging of a request/response body
- Any new third-party dependency or vendor — **it may need a BAA**
- Any migration that drops or alters a column

**A PR that weakens a guard needs an ADR, not an approval.** If the argument is "this
check is too strict and slows people down," that's a design decision with compliance
consequences, and it gets written down with its alternatives — not merged because it
unblocked someone's Tuesday.

## CI/CD

**⬜ None exists.** Phase 0 establishes:

| Stage | Runs | On |
| --- | --- | --- |
| Lint | ESLint + the layering rule | every PR |
| Syntax | `node --check` on extracted scripts | every PR |
| Unit | `domain/` tests, TZ matrix | every PR |
| Integration | routes + Postgres service | every PR |
| Security | bypass tests | every PR |
| Audit | `npm audit`, fail on high | every PR + weekly |
| Secret scan | no keys, no PHI-shaped strings | every PR |

**Nothing merges red.** A branch that "just needs the flaky test rerun" is a branch with
a flaky test to fix.

Deployment: manual, gated, with a rollback plan — see
[operations-runbook.md](operations-runbook.md). No auto-deploy to anything holding PHI.
The five seconds you save aren't worth the breach you can't undo.

## Environments

| Env | Purpose | Data | Exists |
| --- | --- | --- | --- |
| Local | Development | Seed only | ✅ (open the file) |
| CI | Automated checks | Fixtures | ⬜ |
| Staging | Pre-prod verification | **Synthetic only — never a prod copy** | ⬜ |
| Production | Real | Real PHI | ⬜ |

**Staging never gets a production data copy.** It's the single most common way PHI ends
up somewhere with weaker controls than production, and the excuse is always "we needed
realistic data." Generate realistic synthetic data instead; the `SEED_*` generators
already do this well.

## Observability

⬜ Not built. When it is:

| Signal | Rule |
| --- | --- |
| Structured logs | JSON. `requestId` on everything. **Never bodies** |
| Metrics | Send volume, guard trip rate, **override rate**, bounce rate, queue depth |
| Traces | Route → domain → DB → ESP |
| Alerts | Send failures, queue stalls, **any suppression violation** (P1), auth anomalies |

**Every observability vendor sees your data. Every one needs a BAA or must never see
PHI.** This includes error trackers and APM, which people forget because they think of
them as tools rather than as third parties holding patient data.

The metric worth building first is the **override rate** — how often operators
acknowledge a guard and send anyway. It's the health check on the product's core premise.
See [roadmap Phase 5](roadmap.md#phase-5--pilot).

## Definition of Done

Same as [project-status.md](project-status.md#definition-of-done). Restated because it's
the thing most often skipped:

1. Behavior matches spec, **verified by running it**
2. Tests cover the new logic and pass
3. No secrets, no real PHI, no real credentials — in code, tests, or fixtures
4. Docs updated in the same PR, including status checkboxes and [CHANGELOG](../CHANGELOG.md)
5. Any ✅ has a command in the PR body
6. If it touches sending/PHI/audit: server-side enforcement exists and is tested

## Related

- [project-structure.md](project-structure.md) — layout and dependency rules
- [security-compliance.md](security-compliance.md) — what "extra review" is protecting
- [CONTRIBUTING.md](../CONTRIBUTING.md) — the day-to-day version
