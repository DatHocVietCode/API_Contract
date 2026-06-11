# FE Integration Plan — Broad Booking + Redis (Phases 1–7)

This is the frontend integration guide for the broad-booking (doctor-less appointment) work and
its Redis-backed notification/presence/locking changes. It complements:
- `api.md` → **Appointment Assignment Tasks** + **Socket / Realtime** sections (authoritative endpoints, errors, events).
- `README_NOTIFICATION_UNIFIED_SOCKET.md` → notification envelope + type map (now includes the assignment types).

**Backward compatibility:** every change below is additive. Existing flows keep working. The FE
can adopt these incrementally; nothing here is a breaking change.

---

## 0. TL;DR — what the FE must do

| # | Change | FE action | Priority |
|---|--------|-----------|----------|
| 1 | New notification types `ASSIGNMENT_TASK_REMINDER`, `ASSIGNMENT_TASK_EXPIRED` | Add 2 handlers to the `NOTIFICATION_RECEIVED` dispatch map (+ `ASSIGNMENT_TASK_CREATED`, `APPOINTMENT_DOCTOR_ASSIGNED` if not already handled) | **High** |
| 2 | New error `TASK_LOCK_HELD` on `accept` / `assign` | Show a transient "being handled by another receptionist" message + allow retry/refresh | **High** |
| 3 | `online` flag on `ASSIGNMENT_TASK_CREATED` (and reminder/expired) | Optional: informational only (e.g. badge). No logic required | Low |
| 4 | Backend auto-joins the email room on connect | Optional: you may drop the `JOIN_ROOM` emit on `/notification` etc. (keeping it is also fine) | Low |
| 5 | Role-aware presence (`online_role:RECEPTIONIST`) | None — BE-internal; FE never reads it | None |

The **DB assignment-task queue is the source of truth.** Realtime is a best-effort nudge. The
receptionist screen must still work purely from polling `GET /appointment/assignment-tasks`.

---

## 1. Receptionist: assignment-task queue (the core screen)

Endpoints (all `RECEPTIONIST` / `ADMIN`; mutations are `RECEPTIONIST`-only) — see `api.md`:
- `GET /appointment/assignment-tasks?status=PENDING&specialty=&page=&limit=` → queue list.
- `GET /appointment/assignment-tasks/:id` → detail (with `history`).
- `POST /appointment/assignment-tasks/:id/accept` → claim a `PENDING` task.
- `POST /appointment/assignment-tasks/:id/release` → return a claimed task to the pool.
- `POST /appointment/assignment-tasks/:id/assign` → body `{ doctorId, timeSlotId, appointmentDate }`, completes the task.

Recommended UX:
1. Poll `GET …?status=PENDING` (e.g. every 15–30s) **and** subscribe to realtime (section 2) to refresh sooner.
2. On a card: `accept` → load doctors/slots for the specialty → `assign`. `release` returns it to the pool.
3. Always re-fetch the list after any mutation (status may have changed under you).

### Error handling (blocked envelope)
Blocked operations return `{ "code": "ERROR", "message": "...", "data": { "blockedReason": "<CODE>" } }`
(HTTP 400, or 404 for `TASK_NOT_FOUND`). Map `blockedReason`:

| `blockedReason` | Meaning | Suggested FE behavior |
|---|---|---|
| **`TASK_LOCK_HELD`** *(new)* | Another receptionist is processing this task **right now** (short Redis lock). Transient. | Toast "This task is being handled by another receptionist. Try again." → keep the card, refresh the list. |
| `TASK_ALREADY_ACCEPTED` | Someone already claimed it (durable). | Remove/refresh the card; it's no longer available. |
| `TASK_NOT_PENDING` / `TASK_NOT_ASSIGNED` | Task moved on. | Refresh the list. |
| `TASK_NOT_OWNED` | You are not the receptionist who accepted it. | Refresh; only the owner can assign/release. |
| `DEPOSIT_NOT_PAID` | DICH_VU deposit not yet paid. | Block assign; surface "waiting for deposit". |
| `SLOT_UNAVAILABLE` / `SLOT_DOCTOR_MISMATCH` / `INVALID_SCHEDULE` | Slot problem. | Re-pick slot. |
| `APPOINTMENT_NOT_ASSIGNABLE` / `TASK_NOT_FOUND` | Gone/invalid. | Refresh. |

