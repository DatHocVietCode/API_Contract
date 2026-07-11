# Admin Patient List (PII-scoped)

Paginated patient list for the admin management screen.

> **Breaking change (req-5 PII scoping).** The admin panel now sees **only** a
> patient's name, active/inactive account status, and an opaque `accountId` handle
> for row actions. All contact and clinical data was removed. See the old→new shape
> below — the FE must stop reading the removed fields.

## `GET /patients/admin/`

### Query params (all optional)

| Param | Type | Notes |
|---|---|---|
| `page` | int ≥ 1 | default 1 |
| `limit` | int ≥ 1 | default 10 |
| `keyword` | string | matches patient profile name/phone/email or account email (server-side search input; matched fields are **not** returned) |

### Response

```jsonc
{
  "code": 200,
  "message": "Get patients successfully",
  "data": [
    {
      "name": "Nguyễn B",       // Profile.name (null if unresolved)
      "status": "ACTIVE",        // Account.status — ACTIVE | INACTIVE (null if unresolved)
      "accountId": "string"      // handle for the status toggle (PATCH /users/:id/status); null if unresolved
    }
  ],
  "pagination": { "total": 42, "page": 1, "limit": 10, "totalPages": 5 }
}
```

### Old → new (breaking)

Each item previously returned the full patient record; it now returns only `{ name, status, accountId }`.

**Removed** from each item: `profileId` object (`phone`, `email`, `gender`, `dob`, `avatarUrl`),
`accountId` object's `email`/`role`, and the patient document's `height`, `weight`,
`bloodType`, `medicalProfileId`, and embedded `medicalRecord`.

**Kept / added:** `name` (from the profile), `status` (from the account), `accountId`
(the account `_id`, used as the row-action handle).

### Notes

- `status` is the active/inactive flag toggled via `PATCH /users/:id/status` (account-keyed),
  which is why `accountId` is the exposed handle rather than the patient `_id`.
- Fields resolve to `null` when the underlying profile/account link is missing (a known
  `Appointment`/patient data-integrity caveat); the endpoint never errors on unresolved links.
