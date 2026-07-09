# CoWork Bug Plan

26 confirmed bugs, found by independent reading + two blind cross-check agent audits (auth/time
cluster and bookings/services cluster) — full agreement across all three passes, no
contradictions. Grouped by difficulty tier; fix in this order (Easy → Medium → Hard). Rule
numbers refer to `README.md` §"Business rules".

Point value if this were exactly the hackathon rubric: 8 Easy × 3 = 24, 7 Medium × 5 = 35,
11 Hard × 10 = 110 → 169 total across 26 bugs (some hackathon-graders may bucket a few of these
together, e.g. the 3 `list_bookings` issues as one bug — treat the count as a ceiling, not a
guarantee).

---

## EASY (8)

### E1. Access token expires in 15 hours, not 900 seconds
**File:** `app/auth.py:50` — `create_access_token`
```python
lifetime = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60)
```
`ACCESS_TOKEN_EXPIRE_MINUTES=15`, so this is `timedelta(minutes=900)` = 54,000s, not 900s.
**Fix:** `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)`. **Rule 8.**

### E2. 5-minute grace window on booking start time
**File:** `app/routers/bookings.py:86`
```python
if start <= now - timedelta(seconds=300):
```
Rule 2 says "no grace window of any size." **Fix:** `if start <= now:`.

### E3. `list_bookings` sorts descending instead of ascending
**File:** `app/routers/bookings.py:137` — `.order_by(Booking.start_time.desc(), Booking.id.asc())`
**Fix:** `.asc()` on `start_time`. **Rule 11.**

### E4. `list_bookings` offset is one page ahead (skips page 1 entirely)
**File:** `app/routers/bookings.py:138` — `.offset(page * limit)`
**Fix:** `.offset((page - 1) * limit)`. **Rule 11.**

### E5. `list_bookings` ignores the `limit` query param
**File:** `app/routers/bookings.py:139` — `.limit(10)` hardcoded.
**Fix:** `.limit(limit)`. **Rule 11.**

### E6. Duplicate username returns 201 with existing user instead of 409
**File:** `app/routers/auth.py:37-43`
```python
if existing is not None:
    return {"user_id": existing.id, ...}
```
**Fix:** `raise AppError(409, "USERNAME_TAKEN", "Username already taken in this organization")`.
**Rule 15.**

### E7. `get_booking` overwrites `start_time` with `created_at`
**File:** `app/routers/bookings.py:166` — `response["start_time"] = iso_utc(booking.created_at)`
`serialize_booking` already sets the correct `start_time`; this line clobbers it.
**Fix:** delete the line. (Data-integrity / contract bug.)

### E8. Refund tier: exactly-48h notice gets 50% instead of 100%
**File:** `app/routers/bookings.py:200-202`
```python
notice_hours = int(notice.total_seconds() // 3600)
if notice_hours > 48:
```
Rule 6 says `notice >= 48h -> 100%`; `>` excludes the boundary.
**Fix:** compare the timedelta directly: `if notice >= timedelta(hours=48): refund_percent = 100`.

---

## MEDIUM (7)

### M1. `parse_input_datetime` strips the UTC offset instead of converting to UTC
**File:** `app/timeutils.py:12-13`
```python
if dt.tzinfo is not None:
    dt = dt.replace(tzinfo=None)
```
`"10:00+05:00"` should become `05:00` UTC; current code keeps `10:00` (just deletes the offset).
This corrupts every downstream time comparison (conflicts, quota, price window, reports).
**Fix:** `dt = dt.astimezone(timezone.utc).replace(tzinfo=None)`. **Rule 1.**

### M2. Duration `0` or negative is accepted (missing minimum-duration / end>start check)
**File:** `app/routers/bookings.py:89-94`
`MIN_DURATION_HOURS = 1` is defined but never checked. `end_time == start_time` → duration 0,
passes; `end_time < start_time` → negative duration and **negative price_cents**, also passes
(only `> MAX_DURATION_HOURS` is checked).
**Fix:** add `if duration_hours < MIN_DURATION_HOURS: raise AppError(400, "INVALID_BOOKING_WINDOW", ...)`.
**Rule 2.**

### M3. Back-to-back bookings wrongly rejected as conflicts
**File:** `app/routers/bookings.py:50`
```python
if b.start_time <= end and start <= b.end_time:
```
Rule 3 requires strict `<` on both sides (back-to-back must be allowed). Example: existing
10:00-11:00, new 11:00-12:00 → currently flagged `ROOM_CONFLICT`, should be allowed.
**Fix:** `if b.start_time < end and start < b.end_time:`.

