---
title: The Real Difference Between pk and id in Django (And Why It Breaks Queries)
date: 2026-01-29 20:00:00 +0200
categories: [Django, ORM]
tags: [python, django, database]
---

## Introduction

In Django, you’ll often see both `pk` and `id` in queries. Most of the time, they behave the same, so it’s easy to assume they are identical.  

But the moment you define a custom primary key or reference a related model, the difference becomes **critical** and can silently break queries, especially joins.

This post builds a mental model for `pk` vs `id`: when they match, when they differ, and why using `pk` consistently will save you from hidden bugs.

---

## Definitions

- **`pk`**: an alias for the primary key field of the model. Always points to whichever field is marked `primary_key=True`.
- **`id`**: a normal field name. Exists only if the model has an `id` field (default created by Django or defined manually).

---

## When `pk` and `id` are the same

If you don’t define a primary key, Django automatically adds one:

- Historically: `AutoField` named `id`  
- Newer projects: `BigAutoField` (depending on `DEFAULT_AUTO_FIELD`)  

In this default case:

```python
# id is the PK
class Book(models.Model):
    title = models.CharField(max_length=200)

Book.objects.get(pk=1)
Book.objects.get(id=1)
```

both are equivalent. `pk` is just an alias for `id`.

---

## When `pk` and `id` differ

Define a natural primary key e.g., ISBN for a book:

```python
class Book(models.Model):
    isbn = models.CharField(max_length=13, unique=True, primary_key=True)
    title = models.CharField(max_length=200)
```

Now:

- `Book.pk` points to `isbn`
- `Book.objects.get(id=1)` raises an error because no `id` field exists
- `Book.objects.get(pk=1)` runs a valid query but raises `DoesNotExist` because `1` is not a valid ISBN
- `Book.objects.get(pk="978-3-16-148410-0")` works

**Rule of thumb:**

- Use `pk` when referring to the primary key.
- Use `id` only if you are certain the model has an `id` field.

---

## What happens to queries

### Default PK (ID)

```python
Book.objects.get(pk=7)
```

```sql
SELECT * FROM library_book WHERE id = 7 LIMIT 1;
```

### Custom PK (ISBN)

```python
Book.objects.get(pk="978-3-16-148410-0")
```

```sql
SELECT * FROM library_book WHERE isbn = '978-3-16-148410-0' LIMIT 1;
```

Your code didn’t change, but the database column used changed. That’s why `pk` is powerful: it’s stable across PK changes.

---

## The “mixing in query” problem

Foreign keys referencing a model without an `id` can break if you use `id`:

```python
class Review(models.Model):
    book = models.ForeignKey(Book, on_delete=models.CASCADE, related_name='reviews')
    rating = models.IntegerField(choices=[(i, i) for i in range(1, 6)])
```

| Expression        | Meaning                           | Safe if PK changes? |
| ----------------- | --------------------------------- | ------------------- |
| `review.book_id`  | Local FK column on `Review`       | Yes                 |
| `review.book__id` | Follow relation to `id` on `Book` | Only if `id` exists |
| `review.book__pk` | Follow relation to PK on `Book`   | Always              |


- `book_id` always refers to the local FK column, not the primary key field name on the related model.

**Example:**

```python
Review.objects.filter(book__pk="978-3-16-148410-0")   # Safe
Review.objects.filter(book__id="978-3-16-148410-0")   # Unsafe if no id
Review.objects.filter(book_id="978-3-16-148410-0")    # Always works regardless of result
```

---

## The Refactoring (Breaking Joins)

This is a dangerous "silent failure" that happens when you change a Primary Key without a data migration.

### 1. The Starting State (ISBN as PK)
Your models and database are in sync. The `Review.book_id` column contains ISBN strings.

```python
# ISBN is the PK
class Book(models.Model):
    isbn = models.CharField(max_length=13, primary_key=True)

class Review(models.Model):
    # This column stores ISBN strings
    book = models.ForeignKey(Book, on_delete=models.CASCADE)
```

### 2. The Refactor (ID as PK)
You change `isbn` to a regular field. Django adds an `id` column to `Book`.

