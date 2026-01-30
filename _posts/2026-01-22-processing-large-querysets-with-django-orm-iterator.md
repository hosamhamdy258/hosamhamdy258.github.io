---
title: "Processing Large QuerySets with Django ORM iterator()"
date: 2026-01-22 20:00:00 +0200
categories: [Django, Python]
tags: [django, orm, performance, python]
---

## Introduction

When working with large datasets in Django, the ORM's default caching behavior can quietly become a liability.

QuerySet caching is convenient until you iterate over hundreds of thousands of rows and watch memory usage spike.

The `.iterator()` method exists to solve this exact problem. Used correctly, it enables memory-efficient, streaming-like processing. Used incorrectly, it breaks assumptions about caching, prefetching, and reuse.

This post explains when `.iterator()` is the right tool and when it isn’t.

## What is `.iterator()`?

In Django, when you evaluate a QuerySet, it usually caches the results. This means if you iterate over the same QuerySet twice, it won't hit the database again. However, for very large results, this cache can consume a lot of memory.

The `.iterator()` method allows you to iterate over a QuerySet without caching the results at the QuerySet level.

Once consumed, an iterator is exhausted and cannot be reused.

## Basic Usage

```python
# Standard way (caches results)
users = User.objects.all()
for user in users:
    print(user.username)

# Using iterator (does not cache results)
users = User.objects.all().iterator()
for user in users:
    print(user.username)
```

## Pros

- **Significant Memory Savings**: Ideal for processing millions of rows where caching the entire result set would lead to an `OutOfMemoryError`.
- **Lower Latency**: Execution can start as soon as the first few rows are returned from the database.
- **Chunking**: You can specify a `chunk_size` to balance memory usage and database round trips.

## Cons

- **No Caching**: If you need to iterate over the same data again, you'll have to hit the database a second time.
- **Prefetching Issues**: `.iterator()` does not work with `prefetch_related()` (**before 4.1 Django versions**), it will ignore prefetching. It only works with `select_related()`. [Django 4.1 Release Notes](https://docs.djangoproject.com/en/6.0/releases/4.1/#models)
- **Database Backend Support**: Some database backends might have specific ways of handling long-running cursors.

## When to Use It?

Use `.iterator()` when you are performing a **single-pass processing** of a large amount of data, such as in a background task, data migration, or export script (like a CSV or Excel export).

## Practical Examples

### Data Export Example

```python
def export_users_to_csv(filepath):
    """Export all users to CSV file using iterator for memory efficiency."""
    with open(filepath, 'w', newline='', encoding='utf-8') as csvfile:
        writer = csv.writer(csvfile)
        writer.writerow(['id', 'username', 'email'])

        # Using iterator to avoid loading all users into memory
        qs = User.objects.values_list('id', 'username', 'email').iterator(chunk_size=2000)

        for user_id, username, email in qs:
            writer.writerow([user_id, username, email])
```

### Chunked Processing Example

```python
# Process 2000 records at a time to balance memory and database hits
for user in User.objects.all().iterator(chunk_size=2000):
    process_user_data(user)  # Your processing logic
```

### Common Pitfall: Multiple Iterations

```python
users = User.objects.only('id', 'username', 'email').iterator(chunk_size=2000)
for user in users:
    print(user.username)

for user in users:  # Second iteration - empty iterator!
    print(user.email)
```

**Important Reminder**: Once consumed, an iterator is exhausted and cannot be reused, because Iterators represent a one-way stream. Once the underlying database cursor is exhausted, there is no way to rewind it.

## Important Clarification: `iterator(chunk_size)` Does Not Mean Multiple Queries

A common misconception about `.iterator(chunk_size=...)` is that it splits the query into multiple `LIMIT` based database queries. It does not.

Consider the following code:

```python
for user in User.objects.iterator(chunk_size=2000):
    process(user)
```

With database query logging enabled, you will still see a single SQL query:

```sql
SELECT "auth_user"."id",
       "auth_user"."username",
       "auth_user"."email"
FROM "auth_user";
```

This behavior is intentional and correct.

### What `chunk_size` Actually Controls

The `chunk_size` parameter does not affect the SQL query itself. Instead, it controls how many rows Django fetches from the database cursor at a time.

In other words:

- The database executes one query
- Django opens a cursor for that query
- Rows are fetched from the cursor in batches of `chunk_size`
- Each batch is:
  - Converted into model instances
  - Yielded one by one
  - Discarded immediately (no QuerySet caching)

Conceptually, the flow looks like this:

```
Single SQL query
        ↓
Database cursor
        ↓
Fetch 2000 rows → process → discard
Fetch 2000 rows → process → discard
Fetch 2000 rows → process → discard
```

### Important Mental Model

`.iterator()` optimizes how long rows live in Python memory, not how many rows the database returns.

## Best Practices

- **Use `chunk_size` parameter**: For most use cases, `chunk_size=2000` provides a good balance between memory usage and database round-trips.
- **Combine with `select_related()`**: Use `.iterator()` with `select_related()` for foreign key relationships to avoid N+1 queries.
- **Be aware of `prefetch_related()`**: Remember that prefetching is ignored with iterators (before Django 4.1 versions).
- **Profile memory usage**: Use memory profiling tools to verify that `.iterator()` is actually helping your specific use case.
- **Consider database-specific optimizations**: Some databases have server-side cursors that might be more efficient.

## Conclusion

The `.iterator()` method is a powerful tool in Django's performance optimization toolkit. It's particularly valuable for data migrations, exports, and background processing of large datasets. However, it's not a silver bullet understanding its limitations, especially around caching and prefetching, is crucial for using it effectively.

When you need to process large amounts of data in a single pass and memory is a concern, `.iterator()` should be your go-to solution. Just remember to profile your specific use case and consider the trade-offs between memory usage and database round-trips.

### Further Reading

- [Django QuerySet API Reference: iterator()](https://docs.djangoproject.com/en/stable/ref/models/querysets/#iterator)

- [Django QuerySet API Reference: select_related() and prefetch_related()](https://docs.djangoproject.com/en/stable/topics/db/optimization/#use-queryset-select-related-and-prefetch-related)
