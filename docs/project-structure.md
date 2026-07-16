# Project structure

> **Audience:** engineers navigating or extending the repo. **Purpose:** what's where,
> what may depend on what, and what to name things.

## Today (verified)

Five tracked files. `git ls-files`:

```
outreach-credify/
├─ index.html                                   4,598 lines  ⚠ see below
├─ CLUSTER_EMAIL_07-10-26_0510PM-PDT_SUMM-LINKS_SUPPORT (1).html
│                                               4,598 lines  ⚠ byte-identical to index.html
├─ home.html                                      329 lines  landing page
├─ test.v1                                          4 lines  ⚠ scratch junk — delete
├─ README.md                                                  hub
└─ docs/                                                      this doc set
```

> ### ⚠ `index.html` and the mockup are the same file
>
> ```
> $ md5sum index.html "CLUSTER_EMAIL_07-10-26_0510PM-PDT_SUMM-LINKS_SUPPORT (1).html"
> 7c02a77a371fad28e57335a2532e5b85 *index.html
> 7c02a77a371fad28e57335a2532e5b85 *CLUSTER_EMAIL_07-10-26_0510PM-PDT_SUMM-LINKS_SUPPORT (1).html
> ```
>
> Two files, one content, no marker declaring which is canonical. Edit either and they
> diverge silently — and a reviewer has no way to know which one mattered. **Resolve
> before editing either.** [Risk R-1](project-status.md#risks), backlog #4.

Absent, and worth naming explicitly: no `package.json`, no `.gitignore`, no `node_modules`,
no CI config, no `Dockerfile`, no `.env`, no tests, no build output. There is nothing to
install and nothing to compile.

## Anatomy of `index.html`

One file, four regions. Line numbers verified but they **will drift** — treat them as
signposts.

```
┌─ lines 1–32      FILE LINEAGE comment
│                    Provenance + changelog, newest first, capped at 10 entries.
│                    Genuinely useful — it's the only history the file itself carries.
├─ lines 33–41     <head> — meta, Google Fonts
├─ lines 42–~1050  <style> — one block
│                    :root design tokens (--green-primary, --mint-pale, …)
│                    then component classes (.btn, .card, .badge, .ms-panel)
├─ lines ~1051–1900  DATA + PURE LOGIC          ◀── the part worth extracting
│                    SEED_* constants (18)
│                    mkForm(), makeContacts(), buildJob()
│                    PHI_PATTERNS, PHI_TAG_KEYS, isPhiTag(), scanPhiText()
│                    PERSIST_KEY, persistState(), loadState()
│                    api() ← line 1898, the stub
├─ line 1907       const state = { … }          ◀── all app state, one object
└─ lines ~1910–4598  RENDER + DISPATCH
                     render<Tab>() per view
                     one delegated click dispatcher (data-action)
                     doQueueSend() ← the guard chain
```

The structure is better than "4,598-line file" suggests. There's a real seam around line
1900: everything above it is close to pure, everything below touches the DOM. That seam
is what makes [backlog #6](project-status.md#prioritized-backlog) tractable rather than a
rewrite.

**Conventions the file already follows** — keep them:

| Convention | Why |
| --- | --- |
| One `state` object, no hidden state | You can read the whole app's state in one place |
| `data-action` + one delegated dispatcher | No inline `on*` handlers; survives re-render |
| `SEED_*` for all seed data | Clearly separated from logic |
| Design tokens in `:root` | No hardcoded colors anywhere |
| FILE LINEAGE header, capped at 10 | Provenance without unbounded growth |

## Target (⬜ not built)

Where Phase 1+ code goes. Nothing below exists.

```
outreach-credify/
├─ public/                    static front-end
│  ├─ index.html              the app (slimmed as logic moves out)
│  └─ home.html               landing page
│
├─ src/
│  ├─ domain/                 ◀── PURE. No DOM, no HTTP, no DB.
│  │  ├─ phi/                     scanner: patterns, tag keys, findings
│  │  ├─ guards/                  the 9 checks from doQueueSend()
│  │  ├─ audience/                the filter chain from clusterRecipients()
│  │  ├─ templates/               merge-tag render
│  │  └─ suppression/             absolute-block rules
│  │
│  ├─ data/                   repositories. The ONLY place SQL lives.
│  ├─ http/
│  │  ├─ gate/                authn → authz → schema → idempotency
│  │  ├─ routes/              thin. Parse, call domain, format.
│  │  └─ errors/              the one error shape
│  ├─ services/               ESP, SMS, queue — behind interfaces
│  └─ config/                 env parsing. Fail fast on missing vars.
│
├─ migrations/                NNNN_description.sql, forward-only
├─ test/
│  ├─ domain/                 fast, pure, the bulk of the suite
│  ├─ integration/            routes + DB
│  └─ security/               ◀── bypass attempts. Guards must fail closed.
├─ docs/
└─ package.json
```

`src/domain/` is the point of the whole layout. It's the code that must be right, and
keeping it free of DOM/HTTP/DB is what makes it testable, reviewable, and portable.

## Dependency rules

**Dependencies point inward. `domain/` depends on nothing.**

```
        http/  ──────┐
                     ├──▶  domain/  ◀── depends on NOTHING
        data/  ──────┤              (no express, no pg, no window)
                     │
    services/  ──────┘
```

| Rule | Rationale |
| --- | --- |
| `domain/` imports **nothing** from `http/`, `data/`, `services/`, or any browser global | It must run in a plain Node test with no server, no DB, no fixtures. That's what makes the guard tests fast and trustworthy |
| `http/` may import `domain/` and `data/` | Routes orchestrate; they don't decide |
| `data/` may import `domain/` **types only**, never its logic | Repos map rows; they don't enforce rules |
| `services/` sit behind interfaces defined in `domain/` | Vendors get replaced. The ESP is not special |
| **SQL only in `data/`** | One place to audit for injection and for missing tenant/suppression predicates |
| **No `window`, `document`, `localStorage` in `domain/`** | The single rule that makes extraction possible |
| Routes are thin | Parse → call domain → format. If a route has an `if` about business rules, it's in the wrong layer |

**How to enforce it:** a lint rule, not a code review habit. `eslint-plugin-import`'s
`no-restricted-paths` catches an inward-pointing violation at CI time; a reviewer catches
it maybe. Add the rule in Phase 0, when there's nothing to fix yet.

The rule that will get tested first is "no `document` in `domain/`" — the extraction in
Phase 1 will surface a dozen places where a guard reaches for a DOM value. Each one is a
real coupling bug that the browser was hiding.

## Naming conventions

| Thing | Convention | Example |
| --- | --- | --- |
| Files (JS) | `kebab-case.js` | `phi-scanner.js` |
| Directories | `kebab-case`, plural for collections | `guards/`, `routes/` |
| Functions | `camelCase`, verb-first | `scanPhiText()`, `resolveAudience()` |
| Predicates | `is`/`has`/`can` + adjective | `isPhiTag()`, `canSend()` |
| Validators returning bool | `<thing>Valid()` | `phiAckValid()` — matches the prototype |
| Constants | `SCREAMING_SNAKE` | `PHI_TAG_KEYS`, `PERSIST_KEY` |
| Seed data | `SEED_<THING>` | `SEED_CONTACT_TYPES` |
| DB tables | `snake_case`, plural | `job_recipients` |
| DB columns | `snake_case`; FKs `<entity>_id` | `contact_type_id` |
| Slugs | stable, prefixed, never displayed | `ct_prospective_patient`, `stg_disch` |
| Routes | plural, kebab-case | `/sms-templates` |
| JSON fields | `camelCase` | `scheduledAt` |
| Migrations | `NNNN_verb_noun.sql` | `0003_add_audit_partitions.sql` |
| Tests | `<subject>.test.js`, mirroring `src/` | `test/domain/guards/phi.test.js` |
| Branches | `type/short-description` | `feat/server-side-guards` |

**Slugs vs labels.** The prototype gets this right and it's worth preserving: `stg_disch`
is the identifier, "Discharged" is the display string. Labels are the agency's language
and will change; slugs are the contract and must not. Never key logic off a label.

**Deliverable HTML files** follow the Credify convention
`{PREFIX}_{MM-DD-YY}_{HHMMAM/PM}-PDT_{CHANGE}_SUPPORT.html` — which is how the
`CLUSTER_EMAIL_…` mockup got its name. That convention is for *versioned deliverables
sent to Jay*, not for files living in the repo. Repo files get plain names (`index.html`).
Conflating the two is how the duplicate happened.

## Related

- [architecture.md](architecture.md) — why the layers are shaped this way
- [engineering-standards.md](engineering-standards.md) — style and testing
- [data-model.md](data-model.md#conventions) — DB conventions
