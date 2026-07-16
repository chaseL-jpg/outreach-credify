# Product overview

> **Audience:** anyone who needs to understand what this is for before touching it.
> **Purpose:** the problem, the users, the vocabulary, and the boundaries.

## Vision

Small behavioral health agencies run outreach on tools built for people selling
software. Mailchimp doesn't know that a discharged client is a different category of
person than a lead. Your CRM's "blast to all contacts" button doesn't know that three of
those contacts are patients whose diagnosis is implied by the mere fact that you're
emailing them.

The gap isn't features — it's that generic tools treat a mistake as a bounced email,
while in behavioral health a mistake is a HIPAA breach with a notification duty attached.
An agency of eight people cannot staff around that. So they either don't do outreach, or
they do it in a spreadsheet and hope.

**outreach-credify's premise: encode the rules into the send path so that the safe
action is the default one, and the unsafe action requires you to stop and acknowledge in
writing what you're about to do.**

The prototype already embodies this. `doQueueSend()` won't queue a job that contains an
unacknowledged PHI merge tag, or that targets a discharged client without a specific
acknowledgment, or that breaches a frequency cap. It doesn't hide those cases — it
surfaces them and makes you sign for them. That's the product.

## Personas

Verified against `REPS` in `index.html` — these are the seeded roles, which reflect how
the maintainer models the customer's org chart.

| Persona | Seeded as | What they do here | What they fear |
| --- | --- | --- | --- |
| **Director of Outreach** | Janet Dobson | Builds audiences, writes templates, schedules partner campaigns, reads open rates | Sending something embarrassing — or reportable — to 200 referral partners |
| **Intake Coordinator** | Paul Pranav | Follows up with prospective clients, manages consent and preferences | Contacting someone who opted out |
| **Care Coordinator** | Balaji Miller, Sherilynne | Reaches active clients about appointments and care | Putting clinical detail in an unencrypted email |
| **Billing Specialist** | Taras Pavlyk | Balance reminders, statements, payment links | Exposing financial + identity data (SSN, account) in a merge tag |
| **Compliance officer / owner** | *not modeled* | Answers to auditors. Needs the audit log to be complete and immutable | That the audit log lives in `localStorage` and any user can clear it |

**The compliance officer is the most important unmodeled persona.** Every feature in
this system is ultimately answerable to them, and there is no UI for them today. The
audit tab is a report, not an oversight tool. Worth fixing before the pilot.

## Domain glossary

Terms mean specific things here. Getting them wrong in code or conversation causes real
confusion — several are near-synonyms in English and distinct in the model.

| Term | Meaning here | Not to be confused with |
| --- | --- | --- |
| **Contact** | Any person or org record. 20 system types exist | "Client" — a contact *type*, not a synonym |
| **Contact type** | One of 20 system labels (`Prospective Patient`, `Referral Source Partner`, …). Each is Prospective (P) or Active (A) | Status |
| **Status** | Where the contact stands operationally. 10 seeded | Stage |
| **Stage** | Clinical/lifecycle position: New Lead → Contacted → Intake Scheduled → In Treatment → Discharged → Closed | Status. Stages are clinical; statuses are operational |
| **Cluster** | An audience: the filtered set a job sends to. The AND-chain of status → rep → type → exclude | "Segment" — a saved audience; **its UI is orphaned** |
| **Job** | One send: template + audience + schedule + tracking, snapshotted at queue time | Template |
| **Template** | Reusable subject+body with `{{form.field}}` merge tags. `text` or `html` | Job |
| **Merge tag** | `{{slug.key}}`, resolved per recipient from form data. **The main PHI leak vector** | |
| **Category** | Message purpose: 6 seeded, each `transactional` or `marketing`. Drives per-contact notify prefs | Contact type |
| **Suppression** | Hard block. Never send, regardless of anything else | Opt-out |
| **Opt-out / preference** | Per-category consent. May receive transactional but not marketing | Suppression — which is absolute |
| **Frequency cap** | Max sends per window + min gap. Seeded: 3 per 7 days, 24h gap | Quiet hours |
| **Quiet hours** | Clock window when sends are held | Business hours — a separate, independent window |
| **PHI** | Protected Health Information. Detected two ways: 5 regex patterns and 13 merge-tag keys + the `clinical` form slug | |
| **Guard chain** | The 9 sequential checks in `doQueueSend()` | |

**The status/stage distinction trips everyone.** A contact can be status "Active" and
stage "Discharged" simultaneously — and that combination is exactly what the discharge
acknowledgment guard exists to catch.

## Feature inventory

