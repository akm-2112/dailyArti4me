---
title: "🌅 Morning Tech Lesson #9 — Database Transactions: ACID Is Really About Protecting Invariants"
order: 17
volume: 3
category: morning
source: Evening Tech Discovery
---

# 🌅 Morning Tech Lesson #9 — Database Transactions: ACID Is Really About Protecting Invariants

We’ve recently covered **idempotency → queues → race conditions → caching**. Today we’ll deepen the database side of that chain.

The central question is:

> **What happens when a business operation changes several pieces of data, but the application crashes halfway through?**

Estimated time: **10–15 minutes**.

## Start with a money-transfer example

Suppose Alice has `$100` and sends `$30` to Bob.

Your PHP code might look like:

```php
$alice->balance -= 30;
$alice->save();

$bob->balance += 30;
$bob->save();
```

Conceptually:

```
Alice: $100 → $70
Bob:    $50 → $80
```

But imagine the process dies here:

```
Alice: $100 → $70  ✓

        💥 CRASH

Bob:    $50 → $80  ✗
```

Thirty dollars effectively disappeared.

The individual SQL statements were valid. The **business operation** was not.

The important observation is:

```
two database writes
        ≠
two independent business operations
```

From the business perspective, this is one operation:

```
TRANSFER $30
```

It should either:

```
Alice -30 AND Bob +30
```

or:

```
nothing changes
```

That's what a transaction gives us.

---

# 🧠 The basic transaction

In SQL:

```
SQLBEGIN;

UPDATE accounts
SET balance = balance - 30
WHERE id = 1;

UPDATE accounts
SET balance = balance + 30
WHERE id = 2;

COMMIT;
```

If something fails:

```
SQLROLLBACK;
```

Conceptually:

```
BEGIN
  │
  ├── change Alice
  │
  ├── change Bob
  │
  ▼
COMMIT
```

Before `COMMIT`, these changes belong to an unfinished transaction.

If the transaction fails:

```
ROLLBACK
   ↓
restore previous state
```

So instead of exposing:

```
Alice = $70
Bob   = $50
```

we preserve the invariant:

```
total money remains constant
```

In Laravel, the same idea is commonly expressed as:

```
PHPDB::transaction(function () use ($alice, $bob) {
    $alice->decrement('balance', 30);
    $bob->increment('balance', 30);
});
```

The syntax is convenient.

The concept matters much more.

---

# ACID without the textbook fog

You've probably heard:

```
A C I D
```

Let's make each letter practical.

## A — Atomicity

Atomicity means:

> **The transaction succeeds as one unit or fails as one unit.**

Our transfer:

```
Alice -30
Bob   +30
```

should never permanently become:

```
Alice -30
Bob    0
```

Think:

```
all
or
nothing
```

This connects directly to the word **atomic** from your race-condition lesson.

---

## C — Consistency

Consistency is often explained badly.

The useful mental model is:

> **A successful transaction should take the database from one valid state to another valid state.**

Suppose your invariant is:

```
balance >= 0
```

or:

```
orders.user_id must reference
an existing user
```

Database mechanisms can help enforce those rules:

```
CHECK constraints
UNIQUE constraints
FOREIGN KEY constraints
NOT NULL
```

But ACID consistency does **not** mean:

> “The database magically understands every business rule.”

If your application incorrectly transfers:

```
$3,000 instead of $30
```

the database cannot know that was wrong unless you've encoded a relevant invariant.

Transactions protect rules you've actually designed.

---

# I — Isolation

This is where today's lesson connects strongly to concurrency.

Suppose two transactions execute simultaneously:

```
Transaction A ─────────────>

      Transaction B ─────────────>
```

Isolation controls how much one transaction can observe or interfere with another transaction's intermediate work.

Ideally, you'd like concurrent transactions to behave as though there were some valid serial ordering:

```
A then B
```

or:

```
B then A
```

rather than some dangerous mixture.

But full isolation can be expensive.

Databases therefore expose different **isolation levels**, making trade-offs between concurrency and protection.

Common SQL names include:

```
READ UNCOMMITTED

READ COMMITTED

REPEATABLE READ

SERIALIZABLE
```

We won't memorize them today.

