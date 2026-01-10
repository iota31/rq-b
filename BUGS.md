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

**Full pytest output:**
```
$ pytest tests/test_utils.py::TestUtils::test_parse_timeout -v
=================================================================================== test session starts ===================================================================================
platform darwin -- Python 3.12.0, pytest-9.0.2, pluggy-1.6.0
rootdir: /tmp/rq-b
collected 1 item

tests/test_utils.py::TestUtils::test_parse_timeout FAILED                                                                                                                           [100%]

======================================================================================== FAILURES =========================================================================================
______________________________________________________________________________ TestUtils.test_parse_timeout _______________________________________________________________________________

self = <tests.test_utils.TestUtils testMethod=test_parse_timeout>

    def test_parse_timeout(self):
        """Ensure function parse_timeout works correctly"""
        self.assertEqual(12, parse_timeout(12))
        self.assertEqual(12, parse_timeout('12'))
        self.assertEqual(12, parse_timeout('12s'))
        self.assertEqual(720, parse_timeout('12m'))
>       self.assertEqual(3600, parse_timeout('1h'))
E       AssertionError: 3600 != 360

tests/test_utils.py:45: AssertionError
================================================================================= short test summary info =================================================================================
FAILED tests/test_utils.py::TestUtils::test_parse_timeout - AssertionError: 3600 != 360
==================================================================================== 1 failed in 0.33s ====================================================================================
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

**Full pytest output:**
```
$ pytest tests/test_utils.py::TestUtils::test_get_redis_version -v
=================================================================================== test session starts ===================================================================================
platform darwin -- Python 3.12.0, pytest-9.0.2, pluggy-1.6.0
rootdir: /tmp/rq-b
collected 1 item

tests/test_utils.py::TestUtils::test_get_redis_version FAILED                                                                                                                       [100%]

======================================================================================== FAILURES =========================================================================================
____________________________________________________________________________ TestUtils.test_get_redis_version _____________________________________________________________________________

self = <tests.test_utils.TestUtils testMethod=test_get_redis_version>

    def test_get_redis_version(self):
        """Ensure get_version works properly"""
        redis = Redis()
        self.assertIsInstance(get_version(redis), tuple)

        # Parses 3 digit version numbers correctly
        class Redis4(Redis):
            def info(self, *args, **kwargs):
                return {'redis_version': '4.0.8'}

        self.assertEqual(get_version(Redis4()), (4, 0, 8))

        # Parses 3 digit version numbers correctly
        class Redis3(Redis):
            def info(self, *args, **kwargs):
                return {'redis_version': '3.0.7.9'}

        self.assertEqual(get_version(Redis3()), (3, 0, 7))

        # Parses 2 digit version numbers correctly (Seen in AWS ElastiCache Redis)
        class Redis7(Redis):
            def info(self, *args, **kwargs):
                return {'redis_version': '7.1'}

>       self.assertEqual(get_version(Redis7()), (7, 1, 0))
E       AssertionError: Tuples differ: (7, 1, 1) != (7, 1, 0)
E
E       First differing element 2:
E       1
E       0
E
E       - (7, 1, 1)
E       ?        ^
E
E       + (7, 1, 0)
E       ?        ^

tests/test_utils.py:149: AssertionError
================================================================================= short test summary info =================================================================================
FAILED tests/test_utils.py::TestUtils::test_get_redis_version - AssertionError: Tuples differ: (7, 1, 1) != (7, 1, 0)
==================================================================================== 1 failed in 0.13s ====================================================================================
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

**Full pytest output:**
```
$ pytest tests/test_job.py::TestJob::test_persistence_of_callbacks -v
=================================================================================== test session starts ===================================================================================
platform darwin -- Python 3.12.0, pytest-9.0.2, pluggy-1.6.0
rootdir: /tmp/rq-b
collected 1 item

tests/test_job.py::TestJob::test_persistence_of_callbacks FAILED                                                                                                                    [100%]

======================================================================================== FAILURES =========================================================================================
__________________________________________________________________________ TestJob.test_persistence_of_callbacks __________________________________________________________________________

self = <tests.test_job.TestJob testMethod=test_persistence_of_callbacks>

    def test_persistence_of_callbacks(self):
        """Storing jobs with success and/or failure callbacks."""
        job = Job.create(
            func=fixtures.some_calculation,
            on_success=Callback(fixtures.say_hello, timeout=10),
            on_failure=fixtures.say_pid,
            on_stopped=fixtures.say_hello,
            connection=self.connection,
        )  # deprecated callable
        job.save()
        stored_job = Job.fetch(job.id, connection=self.connection)

        self.assertEqual(fixtures.say_hello, stored_job.success_callback)
        self.assertEqual(10, stored_job.success_callback_timeout)
        self.assertEqual(fixtures.say_pid, stored_job.failure_callback)
        self.assertEqual(fixtures.say_hello, stored_job.stopped_callback)
        self.assertEqual(CALLBACK_TIMEOUT, stored_job.failure_callback_timeout)
        self.assertEqual(CALLBACK_TIMEOUT, stored_job.stopped_callback_timeout)

        # None(s)
        job = Job.create(func=fixtures.some_calculation, on_failure=None, connection=self.connection)
        job.save()
        stored_job = Job.fetch(job.id, connection=self.connection)
        self.assertIsNone(stored_job.success_callback)
>       self.assertEqual(CALLBACK_TIMEOUT, job.success_callback_timeout)  # timeout should be never none
        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
E       AssertionError: 60 != 0

tests/test_job.py:305: AssertionError
================================================================================= short test summary info =================================================================================
FAILED tests/test_job.py::TestJob::test_persistence_of_callbacks - AssertionError: 60 != 0
============================================================================== 1 failed, 2 warnings in 0.14s ==============================================================================
```