Twelve reachable tabs, verified by `grep -o 'data-tab="[a-z-]*"' index.html | sort -u`.

| # | Tab | What it does | State |
| --- | --- | --- | --- |
| 1 | **Send** | Audience builder, template picker, delivery mode, guard chain, confirm | ✅ UI complete, sends nothing |
| 2 | **Contacts** | Browse/filter contacts; status, rep, type, exclude filters | ✅ UI on seed data |
| 3 | **Templates** | Email + SMS templates, merge-tag inserter, PHI scan, preview | ✅ UI complete |
| 4 | **Types** | Manage the 20 contact types | ✅ UI complete |
| 5 | **Scheduled** | Queued/sent job list, cancel, per-recipient status | ✅ UI, simulated |
| 6 | **Rates** | Open rates by contact / type / user; best-send-hour | ✅ UI, simulated data |
| 7 | **Clicks** | Click tracking by job/link | ✅ UI, simulated |
| 8 | **Deliverability** | SPF/DKIM/DMARC check, bounce/complaint events, seed test | ✅ UI, **fully simulated** |
| 9 | **Suppression** | Email + SMS suppression lists, add/remove | ✅ UI complete |
| 10 | **Preferences** | Per-contact category consent | ✅ UI complete |
| 11 | **Triggers** | Event-driven sends. "+ New Trigger" **deliberately disabled** | 🟡 Intentionally parked |
| 12 | **Audit** | Filterable event log: users, types, sources, event type, date range | ✅ UI, **in-memory only** |
| — | **Segments** | Saved audiences. `view-segments` exists, **no nav tab** | ⛔ Unreachable dead code |

Two things to notice. **Deliverability is the most misleading tab** — it renders SPF/DKIM
/DMARC verdicts and bounce events that are entirely fabricated client-side. A
stakeholder demo could easily be read as "deliverability works." It doesn't exist.
**Triggers is disabled on purpose** — the dispatcher case is commented out rather than
deleted, so it can be re-enabled by uncommenting.

## Non-goals

Explicitly out of scope. Each has a reason; when someone asks "can it also…", start here.

| Non-goal | Why |
| --- | --- |
| **Being an EHR** | Clinical records live in the EHR. This module reads contact/demographic data for outreach and nothing more |
| **Being a general marketing platform** | Every safety assumption here comes from behavioral health. Genericizing removes the reason to exist |
| **Two-way inbox / conversation threading** | Out. SMS STOP handling is consent enforcement, not a chat feature |
| **Sending clinical content in email bodies** | Proposed as a hard rule — [ADR-0005](adr/0005-phi-minimization-in-outreach.md). Portal notice pattern instead: "you have a message," link to the portal |
| **Self-serve signup** | Agencies onboard with a BAA and a conversation. There is no world where this has a "sign up free" button |
| **Multi-tenancy (for now)** | Single agency per deployment until the isolation model is designed. See [deployment.md](deployment.md) |

## Success metrics

**No baseline exists.** Nothing is deployed, so every number below is a target to
instrument, not an achievement. The landing page (`home.html`) cites "62% open rate" and
"6 hrs saved per week" — **those are marketing copy with no measurement behind them.**
Do not treat them as product metrics; they were written to sell, not to report.

Once there's a pilot, these are worth instrumenting:

| Metric | Why it matters | Target |
| --- | --- | --- |
| **PHI guard trip rate** | How often the product stops a real mistake. The core value prop, measurable | Track, don't target — a rate of zero means the scanner is broken or nobody's using it |
| **Acknowledged-and-sent-anyway rate** | Every guard has an override. If overrides are routine, the guard is noise and the design is wrong | < 5% of trips |
| **Suppression violations** | Must be zero. Any non-zero is a bug and possibly a complaint | **0** |
| Time to build + send a campaign | The "6 hrs saved" claim, actually measured | Baseline first |
| Open rate by contact type | Whether outreach lands | Baseline first |
| Bounce / complaint rate | Sender reputation; too high and the ESP drops you | < 2% bounce, < 0.1% complaint |

The second metric is the one to watch. A guard everyone clicks through isn't a guard —
it's a speed bump that teaches people to click through guards. If the
acknowledge-and-send-anyway rate is high in the pilot, the answer is to make the safe
path easier, not to make the warning louder.

## Related

- [project-status.md](project-status.md) — real state and backlog
- [security-compliance.md](security-compliance.md) — the rules behind the guards
- [data-model.md](data-model.md) — how these entities are structured
- [ADR-0005](adr/0005-phi-minimization-in-outreach.md) — the PHI-in-email boundary
