---
title: Django ORM Performance all() vs values() vs values_list()
date: 2026-02-05 20:00:00 +0200
categories: [Django, Performance]
tags: [python, django, orm, database]
---

## Introduction

In Django, the ORM is your best friend when it comes to productivity, but it can quickly become your worst enemy for performance. The "default" way to query data is `all()`, which returns full model instances. While convenient, this comes with a heavy price in both memory and execution time.

we have talked about `all()` and `only()` in previous posts ([Django ORM at Scale Why .all() Is a Hidden Performance Trap]({% post_url 2026-01-15-django-orm-at-scale-why-all-is-a-hidden-performance-trap %})), but today we will talk about `values_list()` and compare it with `all()` and `values()`.

If you don't need the full object with its methods and lazy loading capabilities, then using `values()` or `values_list()` can save your application’s performance and reduce its memory footprint significantly.

In this post, we’ll benchmark these three main contenders and see why choosing the right data structure matters.

---

## The Three Main Contenders

Most Django queries fall into one of these three categories based on what they return:

### 1. `all()` (Full Model Objects)
The default approach. It returns a `QuerySet` of model instances.
- **What it returns**: A list-like object of classes ( A lazy QuerySet of model instances ).
- **When to use**: When you need to call model methods or perform updates (`.save()`).

### 2. `values()` (Dictionaries)
Returns a `QuerySet` that evaluates to a list of dictionaries.
- **What it returns**: `[{'field': 'value'}, ...]`
- **When to use**: For data serialization (e.g., JSON APIs) where keys are needed.

### 3. `values_list()` (Tuples)
Returns a `QuerySet` that evaluates to a list of tuples.
- **What it returns**: `[('value1', 'value2'), ...]`
- **When to use**: For high-performance processing where the overhead of dictionary keys is too much.

---

## The Performance Extras

To push performance even further, Django offers specialized tools that work alongside our main contenders:

- **`only(*fields)`**: A variant of `all()` that fetches specific columns but still returns model instances. It's the "middle ground" between `all()` and `values()`.
- **`flat=True`**: An option for `values_list()` when fetching a single field. It returns a flat list (e.g., `[1, 2, 3]`) instead of a list of single-item tuples.

---

## Benchmark Comparison


---

## The Trick: Safe Access of values_list

The problem with `values_list()` is usability. Hardcoding indices like `row[1]` is a recipe for disaster if the query changes; your code may break silently or process the wrong data.

To bridge the gap between **dictionary readability** and **tuple performance**, we can use an **Index Map** pattern.

### How it Works
By using a single source of truth for your fields, you can generate both your query and your accessor map dynamically with zero runtime overhead.

```python
# 1. Single source of truth for fields
MY_FIELDS = ['id', 'title', 'author_email']

# 2. Safely generate index map
INDEX_MAP = {name: i for i, name in enumerate(MY_FIELDS)}

# 3. Query using the same list
data = Record.objects.values_list(*MY_FIELDS)

# 4. Access with named-like clarity and tuple speed
for row in data:
    process_title(row[INDEX_MAP['title']])
```
---

## Performance vs. Simplicity: A Fair Warning

It is important to note that these benchmarks represent extreme cases with millions of records. 

### When to Keep it Simple
For the majority of applications where you are dealing with a few low amounts of records per request, the default `.all()` is perfectly fine. It is readable, flexible, and gives you the full power of the ORM. Don't optimize prematurely; focus on code clarity first.

### When to Seek Every Byte
You should reach for `values_list` and the **Index Map** pattern only when you are in "performance critical" paths:
- Exporting millions of rows to CSV/XLSX.
- Large-scale data migrations.
- Heavy background processing worker tasks.
- High-traffic APIs where every kilobyte of memory pressure counts.

If you aren't feeling the pain of memory limits or slow responses, keep your code simple. But when the scale hits, you now know exactly where the finish line is.

---

## Access Efficiency Comparison


> Benchmarks were run on SQLite. While relative Python-side overhead remains consistent across databases, absolute timings will differ on PostgreSQL or MySQL.

### Experiment 1: Access Overhead (Dict vs. Tuple)

To see the real-world impact at scale, I ran a benchmark using **1,000,000 records** on a `Record` model with 5 fields in an on-disk SQLite database.

| Method                                              | Retrieval Time | Memory Usage |
| :-------------------------------------------------- | :------------- | :----------- |
| **`.all()` (Full Objects)**                         | 10.84s         | 976.56 MB    |
| **`.values()` (Dictionaries)**                      | 6.47s          | 938.41 MB    |
| **`.values_list()` (Tuples)**                       | 6.40s          | 762.93 MB    |
| **`.only('id', 'title')` (Selective)**              | 4.55s          | 389.10 MB    |
| **`.values_list('id', flat=True)` (Single column)** | 0.51s          | 38.15 MB     |

Notice the **175MB difference** between `.values()` and `.values_list()`. Even more striking is the **213MB saving** when moving from `.all()` directly to `.values_list()`. Even without the overhead of model instances, converting rows into dictionaries costs a significant amount of RAM just to store the string keys for every single row.

---

### Experiment 2: Access Overhead (Dict vs. Tuple)

Fetching the data is only half the battle. What happens when you actually *process* those 1,000,000 rows? I ran a separate experiment to measure the overhead of accessing a single field (`title.upper()`) across different data structures.

| Access Pattern                | Time (s) | Efficiency  |
| :---------------------------- | :------- | :---------- |
| **`row['title']`**            | 0.052s   | Baseline    |
| **`row[INDEX_MAP['title']]`** | 0.055s   | **Optimal** |

By using a mapping derived from your field list, you get the absolute best performance Python allows while keeping your code maintainable and robust.

---

## Source Code

You can find the full benchmarking script used in this post here:
[benchmark_values_and_values_list.py](https://github.com/hosamhamdy258/benchmarks/blob/main/benchmark_values_and_values_list.py)

This script is optimized to:
- Persist 1,000,000 records to a local SQLite database.
- Compare memory usage using `pympler`.
- Measure access overhead for various retrieval patterns.

---

## The Golden Rules

1. **Identity vs. Data**: Use `all()` when you need model methods or `.save()`. Use `values()` or `values_list()` when you just need data.
2. **Performance at Scale**: When processing millions of rows, use `values_list(*FIELDS)` and access by an **Index Map** to save hundreds of megabytes of RAM without losing safety.
3. **The Flat List Shortcut**: Use `values_list(..., flat=True)` whenever you're fetching a single column. It's the "hidden" speed hack of Django.

---

## Summary

Django's ORM is designed for convenience, but it provides powerful escape hatches when you hit scale. Moving from `all()` to `values_list()` isn't just about database columns, it's about how Python manages memory. By avoiding dictionary hashing and model instantiation, you can process millions of records even on modest hardware.

### Key Takeaways:
1. **Memory Matters**: At the scale of 1M records, `values_list()` saves nearly 200MB of RAM over `values()` just by avoiding dictionary overhead.
2. **Safety First**: Never use raw indices in loops. Always use a mapping or unpacking pattern to keep your `values_list` access robust.
3. **Spectrum of Speed**: `all()` is the most flexible, `values()` is the middle ground, and `values_list()` is the high-performance choice for heavy data pipelines.

---

## Conclusion
While `all()` is the easiest to use and `values()` is the middle ground, `values_list()` combined with an **Index Map** is the high-performance finish line for heavy data pipelines. By avoiding model instantiation and dictionary hashing, you can process millions of records on modest hardware without sacrificing code clarity.

