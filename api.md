# API Contract

Base URL: `/api` (global prefix set in `src/main.ts`)

This document is generated from controller definitions in the codebase. Auth refactor note: many endpoints now derive user identity from JWT (`req.user`) rather than request params/body.

Datetime validation note:
- All datetime request fields must be ISO 8601 with timezone (`Z` or `+/-HH:mm`).
- Datetime values without timezone are rejected with `400 Bad Request`.
- Internal processing normalizes datetimes to UTC and converts to epoch milliseconds.
- Exception: path parameters that represent date-only filters may require `YYYY-MM-DD` by endpoint contract.

## Notification Realtime Contract

- Namespace: `/notification`
- Unified socket event: `NOTIFICATION_RECEIVED`
- Payload: `NotificationPayload` discriminated union (see `README_NOTIFICATION_UNIFIED_SOCKET.md`)
- Transport flow: `RabbitMQ(notification.jobs) -> Notification Consumer -> Handler Registry -> Redis(notification) -> Socket emit`

Typed keys currently supported by `NotificationMap`:
- `COIN_EXPIRY_REMINDER`
- `APPOINTMENT_SUCCESS`
- `APPOINTMENT_CANCELLED`
- `PAYMENT_SUCCESS`

Legacy socket events remain temporarily for backward compatibility but are deprecated for notification bell sync:
- `COIN_EXPIRY_REMINDER`
- `APPOINTMENT_BOOKING_SUCCESS`
- `APPOINTMENT_BOOKING_PENDING`
- `APPOINTMENT_BOOKING_FAILED`
- `APPOINTMENT_CANCELLED`
- `SHIFT_CANCELLED`
- `PAYMENT_UPDATE`
- `PAYMENT_VNPAY_URL_CREATED`

## Auth

### POST /auth/register
Description: Register a new user account.
Auth: Public
Request Body:
- `email`: string
- `password`: string
- `role`: `PATIENT | DOCTOR | RECEPTIONIST` (optional, default `PATIENT`)
- `chuyenKhoaId`: string (optional)
- `degree`: string (optional)
- `yearsOfExperience`: number (optional)
Response (Success):
```json
{ "code": "SUCCESS", "message": "User registered successfully", "data": null }
```
Response (Error):
```json
{ "code": "ERROR", "message": "...", "data": null }
```
Example Request:
```http
POST /api/auth/register
Content-Type: application/json

{ "email": "user@example.com", "password": "secret", "role": "PATIENT" }
```

### POST /auth/login
Description: Login and receive tokens.
Auth: Public
Request Body:
- `email`: string
- `password`: string
Response (Success):
```json
{
  "code": "SUCCESS",
  "message": "Login Successful",
  "data": {
    "accessToken": "...",
    "refreshToken": "...",
    "role": "PATIENT",
    "id": "...",
    "patientId": "...",
    "doctorId": null,
    "profileId": "..."
  }
}
```
Response (Error):
```json
{ "code": "ERROR", "message": "Invalid password", "data": null }
```
Example Request:
```http
POST /api/auth/login
Content-Type: application/json

{ "email": "user@example.com", "password": "secret" }
```

### POST /auth/send-otp
Description: Send OTP to email.
Auth: Public
Request Body:
- `email`: string
Response (Success):
```json
{ "code": "SUCCESS", "message": "Succesfully resent otp!", "data": { "otp": "..." } }
```
Response (Error):
```json
{ "code": "ERROR", "message": "...", "data": null }
```

### POST /auth/verify-otp
Description: Verify OTP.
Auth: Public
Request Body:
- `email`: string
- `otp`: string
Response (Success):
```json
{ "code": "SUCCESS", "message": "OTP verify successfully!", "data": null }
```
Response (Error):
```json
{ "code": "ERROR", "message": "OTP invalid or not exist!", "data": null }
```

### POST /auth/refresh
Description: Refresh access token.
Auth: Public
Request Body:
- `refreshToken`: string
HTTP Status:
- `200`: refresh success
- `401`: refresh token invalid/expired/mismatch
- `500`: unexpected server error
Response (Success):
```json
{ "code": "SUCCESS", "message": "Access token refreshed", "data": { "accessToken": "...", "refreshToken": "..." } }
```
Response (Error):
```json
{ "code": "ERROR", "message": "Invalid refresh token", "data": null }
```

Access token expiry handling for FE:
- Protected endpoints return `401` when access token is invalid/expired.
- FE should call `POST /auth/refresh` with refresh token, update access token, then retry the failed request.

JWT payload notes for FE:
- Access token payload includes `role`.
- Receptionist-only APIs require `role = RECEPTIONIST`.
- If token is valid but role is not allowed, API returns `403 Forbidden`.

## Users / Accounts

### GET /users
Description: List all users.
Auth: Public
Query: none
Response (Success): list of accounts
Response (Error): standard error

### GET /users/:id
Description: Get user by id.
Auth: Public
Params:
- `id`: string
Response (Success): account
Response (Error): not found

### PUT /users/profile
Description: Update current user's profile, optionally upload avatar.
Auth: Required (JWT)
Body: partial `AccountProfileDto` (fields include name, phone, avatarUrl, etc.)
Multipart: `avatar` file (optional)
Response: updated profile

### PUT /users/password
Description: Change current user's password.
Auth: Required (JWT)
Body:
- `currentPassword`: string
- `newPassword`: string
Response: status

### PATCH /users/:id
Description: Update account by id.
Auth: Public
Body: partial `Account`

### DELETE /users/:id
Description: Delete account by id.
Auth: Public

### PATCH /users/:id/status
Description: Update account status.
Auth: Public
Body:
- `status`: enum

## Receptionist

Role authorization:
- All endpoints below require JWT and `role = RECEPTIONIST`.
- Unauthorized responses:
  - `401`: missing/invalid/expired JWT
  - `403`: authenticated but role is not `RECEPTIONIST`

### GET /receptionist/test
Description: Health-check endpoint for receptionist module integration.
Auth: Required (JWT, RECEPTIONIST)
Response:
```json
{ "message": "Receptionist module working" }
```

### GET /receptionist/visits
Description: Fetch today's receptionist visit list for check-in workflow.
Auth: Required (JWT, RECEPTIONIST)
Query: none
Filter rule:
- Uses `appointment.scheduledAt` as the source of truth.
- Returns visits scheduled for the current day in UTC.
Sort:
- ascending by `scheduledAt`
Response (Success):
```json
{
  "code": "SUCCESS",
  "message": "Fetched receptionist visits successfully",
  "data": [
    {
      "visitId": "...",
      "appointmentId": "...",
      "status": "CREATED",
      "scheduledAt": 1776650400000,
      "patientName": "Nguyen Van A",
      "doctorName": "Dr. B",
      "appointmentStatus": "CONFIRMED"
    }
  ]
}
```

### PATCH /receptionist/visits/:visitId/check-in
Description: Mark a visit as checked in.
Auth: Required (JWT, RECEPTIONIST)
Body: empty object `{}`

Validation rules:
- Visit must exist.
- Visit status must be `CREATED`.
- Linked appointment status must be `CONFIRMED`.

Response (Success):
```json
{
  "code": "SUCCESS",
  "message": "Visit checked in successfully",
  "data": {
    "visitId": "...",
    "status": "CHECKED_IN"
  }
}
```

Response (Error):
```json
{
  "statusCode": 409,
  "message": "Visit can only be checked in from CREATED",
  "error": "Conflict"
}
```

## Doctor (Phase 3 Visit-Based Workflow)

Role authorization:
- All endpoints below require JWT and `role = DOCTOR`.
- Unauthorized responses:
  - `401`: missing/invalid/expired JWT
  - `403`: authenticated but role is not `DOCTOR` or doctor does not own the visit

### GET /doctor/visits/today
Description: Fetch today's visits for the authenticated doctor. Uses `Visit` as the primary data source and joins `Appointment` only for display fields (e.g., `scheduledAt`).
Auth: Required (JWT, DOCTOR)
Query: none

Filter rules:
- Only return visits where `status` IN [`CHECKED_IN`, `IN_PROGRESS`].
- Exclude `CREATED` and `COMPLETED` visits.
- Only return visits whose linked `appointment.scheduledAt` falls within the current UTC day.

Response (Success):
```json
{
  "code": "SUCCESS",
  "message": "Fetched today visits for doctor",
  "data": [
    {
      "visitId": "...",
      "appointmentId": "...",
      "status": "CHECKED_IN",
      "scheduledAt": 1776650400000,
      "patientName": "Nguyen Van A",
      "doctorName": "Dr. B",
      "appointmentStatus": "CONFIRMED"
    }
  ]
}
```

### PATCH /doctor/visits/:visitId/start
Description: Mark a visit as started (doctor begins examination).
Auth: Required (JWT, DOCTOR)
Params:
- `visitId`: string
Body: empty

