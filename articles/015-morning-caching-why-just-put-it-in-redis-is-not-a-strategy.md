---
title: "🌅 Morning Tech Lesson #8 — Caching: Why “Just Put It in Redis” Is Not a Strategy"
order: 15
volume: 3
category: morning
source: Evening Tech Discovery
---

# 🌅 Morning Tech Lesson #8 — Caching: Why “Just Put It in Redis” Is Not a Strategy

We’ve recently worked through **idempotency → queues → race conditions**. Today we’ll switch from correctness/concurrency to **performance and scalability**, while revisiting database indexes from another angle.

The central question is:

> **If 10,000 users repeatedly ask for the same expensive data, should your application recompute or reload it 10,000 times?**

Estimated time: **10–15 minutes**.

## Start with a real API

Imagine your PHP API exposes:

```
httpGET /api/products/123
```

The controller eventually performs:

```php
$product = Product::with([
    'category',
    'images',
    'reviews',
    'inventory'
])->findOrFail(123);

return $product;
```

Suppose the database work takes:

```
40 ms
```

That's perfectly acceptable for one request.

But now product `123` goes viral:

```
5,000 requests / second
```

Your database suddenly receives roughly:

```
5,000 similar queries / second
```

even though most users are asking for essentially the **same information**.

The first optimization question shouldn't necessarily be:

```
How can I make this SQL 3 ms faster?
```

Sometimes the better question is:

> **Why am I querying the database at all for every request?**

That's where caching enters.

## 🧠 What is a cache?

A cache stores a copy of data somewhere that's cheaper or faster to access.

Instead of:

```
Request
   ↓
Application
   ↓
Database
   ↓
expensive query
   ↓
Response
```

we introduce:

```
Request
   ↓
Application
   ↓
Cache
  /   \
HIT   MISS
 │      │
 │      ▼
 │   Database
 │      │
 │   save cache
 │      │
 └──────┴──→ Response
```

A **cache hit** means:

```
I already have the answer.
```

A **cache miss** means:

```
I don't have it.
Go to the original source.
```

For example:

```php
$product = Cache::remember(
    "product:{$id}",
    now()->addMinutes(5),
    function () use ($id) {
        return Product::with([
            'category',
            'images'
        ])->findOrFail($id);
    }
);
```

Conceptually:

```
product:123
      ↓
{
  id: 123,
  name: "...",
  price: 4999
}
```

If the cache is backed by Redis, subsequent requests can often retrieve that representation without repeating the database query.

## Why is cache access often faster?

A relational database may need to do work involving:

```
parse query
   ↓
plan query
   ↓
navigate indexes
   ↓
read pages
   ↓
join tables
   ↓
construct rows
   ↓
return results
```

Your cache may instead perform something conceptually closer to:

```
key
 ↓
lookup
 ↓
value
```

Remember your hash-table lesson?

```
key → value
```

That idea appears again.

Caching doesn't magically make algorithms disappear, but we're often replacing expensive computation or storage access with a cheaper lookup.

## 🔗 Cache versus database index

This distinction is important.

An **index** helps the database locate data efficiently:

```
Application
    ↓
Database
    ↓
INDEX
    ↓
Rows
```

A **cache** can avoid asking the database altogether:

```
Application
    ↓
CACHE
    ↓
answer
```

Suppose:

```sql
SELECT *
FROM products
WHERE id = 123;
```

already uses a primary-key index.

The query may be extremely efficient.

But:

```
efficient query × 100,000 requests
```

is still work.

Caching attacks a different dimension:

```
Index:
make each DB lookup cheaper

Cache:
reduce how many DB lookups happen
```

You'll frequently use both.

---

# The easiest caching pattern: cache-aside

The PHP example above uses a common strategy called **cache-aside**.

The application owns the logic:

```
1. Look in cache.

2. If found:
      return it.

3. If missing:
      query database.

4. Store result in cache.

5. Return it.
```

TypeScript might look like:

```
TypeScriptasync function getProduct(id: number) {
    const key = `product:${id}`;

    const cached = await redis.get(key);

    if (cached) {
        return JSON.parse(cached);
    }

    const product =
        await database.products.find(id);

    await redis.set(
        key,
        JSON.stringify(product),
        { EX: 300 }
    );

    return product;
}
```

The important idea is framework-independent.

The database remains the **source of truth**.

The cache contains a temporary copy.

---

# And now we meet the hard problem: stale data

Suppose we cache:

```
product:123

price = $100
```

Then an administrator changes the database:

```
price = $80
```

Database:

```
$80
```

Cache:

```
$100
```

