---
title: Django ORM at Scale Why .all() Is a Hidden Performance Trap
date: 2026-01-15 20:00:00 +0200
categories: [Django, ORM]
tags: [python, django, database]
---

## Introduction

Most Django tutorials start with something like:

```python
User.objects.all()
```

It works, it’s clean, and it feels “Pythonic”.

But if you keep that habit in real projects, you’ll eventually ship a slow page, blow up memory in a worker, or accidentally pull 100k rows from the database just to show 20 items on a screen.

This post is about using Django ORM: understanding what it returns, what SQL it roughly generates, and why being **selective** (explicit) is often better than being implicit.

## What is Django ORM?

Django's Object-Relational Mapper (ORM) is a layer that lets you work with database rows as Python objects.

- You write Python expressions.
- Django converts them into SQL.
- The DB executes the SQL and returns rows.
- Django turns rows into model instances (or dicts/tuples depending on the query).

## The `.all()` habit: fine in tutorials, risky at scale

Tutorials use `.all()` because it’s simple and teaches the idea of “querying”.

But in real systems, `.all()` is often a red flag because it means:

- You didn’t choose columns (often `SELECT *`).
- You didn’t limit rows (`LIMIT`).
- You didn’t think about filters.
- You might load big text fields / JSON fields without needing them.

### SQL 101 flashback: `SELECT *` is convenient until it isn’t

Remember SQL classes?

- `SELECT * FROM users;` brings **every column** for every matched row.
- Later you realize you only need names:
  - `SELECT name FROM users;`

That difference matters because database resources and network bandwidth aren't free.

Every extra column increases the volume of data traveling across the wire, leading to higher latency and increased memory pressure. When each request consumes more memory than necessary, you effectively lower your system’s concurrency meaning your server can handle fewer simultaneous requests.

## Start being selective

If you still want **model instances** (a real `QuerySet` of `User` objects), but you don’t need every column, start with `.only()`.

### When you still want model instances: `.only()`

Use `.only()` when you want model objects but not every field.

```python
qs = User.objects.only("id", "username")
```

Typical effect:

- Django fetches only those columns initially.
- If later you access a deferred field (like `email`), Django may issue another query per object.

So `.only()` is good when:

- You truly only access those fields.
- You’re careful not to trigger extra per-row queries.

### When you don’t need model instances at all: `.values()`

A sharp reader might ask: “Why not just use `values()` and skip model instances entirely?”

You can:

```python
qs = User.objects.values("id", "username")
```

### .only() vs .values(): where ORM power really stops

Both `.only()` and `.values()` are tools for being selective, but they make a very different trade-off.

Key contrast:

- `.only()` → returns real model instances with deferred fields.
  ORM behavior continues after the query: model methods, properties, relations, and updates are still available.

- `.values()` → returns plain dictionaries.
  ORM power exists only before the query runs (filtering, annotating, ordering), but once data is fetched, model behavior is gone.

To be precise: `.values()` still returns a `QuerySet`, so you can freely chain ORM operations before execution:

```python
User.objects.values("id", "username").filter(is_active=True).order_by("username")
```

But after evaluation, each row is just a dict:

```python
for u in User.objects.values("id", "username"):
    u["username"]
    u.get_full_name()
    u.save()
```

There are no model methods, no lazy loading, no relationship traversal.

When to use which:

- Use `.values()` when you only need read-only data (APIs, exports, reports). It’s the leanest option and avoids ORM overhead entirely.
- Use `.only()` when you still need model behavior, but want to avoid pulling unnecessary columns.

A useful mental rule:

- `.values()` keeps ORM power before the query executes.
- `.only()` keeps ORM power after the data is fetched.

Choosing between them is less about performance tricks and more about being honest about what your code actually needs.

### Pros & Cons of `.only()`

- **Pros**:
  - Reduces database load and network traffic for wide tables.
  - Lowers memory usage by creating "slimmer" model instances.
- **Cons**:
  - Risk of **N+1 queries** if you accidentally access a deferred field later.
  - Can be harder to maintain if fields are frequently added/removed.

## “.Defer() is good, but I prefer explicit over implicit”

With `.only()`, you’re being explicit about what fields you intend to use. That’s good, but it comes with a responsibility:

- If you later access a deferred field, Django may perform extra queries.

So the rule is simple:

- Use `.all()` when you truly need full objects.
- Use `.only()` when you know exactly which fields you’ll touch.

## Django example: implicit `SELECT *` vs explicit `SELECT name` (100k rows)

Assume you have a `User` model with a lot of fields and ~100k records.

### Implicit style (common in tutorials)

```python
qs = User.objects.all()
for u in qs:
    do_something(u.username)
```

This is conceptually equivalent to:

```sql
SELECT * FROM auth_user;
```

