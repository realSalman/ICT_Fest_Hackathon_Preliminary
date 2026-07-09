# CoWork API - Final Bug Fix Report (Detailed)

This document outlines the problems identified in the original `main` codebase and exactly how they were successfully resolved in the `fixed` codebase, providing code comparisons for clarity.

## 1. Authentication & Identity

**1. Access Token Lifetime**
- **Problem:** Access tokens were being issued with a lifetime of 15 hours instead of 15 minutes.
```python
# main
lifetime = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60)
```
- **Fix:** Removed the `* 60` multiplier.
```python
# fixed
lifetime = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
```

**2. Token Revocation Check (Logout)**
- **Problem:** `logout` revoked tokens using the user ID (`sub`), accidentally blacklisting all sessions for a user globally instead of just the presented token.
```python
# main
if payload.get("sub") in _revoked_tokens:
    raise AppError(401, "UNAUTHORIZED", "Token has been revoked")
```
- **Fix:** Changed the payload check to properly blacklist the token's unique ID (`jti`).
```python
# fixed
if payload.get("jti") in _revoked_tokens:
    raise AppError(401, "UNAUTHORIZED", "Token has been revoked")
```

**3. Single-Use Refresh Tokens**
- **Problem:** Refresh tokens could be reused indefinitely.
- **Fix:** Implemented a memory set `_revoked_refresh_tokens` and added validation.
```python
# fixed
jti = data.get("jti")
if jti in _revoked_refresh_tokens:
    raise AppError(401, "UNAUTHORIZED", "Token has been revoked")
_revoked_refresh_tokens.add(jti)
```

**4. Organization Registration Race Condition**
- **Problem:** Concurrent org registrations bypassed the `None` check and crashed with an HTTP 500 `IntegrityError` due to unique name constraints.
```python
# main
if org is None:
    org = Organization(name=payload.org_name)
    db.add(org)
    db.commit()
    db.refresh(org)
    role = "admin"
```
- **Fix:** Wrapped the commit and gracefully handled the fallback to `"member"` status if the org was created concurrently.
```python
# fixed
if org is None:
    org = Organization(name=payload.org_name)
    db.add(org)
    try:
        db.commit()
        db.refresh(org)
        role = "admin"
    except IntegrityError:
        db.rollback()
        org = db.query(Organization).filter(Organization.name == payload.org_name).first()
        if org is None:
            raise AppError(500, "INTERNAL_ERROR", "Organization creation failed unexpectedly")
        role = "member"
```

**5. User Registration Race Condition (Duplicate Username)**
- **Problem:** Concurrent duplicate usernames bypassed the initial DB query and crashed with an HTTP 500 `IntegrityError`.
- **Fix:** Added a `try...except` wrapper.
```python
# fixed
try:
    db.commit()
    db.refresh(user)
except IntegrityError:
    db.rollback()
    raise AppError(409, "USERNAME_TAKEN", "Username already taken")
```

## 2. Booking Constraints & Core Logic

**6. UTC Timezone Offset Drop**
- **Problem:** `parse_input_datetime` completely stripped offsets from offset-aware strings, mutating the stored time.
```python
# main
if dt.tzinfo is not None:
    dt = dt.replace(tzinfo=None)
```
- **Fix:** Normalized them to UTC first before stripping.
```python
# fixed
if dt.tzinfo is not None:
    dt = dt.astimezone(timezone.utc).replace(tzinfo=None)
```

**7. Grace Window Violation**
- **Problem:** Allowed a 5-minute window into the past for `start_time`.
```python
# main
if start <= now - timedelta(seconds=300):
```
- **Fix:** Strictly enforced future starts.
```python
# fixed
if start <= now:
```

**8. Missing Minimum/Maximum Duration**
- **Problem:** Users could create bookings spanning backward in time or exceeding quotas.
- **Fix:** Added boundary checks.
```python
# fixed
if end <= start:
    raise AppError(400, "INVALID_BOOKING_WINDOW", "end_time must be after start_time")
if duration_hours < MIN_DURATION_HOURS or duration_hours > MAX_DURATION_HOURS:
    raise AppError(400, "INVALID_BOOKING_WINDOW", "duration out of range")
```

**9. Back-to-Back Overlaps Blocked**
- **Problem:** Used `<=` locking out valid adjacent bookings.
```python
# main
if b.start_time <= end and start <= b.end_time:
```
- **Fix:** Changed to strictly `<`.
```python
# fixed
if b.start_time < end and start < b.end_time:
```