Validation / Behavior:
- Visit must exist.
- Doctor ownership: the `doctorId` on the `Visit` (or linked `Appointment`) must match the authenticated doctor's id.
- Allowed transition: `CHECKED_IN -> IN_PROGRESS` only.

Idempotency:
- If visit already `IN_PROGRESS` → return success (no-op).
- If visit already `COMPLETED` → return `409 Conflict`.

Response (Success):
```json
{
  "code": "SUCCESS",
  "message": "Visit started",
  "data": { "visitId": "...", "status": "IN_PROGRESS" }
}
```

Response (Errors):
- `400 Bad Request` when `visitId` invalid or precondition not met.
- `403 Forbidden` when doctor does not own the visit.
- `404 Not Found` when visit not found.
- `409 Conflict` when visit already completed.

### POST /doctor/visits/:visitId/complete
Description: Complete a visit and persist the medical encounter (diagnosis + prescriptions). This operation is wrapped in a MongoDB transaction.
Auth: Required (JWT, DOCTOR)
Params:
- `visitId`: string
Body (CompleteVisitDto):
- `diagnosis`: string (optional)
- `note`: string (optional)
- `prescriptions`: array of items {
  - `medicineId?`: string | null
  - `medicineName?`: string | null
  - `quantity`: number
  - `note?`: string
}

Validation / Behavior:
1. Visit must exist and `status === IN_PROGRESS`.
2. Doctor ownership: `doctorId` must match the authenticated doctor.
3. The backend MUST reuse the shared `MedicalEncounterService` to persist the encounter and prescriptions (preserve existing behavior such as medicine name fallback).
4. The completion MUST run in a transaction that:
   - calls the shared encounter writer
   - updates `Visit.status` to `COMPLETED` and sets `completedAt` (epoch ms UTC)
   - updates linked appointment/time slot states as needed
5. After successful commit, emit event `domain.visit.completed` with payload `{ visitId, encounterId, completedAt }`.

Idempotency:
- If visit already `COMPLETED` → return `409 Conflict`.

Response (Success):
```json
{
  "code": "SUCCESS",
  "message": "Visit completed",
  "data": { "visitId": "...", "encounterId": "..." }
}
```

Response (Errors):
- `400 Bad Request` when preconditions fail (e.g., visit not IN_PROGRESS).
- `403 Forbidden` when doctor does not own the visit.
- `404 Not Found` when visit not found.
- `409 Conflict` when visit already completed.

Notes for FE integration:
- FE must call the doctor endpoints using the authenticated doctor's token (contains `doctorId`).
- FE should not act on visits with status `CREATED` or `COMPLETED`.
- For completing a visit, FE should POST the diagnosis/prescription bundle and expect `encounterId` in response for navigation to medical records or prescription printing.

### Appointment compatibility
Description: Existing appointment-based completion endpoints are preserved for backward compatibility but act as wrappers only. They MUST:
- resolve `visitId` from `appointmentId` when necessary
- delegate to visit-based completion (`VisitService.completeVisit`)
- NOT perform direct encounter persistence themselves (no duplication of business logic)


### Receptionist — Billing APIs
Purpose: Endpoints for receptionist to view and prepare billing for a completed visit. All endpoints below require JWT and `role = RECEPTIONIST`.

Notes on paths used by the backend:
- Fetch billing snapshot: `GET /receptionist/billing/:visitId` (visit-scoped)
- Mutations operate on a persisted `Billing` resource by id: `PATCH /receptionist/billings/:billingId/*` and `POST /receptionist/billings/:billingId/finalize`.

### Billing wallet summary for finalization
Description: Staff-only billing-scoped wallet summary used by receptionist and admin finalize flows. The lookup is always constrained by `billingId`; clients cannot query an arbitrary patient wallet directly.
Auth: Required (JWT, RECEPTIONIST or ADMIN)
Path: `GET /billing/:billingId/wallet-summary`

Response (Success):
```json
{
  "code": "SUCCESS",
  "message": "Fetched wallet summary successfully",
  "data": {
    "availableCoins": 120000,
    "availableCredit": 450000,
    "maxApplicableDiscount": 30000
  }
}
```

Notes:
- The backend resolves the billing record first, then the linked patient, then fetches a sanitized wallet summary internally.
- `maxApplicableDiscount` is optional and only returned when the billing workflow can compute the current coin discount cap.
- No wallet transaction history, ledger metadata, or reward details are exposed by this endpoint.

### GET /receptionist/billing/:visitId
Description: Fetch billing snapshot for one visit. If a `Billing` document has been created for the visit it will be returned; otherwise the endpoint may return a best-effort snapshot depending on implementation.
Auth: Required (JWT, RECEPTIONIST)
Params:
- `visitId`: string (appointment / visit id)
Response (Success):
```json
{
  "code": "SUCCESS",
  "message": "Fetched billing successfully",
  "data": {
    "billingId": "...",        
    "visitId": "...",
    "status": "DRAFT",

    "consultationFee": 100000,
    "medicationFee": 50000,
    "totalAmount": 150000,

    "insuranceAmount": 45000,
    "depositUsed": 0,

    "creditUsed": 0,
    "coinUsed": 0,

    "finalPayable": 105000,
    "paymentCategory": "BHYT",

    "medications": [
      {
        "medicineId": "...",
        "medicineName": "Paracetamol 500mg",
        "prescribedQty": 10,
        "dispensedQty": 10,
        "unitPrice": 5000,
        "source": "CLINIC",
        "lineTotal": 50000
      }
    ]
  }
}
```
Response (Error):
```json
{
  "statusCode": 404,
  "message": "Visit not found",
  "error": "Not Found"
}
```

### PATCH /receptionist/billings/:billingId/apply-credit
Description: Receptionist applies `credit` amount to an existing billing. This only modifies the billing draft and does NOT deduct credit from the wallet.
Auth: Required (JWT, RECEPTIONIST)
Params:
- `billingId`: string
Request Body:
```json
{ "creditToUse": 50000 }
```
Validation rules:
- `creditToUse` must be a number >= 0
- Billing `status` must be `DRAFT`
- `creditToUse` must be <= remaining payable computed from stored base values (totalAmount, insuranceAmount, depositUsed, coinUsed)
- Patient must have sufficient credit balance (checked, but not deducted)

Response (Success):
```json
{
  "code": "SUCCESS",
  "message": "Credit applied",
  "data": {
    "billingId": "...",
    "creditUsed": 50000,
    "finalPayable": 55000
  }
}
```

### PATCH /receptionist/billings/:billingId/apply-coin
Description: Receptionist applies `coin` amount to an existing billing. This only modifies the billing draft and does NOT deduct coin from the wallet.
Auth: Required (JWT, RECEPTIONIST)
Params:
- `billingId`: string
Request Body:
```json
{ "coinToUse": 20000 }
```
Validation rules:
- `coinToUse` must be a number >= 0
- Billing `status` must be `DRAFT`
- `coinToUse` must be <= remaining payable after credit (computed from stored base values)
- Patient must have sufficient available coin balance (checked, but not deducted)

Response (Success):
```json
{
  "code": "SUCCESS",
  "message": "Coin applied",
  "data": {
    "billingId": "...",
    "coinUsed": 20000,
    "finalPayable": 35000
  }
}
```

### POST /receptionist/billings/:billingId/finalize
Description: Finalize a billing with optional medication fulfillment adjustments. Receptionist confirms actual dispensed quantities and sources before finalization. Finalize locks the billing so pricing and applied discounts cannot be modified.
Auth: Required (JWT, RECEPTIONIST)
Params:
- `billingId`: string
Request Body:
```json
{
  "medications": [
    {
      "medicineId": "...",
      "dispensedQty": 10,
      "source": "CLINIC"
    },
    {
      "medicineId": "...",
      "dispensedQty": 0,
      "source": "CLINIC"
    },
    {
      "medicineId": "...",
      "dispensedQty": 5,
      "source": "OUTSIDE_PURCHASE"
    }
  ]
}
```
Validation rules:
- Billing must exist
- Billing `status` must be `DRAFT` to perform finalize (otherwise `BadRequest` unless already `FINALIZED`)
- Each medication in fulfillment:
  - `dispensedQty` must be >= 0
  - `source` must be one of: `CLINIC`, `OUTSIDE_PURCHASE`
- `finalPayable` computed after fulfillment must be >= 0

Behavior:
- Applies fulfillment changes to medications[] in billing:
  - Updates `dispensedQty` and `source` for each medication matched by `medicineId`
  - Recalculates `lineTotal` per medication:
    - If `source = OUTSIDE_PURCHASE`: `lineTotal = 0` (patient paid outside, no clinic charge)
    - If `dispensedQty = 0`: `lineTotal = 0` (not dispensed)
    - Otherwise: `lineTotal = dispensedQty * unitPrice` (from price snapshot)
