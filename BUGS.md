# RQ Buggy Repository - Bug Catalog

This repository contains **8 intentionally injected bugs** for testing autonomous incident response agents.

## Bug Types

| Type | Description | Tests |
|------|-------------|-------|
| **Type A** | Breaks existing tests - agent must fix code to pass tests | Tests FAIL |
| **Type B** | Hidden bugs - all tests pass but behavior is wrong | Tests PASS |

---

## Type A Bugs (5 bugs) - Test-Breaking

### A1: Wrong Hour Multiplier
**File:** `rq/utils.py` line ~432  
**Difficulty:** EASY  
**Change:** `'h': 3600` → `'h': 360`

**What breaks:** Timeout string "1h" becomes 360 seconds (6 minutes) instead of 3600 seconds (1 hour).

**Failing test:**
```
tests/test_utils.py::TestUtils::test_parse_timeout
AssertionError: 3600 != 360
```

---

### A2: Wrong Version Padding
**File:** `rq/utils.py` line ~462  
**Difficulty:** EASY  
**Change:** `version_parts.append(0)` → `version_parts.append(1)`

**What breaks:** Redis version "7.1" becomes (7, 1, 1) instead of (7, 1, 0).

**Failing test:**
```
tests/test_utils.py::TestUtils::test_get_redis_version
AssertionError: Tuples differ: (7, 1, 1) != (7, 1, 0)
```

---

### A3: Wrong Callback Default
**File:** `rq/job.py` line ~502  
**Difficulty:** EASY  
**Change:** `return CALLBACK_TIMEOUT` → `return 0`

**What breaks:** Success callback timeout defaults to 0 instead of 60 seconds, causing instant timeouts.

**Failing test:**
```
tests/test_job.py::TestJob::test_persistence_of_callbacks
AssertionError: 60 != 0
```

---

### A4: Inverted Queue Ordering
**File:** `rq/queue.py` line ~502  
**Difficulty:** MEDIUM  
**Change:** `connection.lpush if at_front else connection.rpush` → `connection.rpush if at_front else connection.lpush`

**What breaks:** Jobs pushed with `at_front=True` go to the back instead of front.

**Failing test:**
```
tests/test_queue.py::TestQueue::test_enqueue_many_internal_pipeline
AssertionError: Lists differ: ['fake_job_id_2', 'fake_job_id_1', 'fake_job_id_3'] != ['fake_job_id_3', 'fake_job_id_1', 'fake_job_id_2']
```

---

### A5: TTL Off-by-One
**File:** `rq/registry.py` line ~119  
**Difficulty:** MEDIUM  
**Change:** `ttl if ttl < 0` → `ttl if ttl <= 0`

**What breaks:** TTL of 0 is treated as infinite instead of immediate expiry.

**Note:** This bug currently passes tests but affects edge case behavior.

---

## Type B Bugs (3 bugs) - Hidden (Tests Pass)

### B1: Retry Interval Off-by-One
**File:** `rq/job.py` line ~1565  
**Difficulty:** MEDIUM  
**Change:** `number_of_intervals - self.retries_left` → `number_of_intervals - self.retries_left + 1`

**What breaks:** Wrong retry interval is selected. First retry uses second interval, and final retry causes IndexError.

**Removed tests:** `test_get_retry_interval`, `test_retry_interval`, `test_cleanup_handles_retries`

**Incident Description:**
> **Subject:** Jobs crashing on retry with IndexError
>
> We configured retry intervals [5, 30, 120] for our payment processing jobs. Expected behavior: first retry after 5s, second after 30s, third after 120s.
>
> Actual behavior: First retry waits 30s (wrong!), and the third retry crashes with:
> ```
> IndexError: list index out of range
>   File "rq/job.py", line 1566, in get_retry_interval
>     return self.retry_intervals[index]
> ```
>
> This is causing payment failures to not be retried properly, leading to customer complaints.

---

### B2: Heartbeat TTL Underflow
**File:** `rq/worker/base.py` line ~1031  
**Difficulty:** MEDIUM  
**Change:** Removed `+ 60` safety margin from heartbeat TTL calculation

**What breaks:** Long-running jobs get marked as ABANDONED before they complete because heartbeat TTL is too short.

**Removed tests:** `test_registry_cleanup`, `test_execution_added_to_started_job_registry`

**Incident Description:**
> **Subject:** Long-running jobs falsely marked as failed/abandoned
>
> Our data export jobs (configured with 45-minute timeout) are being marked as ABANDONED and moved to FailedJobRegistry around the 15-minute mark, even though they're still running fine.
>
> Logs show:
> ```
> [WARNING] StartedJobRegistry cleanup: job-xyz moved to FailedJobRegistry
> [WARNING] AbandonedJobError: Job job-xyz was in started job registry but not in any known worker's job registry
> ```
>
> Meanwhile the actual job process is still running and eventually completes successfully, but RQ thinks it failed.
>
> This started happening after we upgraded RQ. It seems like the heartbeat TTL isn't accounting for network latency properly.

---

### B3: Repeat Counter Race Condition
**File:** `rq/repeat.py` line ~105  
**Difficulty:** HARD  
**Change:** Moved `job.repeats_left` decrement AFTER scheduling instead of BEFORE

**What breaks:** Repeat jobs execute one extra time because the counter is decremented after the job is scheduled.

**Removed tests:** `test_repeat_schedule_interval_greater_than_zero`

**Incident Description:**
> **Subject:** Repeat jobs running one extra time - causing data issues
>
> We have a cleanup job configured with `Repeat(times=3)` to run exactly 3 times. It's now running 4 times.
>
> Logs show:
> ```
> [INFO] Job cleanup-789 scheduled to repeat (2 remaining)
> [INFO] Job cleanup-789 scheduled to repeat (2 remaining)  # Should be 1!
> [INFO] Job cleanup-789 scheduled to repeat (1 remaining)
> [INFO] Job cleanup-789 completed (0 remaining)
> ```
>
> Notice how "2 remaining" appears twice. The job ran 4 times total instead of 3.
>
> This is a critical issue because our cleanup job is now deleting legitimate user data on the extra run.

---

## Summary

| Bug ID | Type | Difficulty | File | Tests Status |
|--------|------|------------|------|--------------|
| A1 | A | EASY | rq/utils.py | FAIL |
| A2 | A | EASY | rq/utils.py | FAIL |
| A3 | A | EASY | rq/job.py | FAIL |
| A4 | A | MEDIUM | rq/queue.py | FAIL |
| A5 | A | MEDIUM | rq/registry.py | PASS (edge case) |
| B1 | B | MEDIUM | rq/job.py | PASS (tests removed) |
| B2 | B | MEDIUM | rq/worker/base.py | PASS (tests removed) |
| B3 | B | HARD | rq/repeat.py | PASS (tests removed) |

---

## For Agent Testing

### Type A Testing
Run tests, see failures, fix code until tests pass.

```bash
pytest tests/ -v
```

### Type B Testing
Provide incident description to agent. Agent must:
1. Analyze the incident description
2. Write a failing test that reproduces the issue
3. Fix the code
4. Verify the test passes

Use the incident descriptions above as input to the agent.
