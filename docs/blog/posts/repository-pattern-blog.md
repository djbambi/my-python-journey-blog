---
title: Understanding the Repository Pattern – Separating Data Access from Business Logic
date: 2026-03-10
categories:
  - Python
authors:
  - david
tags:
  - pybites
  - pdm
  - design-patterns
  - repository-pattern
description: >
  After learning about abstract base classes, I explored the repository pattern
  by building a weather data pipeline with swappable pandas and polars backends.
  This post explains what the repository pattern is, why it matters, and how I
  implemented it step by step.
---

# Understanding the Repository Pattern: Separating Data Access from Business Logic

After learning about abstract base classes, I wanted to see them used in a
real design pattern. The **repository pattern** was the perfect next step.

The idea is simple: your business logic shouldn't know or care *where* the data
comes from. It just asks a repository for it.

This post explains how I implemented the repository pattern using weather data,
with swappable **pandas** and **polars** backends.

<!-- more -->

---

## The problem: business logic tied to data access

Imagine I have a function that reads weather data from a JSON file and
calculates the average temperature:

```python
import json
import pandas as pd

def weather_summary():
    with open("weather_data.json") as f:
        raw = json.load(f)

    df = pd.DataFrame(raw["data"])
    avg_temp = df["temp"].mean()
    print(f"Average temperature: {avg_temp:.1f}°C")
```

This works—but it has problems:

- **Tightly coupled**: the function depends directly on pandas *and* the file format
- **Hard to test**: you can't test the summary logic without a real JSON file
- **Hard to change**: switching to polars means rewriting the whole function
- **Mixed responsibilities**: data loading and business logic are tangled together

---

## The repository pattern: a middleman for your data

The repository pattern introduces a layer between your business logic and your
data source.

Think of it like ordering food at a restaurant:

- You (the business logic) ask the **waiter** (the repository) for food
- You don't care whether the kitchen uses a gas oven or an electric one
- You just get your meal in a consistent format

In code terms:

- Your business logic asks a **repository** for data
- The repository returns it in a consistent format (dataclasses)
- You don't care whether it used pandas, polars, SQL, or an API behind the scenes

---

## Step 1: define the data model

First, I needed a simple data container for a weather record. This has no
dependencies on any library—it's just a plain dataclass:

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class WeatherRecord:
    dt: int             # Unix timestamp
    temp: float         # Temperature in Celsius
    humidity: int       # Humidity percentage
    description: str    # e.g., "overcast clouds"
```

Using `frozen=True` makes the records immutable—once created, they can't be
changed. This prevents accidental data corruption downstream.

---

## Step 2: define the repository interface

Next, I defined the **contract** that any repository must follow. This uses an
abstract base class:

```python
from abc import ABC, abstractmethod

class WeatherRepository(ABC):
    @abstractmethod
    def get_weather(self) -> list[WeatherRecord]:
        """Return a list of WeatherRecords from some data source."""
        ...
```

This says: "Any class that inherits from `WeatherRepository` **must** implement
a `get_weather()` method that returns a list of `WeatherRecord` objects."

It doesn't say *how* to get the data. That's the whole point.

---

## Step 3: implement a pandas repository

The first concrete implementation uses pandas to load and parse the JSON data:

```python
import json
import pandas as pd

class PandasWeatherRepository(WeatherRepository):
    def __init__(self, filepath: str):
        self.filepath = filepath

    def get_weather(self) -> list[WeatherRecord]:
        with open(self.filepath) as f:
            raw = json.load(f)

        df = pd.DataFrame(raw["data"])

        records = []
        for _, row in df.iterrows():
            records.append(
                WeatherRecord(
                    dt=row["dt"],
                    temp=row["temp"],
                    humidity=row["humidity"],
                    description=row["weather"][0]["description"],
                )
            )
        return records
```

The key detail: the nested `"weather"` field in the JSON contains a list of
dictionaries. We extract the description with `row["weather"][0]["description"]`.

---

## Step 4: implement a polars repository

The second implementation does the same thing using polars:

```python
import json
import polars as pl

class PolarsWeatherRepository(WeatherRepository):
    def __init__(self, filepath: str):
        self.filepath = filepath

    def get_weather(self) -> list[WeatherRecord]:
        with open(self.filepath) as f:
            raw = json.load(f)

        # Extract descriptions before creating the DataFrame
        for entry in raw["data"]:
            entry["description"] = entry["weather"][0]["description"]

        df = pl.DataFrame(raw["data"])

        records = []
        for row in df.iter_rows(named=True):
            records.append(
                WeatherRecord(
                    dt=row["dt"],
                    temp=row["temp"],
                    humidity=row["humidity"],
                    description=row["description"],
                )
            )
        return records
