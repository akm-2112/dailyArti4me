---
title: "🌅 Morning Tech Lesson #12 — Database Indexes: Why  WHERE  Can Be Either 2 ms or 2 Seconds"
order: 23
volume: 4
category: morning
source: Evening Tech Discovery
---

# 🌅 Morning Tech Lesson #12 — Database Indexes: Why `WHERE` Can Be Either 2 ms or 2 Seconds

We’ve recently covered **HTTP/authentication, the event loop, caching, queues, transactions, and concurrency**. Today we’ll move into **database internals + algorithms**, while connecting back to complexity.

The central question:

> **When you run `SELECT ... WHERE email = ?`, how does the database find the row without checking every row?**

Estimated time: **10–15 minutes**.

## Start with a simple Laravel query

Suppose your `users` table has 10 million rows:

```php
$user = User::where(
    'email',
    'zoe@example.com'
)->first();
```

At the SQL level:

```sql
SELECT *
FROM users
WHERE email = 'zoe@example.com'
LIMIT 1;
```

Without a useful index, the database may need to do something conceptually like:

```
row 1     → not it
row 2     → not it
row 3     → not it
...
row 8,741,921 → FOUND
```

That's a **table scan**.

As the table grows:

```
1,000 rows       → probably fine
100,000 rows     → noticeable
10,000,000 rows  → potentially expensive
```

Algorithmically, you're approaching:

```
O(n)
```

More rows → more work.

An index changes how the database searches.

## 📖 Think of a textbook

Imagine a 1,000-page programming book.

You want:

```
"Binary Search Trees"
```

Without an index:

```
page 1
page 2
page 3
...
page 734
```

With the index at the back:

```
Binary Search Trees → page 734
```

You perform a small lookup and jump near the desired information.

A database index serves a similar purpose.

Instead of arranging everything around the physical table scan:

```
users table

id | name | email
-------------------------
1  | A    | a@example.com
2  | B    | b@example.com
3  | C    | c@example.com
...
```

the database maintains an additional structure roughly resembling:

```
email index

a@example.com → row location
b@example.com → row location
c@example.com → row location
...
```

But real relational database indexes generally aren't implemented as a giant hash map.

A very common structure is a **B-tree/B+ tree family**.

---

# 🌳 Why a tree?

Imagine the emails are organized like this:

```
                  m@example.com
                /               \
               /                 \
      f@example.com           t@example.com
       /       \               /       \
      ...      ...            ...      ...
```

To find:

```
zoe@example.com
```

you don't inspect everything.

You repeatedly eliminate large portions of the search space:

```
zoe > m
     ↓
go right

zoe > t
     ↓
go right

...
```

This should remind you of **binary search**.

Binary search over sorted data has:

```
O(log n)
```

behavior.

For intuition:

```
n = 1,000,000
```

A linear search could theoretically inspect around:

```
1,000,000 items
```

Binary-style searching needs roughly:

```
log₂(1,000,000)
≈ 20 decisions
```

Database B-trees aren't literally ordinary binary trees—they usually have **many children per node**.

That's intentional.

---

# Why B-trees have many children

Databases care heavily about:

```
disk / SSD access
memory pages
```

Suppose a tree node can contain hundreds of keys:

```
                    [100 | 200 | 300 | ...]
                 /      |      |       \
                ▼       ▼      ▼        ▼
```

Then one node lets the database eliminate enormous ranges.

A huge index might have a height of only:

```
3
4
5
```

levels.

Conceptually:

```
root page
   ↓
internal page
   ↓
leaf page
   ↓
matching row
```

This is important because storage access is much more expensive than simple CPU comparisons.

The structure is designed partly to minimize the number of storage pages the database needs to touch.

So when someone says:

> “Indexes make queries faster.”

The deeper explanation is:

> **Indexes create an auxiliary data structure that lets the database locate relevant rows while examining far less data.**

---

# 🛠️ Creating an index

Suppose you frequently query:

```sql
SELECT *
FROM users
WHERE email = ?;
```

You might create:

```sql
CREATE INDEX idx_users_email
ON users(email);
```

In Laravel:

```
PHPSchema::table('users', function (Blueprint $table) {
    $table->index('email');
});
```

For something like user email, you may actually require uniqueness:

```php
$table->unique('email');
```

Now the database can potentially use the index when executing:

```
SQLWHERE email = ?
```

But here's where developers often develop the wrong mental model.

---

# ⚠️ An index is not free performance

Imagine inserting:

```
SQLINSERT INTO users (...)
```

Without indexes:

```
write row
```

With several indexes:

```
write row

update email index

update username index

update created_at index

update country index

...
```

Every index introduces costs:

```
storage
memory/cache pressure
INSERT work
UPDATE work
DELETE work
maintenance
```

So:

```
more indexes
≠
automatically better database
```

Indexes trade:

```
extra storage + write cost
```

for:

```
faster reads
```

This is another recurring engineering trade-off.

---

# Composite indexes are where things get interesting

