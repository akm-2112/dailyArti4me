---
title: "🌅 Morning Tech Lesson #3 — Database Indexes: Why the Right Index Can Turn a Slow Query Into a Fast One"
order: 5
volume: 1
category: morning
source: Evening Tech Discovery
---

# 🌅 Morning Tech Lesson #3 — Database Indexes: Why the Right Index Can Turn a Slow Query Into a Fast One

Your last two lessons built a useful foundation:

**Big-O → how work grows**

**Hash tables → how data structures avoid scanning everything**

Today we'll connect those ideas to something you use constantly in backend development:

> **Database indexes**

Estimated time: **10–15 minutes**

---

## 🧠 Start with a real backend problem

Imagine your `orders` table has **10 million rows**.

```sql
SELECT *
FROM orders
WHERE customer_id = 8472;
```

Without an appropriate index, the database may have to examine a huge number of rows:

```
row 1
row 2
row 3
row 4
...
row 10,000,000
```

This is a **sequential/table scan**.

Conceptually, it resembles the array search we learned earlier:

```
TypeScriptusers.find(user => user.id === id);
```

Worst-case thinking:

```
O(n)
```

Now imagine the database has:

```sql
CREATE INDEX idx_orders_customer_id
ON orders(customer_id);
```

Instead of searching millions of rows, the database can navigate an additional data structure that helps it locate matching rows.

That's the fundamental idea:

> **An index is an extra data structure maintained by the database to make certain lookups faster.**

---

# 📖 Think about a physical book

Suppose I give you a 1,000-page programming book and ask:

> Find every page discussing dependency injection.

Without an index:

```
page 1
page 2
page 3
...
page 1000
```

With the index at the back:

```
Dependency Injection → 142, 219, 455
```

You first search a much smaller organized structure, then jump to the relevant pages.

A database index works on roughly the same principle.

```
                INDEX
                  │
        customer_id = 8472
                  │
                  ▼
             row locations
                  │
          ┌───────┼───────┐
          ▼       ▼       ▼
        Order   Order    Order
```

---

# 🌳 How does the index itself stay searchable?

A common database index structure is a **B-tree/B+ tree**.

Don't worry about the exact implementation today.

Understand the shape.

Imagine:

```
                     [500]
                    /     \
                   /       \
              [100,300]   [700,900]
              /   |   \    /   |   \
```

Suppose you're searching for:

```
customer_id = 750
```

You don't examine every value.

You navigate through the tree:

```
750 > 500
      ↓
go right

750 between 700 and 900
      ↓
follow appropriate branch

find 750
```

Each decision eliminates large portions of the search space.

This is conceptually similar to the **O(log n)** behavior we discussed in Lesson #1.

---

# 🤯 Why `O(log n)` matters

Imagine:

```
1,000,000 rows
```

A linear scan could inspect roughly:

```
1,000,000
```

items.

A balanced tree search might require only a small number of levels.

As the table becomes:

```
10 million
100 million
1 billion
```

the tree grows relatively slowly.

That's one reason indexes are so powerful.

But there's something even more important:

**Databases care heavily about disk/page access.**

Reading one useful page from memory or storage can be far cheaper than reading thousands of irrelevant pages.

So real database performance is more complicated than simply saying:

```
index = O(log n)
```

The mental model is still extremely useful, though.

---

# 🐘 Practical PHP example

Imagine a Laravel-style application:

```php
$orders = Order::where(
    'customer_id',
    $customerId
)->get();
```

Developers sometimes look at this and think:

> “That's only one query, so it should be fast.”

But your PHP code tells you almost nothing about the query's actual cost.

The important question is:

```
Does orders.customer_id have an appropriate index?
```

These two identical PHP statements:

```
PHPOrder::where('customer_id', $id)->get();
```

could behave dramatically differently depending on:

```
table size
indexes
data distribution
selected columns
query plan
database cache
```

This is why understanding databases underneath your ORM matters.

---

# 🔬 Meet `EXPLAIN`

When investigating database performance, don't guess.

Ask the database.

For example, in MySQL:

```
SQLEXPLAIN
SELECT *
FROM orders
WHERE customer_id = 8472;
```

The query planner can show whether it's using an index and how it intends to access the data.

This is an important engineering habit:

> **Measure and inspect before optimizing.**

You'll eventually want to become comfortable reading query plans.

That skill is far more valuable than blindly adding indexes.

---

# ⚠️ Why not index every column?

Because indexes aren't free.

Suppose you create:

```sql
CREATE INDEX idx_orders_customer
ON orders(customer_id);

CREATE INDEX idx_orders_status
ON orders(status);

CREATE INDEX idx_orders_created
ON orders(created_at);
```

Every time you insert an order:

```
SQLINSERT INTO orders (...)
VALUES (...);
```

the database doesn't only update the table.

It may also need to update:

```
table
+
customer_id index
+
status index
+
created_at index
```

Indexes therefore introduce trade-offs.

### Benefits

```
reads/searches faster
sorting may become faster
joins may become faster
```

### Costs

```
more storage
slower writes
more memory/cache pressure
maintenance overhead
```

This connects directly to our previous lesson:

> **Data structures often trade memory/storage and maintenance work for faster lookup.**

---

# 🧩 Composite indexes

Now things get more interesting.

Imagine a common query:

```sql
SELECT *
FROM orders
WHERE customer_id = 8472
AND status = 'pending';
```

You might create:

```sql
CREATE INDEX idx_orders_customer_status
ON orders(customer_id, status);
```

This is a **composite index**.

