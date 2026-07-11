# Admin Analytics (read-only)

Read-only ADMIN dashboard metrics, one endpoint per metric. Each is a single aggregation
over existing collections (no analytics store).

## Access

All endpoints require:

- `Authorization: Bearer <jwt>`
- Role **ADMIN** only (`JwtAuthGuard` + `RoleGuard` + `@Roles(ADMIN)`).

Responses use the standard envelope `{ code, message, data }` (`code = SUCCESS`).

## Counting unit & time windows

- The countable unit is a **completed appointment** (`Appointment.appointmentStatus === COMPLETED`) — not the `Visit` entity, not a booking.
- **Reqs 1, 2, 4** window on `Appointment.scheduledAt`. Default when **no** `from`/`to` is passed: **last 3 months** (`now − 3 months → now`). Passing `from`/`to` (epoch ms) overrides it; an all-time query is expressed by passing a sufficiently wide range.
- **Req 3 diverges:** it is **all-time by default** and windows on **`Review.createdAt`** only when `from`/`to` are passed. Its `window` is `null` unless a range was supplied. The two axes (`scheduledAt` vs `Review.createdAt`) are not comparable — the FE must not assume a shared window.

## Common query params (all optional)

| Param | Type | Notes |
|---|---|---|
| `from` | int (epoch ms) | window lower bound |
| `to` | int (epoch ms) | window upper bound |
| `limit` | int | top-N size; default **5**, clamped to `[1, 50]` |

---

## 1. `GET /admin/analytics/top-specialties`

Most-examined specialties, keyed on the **treating doctor's** specialty
(`Doctor.chuyenKhoaId`, falling back to `Appointment.specialtyId`). Completed exams whose
specialty cannot be attributed (both null) are excluded from the ranking and counted in
`unattributedCount`.

```jsonc
{
  "window": { "from": 1710000000000, "to": 1717776000000 },
  "items": [ { "specialtyId": "string", "name": "Trung tâm tim mạch", "examCount": 42 } ],
  "unattributedCount": 0
}
```

## 2. `GET /admin/analytics/top-doctors`

Most-examined doctors by examination **volume**. `doctorName` is resolved from the doctor's
linked profile (`Doctor.profileId → Profile.name`).

```jsonc
{
  "window": { "from": 1710000000000, "to": 1717776000000 },
  "items": [ { "doctorId": "string", "doctorName": "BS. Nguyễn Đạt", "examCount": 42 } ]
}
```

## 3. `GET /admin/analytics/top-rated-doctors`

Highest **average** `Review.rating` (1–10 scale), always paired with `reviewCount`. Doctors
with 0 reviews are excluded; ties break by `reviewCount` desc. All-time by default (see above).

```jsonc
{
  "window": null,
  "items": [ { "doctorId": "string", "doctorName": "BS. Nguyễn Đạt", "avgRating": 9.3, "reviewCount": 12 } ]
}
```

## 4. `GET /admin/analytics/frequent-patients`

Most-examined patients by **volume**. PII-compliant by construction — only name + status +
`accountId` handle + count (same rule as the admin patient list). Fields are `null` when the
patient/account link does not resolve (a known data-integrity caveat); the endpoint never errors.

```jsonc
{
  "window": { "from": 1710000000000, "to": 1717776000000 },
  "items": [ { "accountId": "string", "name": "Vo Phan Tan Dat", "status": "ACTIVE", "examCount": 44 } ]
}
```

## 5. `GET /admin/analytics/revenue`

Revenue (consultation + medication), by doctor and by specialty, plus two medication
rankings. **All amounts are plain integer VND — there is no `currency` field and no
decimals; do not treat them as cents.**

**Base population — diverges from reqs 1/2/4:** revenue is computed exclusively from
`Billing` records with **`status = 'PAID'`** (DRAFT/FINALIZED billings are excluded
entirely). Consultation revenue is `Σ Billing.consultationFee`, **not**
`examCount × 100,000` — this is an independent pipeline from `top-doctors`/
`top-specialties` above and may rank doctors differently in principle, even though in
current data the ordering happens to agree.

**Time-window axis — also diverges:** windows on **`Billing.createdAt`** (a real Date
field), not `Appointment.scheduledAt`. `createdAt` reflects when the billing **draft**
was created (at visit completion), not when it was actually paid — there is no separate
paid-at timestamp. Default is still the same **last-3-months** shape as reqs 1/2/4
(`from`/`to` epoch ms override; `limit` applies to `byDoctor`, `bySpecialty`, and both
medication rankings, same default/clamp as the other endpoints).

Medication revenue only counts `medications[]` lines with `source = 'CLINIC'`
(`Σ lineTotal`); `source = 'OUTSIDE_PURCHASE'` lines are never charged
(`lineTotal` is always `0` by design) and are reported separately (below) as a
frequency ranking, not a revenue one.

`doctorsWithRevenueCount` / `specialtiesWithRevenueCount` are the **pre-limit** distinct
counts of doctors/specialties with any PAID revenue — use them to detect when `byDoctor`/
`bySpecialty` have been truncated by `limit`, not the returned array length. Doctors/
specialties with no PAID billing simply do not appear (no zero rows, no error).

```jsonc
{
  "window": { "from": 1710000000000, "to": 1717776000000 },
  "totalConsultationRevenue": 1100000,
  "totalMedicationRevenue": 630000,
  "totalRevenue": 1730000,
  "paidBillingCount": 11,
  "doctorsWithRevenueCount": 2,
  "specialtiesWithRevenueCount": 2,
  "byDoctor": [
    { "doctorId": "string", "doctorName": "BS. Nguyễn Đạt", "consultationRevenue": 1000000, "medicationRevenue": 485000, "revenue": 1485000, "billingCount": 10 }
  ],
  "bySpecialty": [
    { "specialtyId": "string", "name": "Trung tâm tim mạch", "consultationRevenue": 1000000, "medicationRevenue": 485000, "revenue": 1485000, "billingCount": 10 }
  ],
  "topDispensedMedications": [
    { "medicineId": "string|null", "medicineName": "Acemuc 200mg", "totalQty": 25, "lineCount": 4, "revenue": 225000 }
  ],
  "topExternalMedications": [
    { "medicineId": "string|null", "medicineName": "string", "totalQty": 0, "lineCount": 0 }
  ]
}
```

- `topDispensedMedications` / `topExternalMedications` sort by **`totalQty` desc, then
  `lineCount` desc**.
- `topExternalMedications` items **omit `revenue`** (always 0 by design — see above).
- `medicineId` is `null` when a line item has no catalog reference (`Medicine._id`);
  grouping falls back to `medicineName` in that case.
- `doctorName` / specialty `name` resolve the same way as endpoints 1–2 (`Doctor.profileId
  → Profile.name`; `ChuyenKhoa.name`) and are `null` if the link doesn't resolve.
