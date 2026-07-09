# CoWork API - Final Bug Fix Report

This document outlines the problems identified in the original `main` codebase and how they were successfully resolved in the `fixed` codebase, including both AI-generated patches and manual race condition fixes.

## 1. Authentication & Identity

**1. Access Token Lifetime**
- **Problem (main):** Access tokens were being issued with a lifetime of 15 hours (`minutes * 60`) instead of 15 minutes.
- **Fix (fixed):** Removed the `* 60` multiplier to strictly enforce the 900-second (15 minutes) limit.

**2. Token Revocation Check (Logout)**
- **Problem (main):** `logout` attempted to revoke tokens using the user ID (`sub`), essentially blacklisting all sessions for a user globally instead of just the presented token.
- **Fix (fixed):** Changed the payload check to properly blacklist the token's unique ID (`jti`).

**3. Single-Use Refresh Tokens**
- **Problem (main):** Refresh tokens had no revocation mechanism and could be reused indefinitely.
- **Fix (fixed):** Implemented a memory set `_revoked_refresh_tokens`. When a refresh token is used, its `jti` is added to the set to block reuse.

**4. Organization Registration Race Condition**
- **Problem (main):** Registering an organization committed the new org without handling unique name constraint violations. If two users concurrently registered the same unknown org, one would hit an unhandled `IntegrityError` (500 Server Error) instead of joining the org as a member.
- **Fix (fixed):** Wrapped the `db.commit()` for Organization creation in a `try...except IntegrityError`. On error, it rolls back, fetches the now-existing org, and safely joins the user as a `"member"`.

**5. User Registration Race Condition (Duplicate Username)**
- **Problem (main):** Similar to the org race condition, registering a duplicate username concurrently could throw a 500 error if it bypassed the initial `existing is not None` check.
- **Fix (fixed):** Safely catches `IntegrityError` on the `User` commit and throws a strict `409 USERNAME_TAKEN`.

## 2. Booking Constraints & Core Logic

**6. UTC Timezone Offset Drop**
- **Problem (main):** `parse_input_datetime` stripped UTC offsets completely from input strings, storing incorrect times if the user provided an offset.
- **Fix (fixed):** Added `.astimezone(timezone.utc)` to properly normalize offset-aware datetimes to UTC before dropping the tzinfo for SQLite storage.

**7. Grace Window Violation**
- **Problem (main):** Allowed a 5-minute grace window into the past for start times.
- **Fix (fixed):** Changed `start <= now - timedelta(seconds=300)` to strictly `start <= now`.

**8. Missing Minimum/Maximum Duration**
- **Problem (main):** Users could create 0-length bookings or bookings spanning backwards in time (`end <= start`), and had no maximum limit.
- **Fix (fixed):** Added validation to ensure `duration >= 1` hour, `duration <= 8` hours, and `end_time` strictly after `start_time`.

**9. Back-to-Back Overlaps Blocked**
- **Problem (main):** The conflict logic used `<=` (`b.start_time <= end and start <= b.end_time`), which falsely blocked valid back-to-back bookings.
- **Fix (fixed):** Adjusted operators to strictly `<` to allow adjacent bookings.

**10. Start Time Overwrite**
- **Problem (main):** `GET /bookings/{id}` erroneously mutated the `start_time` field, replacing it with `created_at`.
- **Fix (fixed):** Removed the rogue assignment.

## 3. Refunds & Cancellation

**11. Refund Math Truncation**
- **Problem (main):** Float multiplication with `int()` truncation resulted in losing half-cents instead of rounding them to the nearest cent.
- **Fix (fixed):** Changed to integer arithmetic `(price_cents * percent + 50) // 100` for proper banker's rounding (half-up).

**12. Refund Tier Boundary Logic**
- **Problem (main):** The 48-hour tier was strict `> 48` instead of `>= 48`. The <24-hour tier defaulted to 50% refund instead of 0%.
- **Fix (fixed):** Replaced logic with strict `timedelta` boundaries (`>= 48` hours -> 100%, `>= 24` hours -> 50%, else 0%).

**13. Refund Response Discrepancy**
- **Problem (main):** The cancel endpoint returned a mathematically computed refund amount that risked diverging from the actual `RefundLog` database entry due to float variations.
- **Fix (fixed):** Adjusted the response to read the finalized `amount_cents` directly from the `RefundLog` entity.

## 4. Multi-Tenancy & Data Isolation

**14. Export Data Leak**
- **Problem (main):** Admin CSV exports with `include_all=True` dropped the `org_id` constraint entirely, leaking booking data from every other tenant in the database.
- **Fix (fixed):** Ensured the `_fetch_scoped` function always enforces `Room.org_id == org_id` regardless of `include_all`.

**15. Member Visibility Bypass**
- **Problem (main):** `GET /bookings/{id}` and cancellation endpoints did not verify ownership for non-admin members, allowing anyone to view or cancel others' bookings.
- **Fix (fixed):** Implemented strict authorization checks: `if user.role != "admin" and booking.user_id != user.id:` -> `404 BOOKING_NOT_FOUND`.

## 5. API Contracts & Caching

**16. Pagination Logic Failure**
- **Problem (main):** `GET /bookings` sorted descending (violating contract), used `page * limit` for the offset (skipping the entire first page), and completely ignored the user's `limit` parameter.
- **Fix (fixed):** Updated to `start_time.asc()`, `offset((page - 1) * limit)`, and `limit(limit)`.

**17. Cache Stale Invalidation**
- **Problem (main):** The availability cache wasn't cleared upon cancellation, and the usage report wasn't cleared upon creation.
- **Fix (fixed):** Placed proper `cache.invalidate_availability` on cancels and `cache.invalidate_report` on creates.

## 6. Concurrency & Race Conditions (Thread Safety)

Because SQLite in Python does not natively block concurrent read-modify-write transactions well, threading locks were introduced to protect critical operations:

**18. Service-wide Race Conditions**
- **Problem (main):** Concurrent requests easily bypassed quotas (Double Booking & Quota Race), double-processed cancellations (Duplicate Refunds), corrupted `_stats` (Room Stats Race), generated duplicate reference IDs, and bypassed rate limits.
- **Fix (fixed):** Injected Python `threading.Lock` wrappers across the service layer:
  - `_booking_lock`: Prevents double bookings and duplicate refunds. 
  - `_stats_lock`: Protects live stats dictionary updates.
  - `_bucket_lock`: Protects rate limiting token buckets.
  - `_ref_lock`: Protects monotonic reference code generator.
  
**19. Notification Deadlock (ABBA)**
- **Problem (main):** `notify_created` locked Email then Audit, while `notify_cancelled` locked Audit then Email, causing cross-thread deadlocks under concurrent load.
- **Fix (fixed):** Reordered locks in `notify_cancelled` to consistently lock Email before Audit.