**10. Start Time Overwrite**
- **Problem:** Erroneously overwrote the API response `start_time` field.
```python
# main
response["start_time"] = iso_utc(booking.created_at)
```
- **Fix:** Removed the line entirely.

## 3. Refunds & Cancellation

**11. Refund Math Truncation**
- **Problem:** Float math and `int()` lost half-cents instead of rounding.
```python
# main
dollars = booking.price_cents / 100.0
refund_dollars = dollars * (percent / 100.0)
amount_cents = int(refund_dollars * 100)
```
- **Fix:** Switched to integer banker's rounding (`half-up`).
```python
# fixed
amount_cents = (booking.price_cents * percent + 50) // 100
```

**12. Refund Tier Boundary Logic**
- **Problem:** Boundary conditions were off. The lowest tier defaulted to `50%` instead of `0%`.
```python
# main
if notice_hours > 48:
    refund_percent = 100
elif notice >= timedelta(hours=24):
    refund_percent = 50
else:
    refund_percent = 50
```
- **Fix:** Enforced strict `timedelta` limits and a `0%` floor.
```python
# fixed
if notice >= timedelta(hours=48):
    refund_percent = 100
elif notice >= timedelta(hours=24):
    refund_percent = 50
else:
    refund_percent = 0
```

**13. Refund Response Discrepancy**
- **Problem:** The cancel endpoint independently calculated the refund amount, risking divergence from the `RefundLog`.
```python
# main
refund_amount_cents = round(booking.price_cents * (refund_percent / 100.0))
log_refund(db, booking, refund_percent)
```
- **Fix:** Used the exact written `RefundLog` entity value.
```python
# fixed
refund_entry = log_refund(db, booking, refund_percent)
refund_amount_cents = refund_entry.amount_cents
```

## 4. Multi-Tenancy & Data Isolation

**14. Export Data Leak**
- **Problem:** Admin exports dropped `org_id` entirely, leaking booking data from all tenants.
```python
# main
if room_id is not None:
    rows = fetch_bookings_raw(db, room_id) # Blind fetch, NO ORG ID
```
- **Fix:** Funneled all exports through `_fetch_scoped`, which strictly asserts `Room.org_id == org_id`.
```python
# fixed
def _fetch_scoped(db: Session, org_id: int, user_id: int | None, room_id: int | None):
    query = db.query(Booking).join(Room).filter(Room.org_id == org_id) 
```

**15. Member Visibility Bypass**
- **Problem:** Standard members could view and cancel any booking globally because ownership checks were omitted.
- **Fix:** Appended strict authorization checks to endpoints.
```python
# fixed
if user.role != "admin" and booking.user_id != user.id:
    raise AppError(404, "BOOKING_NOT_FOUND", "Booking not found")
```

## 5. API Contracts & Caching

**16. Pagination Logic Failure**
- **Problem:** Handled default parameters incorrectly (wrong sorting, wrong offset math, hardcoded limit).
```python
# main
base.order_by(Booking.start_time.desc(), Booking.id.asc())
    .offset(page * limit)
    .limit(10)
```
- **Fix:** Corrected to ascending sorting and mathematical offsets.
```python
# fixed
base.order_by(Booking.start_time.asc(), Booking.id.asc())
    .offset((page - 1) * limit)
    .limit(limit)
```

**17. Cache Stale Invalidation**
- **Problem:** Modifying bookings left stats and availability caches stale.
- **Fix:** Injected cache clearing dynamically:
```python
# fixed (in create_booking)
cache.invalidate_report(user.org_id)

# fixed (in cancel_booking)
cache.invalidate_availability(booking.room_id, booking.start_time.date().isoformat())
```

## 6. Concurrency & Race Conditions (Thread Safety)

**18. Service-wide Race Conditions**
- **Problem:** Because SQLite lacks row-level transactions, concurrent requests triggered double bookings, double cancellations, and duplicate monotonic IDs.
- **Fix:** Introduced Python `threading.Lock` patterns to enforce atomicity.
```python
# fixed (Example: bookings.py)
_booking_lock = threading.Lock()

db.rollback()
with _booking_lock:
    if _has_conflict(db, room.id, start, end):
        raise AppError(409, "ROOM_CONFLICT", "Room already booked for this interval")
```

**19. Notification Deadlock (ABBA)**
- **Problem:** Competing endpoints acquired locks in reverse orders (`Email -> Audit` vs `Audit -> Email`), causing server hangs.
```python
# main (in notify_cancelled)
with _audit_lock:
    with _email_lock:
```
- **Fix:** Reordered the lock acquisition to match system-wide consistency.
```python
# fixed (in notify_cancelled)
with _email_lock:
    with _audit_lock:
```