- Recomputes `medicationFee` as sum of all `lineTotal` values
- Recomputes `totalAmount = consultationFee + medicationFee`
- Recomputes `insuranceAmount` and `finalPayable` based on new `totalAmount`
- Sets `billing.status = FINALIZED` (immutable snapshot)
- Creates a payment record and prepares for payment flow
- Idempotent: if already `FINALIZED`, returns success (no-op)
- After `FINALIZED` the following are NOT allowed to change via receptionist endpoints: `creditUsed`, `coinUsed`, `medications[]`, or any pricing fields

Response (Success):
```json
{
  "code": "SUCCESS",
  "message": "Billing finalized",
  "data": {
    "billingId": "...",
    "status": "FINALIZED",
    "paymentId": "...",
    "paymentStatus": "PENDING",
    "amount": 85000,
    "method": "QR"
  }
}
```

### DTOs (contract)
Define the request/response shapes used by FE. Field names are camelCase and monetary values are numbers. Do NOT include internal Mongo fields like `_id` or `__v`.

- `BillingResponseDto` (used by `GET /receptionist/billing/:visitId`)
```ts
interface BillingResponseDto {
  billingId: string;
  visitId: string;
  status: 'DRAFT' | 'FINALIZED' | 'PAID';

  consultationFee: number;
  medicationFee: number;
  totalAmount: number;

  insuranceAmount: number;
  depositUsed: number;

  creditUsed: number;
  coinUsed: number;

  finalPayable: number;
  paymentCategory: 'BHYT' | 'DICH_VU' | null;
  medications: BillingMedicationDto[];
}

interface BillingMedicationDto {
  medicineId: string | null;
  medicineName: string;
  prescribedQty: number;
  dispensedQty: number;
  unitPrice: number;
  source: 'CLINIC' | 'OUTSIDE_PURCHASE';
  lineTotal: number;
}
```

Deposit semantics:
- `depositUsed` is derived only from a verified appointment deposit.
- Backend uses `Appointment.depositPaidAmount` only when `Appointment.depositStatus = PAID`.
- `Appointment.depositAmount` is only the required/intended deposit and must not be treated as collected money by FE.
- If deposit is unpaid, failed, or not required, billing draft uses `depositUsed = 0`.

- `WalletSummaryDto` (used by `GET /billing/:billingId/wallet-summary`)
```ts
interface WalletSummaryDto {
  availableCoins: number;
  availableCredit: number;
  maxApplicableDiscount?: number;
}
```

- `ApplyCreditRequestDto`
```ts
interface ApplyCreditRequestDto {
  creditToUse: number; // >= 0
}
```

- `ApplyCoinRequestDto`
```ts
interface ApplyCoinRequestDto {
  coinToUse: number; // >= 0
}
```

Rules:
- DTO fields MUST match backend exactly
- Use `number` for all monetary values
- `visitId` and `billingId` are strings
- Response wrapper remains `{ code, message, data }` and must be used for all responses above

Backward compatibility:
- These endpoints are additive and do not change existing appointment APIs.


### POST /receptionist/payment/mock
Description: Temporary endpoint to simulate payment success for receptionist workflow.
Auth: Required (JWT, RECEPTIONIST)
Body:
- `visitId`: string (optional)
- `amount`: number (optional)

### GET /receptionist/payments/:billingId/qr
Description: Create or return an existing QR payment URL for a finalized billing. The endpoint will create a single active `Payment` record for the `billingId` if none exists, then return a VNPay URL built with `billingId` as the canonical `txnRef` (no `amount` accepted from FE).
Auth: Required (JWT, RECEPTIONIST)
Params:
- `billingId`: string
Behavior / Validation:
- `billing.status` must be `FINALIZED` before creating a QR payment (server will create the payment record on demand when needed).
- FE MUST NOT send an `amount` — the backend derives `amount` from `billing.finalPayable`.
Response (Success):
```json
{
  "paymentId": "...",
  "paymentUrl": "https://sandbox.vnpayment.vn/....",
  "amount": 100000
}
```

### POST /receptionist/payments/:paymentId/mark-paid
Description: Receptionist marks a payment as paid via CASH. This endpoint is used for offline (cash) settlement and will, on success, mark `Payment.status = SUCCESS` and `Billing.status = PAID` and commit wallet deductions/rewards transactionally.
Auth: Required (JWT, RECEPTIONIST)
Params:
- `paymentId`: string
Behavior:
- Idempotent: repeated calls on an already-committed payment return success without double-deducting wallets.
Response (Success):
```json
{
  "code": "SUCCESS",
  "message": "Cash payment marked paid",
  "data": {
    "paymentId": "...",
    "billingId": "...",
    "status": "SUCCESS",
    "amount": 100000,
    "method": "CASH"
  }
}
```

Behavior:
- If `visitId` is omitted: returns simulated success payload without DB update.
- If `visitId` is provided:
  - resolves the visit from appointment collection
  - sets mock payment metadata (`paidAt`, `paymentResponseCode=MOCK_SUCCESS`, `paymentTransactionStatus=MOCK_COMPLETED`)
  - upgrades status from `PENDING|FAILED -> CONFIRMED` in temporary mock flow

Response (Success without visitId):
```json
{
  "code": "SUCCESS",
  "message": "Mock payment simulated",
  "data": {
    "visitId": null,
    "status": "COMPLETED",
    "amount": 100000,
    "simulated": true
  }
}
```

Response (Success with visitId):
```json
{
  "code": "SUCCESS",
  "message": "Mock payment completed",
  "data": {
    "visitId": "...",
    "status": "CONFIRMED",
    "amount": 90000,
    "paidAt": "2026-04-27T08:00:00.000Z",
    "simulated": true
  }
}
```

## Visits (Phase 2 Additive)

Purpose:
- Introduce `Visit` as a dedicated examination-lifecycle entity.
- Keep appointment booking/payment flow unchanged in this phase.
- One appointment has exactly one visit (`appointmentId` unique index).

Visit schema (Mongo):
- `id`
- `appointmentId` (unique)
- `doctorId`
- `patientId`
- `status`: `CREATED | CHECKED_IN | IN_PROGRESS | COMPLETED`
- `startedAt` (epoch ms UTC)
- `completedAt` (epoch ms UTC)

Creation trigger:
- Visit is created by event listener on `appointment.booking.success`.
- Creation is idempotent by `appointmentId` (duplicate-safe on retries/re-delivery).

Status flow:
- `CREATED -> CHECKED_IN -> IN_PROGRESS -> COMPLETED`

Validation rules:
- Cannot `CHECKED_IN` if linked `appointment.appointmentStatus !== CONFIRMED`.
- Cannot move to `IN_PROGRESS` unless current visit status is `CHECKED_IN`.
- Cannot move to `COMPLETED` unless current visit status is `IN_PROGRESS`.

Timestamp rules:
- `startedAt` is set when visit status transitions to `IN_PROGRESS`.
- `completedAt` is set when visit status transitions to `COMPLETED`.

### PATCH /receptionist/visits/:visitId/check-in
Description: Receptionist check-in operation for a created visit.
Auth: Required (JWT, RECEPTIONIST)
Body: empty object `{}` (reserved for forward compatibility)

Behavior:
- Allowed only for `CREATED -> CHECKED_IN` transition.
- Rejects if the linked appointment is not `CONFIRMED`.

Response (Success):
```json
{
  "code": "SUCCESS",
  "message": "Visit checked in successfully",
  "data": {
    "visitId": "...",
    "status": "CHECKED_IN"
  }
}
```

Response (Error - invalid transition):
```json
{
  "statusCode": 400,
  "message": "Visit can only be CHECKED_IN from CREATED",
  "error": "Bad Request"
}
```

## Appointments

### GET /appointment/completed/doctor
Description: Get completed appointments for the authenticated doctor.
Auth: Required (JWT)
Query:
- `page`: number (default 1)
- `limit`: number (default 10)
- `keyword`: string (optional)
Response (Success): paginated completed appointments
Example Request:
```http
GET /api/appointment/completed/doctor?page=1&limit=10
Authorization: Bearer <token>
```

### GET /appointment/admin
Description: Admin search/list appointments.
Auth: Public
Query: `doctorId`, `patientId`, `appointmentStatus`, `keyword`, `page`, `limit`

### GET /appointment
Description: List all appointments.
Auth: Public

### GET /appointment/patient
Description: Get appointments for authenticated patient.
Auth: Required (JWT)
Query:
- `page`: number
- `limit`: number
Response: `{ code, message, data }`

