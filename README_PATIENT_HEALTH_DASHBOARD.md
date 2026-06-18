# Patient Health Dashboard — Vital Signs & Health Summary (FE/BE Contract)

## 0. Context

The patient "General Health" tab is now a **read-only health dashboard**. It displays
clinical vital-sign measurements taken by a receptionist at/after check-in — **not**
patient self-entered lifestyle data.

- **Read** (patient-facing): `GET /patients/me/health-summary`
- **Write** (receptionist-facing): `POST /receptionist/visits/:visitId/vital-signs`

> All paths below are relative to the API root. The server mounts everything under the
> global `/api` prefix, so the real URLs are `/api/patients/me/health-summary` etc.

All timestamps in this contract are **epoch milliseconds** (UTC). The backend never returns
ISO strings for this feature.

The backend **owns all clinical classification** (BMI, statuses, overall status) and all
thresholds. The FE must render backend-provided statuses and must not compute thresholds or
BMI itself.

---

## 1. Read — Patient health summary

```http
GET /patients/me/health-summary?limit=10
Authorization: Bearer <patient JWT>
```

- Role: **PATIENT only**. The patient is resolved from the JWT account; a patient can never
  read another patient's summary (no id is accepted from the client).
- `limit`: optional. Default `10`, maximum `50` (values above are clamped, invalid/≤0 fall
  back to the default). Returns only `ACTIVE` records.

### Response envelope: `DataResponse<PatientHealthSummaryDto>`

Empty (account has a patient profile but no measurements):

```json
{
  "code": "SUCCESS",
  "message": "Fetched patient health summary successfully",
  "data": {
    "patientId": "patient-id",
    "latest": null,
    "history": [],
    "overallStatus": "UNEVALUATED",
    "generatedAt": 1781740800000
  }
}
```

- `history` contains only `ACTIVE` records, ordered by `measuredAt` desc, then `createdAt` desc.
- `latest === history[0]` when history is non-empty; otherwise `latest = null` and
  `overallStatus = "UNEVALUATED"`.
- `SUPERSEDED` / `VOIDED` records are never returned here. A full audit history will be a
  separate paginated endpoint in the future.

### Error semantics

| Case | HTTP | Body |
| --- | --- | --- |
| Patient has no measurements | `200` | empty summary above (NOT a 404) |
| Authenticated account has **no patient profile** | `404` | `{ "code": "PATIENT_NOT_FOUND", "message": "...", "data": null }` |
| Endpoint not deployed / unknown route | `404` (Nest default shape, no `code`) | FE-only concern — see below |

- `PATIENT_NOT_FOUND` is a stable, backend-emitted code in the standard envelope.
- `ROUTE_NOT_FOUND` is **not** emitted by the backend (an unmatched route never reaches this
  handler). It is reserved for the FE mock-fallback layer: a default-shaped Nest 404
  (`Cannot GET ...`, no `code` field) may be treated as "endpoint not deployed" **only when a
  mock-fallback flag is enabled**. Once this endpoint is deployed, `PATIENT_NOT_FOUND` must
  never trigger mock fallback.

---

## 2. Write — Record a vital sign

```http
POST /receptionist/visits/:visitId/vital-signs
Authorization: Bearer <receptionist JWT>
```

- Role: **RECEPTIONIST only**.
- Allowed when the visit is `CHECKED_IN` or `IN_PROGRESS`. Rejected for `CREATED`
  (patient not arrived), `COMPLETED`, `CANCELLED`.
- **Check-in does not require vital signs.** This endpoint is optional and append-only; it can
  be called any time the visit is active.

### Request: `CreatePatientVitalSignRequestDto`

```ts
export interface CreatePatientVitalSignRequestDto {
  heightCm?: number;
  weightKg?: number;
  bloodPressureSystolic?: number;
  bloodPressureDiastolic?: number;
  heartRateBpm?: number;
  bloodType?: string;     // A | B | AB | O  (no Rh factor in MVP)
  measuredAt?: number;    // epoch ms; defaults to server time when omitted
  note?: string;
}
```

The FE **must not** send `patientId`, `appointmentId`, `visitId`, `measuredBy`, `source`,
`status`, `recordState`, `bmi`, `createdAt`, or `updatedAt`. These are server-derived; sending
any of them is rejected (HTTP 400) by request validation.

### Server-derived on create

- `patientId`, `appointmentId`, `visitId` — from the visit.
- `measuredBy` — `{ id: accountId, role: "RECEPTIONIST", name? }` snapshotted from the JWT.
- `source` — `"RECEPTIONIST_CHECK_IN"` (MVP).
- `bmi`, `status`, `recordState` (`ACTIVE`), `createdAt`, `updatedAt`.

### Response: `DataResponse<CreatePatientVitalSignResponseDto>`

```ts
export interface CreatePatientVitalSignResponseDto {
  vitalSign: PatientVitalSignRecordDto;
}
```

### Validation (HTTP 400 on violation)

- **At least one real measurement** is required: `heightCm`, `weightKg`, a **complete** blood
  pressure pair, or `heartRateBpm`. `bloodType` alone is **not** a valid record.
