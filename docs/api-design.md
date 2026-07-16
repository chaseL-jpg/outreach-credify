# API design

> **Audience:** engineers building the backend. **Purpose:** the routes the client already
> calls, and the gate every one of them must pass through.

> ## ⚠ No server implements any of this.
>
> The **route inventory** is verified — extracted mechanically from the 54 `api()` call
> sites in `index.html`. The client genuinely calls these paths today; they just dead-end
> in a stub at `index.html:1898`. Everything else here — the gate, error shapes,
> pagination, idempotency — is **design, not implementation**.

## The seam that already exists

```js
// index.html:1898
async function api(method, path, body) {
  if (typeof console !== "undefined") console.debug("[api stub]", method, path, body || "");
  persistState();
  return { ok: true, method, path, body: body || null };
}
```

This is the most useful line of code in the repo. It's `async`, it takes REST semantics,
and it's called from 54 sites with real paths. The client has already written the server's
specification — the backend's job is to make these calls true.

**When you replace it, delete the `console.debug`.** It prints full request bodies. In a
browser with fake data that's harmless. In Node, with a log aggregator that has no BAA,
it's a breach. See [security-compliance.md](security-compliance.md#logging).

The swap is one line:

```js
async function api(method, path, body) {
  const res = await fetch(`${API_BASE}${path}`, {
    method,
    headers: { "Content-Type": "application/json", ...idempotencyHeader(method) },
    credentials: "include",
    body: body ? JSON.stringify(body) : undefined,
  });
  if (!res.ok) throw await toApiError(res);
  return res.json();
}
```

## Surfaces

| Surface | Audience | Auth | Status |
| --- | --- | --- | --- |
| **Internal JSON API** | The app itself, same origin | Session cookie | ⬜ Designed here |
| **Provider webhooks** | ESP/SMS vendor → us | Signature verification | ⬜ Required for Phase 4 |
| **Public unsubscribe** | Email recipients, unauthenticated | Signed token in the link | ⬜ Legally required |
| **Public preference center** | Recipients | Signed token | ⬜ `state.unsubPage` models it |
| Partner/external API | third parties | — | **Non-goal.** Not building this |

The public unsubscribe endpoint is the one people forget. It is unauthenticated by
necessity — someone who no longer wants your email will not log in to say so. It must be
one click, no auth, no confirmation step, and it must work forever.

## Route inventory (verified)

All 41 distinct routes the client calls today (trailing slashes normalized). Extracted
with:

```bash
# 37 routes from literal calls:
grep -o -E 'api\("(GET|POST|PUT|PATCH|DELETE)","[^"]+"' index.html | sort -u

# + 4 more from two DYNAMIC call sites at index.html:1899-1900, which the
#   grep above misses because the method is a ternary:
#     api(t.id ? "PUT" : "POST", "/templates/"     + (t.id||""), t)
#     api(t.id ? "PUT" : "POST", "/contact-types/" + (t.id||""), t)
#   → POST/PUT /templates and POST/PUT /contact-types
```

Call-site accounting: 57 textual occurrences of `api(` = 1 function definition
(`index.html:1898`) + 2 mentions in comments + **54 real call sites** (52 literal,
2 dynamic).

### Contacts & preferences

| Method | Path | Purpose |
| --- | --- | --- |
| `PUT` | `/contacts/{id}/prefs` | Update per-category consent |
| `POST` | `/contacts/{id}/...` | Contact mutations |
| `PUT` | `/notify-prefs` | Bulk notification prefs |
| `DELETE` | `/notify-prefs/{id}` | Clear a pref |

### Suppression

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/suppressions` | Add email suppression |
| `DELETE` | `/suppressions/{id}` | Remove |
| `POST` | `/sms-suppressions` | Add SMS suppression (STOP) |
| `DELETE` | `/sms-suppressions/{id}` | Remove |
| `PUT` | `/sms-stop-reply` | STOP auto-reply text |
| `PUT` | `/unsub-page` | Unsubscribe page content |

### Templates & signatures

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/templates` · `PUT` `/templates/{id}` | Create/update email template (**dynamic call**, `index.html:1899`) |
| `DELETE` | `/templates/{id}` | Delete email template |
| `POST` | `/sms-templates` · `DELETE` `/sms-templates/{id}` | SMS templates |
| `POST` | `/signatures` · `PUT` `/signatures/{id}` · `DELETE` `/signatures/{id}` | Signatures |
| `POST` | `/categories` · `PUT` `/categories/{id}` · `DELETE` `/categories/{id}` | Categories |
| `POST` | `/contact-types` · `PUT` `/contact-types/{id}` | Create/update contact type (**dynamic call**, `index.html:1900`) |
| `DELETE` | `/contact-types/{id}` | Delete contact type |

### Sending

| Method | Path | Purpose |
| --- | --- | --- |
| **`POST`** | **`/sends`** | **Queue a job. The critical route.** |
| `DELETE` | `/sends/{id}` | Cancel a queued job |