### POST /appointment/book
Description: Book an appointment. Patient identity is derived from JWT.
Auth: Required (JWT)
Body: `AppointmentBookingRequestDto`
- `hospitalName`: string
- `appointmentDate`: string (required, scheduled date selected by user; ISO 8601 with timezone)
- `bookingDate`: string (optional, booking creation timestamp; ISO 8601 with timezone). If omitted, server uses request processing time.
- `date`: string/date (deprecated, legacy alias for `appointmentDate`)
- `specialty`: string (optional)
- `timeSlotId`: string
- `doctor`: { `id`, `name`, `email` } (doctor id required)
- `serviceType`: enum
- `paymentMethod`: `ONLINE | VNPAY | CREDIT | CASH | OFFLINE` (`COIN` is deprecated)
- `visitType`: `OFFLINE` (optional, defaults to `OFFLINE`)
- `paymentCategory`: `BHYT | DICH_VU` (optional; defaults to `DICH_VU`)
- `depositAmount`: number (required and > 0 for `DICH_VU`; normalized to `0` for `BHYT`)
- `reasonForAppointment`: string (optional)
- `amount`: deprecated optional number. New clients must not send it; backend ignores it for deposit/payment evidence.
- `coinsToUse`: deprecated optional number. Booking-time coin discount is not part of the deposit payment flow.
- `useCoin`: deprecated optional boolean. Billing has its own coin application flow.
Semantics:
- `appointmentDate`: Represents when the medical visit is scheduled to happen.
- `bookingDate`: Represents when the booking request is created/recorded.
- Backward compatibility: `date` is still accepted temporarily and treated as `appointmentDate` when `appointmentDate` is missing.
- `amount` is no longer part of the new booking contract and is not proof of paid money.
- Consultation fee comes from server-side policy/config, not client `amount`.
- Deposit rules:
  - If `paymentCategory = BHYT`, backend normalizes `depositAmount = 0`.
  - If `paymentCategory = DICH_VU`, `depositAmount > 0` is required and is paid through VNPay before confirmation.
  - `depositAmount` is the required/intended deposit.
  - `depositPaidAmount` with `depositStatus=PAID` is the actual deposit evidence used by billing.
Core flow:
- Validate request.
- Acquire Redis slot lock `SET slot:{doctorId}:{timeSlotId} NX EX 300`.
- Pre-check slot in database for same `(doctorId, appointmentDate, timeSlot)` with status in `PENDING|CONFIRMED`.
- Create appointment with `PENDING`, persist deposit fields, and mark timeslot `booked` in a MongoDB transaction.
- `BHYT`: `depositStatus=NOT_REQUIRED`; appointment is confirmed immediately; `appointment.booking.success` is emitted; `Visit(CREATED)` is created.
- `DICH_VU`: `depositStatus=PENDING`; create `Payment(purpose=APPOINTMENT_DEPOSIT, appointmentId=...)`; return VNPay `paymentUrl`; appointment remains `PENDING`; Visit is not created yet.
- DICH_VU deposit success: set `depositStatus=PAID`, `depositPaidAmount`, `depositPaidAt`, `depositPaymentId`; set appointment `CONFIRMED`; emit `appointment.booking.success`; create Visit.
- Deposit failure/expiry: set `depositStatus=FAILED`, appointment `FAILED`, release slot, and do not create Visit.

Request examples:
1. DICH_VU payload requiring deposit payment
```json
{
  "hospitalName": "UTE Clinic",
  "appointmentDate": "2026-04-15T02:00:00Z",
  "timeSlotId": "67f2b1...",
  "doctor": {
    "id": "67f1a0...",
    "name": "Nguyen Van A",
    "email": "doctor@example.com"
  },
  "serviceType": "KHAM_DICH_VU",
  "visitType": "OFFLINE",
  "paymentCategory": "DICH_VU",
  "depositAmount": 100000,
  "paymentMethod": "VNPAY",
  "reasonForAppointment": "Tai kham dinh ky"
}
```

2. BHYT payload without deposit
```json
{
  "hospitalName": "UTE Clinic",
  "appointmentDate": "2026-04-15T02:00:00Z",
  "timeSlotId": "67f2b1...",
  "doctor": {
    "id": "67f1a0...",
    "name": "Nguyen Van A",
    "email": "doctor@example.com"
  },
  "serviceType": "KHAM_BHYT",
  "visitType": "OFFLINE",
  "paymentCategory": "BHYT",
  "paymentMethod": "OFFLINE",
  "reasonForAppointment": "Kham BHYT"
}
```

Response examples:
1. Slot locked by another booking
```json
{ "code": "ERROR", "message": "Slot already booked", "data": null }
```

2. DICH_VU deposit payment required
```json
{
  "code": "PENDING",
  "message": "Appointment created. Complete deposit payment to confirm booking.",
  "data": {
    "appointmentId": "<appointmentId>",
    "depositStatus": "PENDING",
    "depositAmount": 100000,
    "depositPaymentId": "<paymentId>",
    "paymentUrl": "https://...",
    "originalAmount": 150000,
    "discountAmount": 0,
    "finalAmount": 150000
  }
}
```

3. BHYT booking confirmed without deposit payment
```json
{
  "code": "SUCCESS",
  "message": "Booking confirmed (payment deferred - use billing flow)",
  "data": {
    "appointmentId": "<appointmentId>",
    "depositStatus": "NOT_REQUIRED",
    "depositAmount": 0,
    "depositPaidAmount": 0,
    "depositPaidAt": null,
    "originalAmount": 150000,
    "discountAmount": 0,
    "finalAmount": 150000
  }
}
```

4. Deposit/booking failed
```json
{
  "code": "ERROR",
  "message": "...",
  "data": {
    "appointmentId": "<appointmentId>",
    "depositStatus": "FAILED",
    "originalAmount": 150000,
    "discountAmount": 0,
    "finalAmount": 150000
  }
}
```

Notes:
- Redis is used only for race-condition protection (TTL aligned with VNPay expiry).
- A background cleanup marks expired `PENDING` bookings as `FAILED` when VNPay expiry window is reached.
- Source of truth for TTL is `VN_PAY_EXPIRE_MINUTES` (default 15).
- Database is the final gate for consistency (not Redis).
- Database enforces uniqueness for active bookings on `(doctorId, appointmentDate, timeSlot)` where status is `PENDING|CONFIRMED`.
- Duplicate key errors (`11000`) are mapped to `Slot already booked`.
- Deprecated field notice: `date` will be removed in a future version. Use `appointmentDate` instead.
- If both `appointmentDate` and deprecated `date` are provided, `appointmentDate` takes precedence.
- `paymentMethod=COIN` is deprecated and rejected.
- `amount`, `coinsToUse`, and `useCoin` are deprecated for booking. FE should use billing endpoints for credit/coin application.
- Billing uses booking deposit only when `depositStatus=PAID`; `depositAmount` alone is not payment evidence.
- `Payment.purpose` distinguishes `BILLING` from `APPOINTMENT_DEPOSIT`.

### GET /appointment/:appointmentId/deposit-status
Description: Poll the read-only appointment deposit state after a DICH_VU booking opens the VNPay popup.
Auth: Required (JWT)
Authorization:
- The owning patient can query their appointment.
- `ADMIN` and `RECEPTIONIST` staff can query an appointment.
- Other users receive `403`.
Params:
- `appointmentId`: appointment id

Response fields:
- `appointmentId`: string
- `appointmentStatus`: `PENDING | CONFIRMED | FAILED | CANCELLED | COMPLETED | RESCHEDULED`
- `paymentCategory`: `BHYT | DICH_VU`
- `depositStatus`: `NOT_REQUIRED | PENDING | PAID | FAILED | REFUNDED | FORFEITED`
- `depositAmount`: required/intended deposit amount. This is not paid-money evidence.
- `depositPaidAmount`: verified paid deposit amount.
- `depositPaidAt`: epoch milliseconds UTC or `null`.
- `depositPaymentId`: linked appointment deposit payment id or `null`.
- `paymentStatus`: linked payment status `PENDING | SUCCESS | FAILED` or `null`.
- `paymentUrl`: always `null`. This endpoint never creates or refreshes payments.
- `isConfirmed`: `true` when appointment is `CONFIRMED` or `COMPLETED`.
- `isTerminal`: `true` when deposit status is terminal or appointment is `FAILED` / `CANCELLED`.

DICH_VU pending response:
```json
{
  "appointmentId": "<appointmentId>",
  "appointmentStatus": "PENDING",
  "paymentCategory": "DICH_VU",
  "depositStatus": "PENDING",
  "depositAmount": 100000,
  "depositPaidAmount": 0,
  "depositPaidAt": null,
  "depositPaymentId": "<paymentId>",
  "paymentStatus": "PENDING",
  "paymentUrl": null,
  "isConfirmed": false,
  "isTerminal": false
}
```