Suppose your e-commerce application frequently executes:

```sql
SELECT *
FROM orders
WHERE user_id = 42
AND status = 'paid';
```

You could create:

```sql
CREATE INDEX idx_orders_user_status
ON orders(user_id, status);
```

This is a **composite index**.

Conceptually, it is ordered something like:

```
user_id
   ↓
status

42 | cancelled
42 | paid
42 | paid
42 | pending
43 | cancelled
43 | paid
...
```

The order of columns matters.

An index on:

```
(user_id, status)
```

is not generally equivalent to:

```
(status, user_id)
```

Why?

Because the data is ordered first according to the first column, then according to subsequent columns within that ordering.

Think of a phone book:

```
(last_name, first_name)
```

Finding:

```
Smith, John
```

is easy.

Finding everyone whose:

```
first_name = John
```

is much less naturally supported by that ordering.

This leads to the **leftmost-prefix idea** in B-tree indexes.

An index:

```
(user_id, status, created_at)
```

can often efficiently support query prefixes such as:

```
user_id
```

and:

```
user_id + status
```

and:

```
user_id + status + created_at
```

But a query using only:

```
created_at
```

usually cannot exploit that index in the same way.

Exact optimizer behavior varies by database, so treat this as a mental model rather than an absolute law.

---

# 🛒 A realistic e-commerce example

Imagine:

```
orders

id
user_id
status
total
created_at
```

Your dashboard runs:

```sql
SELECT *
FROM orders
WHERE user_id = 42
ORDER BY created_at DESC
LIMIT 20;
```

You create:

```sql
CREATE INDEX idx_orders_user_created
ON orders(user_id, created_at);
```

Now the index structure is naturally useful for:

```
find user 42
      ↓
already ordered around created_at
      ↓
take recent 20
```

Without the right index, the database may need to:

```
find many matching rows
      ↓
sort them
      ↓
take 20
```

With an appropriate index, it may be able to avoid much of that work.

This illustrates an important lesson:

> **Good indexes are designed around query patterns, not merely individual columns.**

Don't look at your schema and ask:

```
Which columns should have indexes?
```

Start with:

```
Which important queries are slow?

How do they filter?

How do they join?

How do they order?

How selective are those predicates?
```

Then design indexes around the access patterns.

---

# Selectivity matters

Suppose 100 million users have:

```
is_active
```

and:

```
95 million = true
5 million  = false
```

You create:

```sql
CREATE INDEX idx_users_active
ON users(is_active);
```

Then query:

```sql
SELECT *
FROM users
WHERE is_active = true;
```

The index points to:

```
95,000,000 rows
```

That's not especially selective.

The database optimizer may decide:

```
Scanning the table is cheaper.
```

Compare with:

```
SQLWHERE email = 'unique@example.com'
```

which probably selects:

```
1 row
```

Very selective.

A useful intuition:

```
email
   ↓
high selectivity


boolean status
   ↓
often low selectivity
```

Indexes are especially powerful when they let the database discard a large fraction of rows quickly.

---

# 🧠 The query optimizer decides

Another misconception:

```
"I created an index,
therefore PostgreSQL/MySQL will use it."
```

Not necessarily.

The database has a **query optimizer**.

It estimates the cost of possible execution plans:

```
Plan A
table scan

Plan B
index lookup

Plan C
different index

Plan D
different join order
```

and chooses what it estimates is cheapest.

You can inspect this.

In MySQL:

```
SQLEXPLAIN
SELECT *
FROM orders
WHERE user_id = 42;
```

In PostgreSQL:

```
SQLEXPLAIN ANALYZE
SELECT *
FROM orders
WHERE user_id = 42;
```

The output can tell you about things such as:

```
scan type
chosen index
estimated rows
actual rows
join strategy
sorting
execution time
```

Learning to read query plans is one of the skills that separates:

```
"I use SQL"
```

from:

```
"I can diagnose database performance."
```

---

# 🔥 Indexes also affect joins

Suppose:

```
users
  |
  | id
  ▼
orders.user_id
```

and:

```sql
SELECT users.name, orders.total
FROM users
JOIN orders
    ON orders.user_id = users.id
WHERE users.id = 42;
```

If:

```
orders.user_id
```

has no useful index, finding this user's orders may require examining a large amount of the `orders` table.

With:

```sql
CREATE INDEX idx_orders_user
ON orders(user_id);
```

the database can locate matching orders much more efficiently.

This is why foreign-key columns frequently benefit from indexes.

A foreign-key **constraint** and an **index** solve different problems:

```
Foreign key
     ↓
referential integrity


Index
     ↓
efficient access
```

Some database systems/configurations may create or require supporting indexes automatically in certain situations; don't assume all do exactly the same thing.

---

# 🔗 Connect this to complexity

Remember:

```
Array search
    ↓
O(n)


Binary search on sorted array
    ↓
O(log n)


Hash lookup
    ↓
approximately O(1) average
```

