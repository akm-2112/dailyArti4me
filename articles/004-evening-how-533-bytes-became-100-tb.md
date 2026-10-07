---
title: "🌙 Evening Tech Discovery — How 533 bytes became 100 TB"
order: 4
volume: 1
category: evening
source: Evening Tech Discovery
---

# 🌙 Evening Tech Discovery — How 533 bytes became 100 TB

Tonight's read is a great example of **systems engineering at scale**.

On **August 27, 2026**, [Cloudflare](https://www.cloudflare.com/?utm_source=chatgpt.com) published an engineering deep dive explaining how it reduced the memory footprint of the DNS cache behind 1.1.1.1 by roughly **100 terabytes of RAM**—while also making it faster. [Cloudflare Blog](https://blog.cloudflare.com/dns-cache-memory-optimization-1111/?utm_source=chatgpt.com)

The surprising part isn't some revolutionary algorithm.

It was largely about **data structures, memory layout, allocations, and CPU cache locality**.

## The scale changes everything

Cloudflare's DNS platform keeps more than **250 billion DNS cache entries** in memory at any given moment. [Cloudflare Blog](https://blog.cloudflare.com/dns-cache-memory-optimization-1111/?utm_source=chatgpt.com)

At that scale:

```
Saving 1 byte per entry

250,000,000,000 × 1 byte
        ↓
≈ 250 GB
```

So something that would be completely irrelevant in a normal PHP application can become enormously important at infrastructure scale.

Before the work, an average cache entry occupied about:

```
953 bytes
```

Afterward:

```
420 bytes
```

That's a **56% reduction**. Across the fleet, working-set memory fell by roughly **100 TB**. [Cloudflare Blog](https://blog.cloudflare.com/dns-cache-memory-optimization-1111/?utm_source=chatgpt.com)

But here's where it gets interesting for us as developers.

## Optimization #1: don't use a growable structure when the data never grows

Cloudflare's DNS cache is written in Rust.

They had structures using Rust's:

```
RustVec<T>
```

Conceptually, a `Vec` stores something like:

```
pointer
length
capacity
```

The **capacity** exists because the collection may grow.

Imagine:

```
length   = 5
capacity = 8

[ A ][ B ][ C ][ D ][ E ][   ][   ][   ]
```

Those empty positions allow future appends without immediately allocating new memory.

That's useful for something like:

```typescript
const messages = [];

messages.push(message1);
messages.push(message2);
messages.push(message3);
```

But Cloudflare noticed something important about its DNS cache:

> Once a DNS response enters the cache, its record lists don't grow.

So why pay for growability?

They replaced several `Vec<T>` structures with fixed-size structures similar to:

```
RustBox<[T]>
```

Now the representation essentially needs:

```
pointer
length
```

No capacity.

Across eight relevant fields, Cloudflare calculated this change alone saved **64 bytes per cache entry**, plus unused heap capacity—more than **15 TB** across 250+ billion entries. [Cloudflare Blog](https://blog.cloudflare.com/dns-cache-memory-optimization-1111/?utm_source=chatgpt.com)

This is a beautiful engineering principle:

> **Choose a data structure based on how the data behaves, not merely because it's convenient.**

---

# Optimization #2: fewer structures

A cached DNS response contains sections such as:

```
Answers
Authority
Additional
```

Originally these were separate lists.

Conceptually:

```
CacheEntry

answers ──────────→ [record][record]

authority ────────→ [record]

additional ───────→ [record][record]
```

Every separate structure needs metadata such as pointers and lengths.

Instead, they combined records:

```
[ answer ][ answer ][ authority ][ additional ][ additional ]
     ↑                    ↑
     offsets identify sections
```

Then small numeric offsets identify where each section starts.

That eliminated more pointers and metadata, saving another **28 bytes per entry**. [Cloudflare Blog](https://blog.cloudflare.com/dns-cache-memory-optimization-1111/?utm_source=chatgpt.com)

Again:

```
28 bytes
```

sounds laughably small.

Until:

```
28 × 250 billion
```

becomes terabytes.

---

# Optimization #3: memory locality

This one is especially worth understanding.

Suppose your program needs:

```
User
Orders
Address
Permissions
```

If those pieces are scattered throughout memory:

```
RAM

[User] ................ [Order] .............. [Address]
                    .............. [Permissions]
```

the CPU repeatedly has to fetch data from different locations.

Modern CPUs don't usually fetch exactly one variable from RAM.

They load chunks called **cache lines** into very fast CPU caches.

So data packed close together:

```
[User][Order][Address][Permissions]
```

can often be processed faster than equivalent data scattered around memory.

Cloudflare changed its DNS representation so more record data was stored **contiguously**, reducing pointer chasing and allocations. [Cloudflare Blog](https://blog.cloudflare.com/dns-cache-memory-optimization-1111/?utm_source=chatgpt.com)

This improved **CPU cache locality**.

And that's where the story becomes counterintuitive.

You might expect:

> “We're optimizing memory, so maybe performance becomes slightly worse.”

Instead, Cloudflare reported:

| Metric | Before | After |
| --- | --- | --- |
| Memory per entry | 953 B | 420 B |
| Insert throughput | 625k/sec | 893k/sec |
| Lookup latency | 828 ns | 670 ns |

So:

**Memory ↓ 56%**

**Insert throughput ↑ 43%**

**Lookup latency ↓ 19%**

[Cloudflare Blog](https://blog.cloudflare.com/dns-cache-memory-optimization-1111/?utm_source=chatgpt.com)

Less memory wasn't merely cheaper.

It was **faster**.

---

# 🧠 Why should a PHP / TypeScript developer care?

You probably aren't managing 250 billion objects.

But the underlying lesson applies directly to backend development.

Imagine you fetch 100,000 database rows into PHP:

```php
$users = User::all();
```

Then create several transformed copies:

```php
$activeUsers = $users->filter(...);

$mappedUsers = $activeUsers->map(...);

$groupedUsers = $mappedUsers->groupBy(...);
```

Conceptually you may now have:

```
Database rows
     ↓
PHP objects
     ↓
Collection
     ↓
Filtered collection
     ↓
Mapped collection
     ↓
Grouped structure
```

Each abstraction may involve memory, objects, metadata, allocations and references.

For 100 records?

Who cares.

For **10 million records?**

Suddenly architecture matters.

Maybe instead:

```sql
SELECT id, email
FROM users
WHERE active = 1;
```

lets the database filter the data before PHP ever sees it.

Or maybe you stream/chunk records:

```
PHPUser::query()
    ->where('active', true)
    ->chunkById(1000, function ($users) {
        // process
    });
```

The general principle is the same:

> **Don't move, allocate, or retain data you don't actually need.**

---

# There's an even bigger system-design lesson

Remember our recent lesson about **hash tables**?

We talked about trading:

```
more memory
    ↓
faster lookup
```

Cloudflare's DNS cache is essentially a gigantic real-world example of those trade-offs.

Caching itself says:

```
Use RAM
   ↓
avoid expensive upstream work
   ↓
respond faster
```

Cloudflare isn't planning to simply remove the RAM it freed.

Instead, it plans to use that freed memory to hold **more DNS cache entries**. [Cloudflare Blog](https://blog.cloudflare.com/dns-cache-memory-optimization-1111/?utm_source=chatgpt.com)

That should increase:

```
cache hit rate ↑
```

which means:

```
upstream DNS queries ↓
```

which can mean:

```
latency ↓
network traffic ↓
upstream load ↓
```

One low-level optimization can therefore ripple upward through the entire distributed system.

---

# 🔗 Connect the last few concepts

We're starting to build an important mental map:

```
Big-O
  │
  ▼
How does work grow?

Hash tables
  │
  ▼
Trade memory for fast lookup

Caching
  │
  ▼
Trade memory for avoiding expensive work

Memory layout
  │
  ▼
Reduce memory + improve CPU locality

System design
  │
  ▼
What happens when this operates
billions of times?
```

This is why learning fundamentals matters even if you spend most of your day writing PHP APIs or TypeScript.

Eventually, **small implementation details become architecture when multiplied by enough traffic.**

## One idea worth remembering tonight

When optimizing software, developers naturally look for the expensive-looking code:

```
big algorithm
slow SQL query
network request
complex function
```

But at sufficient scale, another question becomes equally powerful:

> **“What tiny thing am I doing billions of times?”**

Saving **533 bytes** doesn't sound impressive.

Saving them **250 billion times** does.

For tonight's deeper read, Cloudflare's original article is excellent and about the right length for your evening session:

[How we saved 100 terabytes of memory by optimizing 1.1.1.1's DNS cache — Cloudflare Engineering, August 27, 2026](https://blog.cloudflare.com/dns-cache-memory-optimization-1111/?utm_source=chatgpt.com)

It's worth reading the diagrams in the original—the sections on Rust enum sizing, heap allocations, serialization, and CPU locality go deeper than I've covered here.
