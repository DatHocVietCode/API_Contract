# Coin Expiry Reminder Contract

## 1. Scope

Feature này chỉ xử lý:
- Reminder trước khi coin hết hạn.
- Gửi email reminder.
- Realtime notification qua Redis Pub/Sub -> Gateway -> WebSocket.

Feature này KHONG xử lý earn business logic.

## 2. Runtime Configuration

Nguồn cấu hình ở [../src/wallet/coin-expiry-reminder/coin-expiry-reminder.constants.ts](../src/wallet/coin-expiry-reminder/coin-expiry-reminder.constants.ts#L8).

- `RABBITMQ_QUEUE_NAME` -> mặc định `coin.expiry.reminder.jobs` (L8)
- `RABBITMQ_EXCHANGE` -> mặc định `coin.expiry.reminder.jobs` (L9)
- `COIN_EXPIRY_REMINDER_QUEUE` -> override queue riêng cho coin reminder (fallback `RABBITMQ_QUEUE_NAME`)
- `COIN_EXPIRY_REMINDER_EXCHANGE` -> override exchange riêng cho coin reminder (fallback `RABBITMQ_EXCHANGE`)
- `COIN_EXPIRY_REMINDER_REDIS_CHANNEL` -> channel pub/sub Redis cho realtime fanout
- `COIN_EXPIRY_REMINDER_DAYS` -> số ngày nhắc trước khi hết hạn (fallback `EXPIRY_REMINDER_DAYS`)
- `COIN_EXPIRY_REMINDER_SCHEDULER_INTERVAL_MS` -> polling interval scheduler (fallback `SCHEDULER_INTERVAL_MS`)
- `COIN_EXPIRY_REMINDER_MAX_RETRY` -> số lần retry worker (fallback `MAX_RETRY`)
- `COIN_EXPIRY_REMINDER_DISPATCH_BUFFER_MS` -> buffer để tránh miss job đúng ngưỡng
- `COIN_EXPIRY_REMINDER_DISPATCH_LOCK_TTL_SECONDS` -> TTL lock dispatch
- `COIN_EXPIRY_REMINDER_PROCESS_LOCK_TTL_SECONDS` -> TTL lock process

## 3. Mongo Source of Truth

### 3.1 Collection

Collection schedule job dùng schema [../src/wallet/coin-expiry-reminder/schemas/coin-job-schedule.schema.ts](../src/wallet/coin-expiry-reminder/schemas/coin-job-schedule.schema.ts#L7).

Field chính:
- `jobId` (unique)
- `transactionId`
- `patientId`
- `type = COIN_EXPIRY_REMINDER`
- `runAt`
- `status = PENDING | DONE | FAILED`
- `retryCount`
- `lastError`
- `createdAt`, `updatedAt`

### 3.2 Index

Index polling theo yêu cầu business nằm tại [../src/wallet/coin-expiry-reminder/schemas/coin-job-schedule.schema.ts](../src/wallet/coin-expiry-reminder/schemas/coin-job-schedule.schema.ts#L44):
- `{ status: 1, runAt: 1 }`

## 4. Lifecycle By Phase

### Phase A - Earn transaction created -> Create reminder job

#### A1. Entry point: add coin
- Hàm: `addCoins(...)`
- Vị trí: [../src/wallet/coin.service.ts](../src/wallet/coin.service.ts#L119)
- Handler:
  - Tạo earn transaction.
  - Gọi `createReminderJobForEarnTransaction(earnTransaction)` tại [../src/wallet/coin.service.ts](../src/wallet/coin.service.ts#L160).

#### A2. Entry point: reward sau complete appointment
- Hàm: `rewardCoinForCompletedAppointment(...)`
- Vị trí: [../src/wallet/coin.service.ts](../src/wallet/coin.service.ts#L610)
- Handler:
  - Tạo reward transaction trong transaction/session.
  - Gọi `createReminderJobForEarnTransaction(rewardTransaction, session)` tại [../src/wallet/coin.service.ts](../src/wallet/coin.service.ts#L711).

#### A3. Job creation logic
- Hàm: `createReminderJobForEarnTransaction(...)`
- Vị trí: [../src/wallet/coin-expiry-reminder/coin-expiry-reminder.service.ts](../src/wallet/coin-expiry-reminder/coin-expiry-reminder.service.ts#L64)
- Logic:
  - Nếu `expiresAt` khong ton tai -> bo qua.
  - Tinh `runAt = expiresAt - EXPIRY_REMINDER_DAYS`.
  - Upsert job status `PENDING`, `retryCount = 0`.
  - `jobId = transactionId` de dam bao idempotency.

### Phase B - Scheduler polling

#### B1. Startup scheduler
- Hàm: `onModuleInit()`
- Vị trí: [../src/wallet/coin-expiry-reminder/coin-expiry-reminder.scheduler.service.ts](../src/wallet/coin-expiry-reminder/coin-expiry-reminder.scheduler.service.ts#L22)
- Logic:
  - `ensureTopology()`.
  - Dispatch ngay 1 lan.
  - Bat interval theo `SCHEDULER_INTERVAL_MS` (L26).

#### B2. Poll due jobs
- Hàm: `dispatchDueJobs()`
- Vị trí: [../src/wallet/coin-expiry-reminder/coin-expiry-reminder.scheduler.service.ts](../src/wallet/coin-expiry-reminder/coin-expiry-reminder.scheduler.service.ts#L38)
- Logic:
  - Query due jobs qua `listDueJobs(...)` (service L94) voi cutoff nho (buffer 1 phut).
  - Batch limit 100.
  - Acquire dispatch lock Redis de tranh duplicate publish.
  - Publish message qua `publishDispatch(...)` (service L152).
  - KHONG mark DONE tai scheduler.

### Phase C - RabbitMQ topology and queue wiring

#### C1. Topology setup
- Hàm: `ensureTopology()`
- Vị trí: [../src/wallet/coin-expiry-reminder/coin-expiry-reminder.service.ts](../src/wallet/coin-expiry-reminder/coin-expiry-reminder.service.ts#L36)
- Logic:
  - Assert DLX exchange.
  - Assert DLQ queue.
  - Bind DLQ queue vao DLX.
  - Kiem tra/tao main queue.

#### C2. Dead-letter publish path
- Hàm: `publishDeadLetter(...)`
- Vị trí: [../src/wallet/coin-expiry-reminder/coin-expiry-reminder.service.ts](../src/wallet/coin-expiry-reminder/coin-expiry-reminder.service.ts#L156)
- Logic:
  - Publish message that bai sang DLX routing key DLQ.

### Phase D - Consumer execution and idempotency

#### D1. Consumer bootstrap
- Hàm: `onModuleInit()`
- Vị trí: [../src/wallet/coin-expiry-reminder/coin-expiry-reminder.queue-consumer.ts](../src/wallet/coin-expiry-reminder/coin-expiry-reminder.queue-consumer.ts#L45)
- Logic:
  - Attach consumer vao `COIN_EXPIRY_REMINDER_QUEUE`.

#### D2. Main handler
- Hàm: `handleDispatch(payload)`
- Vị trí: [../src/wallet/coin-expiry-reminder/coin-expiry-reminder.queue-consumer.ts](../src/wallet/coin-expiry-reminder/coin-expiry-reminder.queue-consumer.ts#L61)
- Idempotency:
  - Load job theo `jobId`.
  - Neu `status = DONE | FAILED` -> skip.
  - Acquire process lock Redis de tranh xu ly dong thoi.

#### D3. Success path
- Gọi `markJobDone(jobId)` tại [../src/wallet/coin-expiry-reminder/coin-expiry-reminder.queue-consumer.ts](../src/wallet/coin-expiry-reminder/coin-expiry-reminder.queue-consumer.ts#L121).

#### D4. Failure path
- Tang retry qua `markJobRetry(...)` tại [../src/wallet/coin-expiry-reminder/coin-expiry-reminder.queue-consumer.ts](../src/wallet/coin-expiry-reminder/coin-expiry-reminder.queue-consumer.ts#L125).
- Neu retry < `MAX_RETRY` -> republish lai queue.
- Neu retry vuot nguong -> `markJobFailed(...)` tại [../src/wallet/coin-expiry-reminder/coin-expiry-reminder.queue-consumer.ts](../src/wallet/coin-expiry-reminder/coin-expiry-reminder.queue-consumer.ts#L143), dong thoi publish DLQ.

### Phase E - Event fanout (Email + Notification)

#### E1. Emit domain events
Trong `handleDispatch(...)` consumer:
- Emit `mail.coin.expiry.reminder`
- Emit `notification.coin.expiry.reminder`

#### E2. Email handler
- Listener: [../src/mail/mail.listenner.ts](../src/mail/mail.listenner.ts#L85)
- Service send mail: [../src/mail/mail.service.ts](../src/mail/mail.service.ts#L184)

#### E3. Notification -> Redis Pub/Sub
- Listener publish redis: [../src/wallet/coin-expiry-reminder/listenners/coin-expiry-reminder-realtime.listenner.ts](../src/wallet/coin-expiry-reminder/listenners/coin-expiry-reminder-realtime.listenner.ts#L11)
- Redis channel: `coin.expiry.reminder`

### Phase F - Redis -> Gateway -> WebSocket

#### F1. Gateway-side subscriber
- Socket listener subscribe Redis: [../src/socket/listenners/coin-expiry-reminder.listenner.ts](../src/socket/listenners/coin-expiry-reminder.listenner.ts#L20)
- Emit to room by patient email: [../src/socket/listenners/coin-expiry-reminder.listenner.ts](../src/socket/listenners/coin-expiry-reminder.listenner.ts#L31)

#### F2. Socket event contract
- Event name: `COIN_EXPIRY_REMINDER`
- Enum: [../src/common/enum/socket-events.enum.ts](../src/common/enum/socket-events.enum.ts#L31)

Payload format (`DataResponse.data`) trả về cho client:
```json
{
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
```

Quy ước thời gian:
- `expiresAt` và `runAt` là epoch milliseconds UTC.
- Client không parse thời gian từ `message`; luôn render từ epoch fields.

## 5. Module Bootstrap Graph

### Wallet module registration
Reminder providers duoc register trong [../src/wallet/wallet.module.ts](../src/wallet/wallet.module.ts#L45):
- `CoinExpiryReminderService`
- `CoinExpiryReminderSchedulerService`
- `CoinExpiryReminderQueueConsumer`
- `CoinExpiryReminderRealtimeListener`

### Socket module registration
Redis -> WebSocket bridge provider:
- [../src/socket/socket.module.ts](../src/socket/socket.module.ts#L21)

## 6. Failure Semantics

- RabbitMQ unavailable:
  - Job van nam trong Mongo `PENDING`.
  - Scheduler se tiep tuc poll va publish lai khi queue available.
- Worker crash:
  - Job chua mark DONE -> co the duoc xu ly lai.
  - Lock + status guard giup tranh double-process.
- Duplicate message:
  - Guard theo `jobId`, `status`, va process lock.
- System restart:
  - Scheduler khoi dong lai, poll tu Mongo source-of-truth.

## 7. Sequence Summary

1. Coin earn transaction duoc ghi.
2. Tao job `PENDING` voi `runAt = expiresAt - N ngay`.
3. Scheduler poll due jobs theo index `{status, runAt}`.
4. Scheduler publish payload len RabbitMQ queue.
5. Consumer xu ly payload, emit event email + notification.
6. Success -> job `DONE`; failure -> retry, het retry -> `FAILED` + DLQ.
7. Notification event publish qua Redis, gateway fanout WebSocket theo room email.