### Segments & triggers

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/segments` · `PUT` `/segments/{id}` · `DELETE` `/segments/{id}` | Saved audiences (**UI orphaned**) |
| `POST` | `/triggers` · `POST` `/triggers/{id}` · `PUT` `/triggers/{id}` · `DELETE` `/triggers/{id}` | Event sends (**UI disabled**) |

### Settings

| Method | Path | Purpose |
| --- | --- | --- |
| `PUT` | `/settings/freqCap` | Frequency cap |
| `PUT` | `/settings/quietHours` | Quiet hours |
| `PUT` | `/settings/businessHours` | Business hours |

### Telemetry

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/audit` | Append audit entry |
| `POST` | `/clicks` | Record click |
| `POST` | `/notifications` | In-app notifications |
| `POST` | `/deliverability/events` | Bounce/complaint |
| `POST` | `/deliverability/auth-check` | SPF/DKIM/DMARC check |
| `POST` | `/deliverability/seed-test` | Inbox placement test |

**`POST /audit` is a design bug.** The client should never be the one deciding what gets
audited — a client that can choose not to call `/audit` can act unaudited. Audit entries
must be written **server-side, inside the same transaction as the action they describe**.
Keep the route for client-observed UI events if you like, but the server must not depend
on it. See [data-model.md](data-model.md).

Two routes are inconsistent with the rest and should be fixed during the port:
`POST /triggers/{id}` (should be `PUT`), and `PUT /notify-prefs` alongside
`PUT /contacts/{id}/prefs` (two paths, one concept — keep the nested one).

## The gate

**Every request passes through the same ordered gate. No exceptions, no opt-outs.**

The failure mode this prevents: someone adds a route, forgets a middleware, and ships an
unauthenticated write. The defense is structural — **the router is only reachable through
the gate.** Don't attach auth per-route where a new route can silently miss it. Mount the
gate above the router and make bypass require deleting code, not forgetting it.

```
  request
     │
     ▼
 ┌─────────────────────────────────────────────────────┐
 │ 1. TRANSPORT   HTTPS only. HSTS. Reject plaintext.  │
 ├─────────────────────────────────────────────────────┤
 │ 2. AUTHN       Valid session? → 401                 │
 │                No session = no route. Allowlist the │
 │                public unsubscribe/webhook paths     │
 │                EXPLICITLY, by exact path.           │
 ├─────────────────────────────────────────────────────┤
 │ 3. AUTHZ       May THIS user do THIS? → 403         │
 │                Deny by default. A route with no     │
 │                policy declared FAILS CLOSED.        │
 ├─────────────────────────────────────────────────────┤
 │ 4. SCHEMA      Validate shape/types → 422           │
 │                Allowlist fields. Reject unknown     │
 │                keys — don't ignore them.            │
 ├─────────────────────────────────────────────────────┤
 │ 5. IDEMPOTENCY Key seen? → replay stored response   │
 ├─────────────────────────────────────────────────────┤
 │ 6. DOMAIN      Guard chain. Server is authority.    │
 ├─────────────────────────────────────────────────────┤
 │ 7. AUDIT       Same transaction as the mutation.    │
 └─────────────────────────────────────────────────────┘
     │
     ▼
  handler
```

Steps 3 and 4 have a shared principle: **fail closed**. An unknown field is a rejection,
not a shrug. A route with no authz policy is a 403, not a 200. The cost of failing closed
is a loud error in development; the cost of failing open is a breach in production.

### `POST /sends` — the guard chain

The client already implements this in `doQueueSend()`. **The server re-runs all of it and
trusts none of it.**

```
POST /sends
Idempotency-Key: <uuid>
{
  "templateId": "...",
  "audienceCriteria": {                 ← CRITERIA, not recipient IDs
    "statusIds": [...], "typeIds": [...],
    "repIds": [...], "repMode": "include",
    "excludeContactIds": [...],
    "connectors": {"status_users":"AND","users_types":"AND"}
  },
  "mode": "scheduled",
  "scheduledAt": "2026-07-23T16:00:00Z",
  "tracking": true,
  "excludeRecent": false,
  "acknowledgments": {
    "phi": false,
    "dischargeSig": null,
    "cap": false
  }
}
```

Server-side, in order:

| # | Guard | Fails with | Mirrors |
| --- | --- | --- | --- |
| 1 | **Resolve audience from criteria against the DB** | 422 | `clusterRecipients()` |
| 2 | Recipients > 0 | 422 `empty_audience` | ✅ |
| 3 | Template exists, readable by user | 404 / 403 | ✅ |
| 4 | Scheduled time present and in the future | 422 `scheduled_in_past` | `isScheduledPast()` |
| 5 | **PHI scan of rendered body** — findings acked? | 422 `phi_unacknowledged` + findings | `phiAckValid()` |
| 6 | **Discharged contacts** — ack sig matches audience? | 422 `discharge_unacknowledged` + ids | `dischargeAckValid()` |
| 7 | **Frequency cap** — violations acked? | 422 `freq_cap_unacknowledged` + contacts | `capAckValid()` |
| 8 | Apply suppression + opt-out; `willSend > 0` | 422 `all_recipients_excluded` | `currentPlan()` |
| 9 | Sender identity resolved from **session** | 401 | `getCurrentUser()` |