DICH_VU paid/confirmed response:
```json
{
  "appointmentId": "<appointmentId>",
  "appointmentStatus": "CONFIRMED",
  "paymentCategory": "DICH_VU",
  "depositStatus": "PAID",
  "depositAmount": 100000,
  "depositPaidAmount": 100000,
  "depositPaidAt": 1780200000000,
  "depositPaymentId": "<paymentId>",
  "paymentStatus": "SUCCESS",
  "paymentUrl": null,
  "isConfirmed": true,
  "isTerminal": true
}
```

DICH_VU failed response:
```json
{
  "appointmentId": "<appointmentId>",
  "appointmentStatus": "FAILED",
  "paymentCategory": "DICH_VU",
  "depositStatus": "FAILED",
  "depositAmount": 100000,
  "depositPaidAmount": 0,
  "depositPaidAt": null,
  "depositPaymentId": "<paymentId>",
  "paymentStatus": "FAILED",
  "paymentUrl": null,
  "isConfirmed": false,
  "isTerminal": true
}
```

BHYT not-required response:
```json
{
  "appointmentId": "<appointmentId>",
  "appointmentStatus": "CONFIRMED",
  "paymentCategory": "BHYT",
  "depositStatus": "NOT_REQUIRED",
  "depositAmount": 0,
  "depositPaidAmount": 0,
  "depositPaidAt": null,
  "depositPaymentId": null,
  "paymentStatus": null,
  "paymentUrl": null,
  "isConfirmed": true,
  "isTerminal": true
}
```

Errors:
- `401`: missing, invalid, or expired JWT.
- `403`: authenticated user does not own the appointment and is not allowed staff.
- `404`: appointment not found.

FE guidance:
1. After DICH_VU `POST /appointment/book` returns `paymentUrl`, open the VNPay popup.
2. Poll `GET /appointment/:appointmentId/deposit-status` from the parent page until `isTerminal=true` or `depositStatus=PAID|FAILED`.
3. Show confirmed only when `depositStatus=PAID` and `appointmentStatus=CONFIRMED`.
4. If the popup closes early, keep polling or offer a refresh-status action.
5. Treat `depositAmount` as intended amount only. Use `depositPaidAmount` with `depositStatus=PAID` as verified payment evidence.

### GET /appointment/today
Description: Get today's appointments for authenticated doctor.
Auth: Required (JWT)
Response: `{ code, message, data }`

### PATCH /appointment/complete
Description: Complete an appointment and create encounter.
Auth: Public
Body: `CompleteAppointmentDto`
- `appointmentId`, `diagnosis`, `note` (optional), `prescriptions[]`

### GET /appointment/:id
Description: Get appointment by id.
Auth: Public

### PATCH /appointment/:id/reschedule
Description: Reschedule an appointment using `appointmentDate + timeSlotId` and persist snapshot fields.
Auth: Required (JWT)
Params:
- `id`: appointment id
Body: `AppointmentRescheduleDto`
- `appointmentDate`: string (ISO 8601 with timezone)
- `timeSlotId`: string
- `reason`: string (optional)

Behavior:
- Uses `AppointmentTimeHelper.resolveTimeWindow()` to compute `scheduledAt`, `startTime`, `endTime` from `appointmentDate + timeSlot`.
- Keeps existing `bookingDate` unchanged.
- Uses Redis slot lock (`slot:{doctorId}:{timeSlotId}`) and conflict check that excludes the current appointment.
- Prevents reschedule to past time.

Response (Success):
```json
{
  "code": "SUCCESS",
  "message": "Appointment rescheduled successfully",
  "data": {
    "appointmentId": "...",
    "appointmentDate": "2026-04-20T02:00:00.000Z",
    "scheduledAt": 1776650400000,
    "startTime": 1776650400000,
    "endTime": 1776652200000,
    "bookingDate": 1775000000000,
    "reason": "..."
  }
}
```

### PATCH /appointment/cancel
Description: Cancel appointment.
Auth: Required (JWT)
Body:
- `appointmentId`: string
- `reason`: string (optional)

Visit-based cancellation rules:
- Appointment status must be `PENDING` or `CONFIRMED`.
- Linked `Visit` must exist and have status `CREATED`.
- Cancellation is blocked once the visit is `CHECKED_IN`, `IN_PROGRESS`, `COMPLETED`, or any status other than `CREATED`.
- Cancellation is blocked within 24 hours of the scheduled appointment time.
- Cancellation is blocked if a `MedicalEncounter`, `Billing`, or related `Payment` exists.
- On success, backend sets `Appointment.status = CANCELLED`, `Visit.status = CANCELLED`, and releases the `TimeSlotLog` to `available`.
- Only the owning patient or staff with `ADMIN` / `RECEPTIONIST` role may cancel.

Deposit refund rules:
- `BHYT`: no deposit and no refund.
- `DICH_VU`: refund to `CreditWallet` only when `depositStatus=PAID` and `depositPaidAmount > 0`.
- Refund amount is `floor(depositPaidAmount * APPOINTMENT_CANCEL_REFUND_RATE)`.
- `APPOINTMENT_CANCEL_REFUND_RATE` defaults to `1`, is clamped to `0..1`, and may disable refund with `0`.
- Refund evidence never comes from `amount`, `paymentAmount`, `depositAmount`, `consultationFee`, or `paidAt`.
- The refund ledger references `appointmentId` and is idempotent. Coin wallet is never credited or restored.
- Pending, ambiguous, or inconsistent appointment-deposit payment records block cancellation to avoid callback races.
- Notification, mail, and socket cancellation side effects may run after a successful commit.

Blocked reason codes:
- `APPOINTMENT_NOT_CANCELABLE`
- `VISIT_ALREADY_STARTED`
- `VISIT_COMPLETED`
- `MEDICAL_ENCOUNTER_EXISTS`
- `BILLING_EXISTS`
- `PAYMENT_EXISTS`
- `APPOINTMENT_DEPOSIT_PAYMENT_PENDING`
- `APPOINTMENT_DEPOSIT_PAYMENT_AMBIGUOUS`
- `APPOINTMENT_DEPOSIT_PAYMENT_INCONSISTENT`
- `TIME_SLOT_RELEASE_FAILED`

### PATCH /appointment/:id/confirm
Description: Confirm appointment.
Auth: Public

## Chat

### POST /chat/conversations
Description: Create or fetch a direct conversation. Authenticated user is injected into participants.
Auth: Required (JWT)
Body:
- `participants`: array of `{ accountId, email?, role }`
- `title`: string (optional)
Response (Success):
```json
{
  "code": "SUCCESS",
  "message": "Conversation created",
  "data": {
    "_id": "<conversationId>",
    "type": "direct",
    "participants": [{ "accountId": "...", "email": "...", "role": "..." }],
    "title": null,
    "lastMessage": null,
    "createdAt": "...",
    "updatedAt": "..."
  }
}
```

### GET /chat/conversations
Description: List conversations for authenticated user.
Auth: Required (JWT)
Query: `skip`, `limit`
Response (Success):
```json
{
  "code": "SUCCESS",
  "message": "Fetched conversations",
  "data": {
    "data": [
      {
        "_id": "<conversationId>",
        "participants": [
          {
            "accountId": "...",
            "email": "...",
            "role": "...",
            "displayName": "...",
            "avatarUrl": null
          }
        ],
        "lastMessage": { "content": "...", "senderId": "...", "at": "..." },
        "updatedAt": "..."
      }
    ],
    "total": 1,
    "skip": 0,
    "limit": 20
  }
}
```

### GET /chat/conversations/:id/messages
Description: List messages in a conversation.
Auth: Required (JWT)
Query: `before`, `limit`
Notes:
- Sorted by `createdAt desc, _id desc` for deterministic ordering.
- `limit` is capped at `50`.

### POST /chat/conversations/:id/read
Description: Mark conversation as read for authenticated user.
Auth: Required (JWT)
Body: none

### GET /chat/contacts/search
Description: Search contacts.
Auth: Required (JWT)
Query: `q`, `role`, `limit`

### Realtime Socket Contract (`/chat` namespace)
Connection auth:
- JWT token in `handshake.auth.token`.

Main rooms:
- Personal room: `user:{accountId}`
- Conversation room: `conv:{conversationId}`

Client -> Server events:
- `CHAT_JOIN_USER`: join personal room for notification fanout
- `CHAT_LEAVE_USER`
- `CHAT_JOIN_CONVERSATION`: payload `{ conversationId }`
- `CHAT_LEAVE_CONVERSATION`: payload `{ conversationId }`
- `CHAT_MESSAGE_SEND`: payload `{ conversationId, content, clientMessageId? }`
- `CHAT_MESSAGE_READ`: payload `{ conversationId }`

