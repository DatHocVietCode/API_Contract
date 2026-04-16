# Notification Socket vs Coin Expiry Socket

## 1. Scope

Tai repo nay, "notification" va "coin expiry socket" khong phai la 2 gateway rieng biet theo cung mot mo hinh.

- Notification system chu yeu luu notification vao Mongo de client co the poll lai.
- Coin expiry socket la realtime fanout layer cho reminder coin sap het han.
- Coin expiry reminder co ca 2 lop:
  - luu notification de fallback
  - push socket de hien thi ngay

## 2. High Level Difference

| Item | Notification flow | Coin expiry socket flow |
| --- | --- | --- |
| Muc tieu | Luu thong bao va cung cap API lay danh sach notification | Push reminder realtime cho client |
| Dau vao | Event business nhu booking success, shift cancel, appointment cancel | Event `notification.coin.expiry.reminder` |
| Cach gui | Store vao Mongo qua `NotificationService` | Emit socket qua namespace `/appointment` va room theo email |
| Push mechanism | Khong co socket gateway rieng cho notification chung | Direct emit + Redis Pub/Sub fanout |
| Fallback | Client poll API `/notifications` | Vua co persist notification, vua co Redis fanout |
| Room key | Khong ap dung | `patientEmail` |
| Socket event | Khong bat buoc | `COIN_EXPIRY_REMINDER` |

## 3. Notification Flow

### 3.1 Business event -> Notification listener

Notification module nhan event tu `@nestjs/event-emitter`.

- Appointment notification listener: [../src/notification/listenners/appointment.notify.listenner.ts](../src/notification/listenners/appointment.notify.listenner.ts)
- Coin expiry notification listener: [../src/notification/listenners/coin-expiry-reminder.notify.listenner.ts](../src/notification/listenners/coin-expiry-reminder.notify.listenner.ts)

Voi coin expiry, listener chi goi service de luu notification:

```ts
this.notificationService.createCoinExpiryReminderNotification(payload);
```

### 3.2 NotificationService -> Mongo

Notification duoc luu trong `NotificationService`:

- File: [../src/notification/notification.service.ts](../src/notification/notification.service.ts)
- Method: `createCoinExpiryReminderNotification(...)`

Du lieu duoc store voi cac truong chinh:

- `receiverEmail`
- `title`
- `message`
- `details`

Voi coin expiry, `details` chua:

- `type = coin_expiry_reminder`
- `jobId`
- `transactionId`
- `amount`
- `expiresAt`
- `runAt`
- `reminderDays`

### 3.3 Client fallback via API

Client co the lay notifications tu API thay vi phu thuoc 100% vao socket.

- Controller: [../src/notification/notification.controller.ts](../src/notification/notification.controller.ts)

Day la phan fallback neu realtime push bi miss.

## 4. Coin Expiry Socket Flow

### 4.1 Queue consumer emits domain event

Khi reminder den han, queue consumer emit event domain:

- File: [../src/wallet/coin/coin-expiry-reminder/coin-expiry-reminder.queue-consumer.ts](../src/wallet/coin/coin-expiry-reminder/coin-expiry-reminder.queue-consumer.ts)
- Event: `notification.coin.expiry.reminder`

Tu event nay, he thong chia ra 2 huong:

1. Luu notification vao Mongo.
2. Push realtime socket.

### 4.2 Persist notification for fallback

Notification listener van luu reminder vao Mongo:

- File: [../src/notification/listenners/coin-expiry-reminder.notify.listenner.ts](../src/notification/listenners/coin-expiry-reminder.notify.listenner.ts)

Muc dich:

- client co the reload trang va poll lai notification
- khong mat thong bao neu socket disconnect

### 4.3 Redis pub/sub for cross-server fanout

Realtime listener publish payload len Redis channel:

- File: [../src/wallet/coin/coin-expiry-reminder/listenners/coin-expiry-reminder-realtime.listenner.ts](../src/wallet/coin/coin-expiry-reminder/listenners/coin-expiry-reminder-realtime.listenner.ts)
- Channel: `coin.expiry.reminder`

Day la bus trung gian cho multi-server.

### 4.4 Redis subscriber -> Socket emit

Redis subscriber nhan payload va emit vao Socket.IO room:

- File: [../src/socket/listenners/coin-expiry-reminder-redis.listenner.ts](../src/socket/listenners/coin-expiry-reminder-redis.listenner.ts)
- Namespace: `/appointment`
- Room: `patientEmail`
- Event: `COIN_EXPIRY_REMINDER`

Cuoi cung payload duoc day toi client dang join room do.

## 5. Direct Socket Path

He thong con co mot direct gateway de push ngay tren cung instance:

- File: [../src/socket/namespace/appointment/appointment-coin-expiry.gateway.ts](../src/socket/namespace/appointment/appointment-coin-expiry.gateway.ts)

Flow:

1. `notification.coin.expiry.reminder` duoc emit.
2. Gateway nhan event va emit thang vao room `patientEmail`.
3. Client trong namespace `/appointment` nhan event `COIN_EXPIRY_REMINDER` ngay lap tuc.

Redis listener van giu vai tro cross-server fallback.

## 6. Push Data Format

Socket push dung `DataResponse` envelope:

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

Quy uoc:

- `expiresAt` va `runAt` la epoch milliseconds UTC.
- Room key luon la `patientEmail`.
- Socket event luon la `COIN_EXPIRY_REMINDER`.

## 7. Notification vs Coin Expiry: How Data Moves

### Notification path

1. Business event phat sinh.
2. Listener tao notification record.
3. Record duoc luu vao Mongo.
4. Client lay lai qua API notification.

### Coin expiry path

1. Queue consumer emit `notification.coin.expiry.reminder`.
2. Notification listener luu notification record.
3. Realtime listener publish payload len Redis.
4. Redis subscriber hoac direct gateway emit socket vao `/appointment`.
5. Client nhan push realtime va van co fallback qua API.

## 8. Why Coin Expiry Needs Both Polling and Socket

- Socket cho response nhanh.
- Mongo notification cho fallback neu client offline.
- Redis pub/sub cho multi-server fanout.
- Direct gateway cho same-instance delivery nhanh hon.

## 9. Sequence Summary

### Notification

`business event -> notification listener -> NotificationService -> Mongo -> API polling`

### Coin expiry socket

`queue consumer -> notification event -> notification storage -> Redis pub/sub -> gateway emit -> client room`

## 10. Practical Note

Neu ban dang tim mot "notification socket" tong quat trong codebase, hien tai phan notification khong co mot socket gateway rieng cho tat ca notification.
Realtime push dang duoc dung theo tung feature, con notification module giu vai tro luu tru va fallback.