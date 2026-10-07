---
title: "🌅 Morning Tech Lesson #1 — Big-O Notation: Thinking About Code as Data Grows"
order: 1
volume: 1
category: morning
source: Evening Tech Discovery
---

# 🌅 Morning Tech Lesson #1 — Big-O Notation: Thinking About Code as Data Grows

**Today’s goal:** Understand why two pieces of code that both “work” can behave completely differently when your application grows.

Estimated time: **10–15 minutes**

## 🧠 The core idea

Imagine your application has **10 users**.

You write some code to find a user by email. It loops through every user until it finds the correct one.

With 10 users? No problem.

Now imagine:

**10 users → 10 checks**

**1,000 users → 1,000 checks**

**10,000,000 users → potentially 10,000,000 checks**

Your algorithm became slower because the amount of work grows with the amount of data.

**Big-O notation describes how the amount of work grows as the input grows.**

We usually call the input size **n**.

So if you have 1,000 users:

`n = 1,000`

Big-O isn't primarily asking:

> “Does this take 3 milliseconds?”

Instead, it asks:

> “What happens when n becomes much larger?”

---

## The Big-O values you should recognize

You don't need advanced mathematics to make Big-O useful in everyday development.

| Complexity | Name | Rough intuition |
| --- | --- | --- |
| O(1) | Constant | Same amount of work regardless of data size |
| O(log n) | Logarithmic | Eliminate large portions of the search each step |
| O(n) | Linear | Potentially inspect every item |
| O(n log n) | Linearithmic | Common for efficient sorting |
| O(n²) | Quadratic | Often comparing everything with everything |
| O(2ⁿ) | Exponential | Becomes enormous very quickly |

For most application development, recognizing **O(1), O(log n), O(n), and O(n²)** already gives you a useful mental model.

---

## 🔎 Example 1 — O(n)

Suppose PHP gives you users like this:

```php
$users = [
    ['id' => 1, 'email' => 'alice@example.com'],
    ['id' => 2, 'email' => 'bob@example.com'],
    ['id' => 3, 'email' => 'charlie@example.com'],
];

function findUserByEmail(array $users, string $email): ?array
{
    foreach ($users as $user) {
        if ($user['email'] === $email) {
            return $user;
        }
    }

    return null;
}
```

Worst case, PHP examines every user.

If there are **n users**, the work grows approximately with n.

So we describe this lookup as:

**O(n)**

Doubling the amount of data can roughly double the amount of work.

---

## ⚡ Example 2 — O(1)

Now imagine restructuring the data:

```php
$usersByEmail = [
    'alice@example.com' => ['id' => 1],
    'bob@example.com' => ['id' => 2],
    'charlie@example.com' => ['id' => 3],
];

$user = $usersByEmail['bob@example.com'] ?? null;
```

PHP arrays use hash-table-like lookup internally.

Instead of scanning every user, we can directly look up the key.

Conceptually, this is **average-case O(1)**.

Whether you have:

```
10 users
1,000 users
1,000,000 users
```

the lookup doesn't need to scan them all.

That's an important lesson:

> **Choosing the right data structure can change the complexity of an operation.**

---

## 🗄️ You already encounter this with databases

This idea connects directly to something you'll see constantly in backend development: **database indexes**.

Imagine:

```sql
SELECT *
FROM users
WHERE email = 'alice@example.com';
```

If `email` isn't indexed, the database may need to examine a large portion—or potentially all—of the table.

Conceptually, think:

**scan → O(n)**

Now add an appropriate index:

```sql
CREATE INDEX idx_users_email
ON users(email);
```

A typical B-tree index lets the database navigate toward the matching value rather than scanning every row.

A useful simplified mental model is:

**indexed search → O(log n)**

Real database performance is more complicated than Big-O because disk access, memory, caching, query planning, index structure, selectivity, and other factors matter.

But the connection is important:

**Data structures → algorithms → database indexes → scalability**

These aren't separate topics.

---

## 🌳 Why O(log n) is surprisingly powerful

Suppose you have **1,000,000 sorted values**.

A linear search might inspect up to:

**1,000,000 values**

Binary search repeatedly cuts the remaining search space in half:

```
1,000,000
500,000
250,000
125,000
62,500
...
```

After only about **20 steps**, you're down to one candidate.

That's roughly:

**O(log n)**

This is why tree structures, indexes, and binary-search-style algorithms are so useful.

---

## 🚨 The complexity that should catch your attention

Consider this TypeScript:

```
TypeScriptfor (const user of users) {
    for (const order of orders) {
        if (order.userId === user.id) {
            // process order
        }
    }
}
```

Suppose you have approximately:

```
10,000 users
10,000 orders
```

Potential comparisons:

**10,000 × 10,000 = 100,000,000**

This pattern is approximately **O(n²)** when both collections scale similarly.

Instead, build a lookup:

```typescript
const ordersByUser = new Map<number, Order[]>();

for (const order of orders) {
    const existing = ordersByUser.get(order.userId) ?? [];
    existing.push(order);
    ordersByUser.set(order.userId, existing);
}

for (const user of users) {
    const orders = ordersByUser.get(user.id) ?? [];
    // process orders
}
```

Building the map is roughly **O(n)**.

Reading each user's bucket is average-case **O(1)** per lookup.

You've traded some **memory** for significantly less repeated searching.

This pattern appears everywhere:

**indexing, caching, hash maps, database joins, lookup tables, grouping data.**

---

## 💡 Why this matters in real software

Big-O helps you recognize code that works perfectly during development but becomes problematic in production.

You might test with:

```
100 records
```

and everything feels instant.

Production might contain:

```
5,000,000 records
```

Then a seemingly innocent nested loop, unindexed query, repeated API call, or inefficient lookup becomes expensive.

Big-O therefore isn't just a coding-interview topic.

It develops the habit of asking:

**“How does this behave when the system gets 10× or 1,000× larger?”**

That's fundamentally a **system-design question**.

---

## ⚠️ Common misconception

**“O(1) is always faster than O(n).”**

Not necessarily.

An O(1) operation could have a large constant cost, while an O(n) loop over five items could be extremely cheap.

For small inputs:

```
O(n) implementation → 1 μs
O(1) implementation → 10 μs
```

The O(n) implementation could still win.

Big-O describes **growth**, not exact runtime.

That's why good engineers consider both:

**algorithmic complexity + actual measurements/profiling**

Don't rewrite simple code merely because another theoretical solution has better Big-O.

---

## 🧩 Quick quiz

Try answering before revealing the mental answer.

**1. What's the complexity of this search?**

```
PHPforeach ($products as $product) {
    if ($product->id === $id) {
        return $product;
    }
}
```

Answer: **O(n)** worst case.

**2. What's suspicious here?**

```
TypeScriptfor (const customer of customers) {
    for (const invoice of invoices) {
        // compare customer and invoice
    }
}
```

Answer: potentially **O(n²)** if both collections grow together.

**3. Why can a hash map improve a lookup?**

Answer: instead of repeatedly scanning a collection in **O(n)**, a hash map can usually provide **average O(1)** key lookup.

---

## 🛠️ Today's 5-minute challenge

Imagine your API receives:

```typescript
const requestedIds = [10, 25, 100, 450];

const products = [
    // imagine 100,000 products here
];
```

You need to return products whose IDs appear in `requestedIds`.

The naïve solution might do something like:

```
TypeScriptproducts.filter(product =>
    requestedIds.includes(product.id)
);
```

Think about two questions:

**What is the complexity as both arrays become large?**

And:

**Which JavaScript data structure could make checking whether an ID exists much faster?**

Hint: you've probably used its cousin, `Map`.

Tomorrow's lessons can build on ideas like this when we reach **hash tables, database indexes, caching, and system design**.

For deeper reading, the [MDN guide to JavaScript keyed collections](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Keyed_collections?utm_source=chatgpt.com) is useful for understanding `Map`, `Set`, `WeakMap`, and `WeakSet`.

**One idea worth remembering from today:**

> Don't only ask, **“Does my code work?”** Ask, **“How does the amount of work change when the data grows?”**