```python
# ID is now the PK
class Book(models.Model):
    isbn = models.CharField(max_length=13, unique=True, primary_key=False)
```

### 3. The Silent Failure
If you run your code now **without a data migration**, your queries will behave differently:

- **`Review.objects.filter(book__pk=ISBN)`**: This usually continues to work because Django resolves book__pk to the local FK column when no JOIN is required, masking the underlying schema mismatch, Since `pk` is a simple lookup, Django filters the `book_id` column on the `Review` table directly. Since that column still contains ISBN strings, it finds the match, but this should not be relied upon as it depends on mismatched foreign key types.
- **`Review.objects.filter(book__title=...)`**: This **breaks completely**. Because it requires an `INNER JOIN`, Django generates this SQL:
  
  ```sql
  SELECT ... FROM review 
  INNER JOIN book ON (review.book_id = book.id) 
  WHERE book.title = '...'
  ```
  The database tries to compare `'978123...'` (String) with `1` (Integer). Since they never match, the query returns an empty `QuerySet`.

---

## My Experiment

To verify this behavior, I ran an experiment with real data. Here's what happened:

**Before Refactor (ISBN as PK):**

```python
# Query: Review.objects.filter(book__title='Cloud Computing Guide')
```

```sql
SELECT "library_review"."id", "library_review"."book_id", "library_review"."rating" 
FROM "library_review" 
INNER JOIN "library_book" ON ("library_review"."book_id" = "library_book"."isbn") 
WHERE "library_book"."title" = 'Cloud Computing Guide'
```

**Results Found: 1**

**After Refactor (ID as PK, no data migration):**

```python
# Same query: Review.objects.filter(book__title='Cloud Computing Guide')
```

```sql
SELECT "library_review"."id", "library_review"."book_id", "library_review"."rating" 
FROM "library_review" 
INNER JOIN "library_book" ON ("library_review"."book_id" = "library_book"."id") 
WHERE "library_book"."title" = 'Cloud Computing Guide'
```

**Results Found: 0** (Silent failure)

Notice the join condition changed from `book_id = isbn` (String = String) to `book_id = id` (String = Integer). The database cannot match `'8901234567890'` with `1`, so the join returns nothing.

Meanwhile, simple lookups still "work":

```python
# Query: Review.objects.filter(book__pk='8901234567890')
# Results Found: 1 (Deceptive success)
```

This works because it queries the local `book_id` column directly, which still contains the ISBN string. But any query requiring a join will silently fail.

---

**Rule of thumb:** If you refactor a Primary Key, you **must** run a data migration to update every Foreign Key column in your database.

---


## The Conflict

When refactoring, you might be tempted to define your own `id` field manually while the database already has one. If you're defining a new model, this is fine. Refactoring an existing one is not.

```python
class Book(models.Model):
    id = models.BigAutoField(primary_key=False)
    isbn = models.CharField(max_length=13, primary_key=True)
```

Django's system checks will catch this error before you even create a migration:

```
SystemCheckError: System check identified some issues:

ERRORS:
library.Book.id: (fields.E100) AutoFields must set primary_key=True.
```

**Why?** Django enforces that `AutoField`, `BigAutoField`, and similar fields must always be primary keys. This prevents you from creating a schema conflict where you'd try to add an `id` column that already exists in the database (which would cause `OperationalError: duplicate column name: id`).

Django's early validation protects you from this mistake before it reaches the database level.

---

## The Reverse Migration (ForeignKey Mismatch)

What if you try to go back? If you attempt to switch from a default `id` back to a custom `primary_key=True` (like `isbn`) on an existing database, you may encounter this blocker:

`django.db.utils.OperationalError: foreign key mismatch - "library_review" referencing "library_book"`

### Why It Happens
In database engines like SQLite, foreign key constraints are strictly enforced during the table re-creation process that occurs during a migration.

1. **The Conflict**: Your `Review` table's `book_id` column is configured as an integer pointing to the `Book.id` primary key.
2. **The Change**: You are trying to delete the `id` column and make `isbn` (a string) the primary key.
3. **The Block**: The database sees that `Review` is referencing a column (`id`) that is about to disappear, while the new primary key (`isbn`) doesn't match the existing foreign key's type or constraints.