Database indexes are these data-structure concepts applied to persistent storage.

But databases have additional realities:

```
disk pages
memory caches
concurrent writes
transactions
locking
statistics
query optimization
```

That's why simply knowing Big-O isn't enough.

Big-O gives you:

```
growth behavior
```

Database engineering adds:

```
real hardware cost
```

For example:

```
O(log n)
```

with four SSD page reads might be slower than scanning a tiny table already sitting entirely in RAM.

So the optimizer uses a **cost model**, not just asymptotic complexity.

---

# 🔄 Connect this to caching

Suppose:

```
GET /products/123
```

takes:

```
600 ms
```

A developer might immediately say:

```
Add Redis.
```

But imagine the real problem is:

```sql
SELECT *
FROM products
WHERE sku = ?;
```

and:

```
sku has no index
```

After adding the correct index:

```
600 ms
  ↓
8 ms
```

Now perhaps you don't need another caching layer at all.

That's why performance work should often proceed:

```
measure
   ↓
find bottleneck
   ↓
fix root cause
   ↓
cache if appropriate
```

rather than:

```
slow
 ↓
Redis!
```

Caching can hide inefficient database access while adding invalidation and consistency complexity.

Indexes can sometimes fix the underlying access problem.

---

# 🤖 Why indexes matter for AI systems too

Imagine your AI agent stores:

```
conversations
messages
tool_calls
embeddings
agent_runs
```

A table:

```
messages

id
conversation_id
role
content
created_at
```

and every agent turn asks:

```sql
SELECT *
FROM messages
WHERE conversation_id = ?
ORDER BY created_at DESC
LIMIT 20;
```

A natural index might be:

```sql
CREATE INDEX idx_messages_conversation_created
ON messages(conversation_id, created_at);
```

Without thoughtful indexing, an AI product may feel slow even though:

```
LLM inference = 800 ms
```

because your application quietly spends:

```
database query = 1,400 ms
```

The glamorous AI layer isn't always the bottleneck.

---

# ⚠️ Common misconception

> **“If queries are slow, add indexes to every column.”**

That can make things worse.

Every index has to be maintained.

Imagine:

```
users table

10 indexes
```

Every insert may require updates to multiple index structures.

A healthier rule is:

> **Index important access patterns, then verify with real query plans and measurements.**

Don't guess.

Use:

```
EXPLAIN
```

and production observability.

---

## 🧩 Quick quiz

**1. Why can an index turn an O(n)-style lookup into something closer to O(log n)?**

Because a tree-based index keeps keys ordered and lets the database eliminate large portions of the search space rather than examining every row.

**2. Why shouldn't you index every database column?**

Indexes consume storage and must be maintained during inserts, updates and deletes; they also add cache/memory pressure.

**3. You have this index:**

```
SQLINDEX(user_id, status, created_at)
```

Which query is naturally better aligned with its ordering?

```
SQLWHERE user_id = 42
AND status = 'paid'
```

rather than:

```
SQLWHERE created_at = '2026-09-09'
```

because the former uses the leftmost portion of the composite index.

---

# 🛠️ Today's practical challenge

You have:

```
orders

id
user_id
status
total
created_at
```

with **50 million rows**.

Your API has three important queries:

```
SQL-- A
SELECT *
FROM orders
WHERE user_id = ?
ORDER BY created_at DESC
LIMIT 20;
```

```
SQL-- B
SELECT *
FROM orders
WHERE user_id = ?
AND status = 'pending';
```

```
SQL-- C
SELECT *
FROM orders
WHERE status = 'paid'
AND created_at >= ?;
```

Design the smallest useful set of indexes.

Don't simply create:

```
index every column
```

Think about whether:

```sql
(user_id, created_at)
```

helps A,

whether:

```sql
(user_id, status)
```

helps B,

and whether C represents a sufficiently important access pattern to deserve something such as:

```sql
(status, created_at)
```

Then run:

```
SQLEXPLAIN
```

against each query on a real MySQL/PostgreSQL table and see whether the optimizer agrees with your prediction.

For deeper reading, [PostgreSQL's official index documentation](https://www.postgresql.org/docs/current/indexes.html?utm_source=chatgpt.com) gives an excellent database-independent foundation despite being PostgreSQL-specific. If you're working primarily with MySQL, [MySQL's official guide to how it uses indexes](https://dev.mysql.com/doc/refman/8.4/en/mysql-indexes.html?utm_source=chatgpt.com) is worth bookmarking.

### 🧠 Keep one mental model today

> **An index is a data structure you pay to maintain so future queries can avoid searching unnecessary data.**

Whenever a database query becomes slow, don't begin with:

```
"Which technology should I add?"
```

Begin with:

```
What rows does this query need?

How many rows is the database examining?

What data structure could let it
discard irrelevant rows earlier?

What execution plan did the
database actually choose?
```

That mindset connects **algorithms, data structures, databases, system performance, and observability**—which is exactly the kind of connection that makes you a stronger software engineer.
