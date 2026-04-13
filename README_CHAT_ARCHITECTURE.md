# Chat System Architecture Refactor (RabbitMQ + Redis)

## 1. Overview

He thong chat ban dau duoc xay dung theo mo hinh monolith voi xu ly dong bo (synchronous).
Cach tiep can nay don gian nhung gap nhieu han che khi he thong can scale.

Giai phap moi ap dung kien truc bat dong bo (asynchronous) su dung RabbitMQ va Redis nham cai thien kha nang mo rong, do tin cay va hieu nang.

## 2. Old Architecture (Direct Socket)

### Flow

User A -> Server -> Emit Socket -> User B

### Drawbacks

- Khong ho tro multi-server
- Message co the bi mat khi server crash
- Khong co co che retry
- DB bi overload khi traffic cao
- Tight coupling giua cac tang (socket, DB, logic)

## 3. New Architecture (RabbitMQ + Redis)

### Flow

1. User gui message toi Gateway
2. Gateway tra ACK ngay lap tuc cho client
3. Gateway publish message vao RabbitMQ
4. Worker consume message tu queue
5. Worker xu ly:
   - Luu message vao database
   - Cap nhat conversation
6. Worker publish event vao Redis
7. Gateway subscribe Redis va emit toi client

## 4. Role of Components

### 4.1 RabbitMQ

RabbitMQ dong vai tro la message queue trung gian.

#### Loi ich

- Buffer traffic (chong spike)
- Retry khi xu ly that bai
- Decouple Gateway va Business Logic
- Dam bao message khong bi mat (durability)

### 4.2 Redis Pub/Sub

Redis duoc su dung nhu mot event bus cho realtime communication.

#### Loi ich

- Dong bo message giua nhieu server
- Ho tro fan-out (1 -> nhieu client)
- Tach worker khoi socket layer
- Scale he thong theo chieu ngang

## 5. Dual Mode (Migration Strategy)

He thong ho tro 2 mode:

### Dual Mode

- Gateway:
  - Ghi DB truc tiep
  - Publish queue
- Worker:
  - Khong xu ly

Muc dich: test queue ma khong anh huong he thong hien tai.

### Worker Mode

- Gateway:
  - Chi publish queue
- Worker:
  - Xu ly toan bo logic

Day la mode production muc tieu sau migration.

## 6. Retry Mechanism

Khi worker xu ly message that bai:

- Tang `retryCount`
- Re-publish message vao queue
- Retry toi da N lan (configurable)
- Sau do chuyen vao Dead Letter Queue (DLQ)

Khong su dung recursion, ma la re-enqueue message.

## 7. Idempotency (Duplicate Handling)

Do co che retry, message co the duoc xu ly nhieu lan.

Giai phap:

- Su dung `clientMessageId` lam unique key
- Kiem tra ton tai truoc khi insert

Dam bao he thong khong tao duplicate message.

## 8. Benefits of New Architecture

- Ho tro multi-server
- Khong mat message
- Giam tai DB
- Xu ly bat dong bo
- Scale tot hon
- Tach biet cac thanh phan he thong

## 9. Trade-offs

- Tang do phuc tap he thong
- Eventual consistency (ACK truoc, DB sau)
- Debug kho hon
- Redis Pub/Sub khong dam bao delivery

## 10. Real-world Scenarios

### Case 1: Multi-server

- Old: message bi miss
- New: Redis dam bao sync

### Case 2: Server crash

- Old: mat message
- New: message van nam trong queue

### Case 3: High traffic

- Old: DB bi overload
- New: queue buffer giup on dinh

## 11. Conclusion

Kien truc moi chuyen he thong tu synchronous monolith sang asynchronous distributed system.

- RabbitMQ dam bao durability va retry
- Redis dam bao realtime communication

Giai phap nay giup he thong scale tot hon va phu hop voi production environment, tuy nhien can danh doi bang viec tang do phuc tap va yeu cau xu ly cac van de nhu idempotency va consistency.
