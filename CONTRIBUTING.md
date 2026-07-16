# Contributing

> **Audience:** anyone about to change something here. **Purpose:** the day-to-day
> workflow. Full standards in
> [docs/engineering-standards.md](docs/engineering-standards.md).

## Before your first change

Read [docs/project-status.md](docs/project-status.md). It's fifteen minutes and it will
stop you from making a wrong assumption about what this repo is.

The assumption to avoid: **this looks like a working application and it isn't.** There is
no backend. `api()` at `index.html:1898` is a stub that writes to `localStorage`. The UI
is complete and convincing; nothing behind it exists.

## Setup

```bash
git clone https://github.com/CredifyFast/outreach-credify.git
cd outreach-credify
```

Open `index.html` in a browser. That's the whole setup — no install, no build, no server.
The app seeds itself with demo data on first load.

Reset to clean state: use the in-app reset, or clear site data for the origin.

## Workflow

```bash
git checkout -b feat/short-description    # feat|fix|docs|chore|refactor
# ... change ...
# ... verify by actually running it ...
git commit
git push -u origin feat/short-description
```

Then open a PR against `main`. No direct pushes.

## The rules that matter here

These are specific to this repo and most have been learned the hard way.

| Rule | Why |
| --- | --- |
| **Never regenerate `index.html` wholesale** | A whole-file rewrite has already silently destroyed work in this project's history. `grep -n` to find the target, replace surgically |
| **`index.html` and the mockup are byte-identical** | md5 `7c02a77a…`. Know which one you're editing and why. [R-1](docs/project-status.md#risks) |
| **Never commit real client data** | Not a CSV, not a screenshot, not an "anonymized" export. The `SEED_*` constants exist so you never need to |
| **Never put real data in `htdocs`** | On the maintainer's machine, Apache serves that whole directory. [Isolation rules](docs/deployment.md#-isolation-rules-for-the-xampp-host--these-apply-today) |
| **Design tokens only** | `var(--green-primary)`, never `#1a8a66` |
| **No inline `on*` handlers** | Use the existing `data-action` dispatcher |
| **Update the FILE LINEAGE header** | Top of `index.html`. Newest first, cap 10, drop the oldest |
| **Match the local style** | The file is dense on purpose. Don't reformat — a whitespace diff across 4,598 lines is unreviewable |
| **Delete dead code you touch** | Exception: the Triggers dispatcher case is commented out **deliberately** |

## Verify before you claim

**✅ means you ran something and watched it pass.** Not "the code looks right." A ✅ with
no command behind it is a lie with a checkmark on it.

For anything non-trivial, in order — cheapest gate first:

1. `node --check` on every extracted `<script>` block
2. CSS brace balance on every `<style>` block
3. Unit tests on any pure function you touched
   ([⬜ blocked until there's a runner](docs/project-status.md#prioritized-backlog))
4. A real browser interaction — perform the actual click path, assert the DOM, check the
   console for **zero new errors or warnings**
5. For date logic (quiet hours, business hours, scheduled sends, frequency caps): test
   under `TZ=America/Los_Angeles`, `TZ=America/New_York`, `TZ=UTC`. All four features are
   date math, and all four are wrong in exactly one timezone if you only test in yours

If a step fails, fix and restart from step 1. Don't hand back partially verified work.

## PR checklist

- [ ] Branch is `type/short-description`, off `main`
- [ ] Diff is surgical — no wholesale regeneration, no drive-by reformatting
- [ ] I ran it. The PR body says what I ran
- [ ] No secrets, no real client data, no PHI — in code, comments, fixtures, or screenshots
- [ ] FILE LINEAGE header updated if `index.html` changed
- [ ] Docs updated **in this PR** if behavior or state changed
- [ ] [CHANGELOG.md](CHANGELOG.md) updated
- [ ] Any `✅` I added has a command behind it
- [ ] Dead code I touched is gone
- [ ] Commit message explains **why**, not what

## Needs extra review

**Two approvals, one from someone who has read
[docs/security-compliance.md](docs/security-compliance.md):**

- The send path or the guard chain (`doQueueSend()` and anything it calls)
- PHI detection, storage, or transmission
- The audit log
- Authentication or authorization
- Any new logging of a request or response body
- **Any new third-party dependency or vendor** — it may need a BAA
  ([ADR-0007](docs/adr/0007-baa-gated-vendor-selection.md))
- Any migration that drops or alters a column

**A PR that weakens a guard needs an ADR, not an approval.** "This check is too strict and
slows people down" is a design decision with compliance consequences. It gets written
down with its alternatives — not merged because it unblocked someone's Tuesday. See
[ADR-0001](docs/adr/0001-record-architecture-decisions.md).

## Definition of Done

From [project-status.md](docs/project-status.md#definition-of-done):

1. Behavior matches spec, **verified by running it**
2. Tests cover the new logic and pass
3. No secrets, no real PHI, no real credentials
4. Docs updated in the same PR
5. Any ✅ has a command in the PR body
6. If it touches sending, PHI, or audit: server-side enforcement exists and is tested

## Questions

If something here contradicts the code, **the code is the truth and this doc is a bug** —
open a PR fixing the doc. Most claims cite a file and line; line numbers drift, so fix
them in the same PR that moves the code.

If a decision seems wrong, check [docs/adr/](docs/adr/) first. It may have been considered
and rejected for a reason — and if it wasn't, that's worth a new ADR.