> Distinguish **`TASK_LOCK_HELD`** (transient, retry) from **`TASK_ALREADY_ACCEPTED`** (durable, gone).

---

## 2. Realtime notifications (`/notification`)

Transport is unchanged: connect `/notification`, listen `NOTIFICATION_RECEIVED`, dispatch by
`payload.type` (see `README_NOTIFICATION_UNIFIED_SOCKET.md`). New types to handle:

```ts
// payload.type -> payload.data
ASSIGNMENT_TASK_CREATED   // receptionists: a new broad booking needs a doctor
ASSIGNMENT_TASK_REMINDER  // receptionists: a pending task is near its deadline (data.reminderCount)
ASSIGNMENT_TASK_EXPIRED   // receptionists: a task passed its deadline -> needs manual handling
APPOINTMENT_DOCTOR_ASSIGNED // patient: a doctor/slot was assigned to their broad booking
```

Receptionist app: on any of the three `ASSIGNMENT_TASK_*` events, refresh the queue and/or bump the
bell. Patient app: on `APPOINTMENT_DOCTOR_ASSIGNED`, refresh the appointment + notify the patient.

Notes:
- `data.online` (on the receptionist events) is informational only — do not branch correctness on it.
- All events are also persisted, so the notification bell/list endpoints return them after the fact.
- Reminders are keyed by `reminderCount`, so a task can legitimately produce several reminder notifications over time (not duplicates).

### Connection (simplification, optional)
The backend now auto-joins your email room on connect, so the `JOIN_ROOM` emit is **optional** on
`/appointment`, `/payment/vnpay`, `/patient-profile`, `/notification`. Keep emitting `JOIN_ROOM` if
convenient (idempotent, still acks `ROOM_JOINED`), or drop it. Keep sending `heartbeat` every
~25–30s; heartbeat now also recovers presence after a brief TTL lapse.

---

## 3. Patient: creating a broad (doctor-less) booking

`POST /appointment/book` with `broadBooking: true` and **no** `doctor` / `timeSlotId`. Required:
`patientId`, `patientEmail`, and at least one of `specialty` / `reasonForAppointment` (routing hint),
plus the normal `serviceType` / `paymentMethod` / `paymentCategory`. (`DICH_VU` still requires a
positive `depositAmount` upfront; `BHYT` requires none.)

Response (`code: "PENDING"`): `{ appointmentId, assignmentTaskId, assignmentStatus: "AWAITING_ASSIGNMENT", depositStatus, depositAmount, paymentUrl? }`.
- For `DICH_VU`, redirect to `paymentUrl` (the deposit must be paid before a receptionist can assign).
- Patient UI should show "Waiting for a receptionist to assign a doctor"; the patient is notified via
  `APPOINTMENT_DOCTOR_ASSIGNED` when assignment completes.

(Normal doctor-selected booking is unchanged and still requires `doctor` + `timeSlotId`.)

---

## 4. Suggested rollout order
1. **Queue screen + polling** (works with zero realtime). Map `blockedReason` incl. `TASK_LOCK_HELD`.
2. **Realtime handlers** for the 3 `ASSIGNMENT_TASK_*` types → live queue refresh.
3. **Patient broad-booking** form + `APPOINTMENT_DOCTOR_ASSIGNED` handler.
4. (Optional) drop `JOIN_ROOM`, lean on auto-join.

## 5. Out of scope / known limitations (so the FE doesn't design around them)
- No Redis/RabbitMQ-down recovery and no Outbox — if realtime is missed, **polling is the fallback**.
- Expiry notifies receptionists only (no admin escalation yet).
- These are intentional for the current (thesis) scope; design the receptionist UI to be polling-correct.
