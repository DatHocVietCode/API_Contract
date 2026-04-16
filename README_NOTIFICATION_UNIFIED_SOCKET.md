# Unified Notification Socket Contract

## 1. Goal

Notification realtime is standardized to one socket event so FE only subscribes once.

- Namespace: `/notification`
- Socket event: `NOTIFICATION_RECEIVED`
- Room strategy: email room from JWT via `JOIN_ROOM`

## 2. Notification Type Map

```ts
export type NotificationMap = {
  COIN_EXPIRY_REMINDER: CoinExpiryDto;
  APPOINTMENT_SUCCESS: AppointmentSuccessDto;
  APPOINTMENT_CANCELLED: AppointmentCancelledDto;
  PAYMENT_SUCCESS: PaymentSuccessDto;
};
```

## 3. Typed Envelope

```ts
export type NotificationPayload = {
  [K in keyof NotificationMap]: {
    type: K;
    data: NotificationMap[K];
    createdAt: number;
    recipientEmail: string;
    idempotencyKey: string;
  }
}[keyof NotificationMap];
```

Rules:
- `type` is discriminant key for FE handler registry.
- `data` keeps domain DTO shape, no flattening.
- `createdAt` is epoch milliseconds UTC.
- `recipientEmail` is target room key.
- `idempotencyKey` is unique business dedup key.

## 4. Transport Pipeline

Notification flow is now:

1. Domain listeners publish `NotificationPayload` to RabbitMQ queue `notification.jobs`.
2. Notification consumer processes queue payload.
3. Handler registry resolves handler by `payload.type`.
4. Handler persists notification in MongoDB with idempotency key.
5. Handler publishes the same typed envelope to Redis channel `notification`.
6. Notification socket bridge subscribes Redis and emits `NOTIFICATION_RECEIVED` to recipient room.

## 5. Backward Compatibility

Legacy domain socket events are temporarily kept for compatibility:
- `COIN_EXPIRY_REMINDER`
- `APPOINTMENT_BOOKING_SUCCESS`
- `APPOINTMENT_BOOKING_PENDING`
- `APPOINTMENT_BOOKING_FAILED`
- `APPOINTMENT_CANCELLED`
- `SHIFT_CANCELLED`
- `PAYMENT_UPDATE`
- `PAYMENT_VNPAY_URL_CREATED`

FE should migrate bell/update logic to only:
- connect `/notification`
- listen `NOTIFICATION_RECEIVED`
- dispatch by `payload.type`

## 6. FE Handler Pattern (No switch-case)

```ts
const handlers = {
  COIN_EXPIRY_REMINDER: handleCoinExpiry,
  APPOINTMENT_SUCCESS: handleAppointmentSuccess,
  APPOINTMENT_CANCELLED: handleAppointmentCancelled,
  PAYMENT_SUCCESS: handlePaymentSuccess,
} as const;

socket.on('NOTIFICATION_RECEIVED', (payload: NotificationPayload) => {
  handlers[payload.type]?.(payload.data, payload);
});
```

## 7. Type Definitions (Current)

```ts
type AppointmentCancelledDto = {
  appointmentId: string;
  patientEmail: string;
  doctorEmail?: string;
  date: string;
  timeSlot: string;
  timeSlotLabel?: string;
  hospitalName?: string;
  reason?: string;
  refundAmount?: number;
  shouldRefund?: boolean;
};

type PaymentSuccessDto = {
  orderId: string;
  status: 'COMPLETED';
};
```

`CoinExpiryDto` and `AppointmentSuccessDto` reuse existing BE DTO structures.
