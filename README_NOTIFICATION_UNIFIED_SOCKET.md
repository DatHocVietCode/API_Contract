# Unified Notification Socket Contract

## 1. Goal

Notification realtime is standardized to one socket event so FE only subscribes once.

- Namespace: `/notification`
- Socket event: `NOTIFICATION_RECEIVED`
- Room strategy: authenticated email room from JWT
- REST list fallback: `GET /notifications/by-email`

The backend returns date/time values as epoch milliseconds UTC. FE owns all date, time, locale, and timezone formatting.

## 2. Notification Enums

```ts
export type NotificationType =
  | 'COIN_EXPIRY_REMINDER'
  | 'APPOINTMENT_SUCCESS'
  | 'APPOINTMENT_CANCELLED'
  | 'APPOINTMENT_RESCHEDULED'
  | 'PAYMENT_SUCCESS'
  | 'ASSIGNMENT_TASK_CREATED'
  | 'ASSIGNMENT_TASK_REMINDER'
  | 'ASSIGNMENT_TASK_EXPIRED'
  | 'APPOINTMENT_DOCTOR_ASSIGNED';

export type NotificationRecipientRole =
  | 'PATIENT'
  | 'DOCTOR'
  | 'RECEPTIONIST'
  | 'ADMIN';
```

## 3. Stored Notification DTO

REST responses and `NOTIFICATION_RECEIVED` both use the saved notification payload:

```ts
export type NotificationDto<TType extends NotificationType = NotificationType> = {
  _id: string;
  type: TType;
  recipientEmail: string;
  recipientRole: NotificationRecipientRole;
  title?: string; // backward-compatible, safe generic fallback
  message?: string; // backward-compatible, must not embed raw epoch/undefined/null
  titleKey?: string; // preferred FE rendering key
  messageKey?: string; // preferred FE rendering key
  data: NotificationMap[TType];
  isRead: boolean;
  createdAt: number; // epoch milliseconds UTC
  idempotencyKey?: string;
};

export type NotificationPayload = NotificationDto;
```

Rules:

- `type` is the discriminant key for FE handler/render registries.
- `recipientEmail` is the exact saved owner and socket room target.
- `recipientRole` is the explicit audience for role-specific rendering.
- `titleKey` and `messageKey` are preferred. FE should render human-facing copy from keys plus `data`.
- `title` and `message` are backward-compatible fallbacks only. New backend messages are generic and safe; they do not contain raw epoch numbers, `undefined`, or `null`.
- `data` carries structured domain values. Date fields remain epoch milliseconds UTC.

## 4. Notification Data Map