Instead, understand *why they exist*.

---

# A concrete isolation problem: lost update

Remember the previous lesson?

Inventory:

```
stock = 10
```

Two requests:

```
A wants 3
B wants 4
```

Naive execution:

```
A reads 10

                 B reads 10

A calculates 7

                 B calculates 6

A writes 7

                 B writes 6
```

Final:

```
stock = 6
```

But we sold:

```
3 + 4 = 7
```

so stock should be:

```
3
```

One update was effectively overwritten.

That's a **lost update**.

A transaction alone doesn't necessarily solve every version of this problem.

You still need the right:

```
SQL operation
locking
isolation level
optimistic concurrency strategy
```

depending on the situation.

Remember this distinction:

> **Transactions provide a framework for correctness; they do not automatically make every concurrent algorithm correct.**

---

# D — Durability

Suppose:

```
SQLCOMMIT;
```

succeeds.

Then the database machine immediately loses power.

Durability means the database promises that a committed transaction won't simply disappear because the process or machine crashes.

Databases typically achieve this using techniques involving persistent storage and logs.

A simplified model:

```
transaction changes
       ↓
write durable log
       ↓
COMMIT acknowledged
       ↓
eventually update data pages
```

This leads to an important database concept:

**Write-Ahead Logging (WAL).**

Instead of relying only on modifying the main database pages immediately, the database records what must happen in a durable log first.

After a crash:

```
Database starts
      ↓
inspect log
      ↓
recover committed operations
```

The exact implementation differs across databases, but the principle is enormously important.

---

# 🧠 Why logs appear everywhere in systems engineering

You've now encountered several ideas that resemble:

```
record what happened
        ↓
use history for recovery
```

Databases:

```
Write-Ahead Log
```

Git:

```
commit history
```

Message systems:

```
event log / offsets
```

Durable workflows:

```
execution history
```

Distributed systems repeatedly discover that **an ordered durable history is incredibly powerful**.

We'll eventually connect this to systems such as Kafka and event sourcing.

---

# 🛒 Now consider checkout

Suppose checkout performs:

```
PHPDB::transaction(function () use ($order) {
    $order->markAsPaid();

    $inventory->reserve($order);

    $coupon->markAsUsed();
});
```

Great.

All three operations use the same relational database.

You can potentially make them one transaction:

```
mark paid
reserve inventory
consume coupon
      │
      ▼
    COMMIT
```

Now add:

```php
$mailer->sendConfirmation($order);
```

Should that happen **inside** the database transaction?

Probably not.

Imagine:

```
BEGIN

update order

reserve inventory

call email API
      │
      │ 8 seconds...
      │
      │ 15 seconds...
      │
      ▼
email provider timeout

ROLLBACK
```

While waiting, your database transaction may be holding:

```
locks
connections
resources
```

That's undesirable.

Worse, imagine:

```
email successfully sent ✓

database COMMIT fails ✗
```

Now the customer receives:

```
"Your order is confirmed!"
```

while your database says:

```
order not confirmed
```

Your database cannot roll back an email.

---

# 🚨 Transactions stop at system boundaries

This is one of today's most important ideas.

Your database transaction can coordinate:

```
MySQL row A
MySQL row B
MySQL row C
```

But it generally cannot simply roll back:

```
Stripe charge
SendGrid email
FedEx shipment
S3 upload
another microservice
```

So:

```
BEGIN;

UPDATE orders...;

sendEmail();       ← external world

COMMIT;
```

creates a difficult failure boundary.

This is where distributed systems become harder than ordinary database programming.

---

# The transactional outbox pattern

Here's a pattern you'll encounter in serious backend systems.

Suppose after creating an order we need to publish:

```
OrderCreated
```

to a queue.

Naively:

```
PHPDB::transaction(function () use ($data) {
    $order = Order::create($data);
});

$queue->publish(
    new OrderCreated($order->id)
);
```

Failure:

```
database commit ✓

        💥 crash

queue publish ✗
```

The order exists, but downstream workers never hear about it.

Reverse the order:

```
publish event ✓

database commit ✗
```

Now workers hear about an order that doesn't exist.

We need:

```
database change
+
event creation
```

to succeed atomically.

The trick is surprisingly simple.

