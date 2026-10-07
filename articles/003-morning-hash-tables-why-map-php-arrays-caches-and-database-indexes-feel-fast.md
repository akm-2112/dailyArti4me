---
title: "🌅 Morning Tech Lesson #2 — Hash Tables: Why  Map , PHP Arrays, Caches, and Database Indexes Feel Fast"
order: 3
volume: 1
category: morning
source: Evening Tech Discovery
---

# 🌅 Morning Tech Lesson #2 — Hash Tables: Why `Map`, PHP Arrays, Caches, and Database Indexes Feel Fast

Yesterday you learned **Big-O notation** and saw that repeatedly scanning an array gives you **O(n)** lookup, while some data structures can give you roughly **O(1)** average lookup.

Today we're going one level deeper:

> **How can a computer find something without searching through everything?**

The answer is one of the most important data structures in software engineering: the **hash table**.

Estimated time: **10–15 minutes**

---

## 🧠 Start with a simple problem

Imagine you have 1 million users:

```typescript
const users = [
    { id: 1, name: "Alice" },
    { id: 2, name: "Bob" },
    // ...
    { id: 1000000, name: "Zoe" }
];
```

You want user `827391`.

One solution:

```typescript
function findUser(id: number) {
    for (const user of users) {
        if (user.id === id) {
            return user;
        }
    }
}
```

Worst case:

```
1 → 2 → 3 → 4 → ... → 827391
```

That's **O(n)**.

Yesterday we saw that a `Map` changes this:

```typescript
const users = new Map<number, User>();

users.set(827391, {
    id: 827391,
    name: "Zoe"
});

const user = users.get(827391);
```

Average lookup:

**O(1)**

But how?

---

# 🔑 Hashing

Imagine a simplified table with 10 storage locations:

```
0
1
2
3
4
5
6
7
8
9
```

We want to store:

```
user ID = 827391
```

A **hash function** takes the key and converts it into something that determines where the value should live.

For teaching purposes, imagine:

```
hash(827391) → 7
```

So:

```
0
1
2
3
4
5
6
7 → { id: 827391, name: "Zoe" }
8
9
```

Later you ask:

```
TypeScriptusers.get(827391)
```

The system calculates:

```
hash(827391) → 7
```

and goes directly to location 7.

It doesn't need:

```
check item 1
check item 2
check item 3
...
```

That's the key idea.

### Array search

```
key
 │
 ▼
search → search → search → search → value
```

### Hash table

```
key
 │
 ▼
hash(key)
 │
 ▼
location
 │
 ▼
value
```

That's why lookup can be **average O(1)**.

---

# 💥 But there's a problem: collisions

Suppose:

```
hash("alice@example.com") → 7

hash("bob@example.com") → 7
```

Two different keys want the same location.

That's called a **hash collision**.

Real hash tables have strategies for handling this.

One conceptual approach is putting multiple entries into the same bucket:

```
Bucket 7
   │
   ├── alice@example.com → Alice
   │
   └── bob@example.com → Bob
```

Another common technique is **open addressing**, where the implementation searches for another available slot.

You don't usually implement this yourself when using PHP or TypeScript.

But understanding collisions explains something important:

> Hash-table lookup is usually **average O(1)**, not magically guaranteed O(1) under every circumstance.

Poor collision behavior can make lookups more expensive.

---

# 🟦 JavaScript / TypeScript

You have two important hash-based structures available.

## `Map`

```typescript
const users = new Map<number, User>();

users.set(10, {
    id: 10,
    name: "Alice"
});

users.set(25, {
    id: 25,
    name: "Bob"
});

const user = users.get(25);
```

Think:

```
key → value
```

Use `Map` when you need to associate something with something else.

Examples:

```
userId → User
productId → Product
requestId → Request
sessionId → Session
```

---

# 🟩 `Set`

Remember yesterday's challenge?

You had:

```typescript
const requestedIds = [10, 25, 100, 450];
```

and:

```
TypeScriptproducts.filter(product =>
    requestedIds.includes(product.id)
);
```

`includes()` potentially scans the array.

If you have:

```
n products
m requested IDs
```

you can end up doing roughly:

**O(n × m)**

Instead:

```typescript
const requestedIds = new Set([
    10,
    25,
    100,
    450
]);

const result = products.filter(product =>
    requestedIds.has(product.id)
);
```

Now membership checking is average **O(1)**.

Building the `Set` costs approximately:

**O(m)**

Filtering products costs:

**O(n)**

Overall:

**O(n + m)**

instead of:

**O(n × m)**

This is exactly the connection to yesterday's Big-O lesson.

---

# 🐘 What about PHP?

Here's something interesting.

PHP's standard `array` isn't just a simple contiguous array like you might encounter in lower-level languages.

It behaves as an **ordered map implemented using a hash table**.

That allows:

```php
$usersById = [
    10 => ['name' => 'Alice'],
    25 => ['name' => 'Bob'],
];

$user = $usersById[25];
```

and associative keys:

```php
$usersByEmail = [
    'alice@example.com' => [
        'id' => 10,
        'name' => 'Alice',
    ],

    'bob@example.com' => [
        'id' => 25,
        'name' => 'Bob',
    ],
];

$user = $usersByEmail['bob@example.com'];
```

This convenience is one reason PHP arrays are extremely flexible.

But there's a trade-off.

Hash tables require extra metadata and memory.

So PHP arrays can consume substantially more memory than a compact low-level array representation.

