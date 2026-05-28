# Reschedule Appointment — API Contract

## Endpoint

```
PATCH /api/appointment/:id/reschedule
Authorization: Bearer <jwt>
```

## Intent

Reschedule changes the **time and slot** of the **same existing appointment** before the visit starts. It is **not** a cancel-and-rebook: the same `appointmentId`, same `visitId`, same doctor, same payment category, and all financial fields are preserved exactly as-is.

---

## Request Body

```json
{
  "appointmentDate": "2025-06-15T09:00:00+07:00",
  "timeSlotId": "64b000000000000000000004",
  "reason": "Patient requested earlier slot"
}
```

| Field             | Type   | Required | Notes                                                         |
|-------------------|--------|----------|---------------------------------------------------------------|
| `appointmentDate` | string | Yes      | ISO 8601 with timezone (`Z` or `+/-HH:mm`). Per AGENTS.md datetime rules — stored as UTC epoch ms. |
| `timeSlotId`      | string | Yes      | MongoDB ObjectId of the new `TimeSlotLog` document.           |
| `reason`          | string | No       | Human-readable reschedule reason; written to audit log.       |

### Fields NOT accepted in this request

The following fields are managed by the booking / billing / payment flows and **must not be sent** in a reschedule request. Any such fields in the body will be rejected by the validation pipe (`forbidNonWhitelisted: true`).

```
hospitalName, doctor, serviceType, visitType,
paymentCategory, depositAmount, depositPaidAmount, depositStatus,
depositPaymentId, depositPaidAt, amount, consultationFee,
paymentAmount, paymentMethod, coinDiscountAmount, coinsToUse, useCoin,
bookingDate, reasonForAppointment, paidAt,
paymentResponseCode, paymentTransactionStatus
```

---

## Allow Conditions

Reschedule is permitted only when **all** of the following are true:

1. `appointmentStatus` is `PENDING` or `CONFIRMED`.
2. Related `Visit` exists and `Visit.status === CREATED` (visit has not started).
3. No `MedicalEncounter` exists for the visit.
4. No `Billing` exists for the visit.
5. No `Payment` linked to a billing exists.
6. New `appointmentDate` is in the future (UTC).
7. New `timeSlotId` slot is available for the same doctor.

---

## Block Conditions and Reason Codes

| Condition | `blockedReason` |
|---|---|
| `appointmentStatus` is CANCELLED / COMPLETED / FAILED / RESCHEDULED | `APPOINTMENT_NOT_RESCHEDULABLE` |
| No Visit found for appointment | `APPOINTMENT_NOT_RESCHEDULABLE` |
| `Visit.status` is `CHECKED_IN` or `IN_PROGRESS` | `VISIT_ALREADY_STARTED` |
| `Visit.status` is `COMPLETED` or `CANCELLED` | `VISIT_COMPLETED` |
| `MedicalEncounter` exists for the visit | `MEDICAL_ENCOUNTER_EXISTS` |
| `Billing` exists for the visit (no payment yet) | `BILLING_EXISTS` |
| `Payment` linked to the billing exists | `PAYMENT_EXISTS` |
| New schedule is in the past | `INVALID_SCHEDULE` |
| New slot is taken (Redis lock or DB conflict) | `SLOT_UNAVAILABLE` |

---

## Response — Success

```json
{
  "code": "SUCCESS",
  "message": "Appointment rescheduled successfully",
  "data": {
    "appointmentId": "64b000000000000000000001",
    "appointmentDate": "2025-06-15T02:00:00.000Z",
    "scheduledAt": 1750039200000,
    "startTime": 1750039200000,
    "endTime": 1750041000000,
    "bookingDate": 1749000000000,
    "appointmentStatus": "CONFIRMED",
    "reason": "Patient requested earlier slot"
  }
}
```

> **`appointmentStatus` is always `CONFIRMED` (or `PENDING`) after a successful reschedule — it is never set to `RESCHEDULED`.**

---

## Response — Blocked

```json
{
  "code": "ERROR",
  "message": "Visit has already started (status: CHECKED_IN); reschedule not allowed",
  "data": {
    "blockedReason": "VISIT_ALREADY_STARTED"
  }
}
```

---

## Financial Preservation Guarantee

A reschedule **never** alters financial state. The following fields on the `Appointment` document are **read-only** for this operation:

| Field | Preserved |
|---|---|
| `paymentCategory` | ✅ |
| `depositAmount` | ✅ |
| `depositPaidAmount` | ✅ |
| `depositStatus` | ✅ |
| `depositPaymentId` | ✅ |
| `depositPaidAt` | ✅ |
| `consultationFee` | ✅ |
| `coinDiscountAmount` | ✅ |
| `paymentAmount` | ✅ |
| `paymentMethod` | ✅ |
| `paidAt` | ✅ |
| `paymentResponseCode` | ✅ |
| `paymentTransactionStatus` | ✅ |
| `bookingDate` | ✅ |

No refund, no credit wallet update, no coin recalculation, no new deposit payment.

---

## Fields Updated by Reschedule

Only the schedule snapshot fields are updated inside a MongoDB transaction:

| Field | Updated |
|---|---|
| `scheduledAt` | New UTC epoch ms |
| `date` | Same as `scheduledAt` (legacy compat) |
| `startTime` | New slot start UTC epoch ms |
| `endTime` | New slot end UTC epoch ms |
| `timeSlot` | New `TimeSlotLog` ObjectId |

---

## Slot Consistency

- Redis lock is acquired on the new slot **before** the transaction opens.
- Old slot is released and new slot is booked atomically inside the same MongoDB transaction.
- Old slot is **not released** if it is the same slot as the new slot.
- Redis lock is released in the `finally` block, regardless of transaction outcome.
- Duplicate key errors (MongoDB index `doctorId + date + timeSlot` for PENDING/CONFIRMED) are mapped to `SLOT_UNAVAILABLE`.

---

## Event Emitted

On success, `appointment.rescheduled` is emitted:

```typescript
{
  appointmentId: string;
  patientEmail: string;
  doctorId: string;
  oldScheduledAt: number;        // epoch ms
  newScheduledAt: number;        // epoch ms
  newStartTime: number;          // epoch ms
  newEndTime: number;            // epoch ms
  oldTimeSlotId: string;
  newTimeSlotId: string;
  reason?: string;
  rescheduledBy?: string;        // email or accountId from JWT
  rescheduledAt: number;         // epoch ms
}
```

`appointment.booking.success` is **not** re-emitted — this prevents creating a new Visit or triggering deposit/payment flows.

---

## What Reschedule Does NOT Do

- Does not cancel the appointment.
- Does not create a new appointment.
- Does not create a new Visit.
- Does not trigger any deposit or payment event.
- Does not change `paymentCategory`, `depositStatus`, or any financial field.
- Does not set `appointmentStatus = RESCHEDULED`.
