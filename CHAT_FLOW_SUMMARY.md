# Chat Feature Summary (Current State)

Last updated: 2026-04-12
Scope: Backend chat implementation in ute-doctor-be and documented contract in api-contract.

## 1. Current Architecture

- HTTP APIs are exposed from ChatController (`/chat/*`).
- Realtime is exposed from ChatGateway under Socket.IO namespace `/chat`.
- Persistence is MongoDB via Mongoose with 2 main collections:
  - `conversations`
  - `messages`
- Queue infrastructure is RabbitMQ (`chat.message.created`) with migration-safe operating modes.
- Realtime fanout can run in direct socket mode or Redis pub/sub mode (`chat.message`).
- Auth:
  - HTTP: `JwtAuthGuard` + `req.user` (`AuthUser`).
  - Socket: JWT is validated by Socket.IO middleware, `socket.data.userId` is set for lifecycle handling, and the normalized payload is exposed to `@WsUser()` through `socket.data.authUser`.

### 1.1 Operating modes (incremental migration)

- `CHAT_WRITE_MODE=dual` (default):
  - Gateway writes DB directly (existing monolith behavior)
  - Gateway also publishes RabbitMQ event for shadow traffic validation
- `CHAT_WRITE_MODE=worker`:
  - Gateway publishes event and ACKs early
  - Worker writes DB, updates conversation snapshot, handles retry/idempotency
- `CHAT_REALTIME_MODE=direct` (default): gateway emits socket directly
- `CHAT_REALTIME_MODE=redis`: worker publishes Redis event, gateway fans out from subscription

## 2. Data Schema (MongoDB)

### 2.1 Conversation schema

Collection fields (from `conversation.schema.ts`):
- `type`: `'direct' | 'group'`, default `'direct'`
- `participants[]`:
  - `accountId` (string, required)
  - `email` (string, optional)
  - `role` (string, required)
  - `lastReadAt` (Date, optional)
- `title` (string, optional)
- `lastMessage` (object, optional)
- `createdAt`, `updatedAt` (timestamps)

Indexes:
- `{ 'participants.accountId': 1, updatedAt: -1 }`

### 2.2 Message schema

Collection fields (from `message.schema.ts`):
- `conversationId` (ObjectId, ref Conversation, required)
- `senderId` (string, required)
- `senderEmail` (string, optional)
- `content` (string, required)
- `type`: `'text' | 'image' | 'file' | 'system'`, default `'text'`
- `clientMessageId` (string, optional)
- `createdAt`, `updatedAt` (timestamps)

Indexes:
- `{ conversationId: 1, createdAt: -1 }`
- `{ clientMessageId: 1 }` unique sparse (idempotency for client retries)

## 3. HTTP Flow (Current)

### 3.1 Create/Upsert conversation
Endpoint: `POST /chat/conversations`

Flow:
1. Validate JWT user from `req.user`.
2. Merge authenticated user into participants (override or append).
3. Call `upsertDirectConversation(participants, title)`.
4. Return existing conversation if found, else create new one.

Important behavior:
- Current lookup uses:
  - `type: 'direct'`
  - `'participants.accountId': { $all: accountIds }`
- This may match a conversation with extra participants if data is corrupted or mixed type.

### 3.2 List conversations
Endpoint: `GET /chat/conversations?skip=&limit=`

Flow:
1. Count and fetch conversations containing current `accountId`.
2. Sort by `updatedAt desc`.
3. Enrich each participant with account/profile info.
4. Return `{ data, total, skip, limit }`.

### 3.3 Get messages
Endpoint: `GET /chat/conversations/:id/messages?before=&limit=`

Flow:
1. Validate ObjectId format.
2. Query messages by `conversationId`.
3. Optional cursor (`before`) is converted to UTC Date.
4. Sort `createdAt desc`, cap `limit` to max 50.

### 3.4 Mark read
Endpoint: `POST /chat/conversations/:id/read`

Flow:
1. Resolve current user from JWT.
2. Update `participants.$[p].lastReadAt = now` with `arrayFilters`.
3. Return updated conversation.

### 3.5 Contact search
Endpoint: `GET /chat/contacts/search?q=&role=&limit=`

Flow:
1. Search accounts by email regex (+ optional role).
2. Search profiles by name regex.
3. Map profile matches back to accounts.
4. Merge and deduplicate results, then return display-ready fields.

## 4. WebSocket Flow (Current)

Namespace: `/chat`

Supported events:
- `CHAT_JOIN_USER`
- `CHAT_LEAVE_USER`
- `CHAT_JOIN_CONVERSATION`
- `CHAT_LEAVE_CONVERSATION`
- `CHAT_MESSAGE_SEND`
- `CHAT_MESSAGE_RECEIVED`
- `CHAT_MESSAGE_READ`

### 4.1 Connection and auth
1. Client connects to `/chat` with JWT.
2. Socket middleware verifies token before the gateway runs.
3. `socket.data.userId` and `socket.data.authUser` are attached to the socket context.

### 4.1.1 Old vs new handshake model (important for FE integration)