Server -> Client events:
- `ROOM_JOINED`: payload `{ room? , conversationId? }`
- `CHAT_MESSAGE_RECEIVED`: DataResponse with persisted message payload
- `CHAT_MESSAGE_READ`: payload `{ conversationId, accountId }`
- `CHAT_MESSAGE_DELIVERED`: DataResponse ACK in worker mode (message queued asynchronously)

Queue migration notes (backward-compatible):
- Queue name: `chat.message.created`.
- `CHAT_WRITE_MODE=dual` (default):
  - Gateway writes MongoDB directly (legacy-safe)
  - Gateway also publishes RabbitMQ event for migration telemetry
- `CHAT_WRITE_MODE=worker`:
  - Gateway publishes queue event and returns ACK (`CHAT_MESSAGE_DELIVERED`)
  - Worker persists message, updates conversation snapshot, then publishes realtime event
- `CHAT_REALTIME_MODE=direct` (default): Gateway emits socket directly.
- `CHAT_REALTIME_MODE=redis`: Worker publishes to Redis channel `chat.message`, gateway subscribes and fanouts.

## Patients

### GET /patients/admin/
Description: Admin list patients.
Auth: Public
Query: `page`, `limit`, `keyword`

### GET /patients/me
Description: Get authenticated patient's profile (by JWT email).
Auth: Required (JWT)

### GET /patients/by-account
Description: Get patient record by authenticated account id.
Auth: Required (JWT)

### GET /patients/profile/:id
Description: Get patient profile by patient id.
Auth: Public

### POST /patients/me/medical-profile
Description: Upsert medical profile for authenticated patient.
Auth: Required (JWT)
Body: partial medical profile fields

### POST /patients/me/allergies
Description: Add allergy record for authenticated patient.
Auth: Required (JWT)
Body: partial allergy record fields

### POST /patients/me/medical-history
Description: Add medical history record for authenticated patient.
Auth: Required (JWT)
Body: partial medical history record fields

## Reviews

### GET /reviews/by-appointment-patient
Description: Get review by appointment for authenticated patient.
Auth: Required (JWT)
Query:
- `appointmentId`: string
Response: review or null

### POST /reviews
Description: Create a review as authenticated patient.
Auth: Required (JWT)
Body:
- `doctorId`: string
- `appointmentId`: string
- `rating`: number
- `comment`: string (optional)

### GET /reviews
Description: List reviews.
Auth: Public
Query: `page`, `limit`

### GET /reviews/:id
Description: Get review by id.
Auth: Public

### GET /reviews/doctor/:doctorId
Description: List reviews by doctor.
Auth: Public

### PATCH /reviews/:id
Description: Update review.
Auth: Public
Body: partial review

### DELETE /reviews/:id
Description: Delete review.
Auth: Public

## Shifts

### POST /shift/register
Description: Register a shift for authenticated doctor.
Auth: Required (JWT)
Body: `RegisterShiftRequestDto`
- `startTime`: string (ISO 8601 with timezone)
- `endTime`: string (ISO 8601 with timezone)
- `legacyAllowMissingTimezone`: boolean (optional, temporary backward-compatibility fallback)
- `shift`: `morning | afternoon | extra`

Validation:
- `endTime` must be greater than `startTime`.
- Missing timezone is rejected by default.
- Temporary fallback mode (`legacyAllowMissingTimezone=true`) assumes `Asia/Ho_Chi_Minh` and logs `[TimeWarning]`.

### GET /shift/doctor/:doctorId/month
Description: Get shifts by doctor and month.
Auth: Public
Query:
- `month`: string
- `year`: string
- `status`: string (optional)

### GET /shift/doctor/:doctorId/date/:date
Description: Get shift by doctor and date.
Auth: Public

### DELETE /shift/:id
Description: Delete shift.
Auth: Public

### PUT /shift/cancel/:id
Description: Cancel shift (authorization enforced in service).
Auth: Required (JWT)
Body: `reason`: string

## Wallet

Reference:
- Detailed lifecycle/phase contract for coin expiry reminder: `COIN_EXPIRY_REMINDER_CONTRACT.md`

### GET /wallet/balance
Description: Get wallet balance for authenticated patient.
Auth: Required (JWT)
Response:
```json
{
  "code": "SUCCESS",
  "message": "Fetched wallet balance",
  "data": {
    "balance": 12000,
    "coinBalance": 12000,
    "creditBalance": 250000
  }
}
```

Notes:
- `balance` is kept for backward compatibility and maps to `coinBalance`.

### GET /wallet/details
Description: Get wallet details and history for authenticated patient.
Auth: Required (JWT)
Query: `page`, `limit`
Response:
```json
{
  "code": "SUCCESS",
  "message": "Fetched wallet details successfully",
  "data": {
    "coinBalance": 12000,
    "totalCoinEarned": 50000,
    "totalCoinUsed": 38000,
    "creditBalance": 250000,
    "totalCredited": 350000,
    "totalDebited": 100000,
    "transactions": [],
    "creditTransactions": [],
    "pagination": { "page": 1, "limit": 10, "total": 20, "totalPages": 2 },
    "creditPagination": { "page": 1, "limit": 10, "total": 8, "totalPages": 1 }
  }
}
```

### GET /wallet/coin/summary
Description: Get expiration-aware coin summary with FEFO consumption breakdown (usable/expired/expiring + per-earn allocation).
Auth: Required (JWT)
Response:
```json
{
  "code": "SUCCESS",
  "message": "Fetched coin summary successfully",
  "data": {
    "totalBalance": 12000,
    "usableCoin": 9000,
    "expiredCoin": 3000,
    "expiringSoon": 1500,
    "breakdown": [
      {
        "transactionId": "...",
        "amount": 5000,
        "used": 2000,
        "remaining": 3000,
        "createdAt": 1773196800000,
        "expiresAt": 1776643200000,
        "category": "active",
        "isExpiringSoon": true
      }
    ]
  }
}
```

Notes:
- **FEFO (First Expire, First Out)**: Spend consumes expiring lots first, sorted by `expiresAt ASC`, then non-expiring lots.
- **Partial consumption**: A single spend can consume from multiple earn lots; the `used` field in each lot shows how much was allocated.
- `createdAt` and `expiresAt` are epoch milliseconds in UTC; `expiresAt` is `null` for non-expiring lots.
- `category` can be `active`, `expired`, or `non_expiring` (when `expiresAt` is `null`).
- Expired lots are never allocated for spend; they remain in `expiredCoin` only.
- `expiringSoon` counts remaining coin from active lots that expire within the next 7 days.
- Breakdown order: expired lots first, then spendable lots in FEFO order.

## Notifications

### GET /notifications/by-email
Description: Get paginated notifications of the authenticated user (plus broadcast notifications).
Auth: Required (JWT)
Query:
- `page`: number (default 1)
- `limit`: number (default 10)
Response:
```json
{
  "code": "SUCCESS",
  "message": "Notifications fetched successfully",
  "data": {
    "data": [
      {
        "_id": "...",
        "title": "Thông báo coin sắp hết hạn",
        "message": "Bạn có 10000 coin sắp hết hạn. Vui lòng sử dụng trước khi hết hạn.",
        "isRead": false,
        "receiverEmail": ["patient@example.com"],
        "isBroadcast": false,
        "details": {
          "type": "coin_expiry_reminder",
          "jobId": "...",
          "transactionId": "...",
          "amount": 10000,
          "expiresAt": 1776269699780,
          "runAt": 1776010499780,
          "reminderDays": 3
        },
        "createdAt": 1776010500123,
        "updatedAt": 1776010500123
      }
    ],
    "total": 12,
    "page": 1,
    "limit": 10,
    "totalPages": 2
  }
}
```

Notes:
- `createdAt` and `updatedAt` are epoch milliseconds (UTC).
- For coin expiry reminders, `details.expiresAt` and `details.runAt` are epoch milliseconds (UTC).
- FE should render reminder time from `details.expiresAt` instead of parsing `message`.

### PATCH /notifications/:id/read
Description: Mark a notification as read.
Auth: Required (JWT)
Response: updated notification object with `createdAt` and `updatedAt` in epoch milliseconds.

## Profiles

### GET /profiles/:id
Description: Get profile by id.
Auth: Public

### PATCH /profiles/me
Description: Update authenticated user's profile.
Auth: Required (JWT)
Body: `UpdateProfileDto`

## Doctors

### GET /doctors/active
Description: List active doctors.
Auth: Public

### POST /doctors
Description: Create a doctor with account/profile (multipart supported).
Auth: Public
Body: `profile`, `degree`, `yearsOfExperience`, etc.

### GET /doctors/admin
Description: Admin list doctors.
Auth: Public

### GET /doctors
Description: List doctors.
Auth: Public

