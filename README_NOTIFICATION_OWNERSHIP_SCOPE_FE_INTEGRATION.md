# Notification Ownership / Scope — FE Integration & Bug-Check Plan

## 0. Context

Reported bug: **the shared notification bell (used by both Patient and Doctor) may show a
Doctor notifications that belong to a Patient** (or vice versa), including an inflated unread
count.

A backend investigation was completed (`ute-doctor-be/docs/notification-backend-analysis.md`).
This document is the **frontend-facing** summary + an integration/verification plan so the FE team
can confirm where the leak actually comes from and react to the upcoming BE hardening.

**No API contract shape changes are introduced by this work.** Endpoints, payloads, and the
`NOTIFICATION_RECEIVED` socket envelope are unchanged. Only backend scoping/guards change.

---

## 1. Backend conclusion (what FE can rely on)

Notification ownership is keyed by **email**, and **account email is unique** → one email = one
account = one role. Consequently:

- `GET /notifications/by-email` and `GET /notifications/count` are **guarded (JWT) and filtered by
  the authenticated user's email**. With distinct accounts (different emails), these **cannot**
  return another user's notifications.
- Realtime delivery (`/notification` namespace, `NOTIFICATION_RECEIVED`) is emitted to a
  **per-user email room derived from the JWT only** — clients cannot join another user's room.
- All notification creation uses the correct recipient email per event (no patient/doctor mix-up).

**Therefore, if the bell shows the wrong user's notifications while two _distinct_ accounts are
involved, the backend scoped endpoints are not returning them — the cause is on the FE side**
(identity/token or cached state), which this plan helps confirm.

> The backend does have two real but **separate** leaks that the bell does not use:
> the global `GET /notifications` (being secured) and an ownership-less `PATCH /:id/read`
> (being fixed). These do not affect the bell, but see §4 for the one behavior change FE should
> tolerate.

---

## 2. Endpoints the bell MUST use (and must NOT use)

| Purpose | Endpoint | Auth | Scope |
|---------|----------|------|-------|
| List notifications | `GET /notifications/by-email?page&limit` | Bearer JWT (required) | Current user's email |
| Unread count | `GET /notifications/count` | Bearer JWT (required) | Current user's email |
| Mark one read | `PATCH /notifications/:id/read` | Bearer JWT (required) | Will be ownership-checked (see §4) |

**Do NOT call `GET /notifications`** (no `/by-email`, no `/count`). That endpoint is global/legacy
and is being secured server-side; the bell and notification center must never use it. Confirm via
grep that the FE has zero references to a bare `/notifications` GET. (As of this writing the FE
already uses only the three scoped calls above — keep it that way.)

Every notification request (REST and the socket handshake `auth.token`) must carry the
**currently-logged-in** user's token.

---

## 3. FE verification checklist (confirm where the leak comes from)

Run with two distinct accounts in the **same browser**, since role-switching in one browser is the
most common trigger.

1. **Token identity on the leaking request**
   - Open DevTools → Network. Trigger the bell as the Doctor.
   - Inspect `GET /notifications/by-email` and `/count`: decode the `Authorization: Bearer` JWT and
     confirm its `email`/`role` is the **Doctor's**, not a stale Patient token.
   - Inspect the `/notification` socket handshake: confirm `auth.token` is the Doctor's.
   - ➜ If a Patient token is being sent while logged in as Doctor: **FE auth/token bug** (token not
     refreshed/replaced on login, or read from a stale store).

2. **Response vs. render**
   - Compare the raw JSON returned by `/by-email` against what the bell renders.
   - ➜ If the response already contains only Doctor notifications but the UI shows Patient ones:
     **FE state/cache bug** (previous user's notifications not cleared).

3. **Cache/state lifecycle**
   - Confirm notification list + unread count are **reset on logout AND on login** (not merged with
     the previous session). Check any React Query/SWR/Zustand/Context store keyed without the user
     identity.
   - ➜ Persisting notifications across account switches under a shared key = **FE cache bug**.

4. **Same-email edge case (not a leak)**
   - If both "users" are actually the **same email/account** acting in two roles, merged
     notifications are expected (same person). Confirm the two accounts truly differ by email.

5. **Global endpoint sanity**
   - Confirm the FE never calls bare `GET /notifications`. If it does anywhere, that path returns
     everyone's notifications today — replace it with `/by-email`.

Record which step reproduces. Steps 1–3 ⇒ FE fix required. None reproducing with distinct accounts
⇒ not reproducible on current FE; keep watching the BE hardening in §4.

---

## 4. Upcoming backend changes FE should tolerate (no contract change)

These are planned BE fixes. They do not change response shapes, but FE should handle them
gracefully:

- **`PATCH /notifications/:id/read` becomes ownership-checked.** Marking a notification the current
  user does not own will return **403/404** instead of succeeding. FE should:
  - only ever call it for items in the user's own list (already the case), and
  - treat a non-2xx as "leave item unread; do not decrement the unread badge."
- **Email matching becomes case-insensitive on the BE.** Users whose email had uppercase letters
  may have previously seen **zero** notifications/`count: 0` over REST while realtime still worked.
  After the fix the list/count will correctly populate — no FE change needed, but expect counts to
  "appear" for affected accounts.
- **`GET /notifications` (global) is being guarded + scoped to the caller.** The FE does not use it;
  no action required beyond never adding a call to it.

---

## 5. Suggested FE fixes (if §3 reproduces)

- **Reset notification state on auth boundary:** clear bell list, unread count, and any cached query
  on both logout and login. If using React Query/SWR, include the user id/email in the cache key so
  a different account cannot read a previous account's cache.
- **Single source of truth for the token:** ensure axios and the socket client both read the
  current access token at request time (interceptor / fresh getter), not a value captured at module
  load or from a previous session.
- **Reconnect the socket on account switch:** disconnect the `/notification` socket on logout and
  reconnect with the new `auth.token` on login so the email room reflects the new identity.
- **Defensive UI:** the bell already filters by `isRead` client-side; keep correctness anchored on
  the scoped REST responses, not on optimistic local merges.

---

## 6. Acceptance scenarios (manual)

- Login as Patient A, generate a notification → bell shows it; count = 1.
- Switch to Doctor B in the same browser → bell shows **only** B's notifications; A's is absent;
  count excludes A's.
- Switch back to Patient A → A still sees their notification.
- As Doctor B, attempt to mark A's notification id read (e.g., via a crafted request) → **rejected**
  (403/404) after the BE fix; A's notification stays unread.
- Both A and B online → a private notification for A arrives only on A's bell in realtime.

---

## 7. References

- Backend analysis: `ute-doctor-be/docs/notification-backend-analysis.md`
- Realtime contract (unchanged): `README_NOTIFICATION_UNIFIED_SOCKET.md`
- Coin-expiry vs notification socket: `README_NOTIFICATION_VS_COIN_EXPIRY_SOCKET.md`
</content>
