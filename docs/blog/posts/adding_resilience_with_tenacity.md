---
title: Adding Resilience with Tenacity – Retry Logic for HTTP Requests
date: 2026-02-07
categories:
  - Python
authors:
  - david
tags:
  - pybites
  - pdm
  - tenacity
  - resilience
description: >
  After building a simple weather API client, the next step was making it resilient to transient failures. This post explains how I added retry logic with exponential backoff using tenacity, and what I learned about designing fault-tolerant systems.
---

# Adding Resilience with Tenacity: Retry Logic for HTTP Requests

After building a simple, testable weather API client, the next logical step was making it **resilient to transient failures**.

Network requests fail. Servers time out. APIs rate-limit. These aren't bugs—they're **expected conditions** that production code needs to handle gracefully.

This post explains how I added retry logic with exponential backoff using the `tenacity` library, and what I learned about designing fault-tolerant systems.

<!-- more -->

---

## The problem: network requests are unreliable

My initial `api_call()` function was simple:

```python
def api_call(session, url, params=None, timeout_s=3.0):
    response = session.get(url, params=params, timeout=timeout_s)
    response.raise_for_status()
    return response
```

This works perfectly—until it doesn't.

What happens when:
- The network hiccups?
- The API returns a 503 "Service Unavailable"?
- The server is temporarily overloaded?

The function would fail immediately, even though **retrying a second later might succeed**.

---

## Why not just wrap it in a loop?

I could have written manual retry logic:

```python
for attempt in range(3):
    try:
        return session.get(...)
    except Exception:
        if attempt == 2:
            raise
        time.sleep(2 ** attempt)  # exponential backoff
```

But this approach has problems:
- **Logic mixed with behaviour** (retry config embedded in code)
- **Hard to test** (retry timing is hardcoded)
- **Not reusable** (every function needs its own retry loop)
- **Error-prone** (easy to get backoff math wrong)

Instead, I used **`tenacity`**—a library designed specifically for this.

---

## Introducing tenacity

`tenacity` is a Python library that provides declarative retry logic through decorators.

The key insight: **retries are a cross-cutting concern**, not part of your core logic.

Instead of embedding retry logic *inside* your function, you declare it *outside*:

```python
from tenacity import retry, stop_after_attempt

@retry(stop=stop_after_attempt(3))
def api_call(session, url, params=None, timeout_s=3.0):
    response = session.get(url, params=params, timeout=timeout_s)
    response.raise_for_status()
    return response
```

The function code stays clean. The retry behaviour is explicit and separate.

---

## Making retry configuration explicit

I didn't want retry settings hardcoded in the decorator. They should live in **configuration**, just like API keys and timeouts.

I extended my `Settings` class:

```python
class Settings(BaseSettings):
    # ...existing fields...
    retry_initial_wait_seconds: int = Field(default=1)
    max_retry_attempts: int = Field(default=3, ge=0, le=10)
    retry_backoff_multiplier: float = Field(default=2.0, gt=0)
    retry_max_wait_seconds: int = Field(default=60, gt=0)
```

Now retry behaviour is:
- **Validated** (attempts must be between 0-10)
- **Documented** (clear defaults and constraints)
- **Configurable** (can be changed via environment variables)

---

## Implementing exponential backoff with jitter

Simple retries can cause **thundering herd problems**—if 1000 clients all retry at the same time, they create another spike of load.

The solution: **exponential backoff with jitter**.

```python
from tenacity import wait_exponential_jitter

@retry(
    stop=stop_after_attempt(settings.max_retry_attempts),
    wait=wait_exponential_jitter(
        initial=settings.retry_initial_wait_seconds,
        max=settings.retry_max_wait_seconds,
        exp_base=settings.retry_backoff_multiplier,
    ),
    reraise=True,
)
def api_call(...):
    ...
```

This means:
- **First retry**: wait ~1 second (+ random jitter)
- **Second retry**: wait ~2 seconds (+ jitter)
- **Third retry**: wait ~4 seconds (+ jitter)
- **Max wait**: capped at 60 seconds

The jitter spreads out retry attempts, preventing synchronized retry storms.

---

## Only retrying retriable errors