### GET /doctors/specialty
Description: Search doctors by specialty.
Auth: Public
Query: `specialtyId`, `keyword`

### GET /doctors/doctor/:doctorId/date/:date
Description: Get doctor timeslots by date.
Auth: Public
Query: `status` (optional)
Params:
- `date`: string (`YYYY-MM-DD`, date-only). This endpoint does not accept full ISO datetime.

### GET /doctors/me
Description: Get doctor by authenticated account id.
Auth: Required (JWT)

### GET /doctors/:id
Description: Get full doctor information by doctor id. Includes populated profile and specialty.
Auth: Public

Params:
- `id`: string (MongoDB ObjectId of the doctor)

Success Response `200`:
```json
{
  "code": "SUCCESS",
  "message": "Fetched doctor successfully",
  "data": {
    "_id": "664a1b2c3d4e5f6789abcdef",
    "profileId": {
      "_id": "664a1b2c3d4e5f6789aaaaaa",
      "name": "Nguyen Van A",
      "email": "doctor@example.com",
      "phone": "0901234567",
      "address": "123 Le Loi, District 1",
      "gender": "male",
      "dob": "1985-03-15T00:00:00.000Z",
      "avatarUrl": "https://res.cloudinary.com/example/image/upload/profiles/avatar.jpg"
    },
    "accountId": "664a1b2c3d4e5f6789bbbbbb",
    "doctorName": "Dr. Nguyen Van A",
    "chuyenKhoaId": {
      "_id": "664a1b2c3d4e5f6789cccccc",
      "name": "Pediatrics",
      "description": "Pediatric specialty",
      "status": true
    },
    "degree": ["General Medicine", "Master of Medicine"],
    "academic": "PhD",
    "bio": "Doctor with over 10 years of experience...",
    "achievements": ["Outstanding Doctor Award 2022"],
    "yearsOfExperience": 10,
    "createdAt": "2024-05-01T00:00:00.000Z",
    "updatedAt": "2024-05-10T00:00:00.000Z"
  }
}
```

Error Responses:
- `404 Not Found`: Doctor id does not exist or is an invalid ObjectId.
  ```json
  { "statusCode": 404, "message": "Doctor not found", "error": "Not Found" }
  ```

### PATCH /doctors/:id
Description: Update doctor (multipart supported).
Auth: Public

## News

### GET /news/public
Description: List public news.
Auth: Public

### POST /news
Description: Create news (multipart image).
Auth: Public

### GET /news
Description: List all news.
Auth: Public

### GET /news/:id
Description: Get news by id.
Auth: Public

### PUT /news/:id
Description: Update news (multipart image).
Auth: Public

### DELETE /news/:id
Description: Delete news.
Auth: Public

## Medicines

### POST /medicines
Description: Create medicine.
Auth: Public

### GET /medicines
Description: List medicines.
Auth: Public
Query: `page`, `limit`, `keyword`, `sort`

### GET /medicines/:id
Description: Get medicine by id.
Auth: Public

### PUT /medicines/:id
Description: Update medicine.
Auth: Public

### DELETE /medicines/:id
Description: Delete medicine.
Auth: Public

## ChuyenKhoa (Specialties)

### GET /chuyenkhoa/admin
Description: Admin list specialties.
Auth: Public
Query: `page`, `limit`, `key`

### GET /chuyenkhoa
Description: List specialties.
Auth: Public

### GET /chuyenkhoa/:id
Description: Get specialty by id.
Auth: Public

### POST /chuyenkhoa
Description: Create specialty.
Auth: Public

### PATCH /chuyenkhoa/:id
Description: Update specialty.
Auth: Public

### DELETE /chuyenkhoa/:id
Description: Delete specialty.
Auth: Public

## Timeslots

### GET /timeslot
Description: List timeslots.
Auth: Public

## Prescription

### POST /prescription/:id/generate-pdf
Description: Generate a prescription PDF file and return a public URL to the generated document.
Auth: Public
Path Params:
- `id`: string (target record / appointment id used to create the output folder)
Body: `CreatePrescriptionPdfDto`
- `diagnosis`: string
- `prescriptions`: array of `{ medicineId?, name, quantity, note? }`
- `note`: string (optional)
- `dateRecord`: string (required, ISO 8601 with timezone; serialized into a Date on the server)
- `patientName`: string (optional)
- `patientAge`: number (optional)
- `doctorName`: string (optional)
Response (Success):
```json
{
  "code": "SUCCESS",
  "message": "PDF generated successfully",
  "data": {
    "url": "http://localhost:3000/prescription/<id>/prescription.pdf"
  }
}
```
Behavior:
- Server renders HTML with Puppeteer, writes the PDF to `public/prescription/<id>/prescription.pdf`.
- The controller returns a fully qualified URL using `BASE_URL` when available, otherwise `http://localhost:<PORT>`.
- The generated file is served statically, so FE can open the URL directly in a new tab or use it as a download link.
Notes:
- Send `dateRecord` as an ISO 8601 string with timezone to avoid validation issues.
- The endpoint returns JSON metadata, not a binary PDF stream.

## Payment (VNPay)

### GET /payment/create_payment_url
Description: Deprecated appointment-payment URL endpoint.
Auth: Public
Query:
- `orderId`: string
- `amount`: number
Response: Error. Booking payment is now created by `POST /appointment/book` for DICH_VU deposits, and billing payment is created by `GET /receptionist/payments/:billingId/qr`.

### GET /payment/vnpay_return
Description: VNPay return handler.
Auth: Public
Query: VNPay return params
Behavior:
- Verify VNPay signature and response code.
- Process result idempotently (duplicate callbacks do not double-update).
- If callback is for `Payment.purpose = APPOINTMENT_DEPOSIT`:
  - success marks the deposit `PAID`, stores `depositPaidAmount/depositPaidAt/depositPaymentId`, confirms the appointment, and emits `appointment.booking.success`.
  - failure marks the deposit `FAILED`, marks the appointment `FAILED`, releases the slot, and does not create a Visit.
- If callback is for `Payment.purpose = BILLING`:
  - success sets `Payment.status = SUCCESS`, sets `Billing.status = PAID`, commits credit/coin deductions and coin reward, and emits `domain.payment.success`.
- If checksum invalid: reject the callback.
- Convert `vnp_PayDate` (VNPay GMT+7 format `yyyyMMddHHmmss`) to UTC before persisting.
- Emit best-effort real-time event `payment:update` with payload `{ orderId, status }`.

Important notes:
- Billing QR payments still use `vnp_TxnRef = billingId`.
- Appointment deposit payments use `vnp_TxnRef = paymentId` so the callback can resolve `Payment.purpose = APPOINTMENT_DEPOSIT`.
- FE MUST NOT rely on client-sent `amount`; payment amount comes from backend-created `Payment.amount`.
- Receptionist billing flow exposes `GET /receptionist/payments/:billingId/qr`.
- DICH_VU booking flow exposes deposit `paymentUrl` directly from `POST /appointment/book`.

Response:
- JSON metadata `{ code, message, data }`.

### GET /payment/:orderId
### GET /payments/:orderId
Description: Deprecated appointment-payment status endpoints.
Auth: Public
Response: Error. Use booking response deposit fields for appointment deposit state, and billing/payment endpoints for finalized billing payment state.

## WebSocket (Socket.IO)

Detailed backend refactor note:
- See `README_SOCKET_REFACTOR_BE.md` for old/new architecture, pros/cons, presence model, and heartbeat diagnostics.

Transport:
- Socket.IO (NestJS WebSocket gateway)
- Default Socket.IO path: `/socket.io`
- Namespace is mandatory when connecting (listed below)

Connection Auth (all namespaces):
- JWT is validated by Socket.IO middleware before the namespace gateway handles the connection.
- Client must provide token in `socket.handshake.auth.token`.
- Middleware attaches `socket.data.userId` and normalized auth payload for socket handlers.
- Missing/invalid/expired token => connection rejected with Unauthorized.

Presence and lifecycle:
- `heartbeat` is a client event that refreshes Redis TTL for the current user's device set.
- Presence is tracked with `user:{userId}:devices` as a multi-device SET and `online_users` as the online-user index.
- TTL is a fallback safety net only; the device set remains the source of truth for online state.
- FE should emit `heartbeat` periodically (recommended every 25-30 seconds) while socket is connected.

Old vs New connection flow:
- Old flow:
  - Connect socket.
  - Emit `JOIN_ROOM`.
  - Wait for `ROOM_JOINED`.
  - Start business event exchange.
- New flow:
  - Connect socket with `handshake.auth.token`.
  - Middleware verifies JWT before gateway lifecycle runs.
  - On success, backend attaches `socket.data.userId` and accepts connection.
  - Gateway lifecycle updates presence in Redis on connect/disconnect.
  - Client can start namespace business events immediately after connect.
  - `JOIN_ROOM` is still required only for email-room namespaces (`/appointment`, `/payment/vnpay`, `/patient-profile`, `/notification`).
  - `JOIN_ROOM` is not the global handshake gate for all namespaces anymore.