```ts
export type NotificationMap = {
  COIN_EXPIRY_REMINDER: CoinExpiryReminderData;
  APPOINTMENT_SUCCESS: AppointmentNotificationData;
  APPOINTMENT_CANCELLED: AppointmentCancelledData;
  APPOINTMENT_RESCHEDULED: AppointmentRescheduledData;
  PAYMENT_SUCCESS: PaymentSuccessData;
  ASSIGNMENT_TASK_CREATED: AssignmentTaskCreatedData;
  ASSIGNMENT_TASK_REMINDER: AssignmentTaskReminderData;
  ASSIGNMENT_TASK_EXPIRED: AssignmentTaskExpiredData;
  APPOINTMENT_DOCTOR_ASSIGNED: AppointmentDoctorAssignedData;
};

export type AppointmentNotificationData = {
  appointmentId?: string | null;
  appointmentDate?: number | null; // epoch ms
  scheduledAt?: number | null; // epoch ms
  bookingDate?: number | null; // epoch ms
  timeRange?: string | null;
  hospitalName?: string | null;
  doctorName?: string | null;
  patientName?: string | null;
  patientEmail?: string | null;
  paymentMethod?: string | null;
  serviceType?: string | null;
  amount?: number | null;
};

export type AppointmentCancelledData = {
  appointmentId?: string | null;
  appointmentDate?: number | null; // epoch ms
  timeRange?: string | null;
  timeSlotId?: string | null;
  hospitalName?: string | null;
  patientEmail?: string | null;
  doctorEmail?: string | null;
  reason?: string | null;
  refundAmount?: number | null;
  shouldRefund?: boolean | null;
};

export type AppointmentRescheduledData = {
  appointmentId?: string | null;
  appointmentDate?: number | null; // epoch ms, same as newScheduledAt
  scheduledAt?: number | null; // epoch ms, same as newScheduledAt
  oldScheduledAt?: number | null; // epoch ms
  newScheduledAt?: number | null; // epoch ms
  timeRange?: string | null;
  timeSlotId?: string | null;
  hospitalName?: string | null;
  doctorName?: string | null;
  patientEmail?: string | null;
  doctorEmail?: string | null;
  reason?: string | null;
};

export type PaymentSuccessData = {
  appointmentId?: string | null;
  orderId?: string | null;
  status: 'COMPLETED';
  appointmentDate?: number | null; // epoch ms
  scheduledAt?: number | null; // epoch ms
  bookingDate?: number | null; // epoch ms
  hospitalName?: string | null;
};

export type AppointmentDoctorAssignedData = {
  appointmentId?: string | null;
  doctorId?: string | null;
  doctorName?: string | null; // NEW: resolved doctor display name (may be null)
  timeSlotId?: string | null;
  appointmentDate?: number | null; // epoch ms
  scheduledAt?: number | null; // epoch ms
  startTime?: number | null; // NEW: slot start, epoch ms (may be null)
  endTime?: number | null; // NEW: slot end, epoch ms (may be null)
  hospitalName?: string | null; // NEW: location/hospital (may be null)
  serviceType?: string | null; // NEW: e.g. KHAM_BHYT | KHAM_DICH_VU | KHAM_ONLINE
  specialty?: string | null; // NEW: requested specialty (may be null)
  patientEmail?: string | null;
};

export type AssignmentTaskCreatedData = {
  taskId?: string | null;
  appointmentId?: string | null;
  specialty?: string | null;
  reasonForAppointment?: string | null;
  deadlineAt?: number | null; // epoch ms
  priority?: string | null;
  online?: boolean | null;
};

export type AssignmentTaskReminderData = {
  taskId?: string | null;
  appointmentId?: string | null;
  deadlineAt?: number | null; // epoch ms
  reminderCount?: number | null;
  online?: boolean | null;
};

export type AssignmentTaskExpiredData = {
  taskId?: string | null;
  appointmentId?: string | null;
  deadlineAt?: number | null; // epoch ms
  online?: boolean | null;
};

export type CoinExpiryReminderData = {
  jobId?: string;
  transactionId?: string;
  amount?: number;
  expiresAt?: number; // epoch ms
  runAt?: number; // epoch ms
  reminderDays?: number;
};
```

## 5. Template Keys And Audiences

```ts
export type NotificationTemplateKey =
  | 'notification.patient.appointmentSuccess.title'
  | 'notification.patient.appointmentSuccess.message'
  | 'notification.doctor.assignedAppointment.title'
  | 'notification.doctor.assignedAppointment.message'
  | 'notification.patient.appointmentCancelled.title'
  | 'notification.patient.appointmentCancelled.message'
  | 'notification.doctor.appointmentCancelled.title'
  | 'notification.doctor.appointmentCancelled.message'
  | 'notification.patient.appointmentRescheduled.title'
  | 'notification.patient.appointmentRescheduled.message'
  | 'notification.doctor.appointmentRescheduled.title'
  | 'notification.doctor.appointmentRescheduled.message'
  | 'notification.patient.doctorAssigned.title'
  | 'notification.patient.doctorAssigned.message'
  | 'notification.patient.paymentSuccess.title'
  | 'notification.patient.paymentSuccess.message'
  | 'notification.receptionist.assignmentTaskCreated.title'
  | 'notification.receptionist.assignmentTaskCreated.message'
  | 'notification.receptionist.assignmentTaskReminder.title'
  | 'notification.receptionist.assignmentTaskReminder.message'
  | 'notification.receptionist.assignmentTaskExpired.title'
  | 'notification.receptionist.assignmentTaskExpired.message';
```