Not all errors should trigger retries.

If I send a `401 Unauthorized`, retrying won't help—my API key is wrong. But a `503 Service Unavailable` is temporary.

I created a decision function:

```python
def _should_retry(exception: BaseException) -> bool:
    """Determine if an exception should trigger a retry."""
    # Always retry network issues
    if isinstance(exception, (requests.Timeout, requests.ConnectionError)):
        return True

    # Only retry specific HTTP error codes
    if isinstance(exception, requests.HTTPError):
        if exception.response is not None:
            return exception.response.status_code in {408, 429, 500, 502, 503, 504}

    return False
```

This distinguishes between:
- **Transient failures** (retry): timeouts, rate limits, server errors
- **Permanent failures** (don't retry): authentication errors, not found, bad requests

---

## Adding observability with logging

When retries happen in production, I want to **know about it**.

`tenacity` provides callbacks for retry events:

```python
from tenacity import before_sleep_log, after_log
import logging

logger = logging.getLogger(__name__)

@retry(
    # ...retry config...
    before_sleep=before_sleep_log(logger, logging.WARNING),
    after=after_log(logger, logging.INFO),
    reraise=True,
)
def api_call(...):
    ...
```

Now when a retry happens:
- **Before sleeping**: log at WARNING level (something failed)
- **After retry**: log at INFO level (retry completed)
- **On final failure**: the original exception is re-raised

This gives me visibility without cluttering my function code.

---

## The complete implementation

Here's the final `api_call()` function:

```python
@retry(
    stop=stop_after_attempt(settings.max_retry_attempts),
    wait=wait_exponential_jitter(
        initial=settings.retry_initial_wait_seconds,
        max=settings.retry_max_wait_seconds,
        exp_base=settings.retry_backoff_multiplier,
    ),
    retry=retry_if_exception(_should_retry),
    before_sleep=before_sleep_log(logger, logging.WARNING),
    after=after_log(logger, logging.INFO),
    reraise=True,
)
def api_call(
    session: Session,
    url: str,
    params: dict[str, str | int | float] | None = None,
    timeout_s: float = 3.0,
) -> Response:
    """Make an HTTP GET request with retry logic."""
    response = session.get(url, params=params, timeout=timeout_s)
    response.raise_for_status()
    return response
```

The core logic is unchanged—three lines of code.

All the retry behaviour is declarative and separate.

---

## What I learned

Adding retry logic taught me several important lessons:

### 1. Resilience is a design concern, not an afterthought
Retries shouldn't be bolted on later—they should be part of your architecture from the start.

### 2. Declarative is better than imperative
Using decorators kept retry logic separate from business logic, making both easier to understand.

### 3. Configuration belongs in configuration
Retry settings are just as important as API keys—they should be validated, documented, and configurable.

### 4. Not all failures are equal
Distinguishing between retriable and non-retriable errors prevents wasted retries and makes debugging easier.

### 5. Observability matters
Logging retry attempts gives you visibility into system health without requiring manual investigation.

---

## Comparing approaches

| Approach | Pros | Cons |
|----------|------|------|
| **Manual retry loop** | Full control, no dependencies | Error-prone, hard to test, not reusable |
| **`tenacity` decorator** | Clean, declarative, well-tested | Additional dependency |
| **Built-in `urllib3` retries** | No extra library | Limited to specific HTTP libraries |

For my use case, `tenacity` was the clear winner—it separated concerns cleanly and made retry behaviour explicit and configurable.

---

## What's next

The retry logic is now solid, but there are still improvements to consider:

- **Circuit breaker pattern**: stop retrying if the service is consistently down
- **Metrics collection**: track retry rates and success rates
- **Adaptive backoff**: adjust retry timing based on API response headers

But those can wait.

For now, I have a **resilient, observable** HTTP client—and that was the goal.

---

## Key takeaways

- Use libraries like `tenacity` for retry logic—don't roll your own
- Make retry configuration explicit and validated
- Use exponential backoff with jitter to prevent thundering herds
- Only retry errors that are actually retriable
- Add logging to make retry behaviour observable
- Keep retry logic separate from business logic

Even simple API clients benefit from thoughtful resilience design.