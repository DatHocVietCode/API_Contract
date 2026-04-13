# Appointment Reschedule Flow

## 1. Overview

API reschedule appointment da duoc tach ro theo luong backend hien tai:
- Controller nhan request va forward sang service
- Service tinh lai snapshot thoi gian moi
- Slot cu duoc release, slot moi duoc book
- Event `appointment.rescheduled` duoc emit de wallet xu ly refund

File tham chieu:
- `src/appointment/appointment.controller.ts`
- `src/appointment/appointment.service.ts`
- `src/appointment/dto/appointment-booking.dto.ts`
- `src/appointment/listenners/reschedule.listener.ts`
- `src/appointment/utils/appointment-time.helper.ts`

## 2. Current Backend Flow

### Entry point

`PATCH /api/appointment/reschedule`

Handler:
- `AppointmentController.rescheduleAppointment()`

What it does:
- Guard bang `JwtAuthGuard`
- Nhan body theo `RescheduleAppointmentDto`
- Goi `AppointmentService.rescheduleAppointment(appointmentId, newDate, newTimeSlotId, reason)`

### Service flow

Handler:
- `AppointmentService.rescheduleAppointment()`

Logic chinh:
- Parse `newDate` mot lan sang UTC
- Load appointment hien tai
- Chi cho phep voi trang thai `PENDING` hoac `CONFIRMED`
- Lay `scheduledAt` snapshot dang luu de tinh so gio con lai
- Chan reschedule neu con `<= 24h`
- Tinh refund theo tier:
  - `> 48h`: 100%
  - `24h - 48h`: 50%
- Load slot moi
- Tinh lai `scheduledAt`, `startTime`, `endTime`
- Update appointment voi snapshot moi
- Release slot cu, book slot moi
- Emit event `appointment.rescheduled`

### Side effect

Handler:
- `RescheduleListener.handleAppointmentRescheduled()`

What it does:
- Nhan event `appointment.rescheduled`
- Goi `walletService.addCoins(...)`
- Neu refund fail, reschedule van thanh cong, chi log loi de admin xu ly sau

## 3. Response Shape

API tra ve:
- `appointmentId`
- `refundAmount`
- `refundReason`
- `newDate`
- `scheduledAt`
- `startTime`
- `endTime`
- `hoursUntilAppointment`

## 4. DTO Contract

`RescheduleAppointmentDto`:
- `appointmentId`: MongoId
- `newDate`: ISO 8601 with timezone
- `newTimeSlotId`: MongoId
- `reason`: optional string

## 5. Important Notes For FE

- FE can only call this API if appointment is still reschedulable.
- FE should not cache old slot time as source of truth; backend uses stored snapshot.
- FE should display refund info from response.
- FE should expect rejection when appointment is too close to start time.

## 6. Current Implementation Progress

Done:
- JWT guard on endpoint
- UTC parsing for new date
- Snapshot update on appointment
- Slot release / slot book
- Refund event emission
- Wallet refund listener

Needs review before FE re-integration:
- Ownership authorization should be re-checked at service level
- Frontend integration has not been completed yet
- UI should confirm the new slot and the refund preview before submit if needed

## 7. Suggested FE Flow

1. User picks new date and time slot
2. FE sends `appointmentId`, `newDate`, `newTimeSlotId`, optional `reason`
3. FE waits for success response
4. FE updates appointment state using returned `scheduledAt` and refund fields
5. If rejected, FE shows the exact backend message
