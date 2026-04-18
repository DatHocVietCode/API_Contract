# BE Socket Refactor Guide

Last updated: 2026-04-18
Scope: Backend Socket.IO architecture refactor in `ute-doctor-be`.
Audience: FE integrators and BE contributors.

## 1. Why This Refactor Exists

The old socket layer mixed authentication, connection lifecycle, and business logic in gateway classes.
That design worked for simple cases but caused fragile integration and poor scale behavior with multi-device users and multi-instance deployment.

The new design separates concerns:
- Middleware: JWT authentication only.
- Gateway: connection lifecycle and event routing only.
- Service: Redis-backed presence business logic.

## 2. Old Design (Before Refactor)

### 2.1 Old Connection Flow

1. Client connected to namespace.
2. Client emitted `JOIN_ROOM`.
3. Server emitted `ROOM_JOINED`.
4. Business events were expected to start after that ack.

### 2.2 Old Pros

- Simple mental model for FE in small systems.
- Easy to debug in one gateway file.
- Fast to implement for email-room pushes.

### 2.3 Old Cons

- Authentication and business logic were coupled in gateway-level code.
- Room join became a global handshake gate even where it was not needed.
- Presence state had no dedicated service boundary.
- Hard to scale and hard to enforce consistency across namespaces.
- Easy to introduce race conditions (request triggers before room is joined).

## 3. New Design (Current)

### 3.1 New Connection Flow

1. Client connects with `handshake.auth.token`.
2. Socket middleware verifies JWT.
3. Middleware attaches identity (`socket.data.userId`, `socket.data.authUser`).
4. Gateway `handleConnection` runs and updates presence in Redis.
5. Client uses namespace-specific room events:
- Email-room namespaces: `JOIN_ROOM` then wait `ROOM_JOINED`.
- Chat namespace: `CHAT_JOIN_USER`, `CHAT_JOIN_CONVERSATION`.
6. Client emits periodic `heartbeat` for long-lived connections.

`JOIN_ROOM` is no longer the global handshake gate.

### 3.2 New Pros

- Clear separation of concerns.
- Authentication is centralized and reusable.
- Presence tracking is Redis-based and multi-instance ready.
- Better support for multi-device users.
- Easier to extend to future realtime domains.
- Better observability with structured logs in middleware and gateway.

### 3.3 New Cons / Trade-offs

- More moving parts (adapter + middleware + gateway + service).
- FE must follow namespace-specific join flow instead of one global flow.
- Integration can fail silently if FE does not send handshake token correctly.
- Presence uses TTL fallback so heartbeat timing must be maintained.

## 4. Where Logic Is Handled

### 4.1 Authentication Layer

- File: `src/socket/middleware/socket-auth.middleware.ts`
- Responsibility:
- Read token from `socket.handshake.auth.token`.
- Verify JWT.
- Attach normalized identity to socket data.
- Reject unauthorized connections.

### 4.2 Socket Adapter Registration

- File: `src/socket/socket.adapter.ts`
- Responsibility:
- Attach auth middleware at Socket.IO server level.
- Attach auth middleware explicitly for known namespaces.

### 4.3 Gateway Lifecycle Layer

- File: `src/socket/base/base.gateway.ts`
- Responsibility:
- `handleConnection` and `handleDisconnect`.
- `JOIN_ROOM` handling for email-room namespaces.
- `HEARTBEAT` reception and delegation to PresenceService.

### 4.4 Presence Business Logic

- File: `src/socket/presence.service.ts`
- Responsibility:
- Add/remove socket connections in Redis sets.
- Refresh TTL on heartbeat.
- Expose online status checks.
- Emit heartbeat diagnostics from Redis scans.

## 5. Redis Presence Model

### 5.1 Data Structures

- `user:{userId}:devices` (SET of socket ids)
- `online_users` (SET of user ids)

### 5.2 Add Connection

1. `SADD user:{userId}:devices socketId`
2. `EXPIRE user:{userId}:devices <ttlSeconds>`
3. `SADD online_users userId`

### 5.3 Remove Connection

1. `SREM user:{userId}:devices socketId`
2. If `SCARD user:{userId}:devices == 0`:
- `DEL user:{userId}:devices`
- `SREM online_users userId`

### 5.4 Heartbeat Refresh

1. Client emits `heartbeat`.
2. Gateway validates `socket.data.userId`.
3. PresenceService calls `EXPIRE` on user device key.
4. PresenceService logs Redis state snapshot:
- TTL after refresh
- Device set count
- Membership in `online_users`
- Current socket ids in device set

TTL is a safety fallback. Source of truth is the device set.

## 6. Current Namespace Rules

### 6.1 Namespaces That Still Use `JOIN_ROOM`

- `/appointment`
- `/appointment/fields-data`
- `/payment/vnpay`
- `/patient-profile`
- `/notification`

Rule: connect -> emit `JOIN_ROOM` -> wait `ROOM_JOINED` -> trigger related business request.

### 6.2 Chat Namespace

- Namespace: `/chat`
- Join model:
- `CHAT_JOIN_USER` for personal room `user:{accountId}`
- `CHAT_JOIN_CONVERSATION` for room `conv:{conversationId}`

Rule: do not treat `JOIN_ROOM` as mandatory handshake for chat.

## 7. Patient Profile Special Case

`GET /patients/me` currently triggers an event-driven pipeline and returns a pending-style response quickly.
The full profile payload is emitted via socket event `PATIENT_PROFILE` to the email room.

Flow summary:
1. FE connects `/patient-profile` with token.
2. FE emits `JOIN_ROOM` and waits `ROOM_JOINED`.
3. FE calls `GET /patients/me`.
4. BE saga builds profile data.
5. BE emits `PATIENT_PROFILE` through `/patient-profile` room.

If step 2 is skipped or raced, FE can miss the push.

## 8. FE Integration Checklist (Recommended)

1. Always pass JWT in `handshake.auth.token`.
2. Wait for socket `connect` before room events.
3. For email-room namespaces, wait `ROOM_JOINED` before triggering dependent HTTP flow.
4. For chat, use chat-specific join events.
5. Emit `heartbeat` every 25-30 seconds on active sockets.
6. Treat disconnect and reconnect as full lifecycle; rejoin required rooms.
7. Keep HTTP fallback for critical data if realtime event is missed.

## 9. Troubleshooting Signals

### 9.1 Auth Not Applied

Symptom:
- Gateway connection log shows `userId=missing`.

Likely cause:
- Missing token in handshake or middleware not applied to that namespace.

### 9.2 Join Acknowledgement Missing

Symptom:
- No `ROOM_JOINED` on FE.

Likely cause:
- `JOIN_ROOM` not emitted, emitted too early, or auth payload has no email.

### 9.3 Heartbeat Not Working

Symptom:
- Frequent disconnect/online flapping.

Checks:
- Confirm `heartbeat` events are received in gateway logs.
- Confirm PresenceService heartbeat logs show positive `expireResult` and valid TTL.

## 10. Summary

The refactor moves socket authentication and presence into a scalable architecture:
- Middleware gates identity.
- Gateway handles lifecycle and routing.
- PresenceService owns Redis behavior.

This improves reliability and extensibility, but FE must follow the updated lifecycle strictly.
