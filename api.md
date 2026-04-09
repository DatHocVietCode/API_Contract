# API Contract

Base URL: `/api` (global prefix set in `src/main.ts`)

This document is generated from controller definitions in the codebase. Auth refactor note: many endpoints now derive user identity from JWT (`req.user`) rather than request params/body.

Datetime validation note:
- All datetime request fields must be ISO 8601 with timezone (`Z` or `+/-HH:mm`).
- Datetime values without timezone are rejected with `400 Bad Request`.
- Internal processing normalizes datetimes to UTC and converts to epoch milliseconds.

## Auth

### POST /auth/register
Description: Register a new user account.
Auth: Public
Request Body:
- `email`: string
- `password`: string
- `role`: `PATIENT | DOCTOR` (optional, default `PATIENT`)
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
Response (Success):
```json
{ "code": "SUCCESS", "message": "Access token refreshed", "data": { "accessToken": "...", "refreshToken": "..." } }
```
Response (Error):
```json
{ "code": "ERROR", "message": "Invalid refresh token", "data": null }
```

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
- `date`: string/date
- `specialty`: string (optional)
- `timeSlotId`: string
- `doctor`: { `id`, `name`, `email` } (doctor id required)
- `serviceType`: enum
- `paymentMethod`: enum
- `amount`: number (optional)
- `reasonForAppointment`: string (optional)
- `coinsToUse`: number (optional)
- `useCoin`: boolean (optional)
Core flow:
- Validate request
- Acquire Redis slot lock `SET slot:{doctorId}:{timeSlotId} NX EX 300`
- Pre-check slot in database for same `(doctorId, date, timeSlot)` with status in `PENDING|CONFIRMED`
- Create appointment with `PENDING` and mark timeslot `booked` in a MongoDB transaction
- Process payment synchronously by payment method
- For online payment, create an idempotent payment record per appointment before generating payment URL
- If payment success -> set `CONFIRMED`
- If payment fails -> set `FAILED` and release slot lock/slot

Response examples:
1. Slot locked by another booking
```json
{ "code": "ERROR", "message": "Slot already booked", "data": null }
```

2. ONLINE payment: create pending booking + payment url
```json
{
  "code": "PENDING",
  "message": "Appointment created. Complete payment to confirm booking.",
  "data": {
    "appointmentId": "<appointmentId>",
    "paymentUrl": "https://..."
  }
}
```

3. COIN payment success
```json
{
  "code": "SUCCESS",
  "message": "Deducted ... coins successfully",
  "data": { "appointmentId": "<appointmentId>" }
}
```

4. Payment/booking failed
```json
{
  "code": "ERROR",
  "message": "...",
  "data": { "appointmentId": "<appointmentId>" }
}
```

Notes:
- Redis is used only for race-condition protection (TTL 5 minutes).
- A background cleanup marks expired `PENDING` bookings as `FAILED` after 5 minutes.
- Database is the final gate for consistency (not Redis).
- Database enforces uniqueness for active bookings on `(doctorId, date, timeSlot)` where status is `PENDING|CONFIRMED`.
- Duplicate key errors (`11000`) are mapped to `Slot already booked`.

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

### PATCH /appointment/reschedule
Description: Reschedule appointment.
Auth: Required (JWT)
Body: `RescheduleAppointmentDto`
- `appointmentId`, `newDate`, `newTimeSlotId`, `reason` (optional)

### PATCH /appointment/cancel
Description: Cancel appointment.
Auth: Required (JWT)
Body:
- `appointmentId`: string
- `reason`: string (optional)

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

### GET /chat/conversations
Description: List conversations for authenticated user.
Auth: Required (JWT)
Query: `skip`, `limit`

### GET /chat/conversations/:id/messages
Description: List messages in a conversation.
Auth: Required (JWT)
Query: `before`, `limit`

### POST /chat/conversations/:id/read
Description: Mark conversation as read for authenticated user.
Auth: Required (JWT)
Body: none

### GET /chat/contacts/search
Description: Search contacts.
Auth: Required (JWT)
Query: `q`, `role`, `limit`

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

### GET /wallet/balance
Description: Get wallet balance for authenticated patient.
Auth: Required (JWT)

### GET /wallet/details
Description: Get wallet details and history for authenticated patient.
Auth: Required (JWT)
Query: `page`, `limit`

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

### GET /doctors/me
Description: Get doctor by authenticated account id.
Auth: Required (JWT)

### GET /doctors/:id
Description: Get doctor by id.
Auth: Public

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
Description: Generate prescription PDF for appointment or record.
Auth: Public
Body: `CreatePrescriptionPdfDto`
Response: `{ code: "SUCCESS", data: { url } }`

## Payment (VNPay)

### GET /payment/create_payment_url
Description: Create VNPay payment URL.
Auth: Public
Query:
- `orderId`: string
- `amount`: number
Response: `{ "paymentUrl": "..." }`

### GET /payment/vnpay_return
Description: VNPay return handler.
Auth: Public
Query: VNPay return params
Behavior:
- Verify VNPay signature and response code.
- Process result idempotently (duplicate callbacks do not double-update).
- If success (`vnp_ResponseCode=00` and `vnp_TransactionStatus=00`): finalize payment as `COMPLETED` and confirm appointment.
- If failed or checksum invalid: finalize payment as `FAILED`.
- Convert `vnp_PayDate` (VNPay GMT+7 format `yyyyMMddHHmmss`) to UTC before persisting.
- Emit best-effort real-time event `payment:update` with payload `{ orderId, status }`.
- Redirect user to frontend result page.

Response:
- No JSON body.
- HTTP redirect to:
  - `${FRONTEND_URL}/payment-result?orderId=<orderId>&status=COMPLETED|FAILED&code=<vnp_ResponseCode>`

### GET /payment/:orderId
### GET /payments/:orderId
Description: Get payment status by order/appointment id.
Auth: Public
Response:
```json
{
  "orderId": "string",
  "status": "PENDING | COMPLETED | FAILED",
  "amount": 100000,
  "paidAt": "2026-04-08T01:23:45.000Z"
}
```

Notes:
- `paidAt` can be `null` for unpaid/failed records.
- `orderId` maps to appointment id (`vnp_TxnRef`).

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
- Booking now uses Redis slot lock key `slot:{doctorId}:{timeSlotId}` with TTL 300s.
- Expired `PENDING` bookings are auto-marked `FAILED` after 5 minutes.
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