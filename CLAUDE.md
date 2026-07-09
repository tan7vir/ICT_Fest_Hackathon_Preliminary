# CoWork — Hackathon Bug-Fix Challenge

Multi-tenant coworking-space booking REST API (FastAPI + SQLAlchemy + SQLite). This is a
**bug-fix-only** challenge: 26 confirmed bugs have already been located and are enumerated in
[`BUGS_PLAN.md`](./BUGS_PLAN.md). Your job is to fix them — nothing else.

## Hard constraints (read before touching anything)

- **Do not change the API contract.** Paths, HTTP status codes, error `code` strings, JSON field
  names, and JWT claim names must stay exactly as documented in `README.md`. Grading is black-box
  against that contract.
- **Do not refactor or add features.** Only touch the specific lines a bug requires. No new
  endpoints, no schema changes beyond what a bug fix strictly needs, no dependency additions.
- **Every fix should be the smallest correct diff.** Most of these are 1-3 line fixes. The
  concurrency bugs are the exception — they need a `threading.Lock` around a critical section,
  matching the pattern `app/services/notifications.py` already uses (`_email_lock` /
  `_audit_lock`). Don't build a lock manager or abstraction; one dedicated `Lock()` per affected
  module/resource is correct and consistent with existing style.
- **SQLite + single process**: this app is not meant to scale horizontally. A single
  `threading.Lock()` per shared mutable resource is the right fix, not a distributed-locking
  scheme.

## Working through `BUGS_PLAN.md`

The plan is grouped by difficulty tier (matches the hackathon's scoring: Easy=3pts,
Medium=5pts, Hard=10pts) and ordered by file, so fixing bugs in `bookings.py` back-to-back
avoids re-reading the file repeatedly. Suggested order: do all Easy bugs first (fast, safe
points), then Medium, then Hard (concurrency/security — these need the most care).

After each fix, sanity-check against the specific rule it violates (rule numbers are cited in
the plan) — don't just make the code "look right," verify it actually satisfies the business
rule's exact boundary conditions (e.g. `>=` vs `>`, inclusive ranges).

## Verifying fixes

- Run `pytest` (`tests/test_smoke.py`) — it only covers the golden path, so passing it is
  necessary but nowhere near sufficient. Most of these bugs won't be caught by it.
- For concurrency bugs, the only real verification is firing concurrent requests at the running
  app (e.g. `asyncio.gather` of many `httpx` calls, or a small threaded script) and confirming
  the invariant holds (no duplicate reference codes, no double-booking, exactly one RefundLog
  row per cancelled booking, etc.).
- After fixing, consider writing `bug_report.md` at the repo root (file/line, what was wrong,
  why, how fixed) — it's the competition's tie-breaker.

## Architecture map

```
app/
├── main.py             # FastAPI app wiring, error handler registration
├── config.py           # env-driven config (JWT secret, token lifetimes)
├── database.py         # SQLAlchemy engine/session
├── models.py           # Organization, User, Room, Booking, RefundLog
├── schemas.py          # Pydantic request bodies
├── serializers.py      # Booking -> response dict
├── auth.py             # password hashing, JWT issue/verify/revoke, auth dependencies
├── cache.py            # in-memory report/availability cache (keyed dict, invalidate-by-key)
├── errors.py           # AppError -> {"detail", "code"} JSON envelope
├── timeutils.py        # ISO datetime parsing/rendering (UTC normalization)
├── routers/
│   ├── auth.py          # register/login/refresh/logout
│   ├── rooms.py          # rooms, availability, stats
│   ├── bookings.py       # create/list/get/cancel — most bugs live here
│   ├── admin.py          # usage-report, export
│   └── health.py
└── services/
    ├── reference.py      # sequential CW-###### reference code counter (unlocked — bug)
    ├── ratelimit.py       # per-user rolling-window bucket (unlocked — bug)
    ├── stats.py           # per-room live count/revenue (unlocked — bug)
    ├── refunds.py         # RefundLog write + amount calc (diverges from router's calc — bug)
    ├── notifications.py   # simulated email/audit side effects (has the deadlock — bug)
    └── export.py          # CSV export (has the org-scoping leak — bug)
```

## Business rules

Full rules are in `README.md` (identical to the hackathon problem statement). Skim it once —
`BUGS_PLAN.md` cites rule numbers but doesn't restate the full rule text every time.