Old model (legacy mindset):
1. Connect socket.
2. Emit `JOIN_ROOM`.
3. Wait for `ROOM_JOINED`.
4. Start business exchange.

New model (current):
1. Connect with `handshake.auth.token`.
2. Middleware is the authentication gate and accepts/rejects before gateway handlers.
3. Presence lifecycle is handled on connect/disconnect using `socket.data.userId`.
4. For `/chat`, client should use `CHAT_JOIN_USER` and `CHAT_JOIN_CONVERSATION` as room events.
5. `JOIN_ROOM` is not required for `/chat` and is not the global gate anymore.

### 4.2 Room model
- Personal room: `user:{accountId}`
- Conversation room: `conv:{conversationId}`

### 4.3 Send message flow (dual mode)
1. Client emits `CHAT_MESSAGE_SEND` with `conversationId`, `content`, optional `clientMessageId`.
2. Gateway uses JWT identity as sender (not client-provided senderId).
3. Service saves message document.
4. Service updates `conversation.lastMessage` and `updatedAt`.
5. Gateway publishes `chat.message.created` event to RabbitMQ.
6. Realtime fanout:
  - direct mode: gateway emits `CHAT_MESSAGE_RECEIVED`
  - redis mode: gateway publishes to Redis channel and fanout happens from subscription callback

### 4.4 Send message flow (worker mode)
1. Gateway publishes `chat.message.created` event and emits `CHAT_MESSAGE_DELIVERED` ACK immediately.
2. Worker consumes queue event.
3. Worker checks idempotency by `clientMessageId`.
4. Worker writes message + updates conversation snapshot.
5. Worker publishes realtime payload to Redis channel `chat.message`.
6. Gateway receives Redis message and emits `CHAT_MESSAGE_RECEIVED` to rooms.

### 4.5 Retry and dead-letter behavior
- Worker retries failed message processing up to `CHAT_QUEUE_MAX_RETRY`.
- When retries are exhausted, event is pushed to `chat.message.created.dlq` for investigation/replay.

### 4.6 Read flow
1. Client emits `CHAT_MESSAGE_READ` with `conversationId`.
2. Service updates participant `lastReadAt`.
3. Gateway broadcasts read event to conversation room.

## 5. Current Strengths

- JWT-based identity on both HTTP and Socket paths.
- Message idempotency support via `clientMessageId` unique sparse index.
- Clear room strategy for realtime fanout.
- Read-state model (`lastReadAt`) is already in participant schema.

## 6. Gaps and Risks

### 6.1 Authorization gaps
- `GET /chat/conversations/:id/messages` does not verify requester is participant.
- `POST /chat/conversations/:id/read` does not explicitly verify participant ownership before update.
- Socket `CHAT_JOIN_CONVERSATION` and `CHAT_MESSAGE_SEND` do not enforce participant membership server-side.

Impact:
- Potential data exposure if a valid user guesses conversation id.

### 6.2 Upsert matching accuracy
- `$all` without participant count guard can be ambiguous for direct conversations.

### 6.3 Performance (N+1)
- `listConversationsByUser` does per-participant account/profile queries.

### 6.4 Contract completeness
- `api.md` chat section currently lists endpoints, but lacks:
  - request/response examples for each endpoint
  - error cases
  - socket event contract table (payload + ack/error)

### 6.5 Read model detail
- `lastReadAt` exists, but unread count is not materialized and not returned explicitly.

## 7. Recommended Improvements

## 7.1 Priority 0 (security + correctness)

1. Enforce participant membership checks on:
- HTTP get messages
- HTTP mark read
- Socket join conversation
- Socket send/read events

2. Harden direct upsert query:
- Add exact participant count condition (e.g. size = 2 for direct)
- Or compute deterministic `directKey` from sorted participant ids and index unique on it.

## 7.2 Priority 1 (performance)

1. Replace N+1 enrichment with aggregate lookup pipeline.
2. Add pagination cursor for conversations (updatedAt + _id) to avoid deep skip.
3. Add optional index support for read-related queries if unread counts are introduced.

## 7.3 Priority 2 (product + DX)

1. Expand contract docs with:
- endpoint examples
- error matrix
- socket event payload contracts
2. Add delivery/typing flow (events already declared but not fully used).
3. Add explicit acknowledgement format for socket send errors.

## 8. Suggested Next Refactor Shape

- Keep message source of truth in `messages`.
- Keep conversation summary in `conversations.lastMessage`.
- Introduce `ConversationParticipantState` style fields if needed:
  - `lastReadAt`
  - `lastDeliveredAt` (optional)
  - `mutedUntil` (optional)

This keeps current model compatible while enabling unread counters and richer inbox behavior.

## 9. Practical Checklist (Implementation)

- Add `assertConversationParticipant(conversationId, accountId)` helper in ChatService.
- Call this helper in all read/send/join/read-receipt paths.
- Patch upsert query to exact match for direct conversation.
- Replace participant enrichment with one aggregation pipeline.
- Update `api-contract/api.md` with chat request/response and socket event sections.
