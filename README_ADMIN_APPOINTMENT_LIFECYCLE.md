# Admin Appointment Lifecycle (read-only)

Read-only ADMIN surface that reconstructs an appointment's full lifecycle as a
phase-grouped **timeline-as-tree**, plus a filtered admin appointment list.

> **Reconstruction is domain-first.** There is no central audit log. Lifecycle
> "events" are **derived** from live domain collections (Appointment, Payment,
> AppointmentAssignmentTask, Visit, MedicalEncounter, Billing, TimeSlotLog,
> Credit/CoinTransaction, Notification) + their timestamps + the assignment-task
> `history[]`. The tree is intentionally fault-tolerant: missing/legacy/weak data
> renders best-effort with warnings and **never** fails the whole response.

## Access

All endpoints require:

- `Authorization: Bearer <jwt>`
- Role **ADMIN** only (`JwtAuthGuard` + `RoleGuard` + `@Roles(ADMIN)`).

Responses use the standard envelope: `{ code, message, data }` where `code` is
`SUCCESS | ERROR | NOT_FOUND | ...`.

---

## 1. `GET /admin/appointments`

Filtered, paginated, **summarized** appointment list for the admin table.

### Query params (all optional)

| Param | Type | Notes |
|---|---|---|
| `page` | int ≥ 1 | default 1 |
| `limit` | int 1–100 | default 20 (clamped) |
| `sort` | string | `bookingDate\|scheduledAt\|updatedAt\|createdAt` + `:asc\|:desc` (default `bookingDate:desc`) |
| `status` | enum | `AppointmentStatus` |
| `paymentCategory` | enum | `BHYT \| DICH_VU` |
| `assignmentStatus` | enum | `NONE \| AWAITING_ASSIGNMENT \| ASSIGNED` |
| `depositStatus` | enum | `NOT_REQUIRED \| PENDING \| PAID \| FAILED \| REFUNDED \| FORFEITED` |
| `doctorId` | ObjectId | ignored if not a valid id |
| `patientEmail` | string | exact match |
| `dateFrom`,`dateTo` | epoch ms | filters `scheduledAt` |

### Response `data`

```jsonc
{
  "items": [
    {
      "appointmentId": "string",
      "patient": { "email": "p@e.com" } | null,
      "doctor": { "id": "string", "name": "string" } | null,
      "appointmentStatus": "CONFIRMED",
      "assignmentStatus": "NONE",
      "depositStatus": "PAID",
      "paymentCategory": "DICH_VU",
      "visitStatus": "COMPLETED" | null,
      "billingStatus": "PAID" | null,
      "bookingDate": 1718000000000 | null,
      "scheduledAt": 1718600000000 | null,
      "hasWarnings": false
    }
  ],
  "page": 1,
  "limit": 20,
  "total": 137
}
```

---

## 2. `GET /admin/appointments/:id/lifecycle`

Full phase-grouped lifecycle tree for one appointment, with **summarized** node
payloads only (no raw payloads / PII — fetch those lazily via endpoint 3).

- Invalid `:id` or missing appointment → **404** (`NOT_FOUND`).

### Response `data` (LifecycleTree)

```jsonc
{
  "appointment": {
    "id": "string",
    "appointmentStatus": "COMPLETED",
    "assignmentStatus": "NONE",
    "paymentCategory": "DICH_VU",
    "depositStatus": "PAID",
    "scheduledAt": 1718600000000 | null,
    "bookingDate": 1718000000000 | null
  },
  "rootNodeId": "BOOKING:appointments:<id>:created",
  "nodes": [ /* LifecycleNode[] */ ],
  "edges": [ /* LifecycleEdge[] */ ],
  "phases": [ { "phase": "BOOKING", "status": "OK", "nodeCount": 1 } ],
  "globalWarnings": [ /* LifecycleWarning[] */ ],
  "reconstruction": { "strategy": "DOMAIN_FIRST", "generatedAt": 1718600000000, "partial": false }
}
```

### LifecycleNode

```jsonc
{
  "id": "DEPOSIT:payments:<paymentId>:paid",
  "phase": "DEPOSIT",
  "eventType": "DEPOSIT_PAID",
  "label": "Deposit paid",
  "labelKey": "lifecycle.deposit.paid",      // optional, i18n hook
  "timestamp": 1718000500000 | null,
  "timestampConfidence": "EXACT | INFERRED | MISSING",
  "statusBefore": "string?",
  "statusAfter": "PAID",
  "nodeStatus": "OK | PARTIAL | MISSING | LEGACY | CONFLICT | UNKNOWN",
  "actor": { /* ActorSummary */ },
  "sourceCollection": "payments",
  "sourceRecordId": "string | null",
  "parentId": "<rootNodeId> | ''",
  "summary": { /* small safe display fields, no PII */ },
  "warnings": [ /* LifecycleWarning[] */ ],
  "hasDetail": true
}
```