Audience rules:

- Patient notifications use `recipientRole: 'PATIENT'` and patient template keys.
- Doctor notifications use `recipientRole: 'DOCTOR'` and doctor template keys.
- Receptionist assignment workflow notifications use `recipientRole: 'RECEPTIONIST'` and receptionist template keys.
- A single appointment event may create multiple notification rows, but each row must have its own recipient, role, keys, and structured data.

## 6. Transport Pipeline

Notification flow:

1. Domain listeners publish typed notification jobs to RabbitMQ queue `notification.jobs`.
2. Notification consumer processes queue payload.
3. Handler registry resolves the handler by `payload.type`.
4. Handler persists a notification row in MongoDB with recipient ownership and an idempotency key.
5. Handler publishes the saved `NotificationDto` to Redis channel `notification`.
6. Notification socket bridge emits `NOTIFICATION_RECEIVED` to `NotificationDto.recipientEmail` only.

## 7. FE Handler Pattern

```ts
const handlers = {
  COIN_EXPIRY_REMINDER: handleCoinExpiry,
  APPOINTMENT_SUCCESS: handleAppointmentSuccess,
  APPOINTMENT_CANCELLED: handleAppointmentCancelled,
  APPOINTMENT_RESCHEDULED: handleAppointmentRescheduled,
  PAYMENT_SUCCESS: handlePaymentSuccess,
  ASSIGNMENT_TASK_CREATED: handleAssignmentTaskCreated,
  ASSIGNMENT_TASK_REMINDER: handleAssignmentTaskReminder,
  ASSIGNMENT_TASK_EXPIRED: handleAssignmentTaskExpired,
  APPOINTMENT_DOCTOR_ASSIGNED: handleAppointmentDoctorAssigned,
} satisfies Record<NotificationType, (payload: NotificationPayload) => void>;

socket.on('NOTIFICATION_RECEIVED', (payload: NotificationPayload) => {
  handlers[payload.type]?.(payload);
});
```

FE should use `payload.titleKey`, `payload.messageKey`, `payload.recipientRole`, and `payload.data` to render localized text. Do not regex-parse `title` or `message` to discover dates.

## 8. Backward Compatibility

Legacy domain socket events are temporarily kept for compatibility:

- `COIN_EXPIRY_REMINDER`
- `APPOINTMENT_BOOKING_SUCCESS`
- `APPOINTMENT_BOOKING_PENDING`
- `APPOINTMENT_BOOKING_FAILED`
- `APPOINTMENT_CANCELLED`
- `SHIFT_CANCELLED`
- `PAYMENT_UPDATE`
- `PAYMENT_VNPAY_URL_CREATED`

FE bell and notification center should use only:

- `/notification`
- `NOTIFICATION_RECEIVED`
- `GET /notifications/by-email`
- `PATCH /notifications/:id/read`

## 9. Changelog

### 2026-06-22 — Richer `APPOINTMENT_DOCTOR_ASSIGNED` (doctor-assigned to broad booking)

When a receptionist assigns a doctor/slot to a broad (unassigned) appointment, the
patient notification is now self-explanatory.

- **`data` gained optional fields** (all backward-compatible, may be `null`/absent):
  `doctorName`, `hospitalName`, `startTime` (epoch ms), `endTime` (epoch ms),
  `serviceType`, `specialty`. Existing fields are unchanged. **Time stays epoch ms** —
  `scheduledAt`/`startTime`/`endTime` are numbers; FE still owns formatting.
- **`message` fallback is now a complete sentence** (no raw epoch / `undefined` / `null`),
  e.g. `"Bác sĩ Trần Văn A sẽ khám cho bạn lúc 09:30–10:00 22/06/2026 tại Bệnh viện UTE."`.
  Preferred rendering is still `messageKey`/`titleKey` + `data`; this is only a safe fallback.
- No type, event name, `idempotencyKey`, or transport change. FE handlers need no
  migration — reading the new `data` fields is optional polish (e.g. show the doctor
  name and a formatted time directly from `data` instead of refetching the appointment).
