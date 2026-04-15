# API Contract

Base URL: `/api` (global prefix set in `src/main.ts`)

This document is generated from controller definitions in the codebase. Auth refactor note: many endpoints now derive user identity from JWT (`req.user`) rather than request params/body.

Datetime validation note:
- All datetime request fields must be ISO 8601 with timezone (`Z` or `+/-HH:mm`).
- Datetime values without timezone are rejected with `400 Bad Request`.
- Internal processing normalizes datetimes to UTC and converts to epoch milliseconds.
- Exception: path parameters that represent date-only filters may require `YYYY-MM-DD` by endpoint contract.

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
- `appointmentDate`: string (required, scheduled date selected by user; ISO 8601 with timezone)
- `bookingDate`: string (optional, booking creation timestamp; ISO 8601 with timezone). If omitted, server uses request processing time.
- `date`: string/date (deprecated, legacy alias for `appointmentDate`)
- `specialty`: string (optional)
- `timeSlotId`: string
- `doctor`: { `id`, `name`, `email` } (doctor id required)
- `serviceType`: enum
- `paymentMethod`: `ONLINE | VNPAY | CREDIT | CASH | OFFLINE` (`COIN` is deprecated)
- `amount`: number (optional)
- `reasonForAppointment`: string (optional)
- `coinsToUse`: number (optional, requested coin discount amount)
- `useCoin`: boolean (optional, apply coin discount)
Semantics:
- `appointmentDate`: Represents when the medical visit is scheduled to happen.
- `bookingDate`: Represents when the booking request is created/recorded.
- Backward compatibility: `date` is still accepted temporarily and treated as `appointmentDate` when `appointmentDate` is missing.
Core flow:
- Validate request
- Acquire Redis slot lock `SET slot:{doctorId}:{timeSlotId} NX EX 300`
- Pre-check slot in database for same `(doctorId, appointmentDate, timeSlot)` with status in `PENDING|CONFIRMED`
- Calculate amount breakdown:
  - `originalAmount = amount`
  - `discountAmount = min(availableCoin, requestedCoin?, originalAmount * 10%, 30000)` when `useCoin=true`
  - `finalAmount = originalAmount - discountAmount`
- Create appointment with `PENDING`, persist `coinDiscountAmount`, `paymentAmount=finalAmount`, and mark timeslot `booked` in a MongoDB transaction
- If `discountAmount > 0`, deduct coin as discount transaction
- Process remaining `finalAmount` by payment method (`ONLINE|VNPAY` async, `CREDIT` sync)
- For online payment, create an idempotent payment record per appointment before generating payment URL
- If payment success (or `finalAmount = 0`) -> set `CONFIRMED`
- If payment fails -> set `FAILED` and release slot lock/slot

Request examples:
1. Preferred payload (new contract)
```json
{
  "hospitalName": "Bệnh viện Đa khoa",
  "appointmentDate": "2026-04-15T02:00:00Z",
  "timeSlotId": "67f2b1...",
  "doctor": {
    "id": "67f1a0...",
    "name": "Nguyen Van A",
    "email": "doctor@example.com"
  },
  "serviceType": "KHAM_DICH_VU",
  "paymentMethod": "ONLINE",
  "amount": 100000,
  "reasonForAppointment": "Tai kham dinh ky"
}
```

2. Backward-compatible legacy payload (deprecated)
```json
{
  "hospitalName": "Bệnh viện Đa khoa",
  "date": "2026-04-15T02:00:00Z",
  "timeSlotId": "67f2b1...",
  "doctor": {
    "id": "67f1a0...",
    "name": "Nguyen Van A",
    "email": "doctor@example.com"
  },
  "serviceType": "KHAM_DICH_VU",
  "paymentMethod": "ONLINE"
}
```

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
    "paymentUrl": "https://...",
    "originalAmount": 100000,
    "discountAmount": 10000,
    "finalAmount": 90000
  }
}
```

3. CREDIT payment success
```json
{
  "code": "SUCCESS",
  "message": "Deducted ... credit successfully",
  "data": {
    "appointmentId": "<appointmentId>",
    "originalAmount": 100000,
    "discountAmount": 10000,
    "finalAmount": 90000
  }
}
```

4. Zero-final booking (fully discounted by policy)
```json
{
  "code": "SUCCESS",
  "message": "Appointment confirmed successfully",
  "data": {
    "appointmentId": "<appointmentId>",
    "originalAmount": 20000,
    "discountAmount": 20000,
    "finalAmount": 0
  }
}
```

5. Payment/booking failed
```json
{
  "code": "ERROR",
  "message": "...",
  "data": {
    "appointmentId": "<appointmentId>",
    "originalAmount": 100000,
    "discountAmount": 10000,
    "finalAmount": 90000
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
- `paymentMethod=COIN` is deprecated and rejected. Use `useCoin=true` with `ONLINE|VNPAY|CREDIT`.
- Coin used in booking is discount-only and never treated as standalone payment.

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
- JWT token in `handshake.auth.token` (or `Authorization: Bearer <token>`).

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

## WebSocket (Socket.IO)

Transport:
- Socket.IO (NestJS WebSocket gateway)
- Default Socket.IO path: `/socket.io`
- Namespace is mandatory when connecting (listed below)

Connection Auth (all namespaces):
- JWT is validated at gateway connection middleware (shared base gateway logic).
- Client must provide token in one of two ways:
  - `auth.token` in Socket.IO connect options
  - `Authorization: Bearer <token>` handshake header
- Missing/invalid/expired token => connection rejected with Unauthorized.

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
- `PAYMENT_VNPAY_URL_CREATED` payload `{ appointmentId, paymentUrl }` (to user email room)
- `payment:update` payload `{ orderId, status }` (broadcast to all clients in namespace)

Event source flow:
- Booking flow emits `payment.vnpay.url.created` => VnPay gateway pushes `PAYMENT_VNPAY_URL_CREATED`
- VNPay return endpoint emits `payment.update` => VnPay gateway broadcasts `payment:update`

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
- Namespace exists and inherits JWT middleware.
- No dedicated push events are currently emitted in the codebase.

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
2. Always pass JWT in handshake (`auth.token` preferred)
3. Immediately emit `JOIN_ROOM` after connected for email-based push namespaces
4. Subscribe to exact event names from `SocketEventsEnum` (case-sensitive)
5. Subscribe thêm `COIN_EXPIRY_REMINDER` và render thời gian từ epoch (`data.expiresAt`, `data.runAt`)
6. Giữ polling `GET /notifications/by-email` như fallback khi mất kết nối socket hoặc app resume nền

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