Conceptually, think of it as ordered primarily by:

```
customer_id
```

then within each customer:

```
status
```

Something like:

```
customer 100
    cancelled
    pending
    shipped

customer 101
    cancelled
    pending
    shipped

customer 102
    ...
```

That means **column order matters**.

An index:

```sql
(customer_id, status)
```

isn't automatically equivalent to:

```sql
(status, customer_id)
```

The best ordering depends on your query patterns and database optimizer.

---

# 🧠 The leftmost-prefix mental model

For a simplified B-tree mental model, suppose you have:

```
SQLINDEX(customer_id, status, created_at)
```

It naturally supports queries beginning with:

```
customer_id
```

such as:

```
SQLWHERE customer_id = ?
```

and:

```
SQLWHERE customer_id = ?
AND status = ?
```

and potentially:

```
SQLWHERE customer_id = ?
AND status = ?
AND created_at > ?
```

But querying only:

```
SQLWHERE status = ?
```

may not be able to use that composite index efficiently in the same way.

Modern database optimizers have additional strategies and exceptions, so don't treat this as an absolute law.

Treat it as a **design mental model**.

---

# 🛒 Real e-commerce example

Imagine your application frequently runs:

```sql
SELECT id, total, created_at
FROM orders
WHERE user_id = ?
AND status = 'paid'
ORDER BY created_at DESC
LIMIT 20;
```

Think about what you're asking the database to do:

```
find one user's orders
        ↓
keep only paid orders
        ↓
order newest first
        ↓
return 20
```

A carefully designed composite index might be:

```sql
(user_id, status, created_at)
```

Conceptually:

```
user
 ↓
status
 ↓
created_at
```

The index structure now closely matches the query's access pattern.

This leads to an extremely important database-design principle:

> **Design indexes around real query patterns, not individual columns in isolation.**

---

# 🚨 Common misconception

### “If a column has an index, queries using that column will always be fast.”

No.

Consider:

```sql
SELECT *
FROM users
WHERE active = true;
```

Suppose:

```
99% of users are active
```

An index on `active` may not help much because the query still needs almost the entire table.

The optimizer might decide:

```
Why navigate the index and then fetch nearly every row?

I'll just scan the table.
```

This introduces another important concept:

## Selectivity

A column like:

```
email
```

might have millions of distinct values.

A column like:

```
is_active
```

might contain only:

```
true
false
```

Indexes tend to be especially useful when they significantly narrow the search.

But again, actual usefulness depends on the query, database, table layout, statistics and workload.

---

# 🔗 Connect our three lessons

Now your mental map should look like this:

```
LESSON 1
Big-O
│
└── How does work grow?


LESSON 2
Hash tables
│
└── Build structures for fast lookup


LESSON 3
Database indexes
│
└── Build structures that help databases
    locate rows efficiently
```

Notice the recurring idea:

```
Instead of:

SEARCH EVERYTHING

build:

STRUCTURE
   ↓
LOCATE
   ↓
FETCH
```

You'll encounter this pattern repeatedly.

Soon it connects naturally to:

```
Redis
caching
search engines
database joins
distributed systems
```

---

# 🧩 Quick quiz

### 1. Why might this become slow?

```sql
SELECT *
FROM orders
WHERE customer_id = 123;
```

The table contains **50 million rows**, and `customer_id` isn't indexed.

**Answer:** The database may need to scan a large portion of the table to find matching rows.

---

### 2. Why shouldn't you simply index every column?

Because indexes consume storage and must be maintained when data changes.

You're trading:

```
storage + write cost
```

for potentially:

```
faster reads
```

---

### 3. You frequently query:

```
SQLWHERE user_id = ?
AND status = ?
ORDER BY created_at DESC
```

Which index would you investigate first?

```
A) (created_at)
B) (status)
C) (user_id, status, created_at)
```

**C** is a strong candidate based on the query shape, though you'd verify it using the actual query plan and workload.

---

# 🛠️ Today's practical challenge

Imagine this table:

```sql
CREATE TABLE messages (
    id BIGINT PRIMARY KEY,
    conversation_id BIGINT,
    sender_id BIGINT,
    content TEXT,
    created_at TIMESTAMP
);
```

Your chat API constantly executes:

```sql
SELECT *
FROM messages
WHERE conversation_id = 9182
ORDER BY created_at DESC
LIMIT 50;
```

The table eventually reaches:

```
500,000,000 messages
```

Think through these questions:

**1. What happens if `conversation_id` has no index?**

**2. What composite index would you consider for this query?**

**3. Why might that index help with both filtering and ordering?**

A strong candidate to investigate is:

```sql
CREATE INDEX idx_messages_conversation_created
ON messages(conversation_id, created_at);
```

But don't stop at creating it.

Run:

```
SQLEXPLAIN
SELECT *
FROM messages
WHERE conversation_id = 9182
ORDER BY created_at DESC
LIMIT 50;
```

and inspect what your database actually does.

For deeper reading, the official [MySQL documentation on how indexes work](https://dev.mysql.com/doc/refman/8.4/en/mysql-indexes.html?utm_source=chatgpt.com) and [PostgreSQL index documentation](https://www.postgresql.org/docs/current/indexes.html?utm_source=chatgpt.com) are excellent references.

### 🧠 Keep one idea today

> **Indexes aren't magic speed switches. They're additional data structures that trade storage and write work for faster access to data.**

And the bigger engineering lesson is even more useful:

**Don't ask only “Does this query work?”**

Ask:

**“How will the database find these rows when this table has 100 million records?”**