```

Polars doesn't handle nested data as easily as pandas, so I extracted the
description **before** creating the DataFrame. The end result is identical—a
list of `WeatherRecord` objects.

---

## Step 5: write business logic that uses the interface

Now the important part—business logic that **only depends on the interface**:

```python
def weather_summary(repo: WeatherRepository) -> None:
    records = repo.get_weather()

    avg_temp = sum(r.temp for r in records) / len(records)
    avg_humidity = sum(r.humidity for r in records) / len(records)

    print(f"Total records: {len(records)}")
    print(f"Average temperature: {avg_temp:.1f}°C")
    print(f"Average humidity: {avg_humidity:.1f}%")
    for r in records:
        print(f"  {r.dt} | {r.temp}°C | {r.humidity}% | {r.description}")
```

Notice what this function **doesn't** import: no `pandas`, no `polars`, no
`json`. It only knows about `WeatherRepository` and `WeatherRecord`.

---

## Swapping repositories without changing business logic

Here's where the pattern pays off:

```python
if __name__ == "__main__":
    # Using pandas
    pandas_repo = PandasWeatherRepository("weather_data.json")
    print("=== Pandas Repository ===")
    weather_summary(pandas_repo)

    # Using polars
    polars_repo = PolarsWeatherRepository("weather_data.json")
    print("\n=== Polars Repository ===")
    weather_summary(polars_repo)
```

Both produce **exactly the same output**. The `weather_summary()` function
didn't change at all. I just swapped the repository.

---

## Project structure

In a real project, each piece would live in its own file:

```
weather_project/
├── models.py                  # WeatherRecord dataclass
├── repository.py              # WeatherRepository ABC (interface)
├── pandas_repository.py       # PandasWeatherRepository
├── polars_repository.py       # PolarsWeatherRepository
├── weather_summary.py         # Business logic
├── main.py                    # Entry point, picks which repo to use
└── weather_data.json          # Data file
```

This separation keeps things clean:

- Someone using polars doesn't need pandas installed (and vice versa)
- Each file has a **single responsibility**
- Adding a new repository (e.g., `sql_repository.py`) doesn't touch any
  existing files

---

## What I learned

Building this taught me several important lessons:

### 1. Interfaces create flexibility

By defining `WeatherRepository` as an abstract class, I created a contract that
any data source can fulfil. The business logic doesn't care about the
implementation details.

### 2. Dataclasses are the glue

`WeatherRecord` acts as a **common language** between the repositories and the
business logic. Everyone agrees on the shape of the data.

### 3. Separation of concerns matters

Keeping data access separate from business logic makes both easier to
understand, test, and change independently.

### 4. The pattern makes testing easy

I could create an `InMemoryWeatherRepository` for testing that returns
hardcoded data—no files, no libraries, no network calls needed:

```python
class InMemoryWeatherRepository(WeatherRepository):
    def __init__(self, records: list[WeatherRecord]):
        self._records = records

    def get_weather(self) -> list[WeatherRecord]:
        return self._records
```

### 5. ABCs vs Protocols

Python offers two ways to define interfaces—ABCs (used here) and Protocols.
ABCs enforce the contract at class definition time through inheritance.
Protocols use structural typing (duck typing)—any class with the right methods
qualifies, without needing to inherit anything.

---

## Comparing approaches

| Approach | Pros | Cons |
|----------|------|------|
| **Everything in one function** | Simple, quick to write | Tightly coupled, hard to test or change |
| **Repository with ABC** | Enforced contract, clear inheritance | Requires subclassing |
| **Repository with Protocol** | No inheritance needed, duck typing | Contract only checked by type checkers |

For learning purposes, the ABC approach made the pattern most explicit and
visible. In production, either approach works well.

---

## Key takeaways

- The repository pattern separates **what data you need** from **how you get it**
- Define a clear interface using an ABC or Protocol
- Use dataclasses as a common data format between layers
- Business logic should depend on interfaces, not implementations
- This makes your code flexible, testable, and easy to extend
- Each repository can live in its own file with its own dependencies

Even for a simple weather data project, the repository pattern made the code
cleaner and more maintainable. It's a pattern I'll reach for again.