To solve this, you often have to:
- Temporarily disable foreign key checks (`PRAGMA foreign_keys = OFF` in SQLite).
- Manually drop and recreate the relationships in a specialized migration.
- Or, more simply, ensure your data migration happens before the schema and constraints are finalized.

---

## Advanced Topics

### The `to_field` Parameter

By default, a `ForeignKey` references the **primary key** of the related model, not necessarily the `id` field. But what if you want to reference a different field?

Django provides the `to_field` parameter to explicitly specify which field the FK should reference. That field must have `unique=True`:

```python
class Book(models.Model):
    # Default id PK exists
    isbn = models.CharField(max_length=13, unique=True)
    title = models.CharField(max_length=200)

class Review(models.Model):
    # References isbn instead of the PK (id)
    book = models.ForeignKey(Book, on_delete=models.CASCADE, to_field='isbn')
    rating = models.IntegerField()
```



**Key implications:**

1. The FK column (`book_id`) stores ISBN strings, even though `Book` has an integer `id` PK
2. Queries using `book__pk` will filter on `Book.id` (the PK), not `book_id` (which contains ISBNs)
3. To filter by the FK column value, use `book_id` directly

**Example queries:**

```python
# Filter by the FK column (ISBN stored in book_id)
Review.objects.filter(book_id='978-3-16-148410-0')  # Works

# Filter by the Book's PK (id field)
Review.objects.filter(book__pk=1)  # Works, joins on isbn then filters Book.id

# Filter by Book's isbn field
Review.objects.filter(book__isbn='978-3-16-148410-0')  # Works, explicit join
```

**Why this matters:** The `to_field` parameter creates a mismatch between what the FK column stores (ISBN) and what the related model's PK is (id). Understanding `pk` vs `id` becomes even more critical here.

**Documentation:** [ForeignKey.to_field](https://docs.djangoproject.com/en/6.0/ref/models/fields/#django.db.models.ForeignKey.to_field)

---

### Composite Primary Keys

Django 5.2 introduced **composite primary keys**, where multiple fields together form the primary key:

```python
class Book(models.Model):
    pk = models.CompositePrimaryKey('isbn', 'edition')
    isbn = models.CharField(max_length=13)
    edition = models.IntegerField()
    title = models.CharField(max_length=200)
```

**Key characteristics:**

1. **No `id` field exists** - the composite PK is the only primary key
2. **`pk` is the only safe reference** - you cannot use `id` at all
3. **ForeignKeys reference the composite** - related models store both values

**Example with ForeignKey:**

```python
class Review(models.Model):
    book = models.ForeignKey(Book, on_delete=models.CASCADE)
    rating = models.IntegerField()
```

**Querying with composite PKs:**

```python
# Get by composite PK
book = Book.objects.get(pk=('978-3-16-148410-0', 1))

# Filter reviews by composite PK
Review.objects.filter(book__pk=('978-3-16-148410-0', 1))

# This would fail - no id field exists
Book.objects.get(id=1)  # FieldError
```

**Why this matters:** Composite PKs make the `pk` abstraction **essential**. There's no single `id` field to fall back on, and `pk` is the only way to reference the primary key in a database-agnostic manner.

**Documentation:** [Composite Primary Keys](https://docs.djangoproject.com/en/6.0/topics/composite-primary-key/)

---

## Outcomes (mental model)

- `pk` is an alias to the primary key.
- `id` is just a field name, it might not exist.
- `book_id` is always local and never changes meaning.
- By default, `pk` == `id` because Django creates an `id` field.
- `related__pk` is safe; `related__id` is an assumption.
- Changing the primary key changes the SQL column, but `pk` queries remain correct.
- `to_field` can make FKs reference non-PK fields, creating mismatches between FK storage and PK.
- Composite PKs eliminate the `id` field entirely, making `pk` the only safe abstraction.

---

## Summary

Understanding the difference between `pk` and `id` is about more than just syntax, it's about building a robust application. While `id` is a specific field name, `pk` is a dynamic alias that always points to the source of truth for an object's identity.



**The Golden Rule**: Use `pk` when you mean "Identity", and use `id` only when you are explicitly referring to an integer column named `id`.
