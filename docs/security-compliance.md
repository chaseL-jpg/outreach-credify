# Security & compliance

> **Audience:** everyone. **Required reading before touching sending, PHI, or the audit
> log.** **Purpose:** the threat model, the HIPAA posture, and the checklist that gates
> go-live.

> ## ⚠ Read this first
>
> **Current posture: nothing is deployed, no BAA exists, and no real PHI has ever entered
> this system.** Those three facts are the *entire* reason the current state is safe.
>
> The prototype is safe because it is inert, not because it is secure. Every guard in it
> is client-side and bypassable in seconds with devtools. **If you deploy this today with
> real contacts, you have built a HIPAA violation with a nice interface.**
>
> Do not connect this to real data until the [go-live checklist](#go-live-checklist)
> passes.

## Scope: PHI is in scope

Confirmed by the code, not just by policy. The system is structurally built to hold PHI:

| Evidence | Location |
| --- | --- |
| `ssn` and `signature` are first-class **field types** | `FIELD_TYPES` (20 types) |
| A **clinical** intake form exists | `SEED_FORMS` → `mkForm("clinical","Clinical Intake & History",…)` |
| Fields include `ssn`, `dob`, `photo_id`, `home_address`, `tax_id`, `income_doc`, `fin_signature` | `SEED_FORMS` |
| Stages include **In Treatment** and **Discharged** | `SEED_STAGES` |
| Contact types include **Patient** and **Prospective Patient** | `SYSTEM_TYPE_LABELS` |
| Templates merge `{{financial.balance_due}}`, `{{demographics.dob}}` | `SEED_TEMPLATES` |
| A PHI scanner exists at all | `index.html:1441-1452` |

**Also note:** the mere fact that a behavioral health agency is emailing someone can
itself imply treatment. Recipient identity plus sender identity is PHI in this context,
even when the body is anodyne. That's why the "Discharged" acknowledgment exists, and
it's why the contact list itself needs protection — not just the clinical fields.

## Threat model

Assets, ranked by what a breach actually costs.

| Asset | Why it matters |
| --- | --- |
| Contact list (identity + agency association) | **Implies treatment relationship.** Highest-value, most-overlooked |
| `contact_field_values` where `is_phi` | SSN, DOB, diagnosis, PHQ-9/GAD-7 scores, signatures, ID photos |
| Audit log | Compliance evidence. Losing it means being unable to *prove* you didn't breach |
| Suppression list | Consent evidence. Also identity |
| Template bodies | May contain PHI merge tags |
| ESP credentials | Send-as-the-agency capability |

### Adversaries and paths

| Adversary | Path | Today | Mitigated by |
| --- | --- | --- | --- |
| **The well-meaning operator** | Clicks through a PHI warning to hit a deadline | **The likeliest breach in this system, by far** | [ADR-0005](adr/0005-phi-minimization-in-outreach.md) hard-block; measure override rate |
| Curious insider | Reads contacts they have no business reading | No authz exists | RBAC + audit |
| Departed employee | Session still valid | No auth exists | Server sessions, instant revocation |
| External attacker | Unauthenticated write, injection | No server to attack (the only real defense right now) | The [gate](api-design.md#the-gate) |
| Client-side bypass | devtools → skip all 9 guards | **Trivially possible** | Server-side guards |
| Log aggregator | PHI written to logs, no BAA with the vendor | `console.debug` of full bodies at `index.html:1898` | [Logging rules](#logging) |
| Wrong-recipient send | Merge tag resolves to the wrong contact | Untested | Tests + preview |
| ESP itself | Vendor stores message bodies | No BAA | [ADR-0007](adr/0007-baa-gated-vendor-selection.md) |

**The first row is the real threat.** Most HIPAA breaches are not attackers — they're a
Tuesday afternoon, a deadline, and a warning dialog that a tired person clicked past.
This is a design problem, not a security-controls problem: if the safe path is slower
than the unsafe one, people take the unsafe one, and no amount of red text fixes that.
That's the argument behind [ADR-0005](adr/0005-phi-minimization-in-outreach.md) — remove
the override rather than make it scarier.

## Compliance posture

| Requirement | Status | Notes |
| --- | --- | --- |
| BAA with ESP | ⬜ | **Longest lead time in the project.** Backlog #1 |
| BAA with SMS vendor | ⬜ | Same |
| BAA with hosting provider | ⬜ | Required before any PHI touches disk |
| BAA with log/error/monitoring vendors | ⬜ | **Routinely forgotten.** Sentry/Datadog see request bodies |
| HIPAA risk assessment | ⬜ | Legally required. Findings change schema and infra — do it *before* the build, not after |
| Written policies & procedures | ⬜ | Access control, sanction, incident response |
| Workforce training | ⬜ | |
| Encryption at rest | ⬜ | No datastore exists |
| Encryption in transit | ⬜ | TLS 1.2+, HSTS |
| Audit controls | ⛔ | `state.auditLog` is in-memory and user-clearable — **fails outright** |
| Access controls | ⬜ | No authn/authz of any kind |
| Minimum necessary | 🟡 | The PHI scanner is a genuine attempt at this; the design intent is right |
| Breach notification plan | ⬜ | See [operations-runbook.md](operations-runbook.md) |

**TCPA/CAN-SPAM** (separate from HIPAA, applies to the outreach itself):

| Requirement | Status | Notes |
| --- | --- | --- |
| Functioning unsubscribe in every marketing email | 🟡 | UI models it (`state.unsubPage`); no endpoint exists |
| Honor opt-out promptly | 🟡 | Suppression logic modeled client-side only |
| Physical address in footer | ✅ | `state.footer` — "600 B Street, Suite 300, San Diego, CA 92101" |
| SMS STOP handling | 🟡 | `state.stopReply` modeled; no vendor wired |
| Prior express consent for SMS | ⬜ | `consent_text` field exists; enforcement doesn't |

## Isolation

**Single-tenant per deployment** until a real isolation model is designed and reviewed.
One agency's data, one database, one app instance.

Multi-tenancy in a PHI system is a design problem, not a `WHERE tenant_id = ?` problem.
A single missing predicate on a single query leaks one agency's clients to another —
which is a reportable breach for both. If and when it's needed: row-level security in
Postgres, enforced by the database rather than by developer discipline, plus a test suite
that proves cross-tenant queries return zero rows.

**On the current XAMPP host:** `C:\xampp\htdocs\` is a shared document root — any file
there is served by Apache to anything that can reach the port. This is a **local preview
convenience only**. Two rules while it stays that way:

1. **Never put real data in `htdocs`.** Not a CSV, not an export, not "just for a test."
2. **Never expose that Apache to a network.** It's serving your entire `htdocs`, not just
   this project.

The production deployment does not involve XAMPP at all — see
[deployment.md](deployment.md) and [ADR-0002](adr/0002-node-express-postgres-backend.md).

## Authentication & authorization

**⬜ Neither exists.** `state.currentUserId` is a dropdown that picks which of 5 seeded
reps you are pretending to be. It is impersonation-by-design, appropriate for a demo and
nothing else.

Target:

| Control | Approach |
| --- | --- |
| Authn | Server-side sessions, `httpOnly` + `Secure` + `SameSite=Lax` cookie. Not JWT — revocation must be instant when someone is terminated |
| Session lifetime | Idle timeout 30 min, absolute 12h. Shared workstations are normal in clinics |
| Password policy | Length over complexity. Check against a breached-password list |
| MFA | Required for any role that can send or read PHI |
| Authz | RBAC, **deny by default**. A route with no declared policy returns 403 |
| Roles | Derived from the seeded titles — Director of Outreach, Intake Coordinator, Care Coordinator, Billing Specialist, + Compliance Officer (read-only, full audit access) |

**Add the Compliance Officer role from day one**, even though the prototype doesn't model
it. Read-only across everything, full audit access, no send capability. They are the
persona the entire system is ultimately accountable to, and retrofitting a read-only role
after the fact is how you end up giving auditors admin accounts.

Minimum necessary means a Billing Specialist doesn't need `{{clinical.primary_dx}}`. The
`form_fields.is_phi` + role matrix is how that gets enforced rather than trusted.

## Data protection

| Layer | Rule |
| --- | --- |
| In transit | TLS 1.2+ everywhere. HSTS. HTTP redirects to HTTPS, never serves |
| At rest — DB | Full-disk/volume encryption **plus** column encryption for `contact_field_values.value` where `is_phi` |
| At rest — backups | Encrypted. **Test the restore** — an untested backup is a rumor |
| Keys | A managed KMS. Not in env vars, not in the repo, not in a wiki |
| In email bodies | **Nothing.** [ADR-0005](adr/0005-phi-minimization-in-outreach.md) |
| Retention | Contacts per agency policy. **Audit log: 6 years minimum** (HIPAA). Delivery events: 1 year |
| Deletion | Soft-delete contacts; audit entries are never deleted, by anyone, ever |

### Logging

**The rule: logs never contain PHI. No exceptions.**

This is the easiest rule to break and the hardest breach to notice, because nothing
fails. The system works perfectly and quietly copies PHI into a third-party service you
don't have a BAA with.

**There is a live example of the anti-pattern in the repo right now:**

```js
// index.html:1898 — DELETE THIS LINE when the stub is replaced
console.debug("[api stub]", method, path, body || "");
```

That prints full request bodies. Today it prints fake SSNs to a browser console and
harms nobody. Ported to Node as-is, it prints real SSNs to stdout, which your log shipper
forwards to a vendor, which retains them for 90 days, in a system with no BAA and
different access controls than the app. That is a reportable breach caused by a debug
line nobody thought about.

| Log | Never log |
| --- | --- |
| `requestId`, timestamp, actor id, route, status, duration | Request/response bodies |
| Job id, recipient **count**, guard outcomes | Email addresses, names, contact ids in message text |
| PHI finding **kinds** (`{kind:"tag", id:"{{clinical.primary_dx}}"}`) | PHI **values** |
| Error codes and types | Stack traces containing data; SQL with bound params |

Log that a guard tripped and which one. Never log what it found.

Applies equally to error trackers, APM, analytics, and crash reports. Each is a vendor.
Each needs a BAA or must never see PHI.

## Incident response

Full procedure in [operations-runbook.md](operations-runbook.md#incident-response). The
compliance-specific part:

**A suspected PHI breach starts a clock — 60 days to notify affected individuals** (HIPAA
Breach Notification Rule). Media and HHS notification may also apply depending on scale.

1. **Contain** — disable sending immediately. There should be a single kill switch that
   halts all outbound; **build it in Phase 1**, not after you need it.
2. **Preserve** — snapshot the audit log before anything else. Do not "clean up."
3. **Assess** — what data, whose, how many, was it actually acquired.
4. **Notify** — counsel first. **Legal makes the notification call, not engineering.**
5. **Remediate** — fix, then write the postmortem.

The audit log is what makes step 3 answerable. If it's incomplete, you cannot scope the
breach, and an unscopeable breach gets scoped as worst-case. That is the concrete cost of
`state.auditLog` living in `localStorage`.

## Go-live checklist

**Every box must be ✅ before real client data enters this system.** Not "mostly." Not
"we'll do the last two after the pilot."

### Legal — start first, they take longest

- [ ] ⬜ Signed BAA with ESP
- [ ] ⬜ Signed BAA with SMS vendor
- [ ] ⬜ Signed BAA with hosting provider
- [ ] ⬜ Signed BAA with **every** vendor that can see a request body (logs, APM, errors)
- [ ] ⬜ HIPAA risk assessment completed and findings addressed
- [ ] ⬜ Written policies: access control, sanction, incident response
- [ ] ⬜ Workforce training delivered
- [ ] ⬜ PHI-in-outreach boundary ratified ([ADR-0005](adr/0005-phi-minimization-in-outreach.md))

### Technical

- [ ] ⬜ All 9 send guards enforced **server-side** ([ADR-0004](adr/0004-server-side-enforcement-of-send-guards.md))
- [ ] ⬜ Audience resolved server-side from criteria; client recipient lists rejected
- [ ] ⬜ Audit log durable, append-only, `UPDATE`/`DELETE` revoked at the grant level
- [ ] ⬜ Authn + RBAC, deny by default; MFA for PHI roles
- [ ] ⬜ Compliance Officer read-only role exists
- [ ] ⬜ Encryption at rest (volume + column for `is_phi`)
- [ ] ⬜ TLS 1.2+, HSTS, no plaintext listener
- [ ] ⬜ `console.debug` body logging removed; log scrubbing verified **by test**
- [ ] ⬜ Suppression enforced in the send transaction, with a test proving it
- [ ] ⬜ Idempotency on `POST /sends`
- [ ] ⬜ Backups encrypted **and a restore actually performed**
- [ ] ⬜ Kill switch for all outbound sending, tested
- [ ] ⬜ SPF/DKIM/DMARC configured for real
- [ ] ⬜ Unsubscribe endpoint live, one-click, unauthenticated, tested
- [ ] ⬜ Dependency vulnerability scan clean
- [ ] ⬜ Google Fonts self-hosted (removes a third-party request from a PHI app)

### Verification

- [ ] ⬜ Guard chain has tests, including **bypass attempts**
- [ ] ⬜ A test proves logs contain no PHI under realistic payloads
- [ ] ⬜ A test proves a suppressed address cannot be sent to via **any** code path
- [ ] ⬜ Penetration test or equivalent review
- [ ] ⬜ Restore-from-backup rehearsed end to end

**Nothing here is ✅ today.** That's expected — the code is three phases from earning any
of them. What matters is that no one mistakes "the UI shows a PHI warning" for "PHI is
controlled."

## Related

- [ADR-0004](adr/0004-server-side-enforcement-of-send-guards.md) — server-side guards
- [ADR-0005](adr/0005-phi-minimization-in-outreach.md) — no PHI in outbound
- [ADR-0007](adr/0007-baa-gated-vendor-selection.md) — BAA-gated vendors
- [api-design.md](api-design.md#the-gate) — the gate
- [operations-runbook.md](operations-runbook.md) — incident procedure
- [project-status.md](project-status.md#prioritized-backlog) — why BAA is #1
