# Changelog

All notable changes to this project are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
This project does not yet use semantic versioning — **nothing has been released**, and
there is no deployed artifact to version. Versioning starts at the first pilot
([roadmap M5](docs/roadmap.md#milestones)).

Entries before 2026-07-16 are reconstructed from `git log` and the FILE LINEAGE header in
`index.html`. Dates are commit dates; the pre-2026-07-16 grouping is approximate because
the early history has no changelog and the lineage block caps at 10 entries.

## [Unreleased]

### Added

- **2026-07-16** — Full project documentation set: [README](README.md) hub,
  [docs/](docs/) (status, product overview, architecture, data model, API design,
  security & compliance, roadmap, project structure, engineering standards, operations
  runbook, deployment), seven [ADRs](docs/adr/), this changelog,
  [CONTRIBUTING.md](CONTRIBUTING.md), and [.env.example](.env.example).
  Documents the prototype as it actually is — **no backend, `api()` is a stub, nothing
  deployed** — and records the Node/Express + PostgreSQL decision, the HIPAA posture, and
  a lead-time-ordered backlog.
- **2026-07-15** — Landing page (`home.html`, 329 lines). Design system lifted from the
  Cluster Email mockup: Sora + Instrument Serif, green/mint palette, card/button/badge
  components. Sections: sticky topbar, hero with product preview card, trust strip,
  features grid, workflow steps, stats band, CTA, footer.
- **2026-07-15** — `index.html` added as the directory entry point (commit `2aff472`).

### Known issues

Carried from [project-status.md](docs/project-status.md#risks). Listed here because they
are pre-existing defects, not planned work.

- **`index.html` duplicates the mockup byte-for-byte** (md5 `7c02a77a371fad28e57335a2532e5b85`).
  Two files, one content, no canonical marker. Editing either diverges them silently.
  ([R-1](docs/project-status.md#risks))
- **`home.html` lineage header is wrong** — line 3 reads `FILE: index.html`.
- **`test.v1` is committed** — 4 lines of scratch text from commits `f1254f0`, `94141b7`.
- **No `.gitignore`, no `package.json`, no tests, no CI.**
- **Segments UI is unreachable** — `view-segments` exists with no corresponding nav tab.
- **All send guards are client-side only** and bypassable via devtools.
  ([ADR-0004](docs/adr/0004-server-side-enforcement-of-send-guards.md))
- **Audit log lives in `localStorage`** — not durable, user-clearable, fails HIPAA audit
  controls. ([ADR-0006](docs/adr/0006-localstorage-persistence-is-prototype-only.md))

---

## Prototype history

Reconstructed from `git log` and the `index.html` FILE LINEAGE block. Prototype only —
none of this shipped anywhere.

### 2026-07-12

- Calendar mockup (`CALENDAR_07-12-26_…`, 3,193 lines) added, then deleted
  (`b041689` → `4139c5b`).

### 2026-07-10

Cluster Email mockup reaching current form (`2c32f8f`, 4,598 lines):

- Summary card: "X unsubscribed" links to Suppression tab; "X opted out" links to
  Preferences. Implemented via a `data-action` dispatcher to avoid template-literal quote
  nesting.
- All emoji removed; functional emoji replaced with text equivalents.
- Audience panel rebuilt as a 4-row accordion (Status, Assigned Rep, Contact Type,
  Exclude Contacts) with live match count, preview modal, and AND connectors.
  `clusterRecipients()` chain: status → rep (include/exclude) → type → exclude.
- Audit Log filters rebuilt: Credify date inputs with MM/DD/YYYY mask, searchable
  multi-select dropdowns, Select All / Clear All.
- Unsubscribe tab: Add New / Saved toggle, plus an Unsubscribe Templates view.
- Template editor: sticky footer, PHI warning surfacing, format + language toggles merged,
  separate "Attach file" (10MB) and "Image" (2MB) inputs.
- Send-to-Cluster: "Clear" resets the whole audience panel; empty-type-selection now
  toasts instead of silently no-op'ing.
- Triggers: "+ New Trigger" deliberately disabled — dispatcher case commented out rather
  than deleted so it can be restored.

### 2026-07-08 and earlier

- Email / SMS template toggle; `state.smsTemplates` with seed data.
- "By User" toggle on Open Rates; `sentBy` added to `buildJob()`.
- Searchable multi-select component (`.ms-panel`).
- Initial commit (`08bbfcf`) — README only.

---

## Conventions for this file

- **Update in the same PR as the change.** A changelog written later is a changelog
  written wrong.
- Group under `Added` · `Changed` · `Deprecated` · `Removed` · `Fixed` · `Security`.
- Write for someone who wasn't there. "Fixed the thing" helps nobody.
- **`Security` entries are mandatory** for anything touching guards, PHI, auth, or audit —
  even when the change is small, and especially when it weakens a control.
- Link the ADR when a change implements a decision.
