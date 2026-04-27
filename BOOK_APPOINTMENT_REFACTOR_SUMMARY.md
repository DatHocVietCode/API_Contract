# Book Appointment Refactor Summary

Date: 2026-04-08
Scope: Backend booking flow hardening in Appointment module + VNPay callback/payment status integration

## Update 2026-04-27 (Visit-Based Workflow Fields)

Scope: Extend booking request contract for receptionist/visit workflow without breaking legacy payment flow.

### What changed

- Added optional booking request fields:
  - `visitType`: `OFFLINE`
  - `paymentCategory`: `BHYT | DICH_VU`
  - `depositAmount`: non-negative number
- Backend normalization rules:
  - If `paymentCategory = BHYT`, `depositAmount` is normalized to `0`.
  - If `paymentCategory = DICH_VU`, `depositAmount` is accepted as provided (after sanitization).
- Legacy fields remain unchanged and supported:
  - `paymentMethod`
  - `coinsToUse`
  - `useCoin`
- Booking response shape is unchanged for compatibility with existing FE clients.

## Update 2026-04-14 (Coin/Credit Separation)

Scope: Separate reward coin from financial credit for safer accounting and FE consistency.

### What changed

- `COIN` is no longer accepted as a standalone booking payment method.
- New booking payment option: `CREDIT` (money-equivalent wallet).
- Coin is now discount-only in booking flow (`useCoin=true` + optional `coinsToUse`).
- Booking response now always includes amount breakdown:
  - `originalAmount`
  - `discountAmount`
  - `finalAmount`
- Refund flows (appointment cancel / shift cancel) now credit financial wallet (`CreditWallet`) instead of adding coin.
- Wallet APIs now expose both domains for FE rendering:
  - `coinBalance` (reward)
  - `creditBalance` (financial)

### Discount policy applied in booking

- Percentage-based discount (current policy: 10%).
- Per-transaction cap (current policy: 30,000).
- Actual discount is bounded by:
  - policy percentage,
  - cap,
  - available non-expired coin,
  - optional `coinsToUse` requested by client.

### Coin expiration behavior

- Coin earn transactions support `expiresAt`.
- Available coin calculation excludes expired earn transactions.
- Legacy wallet balance is still supported via migration seeding into coin transaction ledger.

### FE integration notes

- For booking UI, always render using `originalAmount`, `discountAmount`, `finalAmount` from response.
- If `finalAmount = 0`, booking can be confirmed without external payment URL.
- For wallet UI, show coin and credit as separate balances and histories.
- After cancel/shift-cancel, refresh `creditBalance` because refund goes to credit.

## Goals

- Prevent double booking under concurrent requests.
- Prevent duplicate payment creation for the same appointment.
- Ensure Redis lock is only a contention reducer, not the final source of truth.
- Improve callback safety and idempotency for VNPay return flow.

## Key Changes

### 1. Database-level slot protection

- Appointment unique index updated to:
  - `(doctorId, appointmentDate, timeSlot)`
  - Unique only for active statuses: `PENDING`, `CONFIRMED`.
- Why:
  - Previous key `(doctorId, timeSlot)` was insufficient across dates.
  - DB uniqueness guarantees correctness even if Redis lock is bypassed/expired.
- Note:
  - Current persistence may still use a legacy field name `date`; semantically this is the scheduled `appointmentDate`.

### 2. Booking pre-check before insert

- Added DB pre-check for same `(doctorId, appointmentDate, timeSlot)` with active statuses.
- If already exists:
  - Release Redis lock.
  - Return `Slot already booked`.

### 3. MongoDB transaction in booking creation

- Wrapped in transaction:
  - Create appointment document with `PENDING`.
  - Mark timeslot as `booked`.
- Any failure rolls back both operations.

### 4. Duplicate key (`11000`) handling

- Duplicate key errors during creation are caught and mapped to:
  - `code: ERROR`
  - `message: Slot already booked`
- Redis lock is released in failure paths.

### 5. Online payment idempotency

- Payment schema was finalized and indexed:
  - Unique index on `appointmentId` (one payment record per appointment).
  - Sparse unique index on `transactionId`.
- Before generating VNPay URL:
  - Check existing payment by `appointmentId`.
  - If found, return `Payment already initiated for this appointment`.
- This prevents duplicate payment URL creation from repeated client clicks/retries.

### 6. Safer lock handling

- Added safe lock release helper with internal try/catch.
- Avoids unhandled rejections if Redis release itself fails.

## VNPay Return Flow (related updates)

- `GET /api/payment/vnpay_return` now:
  - Verifies VNPay checksum/signature.
  - Interprets success only when:
    - `vnp_ResponseCode = 00`
    - `vnp_TransactionStatus = 00`
  - Parses `vnp_PayDate` (GMT+7 format `yyyyMMddHHmmss`) to UTC.
  - Updates appointment/payment state idempotently.
  - Emits best-effort real-time event `payment:update`.
  - Redirects to frontend result page instead of returning JSON.

## Data Model Additions

### Appointment fields

- `paymentAmount`
- `paidAt`
- `paymentResponseCode`
- `paymentTransactionStatus`

### Payment model

- Exported usable schema with indexes.
- `method`/`status` fixed to Mongoose enum form:
  - `type: String`, `enum: ...`

## API Contract Alignment

Updated in `api-contract/api.md`:

- Booking flow now documents:
  - DB pre-check
  - Transactional create + mark timeslot
  - Idempotent payment creation
  - DB-as-final-gate principle
  - Duplicate key mapping to `Slot already booked`
- Payment endpoints documented:
  - `GET /payment/vnpay_return` redirect behavior
  - `GET /payment/:orderId`
  - `GET /payments/:orderId`

## Operational Notes

- After changing indexes, ensure MongoDB old indexes are migrated correctly.
- Existing stale index `(doctorId, timeSlot)` should be reviewed/dropped if still present.
- Redis remains useful for reducing contention, but correctness is enforced by DB constraints and transaction boundaries.

## Testing Checklist

- Concurrent booking requests for same doctor/date/timeslot -> only one succeeds.
- Repeated online booking payment-trigger action -> no duplicate payment record/URL creation.
- Duplicate callback from VNPay -> idempotent status update, no double state transition.
- Callback with invalid checksum -> marked failed path, redirected to frontend result.
- API compatibility: legacy `date` request input still works temporarily, but new clients should send `appointmentDate`.