---

### A4: Inverted Queue Ordering
**File:** `rq/queue.py` line ~502  
**Difficulty:** MEDIUM  
**Change:** `connection.lpush if at_front else connection.rpush` → `connection.rpush if at_front else connection.lpush`

**What breaks:** Jobs pushed with `at_front=True` go to the back instead of front.

**Failing test:**
```
tests/test_scheduler.py::TestQueue::test_enqueue_at_at_front
AssertionError: 0 != 1
```

**Full pytest output:**
```
$ pytest tests/test_scheduler.py -k "at_front" -v
=================================================================================== test session starts ===================================================================================
platform darwin -- Python 3.12.0, pytest-9.0.2, pluggy-1.6.0
rootdir: /tmp/rq-b
collected 25 items / 24 deselected / 1 selected

tests/test_scheduler.py::TestQueue::test_enqueue_at_at_front FAILED                                                                                                                 [100%]

======================================================================================== FAILURES =========================================================================================
___________________________________________________________________________ TestQueue.test_enqueue_at_at_front ____________________________________________________________________________

self = <tests.test_scheduler.TestQueue testMethod=test_enqueue_at_at_front>

    def test_enqueue_at_at_front(self):
        """queue.enqueue_at() accepts at_front argument. When true, job will be put at position 0
        of the queue when the time comes for the job to be scheduled"""
        queue = Queue(connection=self.connection)
        registry = ScheduledJobRegistry(queue=queue)
        scheduler = RQScheduler([queue], connection=self.connection)
        scheduler.acquire_locks()
        # Jobs created using enqueue_at is put in the ScheduledJobRegistry
        # job_first should be enqueued first
        job_first = queue.enqueue_at(datetime(2019, 1, 1, tzinfo=timezone.utc), say_hello)
        # job_second will be enqueued second, but "at_front"
        job_second = queue.enqueue_at(datetime(2019, 1, 2, tzinfo=timezone.utc), say_hello, at_front=True)
        self.assertEqual(len(queue), 0)
        self.assertEqual(len(registry), 2)

        # enqueue_at set job status to "scheduled"
        self.assertEqual(job_first.get_status(), 'scheduled')
        self.assertEqual(job_second.get_status(), 'scheduled')

        # After enqueue_scheduled_jobs() is called, the registry is empty
        # and job is enqueued
        scheduler.enqueue_scheduled_jobs()
        self.assertEqual(len(queue), 2)
        self.assertEqual(len(registry), 0)
>       self.assertEqual(0, queue.get_job_position(job_second.id))
E       AssertionError: 0 != 1

tests/test_scheduler.py:476: AssertionError
================================================================================= short test summary info =================================================================================
FAILED tests/test_scheduler.py::TestQueue::test_enqueue_at_at_front - AssertionError: 0 != 1
============================================================================ 1 failed, 24 deselected in 0.12s =============================================================================
```

---

### A5: TTL Off-by-One
**File:** `rq/registry.py` line ~119  
**Difficulty:** MEDIUM  
**Change:** `ttl if ttl < 0` → `ttl if ttl <= 0`

**What breaks:** Jobs with TTL=0 are stored with score=0 (Unix epoch 1970-01-01) instead of current timestamp. While both result in immediate expiry on cleanup, the score semantics are wrong.

**Note:** This bug passes tests because no test explicitly validates TTL=0 behavior. It requires an incident description.

**Incident Description:**
> **Subject:** Jobs with result_ttl=0 appearing in finished registry then disappearing
>
> We have high-volume fire-and-forget jobs (logging, analytics events) that we enqueue with `result_ttl=0` to avoid storing results. According to the docs, this should mean results are "deleted immediately."
>
> After upgrading RQ, we noticed strange behavior. Our monitoring dashboard shows jobs briefly appearing in the FinishedJobRegistry, then vanishing on the next poll:
>
> ```
> [DEBUG] Job analytics-evt-123 finished, adding to registry with ttl=0
> [DEBUG] FinishedJobRegistry count: 1
> [DEBUG] Running cleanup...
> [DEBUG] FinishedJobRegistry count: 0  # Gone immediately!
> ```
>
> We dug into Redis and found jobs are being stored with score=0:
> ```
> redis> ZSCORE rq:finished:default analytics-evt-456
> "0"
> ```
>
> That's the Unix epoch (1970-01-01)! It should be the current timestamp if we want immediate expiry, or it shouldn't be added at all.
>
> This is causing two issues:
> 1. **Race conditions** - dependent jobs sometimes see the parent as "finished" and sometimes don't, depending on cleanup timing
> 2. **Monitoring confusion** - our Grafana dashboards show phantom jobs appearing and disappearing
>
> We're currently working around this by using `result_ttl=1` instead, but that feels wrong.

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