Room model:
- User-targeted pushes are mostly sent to room by email.
- After connected, client should emit `JOIN_ROOM` (no payload required).
- Server resolves email from JWT payload and joins room `<email>`.
- Server ack event: `ROOM_JOINED` with payload `{ email }`.

### Namespace `/appointment`
Purpose: booking lifecycle and cancellation notifications.

Server push events:
- `APPOINTMENT_BOOKING_SUCCESS`
- `APPOINTMENT_BOOKING_PENDING`
- `APPOINTMENT_BOOKING_FAILED`
- `APPOINTMENT_CANCELLED`
- `SHIFT_CANCELLED`

Event source flow:
- Appointment booking service emits domain events `appointment.booking.pending|success|failed`
- Booking listener maps to socket events `socket.appointment.pending|success|failed`
- Appointment gateway (`/appointment`) pushes to email room(s)
- Shift cancellation and appointment cancellation also emit `socket.shift.cancelled` and `socket.appointment.cancelled` => pushed via the same `/appointment` namespace

### Namespace `/appointment/fields-data`
Purpose: supporting data for booking form.

Client events:
- `get_timeslots_by_doctor` (currently placeholder; no payload returned from gateway)

Server push events:
- `hospital-specialties.fetched`
- `DOCTOR_LIST_FETCHED`

Important:
- These server pushes are sent to email room, so client still needs `JOIN_ROOM`.

### Namespace `/payment/vnpay`
Purpose: payment url and payment status updates.

Server push events:
- `payment:update` payload `{ orderId, status }` (broadcast to all clients in namespace)

Event source flow:
- DICH_VU booking returns the deposit `paymentUrl` directly in `POST /appointment/book`.
- Billing QR creation returns the billing `paymentUrl` directly in `GET /receptionist/payments/:billingId/qr`.
- VNPay return endpoint emits `payment.update` => VnPay gateway broadcasts `payment:update`.

### Namespace `/patient-profile`
Purpose: push assembled patient profile data.

Server push events:
- `PATIENT_PROFILE`

Event source flow:
- Profile saga emits `socket.push.patient-profile` with `{ roomEmail }`
- Patient profile gateway pushes to that email room

### Namespace `/chat`
Purpose: realtime chat messaging.

Client events:
- `CHAT_JOIN_USER` / `CHAT_LEAVE_USER`
- `CHAT_JOIN_CONVERSATION` / `CHAT_LEAVE_CONVERSATION`
- `CHAT_MESSAGE_SEND`
- `CHAT_MESSAGE_READ`

Server push events:
- `CHAT_MESSAGE_RECEIVED`
- `CHAT_MESSAGE_READ`
- `ROOM_JOINED`

Room model:
- User room: `user:<accountId>`
- Conversation room: `conv:<conversationId>`

### Namespace `/auth`
Purpose: authenticated socket connection scope for auth-related realtime extensions.

Current status:
- Namespace exists and receives auth from Socket.IO middleware.
- No dedicated push events are currently emitted in the codebase.

Presence event names reserved in the shared socket enum:
- `heartbeat` (client -> server)
- `user_online` (server event, reserved for future extensions)
- `user_offline` (server event, reserved for future extensions)

Current note for FE:
- `user_online` and `user_offline` are reserved and not emitted yet.
- Do not block presence UI on these two events; rely on existing domain events + HTTP fallback.

### Coin Expiry Reminder Realtime
Purpose: push expiring-coin reminders without waiting for HTTP polling.

Backend fanout path:
- Worker emits internal event `notification.coin.expiry.reminder`
- Realtime listener publishes payload to Redis channel `coin.expiry.reminder`
- Socket listener subscribes Redis and emits Socket.IO event `COIN_EXPIRY_REMINDER` to room `<patientEmail>`

Socket event:
- `COIN_EXPIRY_REMINDER`

Server payload:
```json
{
  "code": "SUCCESS",
  "message": "Coin expiry reminder",
  "data": {
    "jobId": "...",
    "transactionId": "...",
    "patientId": "...",
    "patientEmail": "patient@example.com",
    "patientName": "Nguyen Van A",
    "amount": 10000,
    "expiresAt": 1776269699780,
    "runAt": 1776010499780,
    "reminderDays": 3,
    "retryCount": 0
  }
}
```

Notes:
- `expiresAt` and `runAt` are epoch milliseconds (UTC).
- Client must connect socket with JWT and emit `JOIN_ROOM` before expecting this event.

## Notification And Wallet Realtime Notes

Notification:
- Notification module persists notification data on `notify.*` domain events.
- It does not expose a dedicated notification socket namespace.
- Realtime appointment/shift-related user alerts are delivered through `/appointment` socket events above.
- Coin expiry reminder push is emitted via global socket event `COIN_EXPIRY_REMINDER` after Redis fanout.

Wallet:
- Wallet module currently has HTTP APIs (`/wallet/balance`, `/wallet/details`) and event-driven updates in service/listeners.
- There is no dedicated realtime balance-delta event yet; FE should re-fetch wallet summary after receiving `COIN_EXPIRY_REMINDER` if balance UI is open.
- FE should still refresh wallet state via HTTP after booking/cancel/shift-cancel/payment update flows.
- Cancellation refund is now credited to `creditBalance`; coin reward/discount is tracked separately.

## FE Integration Checklist (Socket)

1. Connect to the exact namespace you need (`/appointment`, `/payment/vnpay`, `/chat`, etc.)
2. Always pass JWT in handshake (`auth.token`)
3. Immediately emit `JOIN_ROOM` after connected for email-based push namespaces
4. Subscribe to exact event names from `SocketEventsEnum` (case-sensitive)
5. Subscribe thêm `COIN_EXPIRY_REMINDER` và render thời gian từ epoch (`data.expiresAt`, `data.runAt`)
6. Giữ polling `GET /notifications/by-email` như fallback khi mất kết nối socket hoặc app resume nền
7. Emit `heartbeat` định kỳ (25-30s) cho các namespace có kết nối dài để giữ presence TTL

## System

### GET /
Description: Health check / hello.
Auth: Public
Response: string

## Breaking Changes

- `patientId` and `email` are no longer accepted from request body/query for multiple endpoints.
- User identity is derived from JWT (`req.user`), including `accountId`, `patientId`, `doctorId`, and `email`.
- Appointment booking core flow is now synchronous in `AppointmentBookingService` (no saga/event chaining for core state transitions).
- New booking status: `FAILED`.
- Booking now uses Redis slot lock key `slot:{doctorId}:{timeSlotId}` with TTL aligned to VNPay expiry (`VN_PAY_EXPIRE_MINUTES`, default 15 minutes).
- Expired `PENDING` bookings are auto-marked `FAILED` on the same VNPay expiry window.
- `paymentMethod=COIN` is deprecated and rejected by booking API.
- New `paymentMethod=CREDIT` allows wallet-based monetary payment.
- Booking response now includes `originalAmount`, `discountAmount`, `finalAmount`.
- Wallet responses now include separate `coinBalance` and `creditBalance`.
- Appointment cancellation/shift-cancellation refunds are credited to credit wallet (financial), not coin wallet (reward).
- Routes changed:
  - `GET /appointment/completed/doctor/:doctorId` ? `GET /appointment/completed/doctor`
  - `GET /appointment/today?doctorId=...` ? `GET /appointment/today`
  - `GET /patients/by-account/:accountId` ? `GET /patients/by-account`
  - `POST /patients/:id/medical-profile` ? `POST /patients/me/medical-profile`
  - `POST /patients/:id/allergies` ? `POST /patients/me/allergies`
  - `POST /patients/:id/medical-history` ? `POST /patients/me/medical-history`
  - `PATCH /profiles/:accountId` ? `PATCH /profiles/me`
  - `GET /doctors/account/:accountId` ? `GET /doctors/me`
  - `POST /shift/register` now requires JWT and uses doctorId from token
  - `POST /shift/register` now requires `startTime` and `endTime` (ISO 8601 with timezone); legacy `date` payload is removed
  - `POST /doctor-posts` now requires JWT and uses doctorId from token
  - `GET /chat/conversations` no longer takes `accountId` in query
  - `POST /chat/conversations/:id/read` no longer accepts `accountId` in body
  - `GET /reviews/by-appointment-patient` now uses authenticated patient instead of `patientId` query
  - `POST /reviews` now uses authenticated patient instead of `patientId` in body
  - `GET /notifications/by-email` no longer accepts `email` query
  - `PATCH /notifications/:id/read` now requires JWT