Even if you only use `username`, you potentially pulled every column for 100k rows.

### Explicit style (select only what you need)

```python
qs = User.objects.only("id", "username")
for u in qs:
    do_something(u.username)
```

This tends to become:

```sql
SELECT "auth_user"."id", "auth_user"."username" FROM "auth_user";
```

### Timing it (A Real-World Benchmark)

Let's walk through a standalone script that populates 50,000 records and compares `.all()` vs. `.only()`. You can download the full script [benchmark_all_vs_only.py](https://github.com/hosamhamdy258/benchmarks/blob/main/benchmark_all_vs_only.py) .

#### Step 1: Standalone Django Setup

To run Django code outside of a project structure (like `manage.py`), we need to manually configure the settings. We use an in-memory SQLite database for speed and simplicity.

```python
import django
from django.conf import settings

# Configure Django settings manually
settings.configure(
    DATABASES={
        "default": {
            "ENGINE": "django.db.backends.sqlite3",
            "NAME": ":memory:",
        }
    },
    INSTALLED_APPS=[
        "__main__",
    ],
)
django.setup()
```

_Ref: [Calling django.setup() for standalone scripts](https://docs.djangoproject.com/en/stable/topics/settings/#calling-django-setup-is-required-for-standalone-scripts)_

#### Step 2: Reusable Helpers

We define a few helpers to make the benchmarking cleaner. The `timer` context manager tracks time, and `measure_memory` uses `pympler` for accurate sizing.

```python
import time
from contextlib import contextmanager
from pympler import asizeof  # pip install pympler

@contextmanager
def timer(label: str):
    start = time.perf_counter()
    try:
        yield
    finally:
        elapsed = time.perf_counter() - start
        print(f"{label}: {elapsed:.4f}s")

def measure_memory(obj, label: str):
    size_mb = asizeof.asizeof(obj) / 1024 / 1024
    print(f"{label} memory: {size_mb:.2f} MB")
```

#### Step 3: Define the Model

We define a `Record` model with a mix of field types to simulate a real object. Note the `TextField` and `JSONField`, which can become large in production.

```python
from django.db import models, connection

class Record(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    metadata = models.JSONField(default=dict)
    author_email = models.EmailField()
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        app_label = "__main__"
```

#### Step 4:Create the table in our in-memory DB

```python
with connection.schema_editor() as schema_editor:
    schema_editor.create_model(Record)
```

#### Step 5: Populate Data

We create 50,000 records. Using `bulk_create` is much faster than saving objects one by one.

```python
def populate_db(count=50000):
    print(f"Populating {count} records...")
    records = [
        Record(
            title=f"Record {i}",
            content="A " * 100,
            metadata={"key": "value", "id": i},
            author_email=f"user{i}@example.com",
        )
        for i in range(count)
    ]
    Record.objects.bulk_create(records)
```

_Ref: [bulk_create() documentation](https://docs.djangoproject.com/en/stable/ref/models/querysets/#bulk-create)_

#### Step 6: The Benchmark

We can now use our helpers to run the comparison cleanly.

```python
def benchmark():
    # Warm up
    _ = Record.objects.count()

    print("\nBenchmarking Record.objects.all()...")
    with timer("Time"):
        all_records = list(Record.objects.all())
    measure_memory(all_records, "Memory")

    print("\nBenchmarking Record.objects.only('id', 'title')...")
    with timer("Time"):
        only_records = list(Record.objects.only("id", "title"))
    measure_memory(only_records, "Memory")
```

_Ref: [QuerySet.only() documentation](https://docs.djangoproject.com/en/stable/ref/models/querysets/#only), [Pympler documentation](https://pythonhosted.org/Pympler/)_

#### Results (50,000 Records)

When we run this script, the difference is stark:

| Query Method                         | Time (seconds) | Memory Usage |
| :----------------------------------- | :------------- | :----------- |
| `Record.objects.all()`               | **0.4708s**    | **62.55 MB** |
| `Record.objects.only('id', 'title')` | **0.2680s**    | **28.23 MB** |

**What we learned**:

- **Speed**: `.only()` was nearly **2x faster**.
- **Memory**: `.only()` used **less than half** the memory.

When you scale this to a production web server handling 100 concurrent requests, that 34MB difference becomes **3.4 GB of wasted RAM**. That is often the difference between a smooth-running site and a server OOM (Out of Memory) crash.

**Important:** The exact numbers depend on DB type, network latency, row size, and server resources.

## Conclusion

Django ORM is powerful and needs awareness.

Be selective, stay explicit.

### Further Reading

If you’re ready to level up, take another look at the official Django ORM documentation for:

- `iterator()` for processing massive datasets with low memory overhead.
- `select_related()` and `prefetch_related()` to avoid N+1 queries across relations.
