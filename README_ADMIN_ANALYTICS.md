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