### M4. `<24h` notice gives 50% refund instead of 0%
**File:** `app/routers/bookings.py:203-206`
```python
elif notice >= timedelta(hours=24):
    refund_percent = 50
else:
    refund_percent = 50
```
Both branches give 50 — the `<24h → 0%` tier is missing entirely.
**Fix:** `else: refund_percent = 0`. **Rule 6.**

### M5. `get_booking` lets any member read any other member's booking in the org
**File:** `app/routers/bookings.py:150-163`
Only `Room.org_id == user.org_id` is checked — no ownership check for non-admins, unlike
`cancel_booking` (line 192) which correctly has one.
**Fix:** add the same guard used in `cancel_booking`:
```python
if user.role != "admin" and booking.user_id != user.id:
    raise AppError(404, "BOOKING_NOT_FOUND", "Booking not found")
```
**Rule 10.**

### M6. `create_booking` never invalidates the usage-report cache
**File:** `app/routers/bookings.py:117-122`
Only `cache.invalidate_availability(...)` is called; `cache.invalidate_report(user.org_id)` is
missing (it's only called in `cancel_booking`). A cached usage-report stays stale after new
bookings.
**Fix:** add `cache.invalidate_report(user.org_id)` alongside the existing invalidation call.
**Rule 12.**

### M7. `cancel_booking` never invalidates the availability cache
**File:** `app/routers/bookings.py:213-218`
Only `cache.invalidate_report(...)` is called; the room's availability cache for that date is
never cleared, so a cancelled booking can keep showing as "busy."
**Fix:** add `cache.invalidate_availability(booking.room_id, booking.start_time.date().isoformat())`.
**Rule 13.**

---

## HARD (11)

### H1. Logout checks the wrong JWT claim — revocation never actually works
**File:** `app/auth.py:86` (`revoke_access_token`, stores `payload["jti"]`) vs `app/auth.py:97`
(`get_token_payload`, checks `payload.get("sub") in _revoked_tokens`).
`sub` (user id, e.g. `"42"`) will never match entries in a set of `jti` UUIDs — logout is a
no-op; the presented token keeps working after logout.
**Fix:** `if payload.get("jti") in _revoked_tokens:`. **Rule 8.**

### H2. Refresh tokens are not single-use
**File:** `app/routers/auth.py:81-93` — no blacklist/check of used refresh-token `jti`s exists
anywhere. A refresh token can be replayed indefinitely until its 7-day expiry.
**Fix:** maintain a used-refresh-jti set (mirrors `_revoked_tokens`), check `data["jti"]` against
it (401 if present) before minting new tokens, and record it after a successful refresh. Needs a
lock around check+record to be truly single-use under concurrent replay of the same token.
**Rule 8.**

### H3. Refund amount can diverge between the cancel response and the stored RefundLog
**File:** `app/routers/bookings.py:208` (`round(price * pct/100.0)`, Python banker's rounding) vs
`app/services/refunds.py:14-17` (`int(dollars * pct/100.0 * 100)`, truncates, and picks up float
error). These are two independent, non-compliant calculations of the same number and can produce
different cents for the same input (e.g. `price_cents=999, pct=50` → response `500`, log `499`).
Rule 6 requires they be equal, and rounding must be half-cents-up.
**Fix:** compute once with integer math, e.g. `(price_cents * pct + 50) // 100`, in one place
(e.g. `refunds.log_refund` returns the `RefundLog`; have `cancel_booking` read
`entry.amount_cents` for its response instead of recomputing).

### H4. Double refund possible on concurrent cancel of the same booking
**File:** `app/routers/bookings.py:184-218`, `app/services/refunds.py`
No row lock around the status-check → `log_refund` → status-update sequence, and
`_settlement_pause()` sleeps between the refund write and the status flip — widening the race.
Two concurrent cancels of the same booking can both pass the `status == "confirmed"` check and
both create a `RefundLog` row. Violates "exactly one RefundLog entry" + "must hold under
concurrent cancel requests."
**Fix:** wrap the whole cancel critical section (status check + refund log + status update) in a
`threading.Lock()` (module-level in `bookings.py`, matching the `notifications.py` pattern), or
`SELECT ... FOR UPDATE` the booking row within one transaction.

### H5. Room-conflict check has a check-then-insert race → double-booking under concurrency
**File:** `app/routers/bookings.py:42-52,100` — `_has_conflict` reads existing bookings, sleeps
0.12s (`_pricing_warmup`), returns; the insert happens later with no lock tying the read to the
write. Two concurrent requests for an overlapping slot can both observe "no conflict" and both
commit. Violates "must hold under concurrent requests" (Rule 3).
**Fix:** same lock as H4 — wrap the whole `create_booking` critical section (conflict check +
quota check + insert) in one `threading.Lock()`. Simplest correct fix for a single-process
SQLite app; don't build per-room lock management for a 4-hour hackathon.

### H6. Quota check has the same check-then-insert race → quota bypass under concurrency
**File:** `app/routers/bookings.py:55-71` — same pattern as H5, `_quota_audit()` sleeps between
the `count()` read and the eventual insert.
**Fix:** covered by the same lock as H5 (both checks + the insert happen inside one critical
section, so fixing H5 fixes this too — verify both invariants hold under a concurrent-burst test
after the fix). **Rule 4.**

### H7. Reference-code counter race → duplicate reference codes under concurrent creation
**File:** `app/services/reference.py:17-21` — unguarded read of `_counter["value"]`, a 0.12s
sleep, then increment. Two concurrent calls can both read the same `current` and return the same
code. `Booking.reference_code` also has no DB `unique=True` as a backstop.
**Fix:** wrap the read-increment in a module-level `threading.Lock()`. **Rule 7.**

### H8. Rate-limit bucket race → cap bypass under concurrent requests
**File:** `app/services/ratelimit.py:18-26` — bucket is read, filtered into a new local list,
slept on (`_settle_pause`), then written back; concurrent calls for the same user overwrite each
other's appended timestamp (lost update), letting more than 20 req/60s through.
**Fix:** wrap the read-filter-append-store sequence in a module-level `threading.Lock()`.
**Rule 5.**

### H9. Room-stats counters race → count/revenue drift under concurrent bursts
**File:** `app/services/stats.py:15-26` — same lost-update pattern in `record_create` and
`record_cancel`, with `_aggregate_pause()` sleeping between read and write.
**Fix:** wrap both functions' read-modify-write in a module-level `threading.Lock()`. **Rule 14.**

### H10. Lock-ordering deadlock between `notify_created` and `notify_cancelled`
**File:** `app/services/notifications.py:24-35`
```python
def notify_created(booking):
    with _email_lock:
        with _audit_lock: ...
def notify_cancelled(booking):
    with _audit_lock:
        with _email_lock: ...
```
Opposite acquisition order. A concurrent create + cancel can deadlock both threads permanently,
hanging the service — directly violates Rule 16 (liveness).
**Fix:** make both functions acquire locks in the same order (e.g. always `_email_lock` then
`_audit_lock` — fix `notify_cancelled` to match `notify_created`).

### H11. `GET /admin/export?include_all=true&room_id=X` leaks another org's bookings
**File:** `app/services/export.py:22-29,41-54`
```python
def fetch_bookings_raw(db, room_id):
    return db.query(Booking).filter(Booking.room_id == room_id)...  # no org filter at all
```
Called whenever `include_all=true` and a `room_id` is given — bypasses org scoping entirely,
unlike `_fetch_scoped` (which joins `Room` and filters by `org_id`). Any admin can pass another
org's `room_id` and receive a full CSV of that org's bookings. Direct Rule 9 violation.
**Fix:** route this case through `_fetch_scoped(db, org_id, None, room_id)` instead of the
unscoped `fetch_bookings_raw` (deleting the now-unused function), or add an org-ownership check
on `room_id` before calling it.

---

## Notes / lower-confidence items (not counted above, mention only)

- `app/routers/auth.py::register` commits a new `Organization` before checking username
  uniqueness; a concurrent register for a brand-new org name could hit the DB's
  `Organization.name` unique constraint and raise an unhandled `IntegrityError` (500, not the
  `{"detail","code"}` envelope). Same risk for a brand-new username under concurrent registration
  (`uq_user_org_username`). Not explicitly required by Rule 15's concurrency language, but worth
  a `try/except` → `AppError` if time allows.
- `JWT_SECRET` defaults to a hardcoded dev value with no startup guard forcing it to be set in
  production — out of scope for this challenge (no production deploy involved) but flagged for
  completeness.