Put the event into the **same database transaction**.

```
BEGIN

INSERT order

INSERT outbox_event

COMMIT
```

Tables:

```
orders
----------------
9281 | paid


outbox
------------------------------------
501 | OrderCreated | {"order":9281}
```

Now both records either exist:

```
order ✓
event ✓
```

or neither does.

A separate process reads the outbox:

```
Outbox table
     │
     ▼
Publisher
     │
     ▼
Queue
     │
     ▼
Workers
```

If publishing fails:

```
event remains in outbox
      ↓
retry later
```

And what concept do retries immediately remind you of?

**Idempotency.**

The receiving consumer should tolerate duplicate publication.

Your earlier lessons are combining.

---

# 🔗 The full chain

Suppose checkout creates an order:

```
HTTP Request
     │
     ▼
Database Transaction
     │
     ├── Order
     ├── Inventory reservation
     └── Outbox event
              │
              ▼
            COMMIT
```

Later:

```
Outbox Publisher
       │
       ▼
      Queue
       │
       ▼
Email Worker
```

Now combine previous lessons:

```
Transaction
    ↓
order + event written atomically


Queue
    ↓
email processing happens asynchronously


Idempotency
    ↓
duplicate delivery is safe


Race-condition protection
    ↓
inventory cannot be oversold


Caching
    ↓
product reads avoid unnecessary DB load
```

This is starting to resemble a real production architecture rather than isolated concepts.

---

# ⚠️ Common misconception

### “A transaction means nothing can go wrong.”

Transactions protect a specific boundary.

Usually:

```
one database
```

They don't magically cover:

```
HTTP APIs
queues
emails
filesystems
other databases
payment providers
```

Another misconception is:

```
make every operation one giant transaction
```

Long-running transactions can create:

```
lock contention
blocked requests
deadlocks
connection exhaustion
lower throughput
```

The goal is not:

> **Use the biggest transaction possible.**

It's:

> **Identify the invariant and create the smallest transaction boundary that protects it correctly.**

---

# 🧩 Quick quiz

**1. Alice's balance is decreased, then the process crashes before Bob's balance increases. Which ACID property should prevent the half-completed transfer from becoming permanent?**

**Atomicity.**

Both changes should commit together or roll back together.

**2. Why is sending an email inside a database transaction dangerous?**

Because the email service isn't part of the database transaction. The email can succeed while the database rolls back, and waiting on the external service can keep locks/resources open.

**3. Why does the outbox pattern write an event to a database table first instead of immediately publishing it?**

Because the business state and the intent to publish can then be committed atomically in the same database transaction.

---

# 🛠️ Today's practical challenge

You're implementing:

```
POST /api/orders
```

The operation must:

```
1. create order
2. decrement inventory
3. mark coupon used
4. publish OrderCreated
5. send confirmation email
```

Design the transaction boundary.

Your starting architecture should probably resemble:

```
DATABASE TRANSACTION

create order
decrement inventory
mark coupon used
write outbox event

        ↓

COMMIT


AFTERWARD

outbox publisher
      ↓
queue
      ↓
email worker
```

Now answer these mentally:

```
What database constraints protect inventory?

What if two checkouts use the final item?

What if the outbox publisher crashes
after publishing but before marking
the event processed?

What if the email worker receives
OrderCreated twice?
```

If your answer to the last two starts with:

```
"Duplicates are possible, so..."
```

you're connecting the material correctly.

For authoritative deeper reading, [PostgreSQL's transaction documentation](https://www.postgresql.org/docs/current/tutorial-transactions.html?utm_source=chatgpt.com) gives a clean introduction to transaction boundaries and savepoints. For the distributed-systems pattern introduced today, [AWS Prescriptive Guidance on the transactional outbox pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html?utm_source=chatgpt.com) gives a practical architecture-level treatment.

### 🧠 Keep one mental model today

> **A transaction protects an invariant across a group of database changes—but the protection ends when you leave that transactional boundary.**

So whenever your code does:

```
database
   ↓
external API
   ↓
queue
   ↓
another database
```

ask:

> **What happens if the process dies between any two arrows?**

That question is one of the gateways from ordinary backend development into reliable distributed-systems engineering.