This gives us another fundamental engineering principle:

> **Performance improvements usually involve trade-offs.**

Hash tables often trade:

**more memory**

for:

**faster lookup**

You'll see this idea again when we study caching.

---

# 🧠 Now connect this to backend engineering

Suppose your API receives 5,000 orders and 5,000 customers.

You write:

```
TypeScriptfor (const order of orders) {
    const customer = customers.find(
        customer => customer.id === order.customerId
    );

    processOrder(order, customer);
}
```

Look carefully.

`find()` scans customers.

You're effectively doing:

```
order 1 → search customers
order 2 → search customers
order 3 → search customers
...
```

Potentially around:

**O(n²)**

for similarly sized collections.

Instead:

```typescript
const customersById = new Map(
    customers.map(customer => [
        customer.id,
        customer
    ])
);

for (const order of orders) {
    const customer =
        customersById.get(order.customerId);

    processOrder(order, customer);
}
```

Now:

```
Build Map
   │
   ▼
O(n)

Process orders
   │
   ▼
O(n)
```

Approximately:

**O(n)** overall.

This tiny refactoring can become extremely important when processing large datasets.

---

# 🗄️ The database connection

There's a deeper idea here.

Suppose you run:

```sql
SELECT *
FROM users
WHERE email = 'alice@example.com';
```

Without an appropriate index, your database may need to inspect many rows.

Yesterday we simplified this as:

```
table scan
≈ O(n)
```

An index creates an additional data structure designed to make finding rows faster.

Databases commonly use structures such as **B-trees/B+ trees**, and some databases also support **hash indexes**.

So conceptually:

```
JavaScript Map
PHP associative array
database index
Redis key lookup
cache
```

aren't identical implementations, but they're related by a powerful idea:

> **Create a structure that helps you locate data instead of repeatedly searching all the data.**

---

# ⚡ This leads directly to caching

Imagine your PHP application repeatedly calculates:

```php
$result = expensiveReport($customerId);
```

Maybe generating the report requires:

```
8 SQL queries
+
aggregation
+
external API
```

Instead, you might store:

```
report:customer:827391
        │
        ▼
cached result
```

Then:

```php
$result = $cache->get(
    "report:customer:$customerId"
);
```

The key itself acts like a lookup identifier.

Conceptually:

```
customerId
   ↓
cache key
   ↓
hash-based lookup
   ↓
value
```

That's one reason key-value stores such as Redis can provide extremely fast access.

Later, when we study caching, you'll already understand the underlying idea.

---

# ⚠️ Common misconception

### “Hashing means encryption.”

No.

They're completely different concepts.

Encryption:

```
data
 ↓
encryption
 ↓
ciphertext
 ↓
decryption
 ↓
original data
```

Hashing:

```
data
 ↓
hash function
 ↓
hash
```

Hash functions are generally **one-way transformations**.

But there's another nuance: the hash functions used internally by hash tables aren't necessarily the same kind of cryptographic hash functions used for security.

For example:

```
hash-table hashing
```

focuses heavily on efficient distribution and lookup.

Cryptographic hashing focuses on security properties.

We'll revisit this when we study **password storage, authentication, and security**.

---

# 🧩 Quick quiz

### 1. What's inefficient here?

```
TypeScriptfor (const user of users) {
    const role = roles.find(
        role => role.id === user.roleId
    );
}
```

Think before continuing.

`roles.find()` performs repeated searches.

If both collections grow similarly, this can approach:

**O(n²)**.

A `Map<roleId, Role>` could reduce repeated lookup cost.

---

### 2. What's the conceptual difference between `Map` and `Set`?

`Map`:

```
key → value
```

Example:

```
userId → User
```

`Set`:

```
unique values
```

Example:

```
allowedUserIds
```

You usually use `Set` when the main question is:

> “Does this value exist?”

---

### 3. Why isn't hash-table lookup simply called guaranteed O(1)?

Because things such as **collisions, resizing, and implementation details** can affect individual operations.

Average lookup is typically treated as **O(1)**.

---

# 🛠️ Today's challenge

You're given:

```typescript
const blockedUserIds = [
    7,
    25,
    88,
    190,
    // imagine 100,000 IDs
];

const requests = [
    { userId: 25, path: "/checkout" },
    { userId: 42, path: "/products" },
    // imagine millions of requests
];
```

Someone writes:

```typescript
const allowedRequests = requests.filter(
    request =>
        !blockedUserIds.includes(request.userId)
);
```

Your challenge:

**1. Identify the scalability problem.**

**2. Rewrite it using the data structure you learned today.**

**3. Think about the complexity before and after.**

Bonus question:

If `blockedUserIds` changes frequently and your application runs across **20 backend servers**, keeping the blocked list only in a JavaScript `Set` introduces another architectural problem.

Where should the shared source of truth live?

Don't worry if you don't know yet—that question leads us toward **Redis, databases, distributed state, and caching**.

For deeper reading, see [MDN's Map documentation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map?utm_source=chatgpt.com), [MDN's Set documentation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set?utm_source=chatgpt.com), and the [PHP manual's explanation of arrays](https://www.php.net/manual/en/language.types.array.php?utm_source=chatgpt.com).

### 🧠 Keep one mental model today

Yesterday:

**Big-O asks how work grows.**

Today:

**Hash tables avoid repeated searching by turning a key into a location for fast lookup.**

That simple idea will come back repeatedly as we move through **algorithms → databases → caching → distributed systems → system design**.
