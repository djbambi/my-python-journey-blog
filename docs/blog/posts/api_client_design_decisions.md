---
title: Starting the PyBites PDM – from “can code” to “can build”
date: 2026-01-11
categories:
  - Python
authors:
  - david
tags:
  - pybites
  - pdm
description: >
  I'm developing an api client to get data from OpenWeather. This is more of a mind dump describing the process. I'm interested in explaining why the design decisions were made, justifying them and also considering alternative approaches.
---

# OpenWeather API Client Design”

__This is a mind dump of my design decisions for the OpenWeather API Client.__

# Designing a Simple, Testable Weather API Client in Python

When I started building a small Python client to fetch data from the OpenWeather API, my goal wasn’t to build a full pipeline or production system.  
It was to **understand the design decisions** that make Python code easier to reason about, test, and evolve.

This post walks through my thought process and the key choices I made while implementing a simple weather client.

<!-- more -->

---

## The problem I was solving

At its core, the task was simple:

- Call an HTTP API  
- Handle success and failure correctly  
- Return parsed JSON data  

However, even simple problems can turn messy quickly if responsibilities aren’t clear. I wanted to avoid:

- hard-coded configuration  
- tightly coupled code  
- untestable HTTP logic  

So I treated this as a **design exercise**, not just a coding task.

---

## Separating configuration from behaviour

One of the first decisions I made was to **separate configuration from code logic**.

Things like:

- API keys  
- base URLs  
- timeouts  

are **not behaviour** — they’re *environment-specific configuration*. Embedding them directly in the client would make the code harder to reuse and harder to test.

### Using a dedicated config module

I created a `config.py` file responsible solely for configuration:

```python
class Settings(BaseSettings):
    openweather_api_key: str
    openweather_timeout_s: float = 10.0
    openweather_base_url: HttpUrl = "https://api.openweathermap.org/data/3.0/onecall/timemachine"
```

This immediately gave me:

- a single source of truth for config  
- clear documentation of required vs optional settings  
- fast failure if configuration is invalid  

---

## Why I used Pydantic for configuration

Rather than reading environment variables manually with `os.environ`, I used **Pydantic Settings**.

This was a deliberate choice.

### Benefits I gained immediately

- **Validation at startup** (e.g. timeout must be > 0)  
- **Type safety** (URLs must be valid URLs)  
- **Cleaner code** (no scattered `os.environ` lookups)  
- **Clear failure modes** (misconfiguration fails fast)  

This pushed configuration errors to the **edges of the system**, where they belong.

---

## Designing the weather client around behaviour

The actual weather client function is intentionally small:

```python
def fetch_openweather_data(*, session, url, params, timeout_s):
    response = session.get(url, params=params, timeout=timeout_s)
    response.raise_for_status()
    return response.json()
```

This function does exactly three things:

1. makes a request  
2. fails if the HTTP status is bad  
3. returns parsed JSON  

No configuration. No retries. No orchestration.

Just behaviour.

---

## Dependency injection: passing in the session

One of the most important decisions I made was **not** to create the HTTP session inside the client.

Instead, I inject it:

```python
fetch_openweather_data(session=session, ...)
```

### Why this matters

By injecting the session:

- the client has **no hidden dependencies**  
- HTTP configuration (headers, adapters, retries) lives elsewhere  
- the function is easy to test in isolation  

The client doesn’t care *where* the session came from — only that it behaves like one.

This follows a simple rule:

> Functions should depend on *what they need*, not *where it comes from*.

---

## How this enables easy testing

Because the session is injected, I can test the client **without making real HTTP calls**.

In tests, I pass in a mock session that:

- pretends to have a `.get()` method  
- returns a fake response object  
- simulates success or failure  

This allowed me to test:

- success paths  
- error propagation  
- correct wiring of arguments  

All without touching the network.

The design made testing straightforward — not something I had to “fight”.

---

## Freezing the scope at an MVP

During development, it became obvious how quickly this could grow:

- retries with backoff  
- multiple locations and dates  
- partitioned storage  
- orchestration via Airflow  

All valid ideas — but not necessary yet.

I deliberately froze the scope at a **minimum viable client** so I could:

- reason clearly about the code  
- validate design choices through testing  
- avoid premature abstraction  

This checkpoint now gives me a solid foundation to build on later.

---

## Key takeaways

What this small exercise reinforced for me:

- Configuration is not behaviour — keep it separate  
- Validate configuration early and explicitly  
- Inject dependencies to keep code testable  
- Small, focused functions are easier to reason about  
- Stopping early is a design decision, not a failure  

Even a simple API client benefits from thoughtful structure.

---

## What’s next

The next steps (when needed) are clear and incremental:

- add retries with exponential backoff  
- extract parameter-building logic  
- integrate orchestration  

But those decisions can wait.

For now, I have a clean, testable core — and that was the real goal.