Then, in one transaction: insert job → insert recipients → append audit → commit →
enqueue.

**Guard 1 is the whole ballgame.** If the client posts recipient IDs and the server
honors them, then suppression, opt-out, and frequency caps are all client-side
suggestions. The client posts *criteria*; the server resolves them. A recipient list in
the request body should be rejected as an unknown field by step 4 of the gate.

**Guard 6's signature.** The prototype hashes the sorted discharged-contact IDs
(`dischargeAckSig()`) so that changing the audience invalidates the acknowledgment. That
is a genuinely good design — it prevents "ack the warning, then add 200 more people."
Port it exactly, and verify the signature server-side against the server-resolved
audience.

**Guard 5 changes under [ADR-0005](adr/0005-phi-minimization-in-outreach.md).** If that
ADR is accepted, PHI findings in an outbound body become a **hard 422 with no override**,
and the acknowledgment path disappears for email entirely.

### Suppression is absolute

No route, no flag, no role, no acknowledgment may send to a suppressed address. Not
"unlikely to" — *cannot*. The `UNIQUE (channel, address)` constraint plus a
pre-send check inside the send transaction is the enforcement. If a code path exists that
could send to a suppressed address, that path is a P1 bug regardless of whether anyone
has hit it.

## Naming

| Rule | Example |
| --- | --- |
| Plural nouns, kebab-case | `/sms-templates`, not `/smsTemplate` |
| Nested resources for owned data | `/contacts/{id}/prefs` |
| Verbs only for non-CRUD actions | `/deliverability/auth-check` |
| Standard verbs: `GET` read · `POST` create · `PUT` replace · `PATCH` partial · `DELETE` remove | |
| `camelCase` JSON bodies | matches the client's existing state |

The existing routes are mostly consistent with this — a good sign about the seam's
quality. Fix the two exceptions noted above during the port, not after.

## Errors

One shape, every time. Never leak stack traces, SQL, or PHI.

```json
{
  "error": {
    "code": "phi_unacknowledged",
    "message": "Template contains PHI that must be acknowledged before sending.",
    "details": {
      "findings": [
        {"kind": "tag",     "id": "{{clinical.primary_dx}}", "label": "Primary diagnosis"},
        {"kind": "pattern", "id": "ssn",                     "label": "SSN / Tax ID"}
      ]
    },
    "requestId": "req_01H..."
  }
}
```

| Status | When |
| --- | --- |
| 400 | Malformed request |
| 401 | No/invalid session |
| 403 | Authenticated, not permitted |
| 404 | Not found — **also** for records the user may not see, to avoid leaking existence |
| 409 | Conflict (duplicate suppression) |
| 422 | Well-formed but fails domain rules — **all guard failures** |
| 429 | Rate limited |
| 500 | Bug. Log with `requestId`, return nothing else |

`details.findings` mirrors `scanPhiText()`'s return shape (`{kind, id, label}`) so the
existing client rendering works unchanged.

**Error messages may name a field but never its value.** `"SSN detected in body"` is
fine; `"SSN 123-45-6789 detected"` puts PHI in an error response, which lands in logs and
error trackers.

## Pagination

Cursor-based for anything unbounded. The prototype has `state.contactsPage`, implying
offset paging — that's fine for hundreds of contacts and wrong for `audit_log`, which is
append-only and grows forever. Offset pagination over a growing table both drifts and
degrades.

```
GET /contacts?limit=50&cursor=eyJpZCI6...
{ "data": [...], "nextCursor": "eyJpZCI6...", "hasMore": true }
```

`limit` default 50, max 200. `audit_log` and `delivery_events` are cursor-only.

## Idempotency

Required on `POST /sends`. Everything else is optional but encouraged.

The scenario: operator clicks Send, the response is slow, they click again. Without an
idempotency key, 200 referral partners get two emails and the frequency cap didn't stop
it because both requests raced past the check.

```
Idempotency-Key: <client-generated uuid v4>
```

Stored on `jobs.idempotency_key UNIQUE`. Replaying a key returns the original response
with the original status. Keys live 24h. The client generates one per send *attempt* —
regenerated when the audience or template changes, so a genuine second send isn't
swallowed.

## Versioning

Not yet. No external consumers, client and server ship together — a version prefix today
is ceremony with no beneficiary.

**When it's needed** (first external consumer, or the unsubscribe links outlive a
breaking change): prefix `/v1`. Note that unsubscribe URLs live in inboxes forever, so
that endpoint's contract is effectively permanent from the first send. Design it once,
carefully, and never break it.

## Related

- [architecture.md](architecture.md#request-lifecycle--target) — the lifecycle
- [data-model.md](data-model.md) — the tables
- [security-compliance.md](security-compliance.md) — authn/authz, logging
- [ADR-0004](adr/0004-server-side-enforcement-of-send-guards.md) — why the server re-runs everything