### LifecycleEdge

```jsonc
{ "from": "<rootNodeId>", "to": "<nodeId>", "edgeStatus": "STRONG_LINK | WEAK_LINK | INFERRED | MISSING", "warnings": [] }
```

### ActorSummary

```jsonc
{
  "actorId": "string?",
  "actorName": "string?",
  "actorEmail": "string?",
  "actorRole": "string?",
  "actorType": "USER | SYSTEM | UNKNOWN",
  "actorConfidence": "EXACT | ROLE_INFERRED | SYSTEM_INFERRED | UNKNOWN",
  "actorSource": "STORED_FIELD | DOMAIN_RELATION | EVENT_TYPE_INFERENCE | FALLBACK",
  "actorWarnings": ["REF_UNRESOLVED"]
}
```

### LifecycleWarning

```jsonc
{ "code": "WEAK_BILLING_LINK", "message": "string", "severity": "INFO | WARN | ERROR", "scope": "NODE | EDGE | TREE", "relatedNodeId": "string?" }
```

### Phases (canonical causal order)

`BOOKING → DEPOSIT → ASSIGNMENT → CONFIRMATION → VISIT → ENCOUNTER → BILLING →
PAYMENT → SLOT → COMMUNICATION → CANCELLATION → RESCHEDULE → UNLINKED`

- **SLOT** nodes are INFERRED (no per-appointment slot history) and carry
  `SLOT_HISTORY_UNAVAILABLE`.
- **COMMUNICATION** (notifications) is a separate collapsed branch with WEAK links;
  it must not be treated as part of the main causal chain.
- **Billing** links via Visit only (no `appointmentId`): edge `WEAK_LINK` +
  `WEAK_BILLING_LINK`. A completed visit with no billing → a `MISSING` placeholder.

### FE rendering rules

- Group nodes by `phase` in the order above; order within a phase by `timestamp`
  (nodes with `timestamp: null` sink to the end of their phase).
- Show actor by confidence: `EXACT` → name/email/role; `ROLE_INFERRED` →
  "Role (inferred)"; `SYSTEM_INFERRED` → "System"; `UNKNOWN` → "Unknown actor".
- Render `globalWarnings` as a tree-level banner; per-node `warnings` as chips.
- `reconstruction.partial = true` ⇒ show a "reconstructed from incomplete data" notice.
- Never assume a phase exists; the tree can be useful while incomplete.

### Conflict warnings (tree never fails)

| code | meaning |
|---|---|
| `CONFLICT_MULTIPLE_ACTIVE_TASKS` | >1 active assignment task |
| `CONFLICT_COMPLETED_WITHOUT_VISIT` | appointment COMPLETED but no Visit |
| `CONFLICT_BILLING_PAID_APPOINTMENT_NOT_COMPLETED` | Billing PAID, appointment not COMPLETED |
| `CONFLICT_DEPOSIT_PAID_WITHOUT_PAYMENT` | depositStatus PAID but no deposit Payment row (e.g. TTL-expired) |

---

## 3. `GET /admin/appointments/:id/lifecycle/nodes/:nodeId`

Lazy, **sanitized** per-node detail (loaded when an admin clicks a node).

- Unknown `nodeId` → **404**.
- Reconstructable-but-incomplete node → **200** with `complete: false` + a
  `NODE_DETAIL_INCOMPLETE` warning (never 500).
- `domainSnapshot` is sanitized by default: sensitive patient fields
  (phone/address/identity/insurance/DOB) are `"[redacted]"`; heavy clinical arrays
  (prescriptions/medications/vitalSigns/history…) are summarized to `{ count }`.
  Raw payloads are **not** exposed (deferred debug-only hardening).

### Response `data` (LifecycleNodeDetail)

```jsonc
{
  "nodeId": "VISIT:visits:<visitId>:completed",
  "eventType": "VISIT_COMPLETED",
  "phase": "VISIT",
  "timestamp": 1718600000000 | null,
  "statusBefore": "IN_PROGRESS",
  "statusAfter": "COMPLETED",
  "actor": { /* ActorSummary */ },
  "domainSnapshot": { "status": "COMPLETED", "completedAt": 1718600000000, "phone": "[redacted]" },
  "sourceMeta": { "collection": "visits", "recordId": "<visitId>" },
  "warnings": [],
  "complete": true
}
```

---

## Known limitations (current demo scope)

- Actor is `system/unknown` or role-inferred for events that don't persist an actor
  (booking, deposit callback, check-in, billing finalize, cash mark-paid). Real
  actor capture is deferred.
- `Billing.appointmentId` is not denormalized yet, so Billing/Payment links are WEAK.
- Slot lifecycle and reschedule history are INFERRED (not stored per appointment).
- Notification linkage is best-effort (payload-embedded `appointmentId`).