Your users may continue seeing:

```
$100
```

You've gained performance by introducing **another copy of state**.

And whenever you have multiple copies of state, you must answer:

> **How do they stay consistent?**

This is why the famous joke exists:

> “There are only two hard things in computer science: cache invalidation and naming things.”

The joke survives because cache invalidation genuinely becomes subtle.

---

# Strategy 1: TTL

TTL means **Time To Live**.

For example:

```
TypeScriptawait redis.set(
    "product:123",
    JSON.stringify(product),
    { EX: 300 }
);
```

means roughly:

```
Keep this value for 300 seconds.
```

After five minutes:

```
cache entry expires
       ↓
next request misses
       ↓
database queried
       ↓
fresh value cached
```

TTL gives you a simple trade-off.

Long TTL:

```
fewer database queries
more stale-data risk
```

Short TTL:

```
more database queries
fresher data
```

There's no universally correct TTL.

A product description might tolerate:

```
10 minutes
```

A stock-market price probably cannot.

Inventory during checkout may require much stronger correctness than either.

Cache policy follows **business semantics**.

---

# Strategy 2: explicit invalidation

Suppose you update a product:

```php
$product->update([
    'price' => $newPrice
]);

Cache::forget(
    "product:{$product->id}"
);
```

Now:

```
Database updated
       ↓
cache deleted
       ↓
next read
       ↓
cache miss
       ↓
database
       ↓
fresh cache
```

This gives fresher behavior.

But what happens if:

```
Database update succeeds ✓

process crashes

Cache::forget() never executes ✗
```

You now have stale cache.

Recognize the pattern?

```
operation A succeeds
operation B fails
```

This is another distributed-systems consistency problem.

The database and Redis don't normally participate in one simple shared ACID transaction.

We'll eventually study patterns for coordinating these kinds of changes.

---

# 🔥 The cache stampede

Now imagine a very popular key:

```
homepage:recommendations
```

It receives:

```
20,000 requests/sec
```

The cached value expires at:

```
12:00:00
```

At exactly that moment:

```
Request 1 → MISS
Request 2 → MISS
Request 3 → MISS
...
Request 10,000 → MISS
```

All of them conclude:

```
Cache empty.
Query database.
```

Instead of protecting your database, the cache suddenly causes:

```
10,000 expensive queries
```

at almost the same time.

That's a **cache stampede** or **thundering herd**.

Notice today's spaced repetition:

> This is another concurrency problem.

Your previous race-condition lesson is already useful.

---

# One defense: single-flight / locking

Conceptually:

```
Cache miss
    │
    ▼
Can I acquire refresh lock?
    │
 ┌──┴───┐
YES     NO
 │       │
 ▼       ▼
query    wait/use stale value
DB
 │
 ▼
populate cache
 │
 ▼
release lock
```

Now perhaps only one worker performs the expensive refresh.

Instead of:

```
10,000 misses
      ↓
10,000 DB queries
```

you get closer to:

```
10,000 misses
      ↓
1 refresh
```

Exact implementations differ, but the engineering principle is:

> **Coordinate expensive cache regeneration under concurrency.**

---

# Another defense: expiration jitter

Suppose you've cached 100,000 products.

Your code gives all of them:

```
TTL = 3600 seconds
```

and they were populated during deployment at approximately the same time.

One hour later:

```
100,000 cache entries expire
simultaneously
```

Bad.

Instead:

```
TTL = 3600 + random(0..300)
```

Now expirations spread across several minutes.

This is **jitter**.

You saw exactly the same idea with queue retries:

```
Retry jitter
    ↓
don't retry everything simultaneously

Cache TTL jitter
    ↓
don't expire everything simultaneously
```

Same principle.

Different problem.

That's the kind of conceptual transfer we're aiming for.

---

# 🚨 What should you *not* blindly cache?

Imagine:

```
GET /product/123
```

Caching makes sense.

Now imagine:

```
GET /account/balance
```

or:

```
GET /inventory/123
```

during checkout.

Stale values might have real business consequences.

You should ask:

```
How stale can this data safely be?
```

For:

```
blog article
```

perhaps:

```
hours
```

For:

```
product description
```

perhaps:

```
minutes
```

For:

```
available inventory shown on listing
```

perhaps slight staleness is acceptable.

For:

```
final inventory reservation
```

you probably need authoritative state and concurrency protection.

This leads to a critical principle:

> **Caching is usually a performance optimization, not your source of truth.**

---

# ⚠️ Common misconception

### “Redis is fast, so I should cache everything.”

That's not good architecture.

Caching introduces:

```
stale data

invalidation logic

extra infrastructure

serialization cost

memory usage

failure modes

stampedes

consistency complexity
```

If your database query takes:

```
2 ms
```

and receives:

```
20 requests/minute
```

adding Redis might make the system **more complicated without meaningfully improving it**.

Measure first.

Remember an earlier lesson:

```
EXPLAIN
```

Perhaps the real problem isn't missing cache.

Perhaps your database query is doing:

```
full table scan
```

because you forgot an index.

The better progression is often:

```
Understand query
      ↓
Fix data access/indexing
      ↓
Measure
      ↓
Introduce caching where workload
actually benefits
```

not:

```
slow endpoint
    ↓
REDIS!
```

---

# 🛒 E-commerce architecture

Imagine your storefront receives huge read traffic.

A simplified design might evolve into:

```
                   Users
                     │
                     ▼
                    CDN
                     │
                     ▼
                 Application
                     │
              ┌──────┴──────┐
              ▼             ▼
            Redis          MySQL
              │             │
         hot product     source of
            data           truth
```

Different caching layers may exist:

```
Browser cache
     ↓
CDN cache
     ↓
Application cache
     ↓
Database buffer cache
```

Yes—the database itself caches heavily too.

Caching isn't really one technology.

It's a recurring systems technique:

> **Keep frequently needed data closer to where it will be used.**

You'll see that same idea in:

```
CPU caches
operating systems
databases
CDNs
DNS
browsers
Redis
AI prompt caching
```

---

# 🤖 Even AI engineering has the same problem

Imagine your application repeatedly sends the same large system prompt:

```
50,000 tokens of documentation
+
user question
```

for every request.

Some AI platforms support forms of **prompt caching**, allowing repeated prompt prefixes to avoid some repeated computation.

Conceptually:

```
same expensive context
        ↓
reuse previously processed work
        ↓
lower latency/cost
```

It's still the same fundamental optimization:

```
expensive repeated work
        ↓
remember previous work
        ↓
reuse
```

The implementation changes.

The idea doesn't.

---

# 🔗 Your mental map

We can now connect several morning lessons:

```
Hash tables
     ↓
Fast key/value lookup

Database indexes
     ↓
Find persistent data efficiently

Caching
     ↓
Avoid repeated expensive retrieval/work

Race conditions
     ↓
What if many cache misses happen together?

Queues
     ↓
Buffer asynchronous work

Idempotency
     ↓
Make repeated work safe
```

You're beginning to see that software-engineering topics aren't isolated interview questions.

They combine inside real systems.

## 🧩 Quick quiz

**1. What's the difference between an index and a cache?**

An index helps the database find data more efficiently; a cache can avoid performing the database operation altogether by returning a previously stored result.

**2. What is a cache stampede?**

Many requests simultaneously miss or observe an expired cache entry and all regenerate the same expensive value, potentially overwhelming the underlying system.

**3. Why isn't a longer TTL automatically better?**

It improves cache hit rate but increases how long users may observe stale data.

---

# 🛠️ Today's practical challenge

You have this endpoint:

```
httpGET /api/products/123
```

Traffic:

```
50,000 requests/hour
```

Product information changes approximately:

```
2–3 times/day
```

The response combines:

```
product
category
images
average review rating
available inventory
```

Design a cache strategy.

But don't simply say:

```
cache everything for 10 minutes
```

Think separately about each field.

Ask:

```
Can product name be stale for 5 minutes?

Can images?

Can review rating?

Can displayed inventory?

What about inventory used during
the actual purchase?
```

Then consider:

```
cache key:
product:123

TTL:
?

invalidation:
?

stampede protection:
?

source of truth:
?
```

Finally imagine:

```
product 123 becomes viral
```

and receives:

```
20,000 requests/sec
```

Your design should prevent cache expiration from suddenly turning all 20,000 requests into database queries.

For practical PHP reading, Laravel's official [Cache documentation](https://laravel.com/docs/cache?utm_source=chatgpt.com) covers TTLs, atomic locks, and cache stores. The [Redis documentation](https://redis.io/docs/latest/?utm_source=chatgpt.com) is worth bookmarking as you move deeper into backend architecture.

### 🧠 Keep one mental model today

> **Caching trades freshness and complexity for speed and reduced load.**

Whenever someone says:

```
"Let's cache it."
```

train yourself to immediately ask:

```
How long can it be stale?

How is it invalidated?

What happens on a miss?

What happens when 10,000 requests
miss simultaneously?

What remains the source of truth?
```

Those questions turn caching from a trick into an engineering decision.