- **Blood pressure is atomic**: send systolic and diastolic both, or neither. `systolic > diastolic`.
- Positive values within physiological bounds: height `30–300`, weight `1–500`, systolic
  `50–300`, diastolic `30–200`, heart rate `20–300`. Blood pressure and heart rate are integers.
- `measuredAt`: rejected if more than 5 minutes in the future, or earlier than
  `appointment.scheduledAt - 6h`.
- Omitted metrics stay **absent** — they are never coerced to `0` or `UNKNOWN`.

---

## 3. DTOs

```ts
export type HealthMetricStatus  = "NORMAL" | "LOW" | "HIGH" | "UNKNOWN";
export type OverallHealthStatus = "STABLE" | "NEEDS_ATTENTION" | "UNEVALUATED";
export type VitalSignRecordState = "ACTIVE" | "SUPERSEDED" | "VOIDED";
export type VitalSignSource = "RECEPTIONIST_CHECK_IN" | "VISIT_INTAKE" | "MIGRATED" | "UNKNOWN";

export interface PatientHealthSummaryDto {
  patientId: string;
  latest: PatientVitalSignRecordDto | null;
  history: PatientVitalSignRecordDto[];
  overallStatus: OverallHealthStatus;
  generatedAt: number;
}

export interface PatientVitalSignRecordDto {
  id: string;
  patientId: string;
  appointmentId?: string;
  visitId?: string;

  bloodType?: string;
  heightCm?: number;
  weightKg?: number;
  bmi?: number;

  bloodPressureSystolic?: number;
  bloodPressureDiastolic?: number;
  heartRateBpm?: number;

  status?: {
    bmi?: HealthMetricStatus;
    bloodPressure?: HealthMetricStatus;
    heartRate?: HealthMetricStatus;
    weight?: HealthMetricStatus;   // reserved; NOT populated in MVP
  };

  source: VitalSignSource;
  recordState: VitalSignRecordState;

  measuredAt: number;

  measuredBy?: {
    id: string;
    name?: string;
    role: "RECEPTIONIST" | "DOCTOR" | "NURSE" | "SYSTEM";
  };

  note?: string;

  supersedesRecordId?: string;
  correctionReason?: string;
  correctedBy?: { id: string; role: "RECEPTIONIST" | "DOCTOR" | "NURSE" | "SYSTEM" };

  createdAt: number;
  updatedAt?: number;
}
```

---

## 4. Backend-owned classification (FE reference only — do NOT reimplement)

These thresholds are owned by the backend and may change without an FE release. Listed so the
FE understands the meaning of each status, not so it can recompute them.

- **BMI** (kg/m², rounded to 1 decimal): `< 18.5` LOW · `18.5–24.9` NORMAL · `≥ 25` HIGH.
  Only derived when both height and weight are present.
- **Blood pressure**: LOW if `systolic < 90` or `diastolic < 60`; HIGH if `systolic ≥ 140` or
  `diastolic ≥ 90`; otherwise NORMAL.
- **Heart rate**: LOW `< 60` · HIGH `> 100` · otherwise NORMAL.
- **`status.weight`** is reserved and **not populated** in MVP — weight alone is not clinically
  classifiable without height. The FE should not render a weight status badge; use BMI status as
  the body-composition indicator. Missing `status.weight` is expected, not malformed.

### Overall status aggregation (from the latest ACTIVE record)

1. No latest ACTIVE record, or no measured classifiable metric → `UNEVALUATED`
2. Any measured classifiable metric is `LOW` or `HIGH` → `NEEDS_ATTENTION`
3. Otherwise any measured classifiable metric is `UNKNOWN`/missing → `UNEVALUATED`
4. Otherwise every measured classifiable metric is `NORMAL` → `STABLE`

An abnormal reading is never masked by an unknown one. Blood type never affects `overallStatus`.

---

## 5. Blood type policy

- `bloodType` is an optional **per-record snapshot**. Receptionists do not need to re-enter it on
  every measurement, and it alone never creates a valid record nor affects `overallStatus`.
- MVP accepts only `A | B | AB | O` (the existing system enum; no Rh factor). The backend never
  returns `A+`/`O-` style values for this feature.
- The summary does **not** expose a single derived blood type. The FE derives the displayed blood
  type from the `bloodType` on ACTIVE records. If active records disagree, the FE must render it as
  unverified ("Chưa xác minh") rather than picking one silently. A dedicated verified
  clinical-profile source may become authoritative later.

---

## 6. Record lifecycle (audit)

Vital signs are **append-only**. New valid records are `ACTIVE`. Corrections (future endpoint)
create a new `ACTIVE` record, mark the old one `SUPERSEDED` with `supersedesRecordId` +
`correctionReason`, and never edit original values; invalid records may be `VOIDED` (value
preserved). The summary endpoint only ever returns `ACTIVE` records.

Deferred (not in this MVP), reserved shape:

```http
POST  /receptionist/vital-signs/:recordId/corrections
PATCH /receptionist/vital-signs/:recordId/void
```
