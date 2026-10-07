---
title: "🌅 Morning Tech Lesson #5 — HTTP Idempotency: Making Retries Safe"
order: 9
volume: 2
category: morning
source: Evening Tech Discovery
---

# 🌅 Morning Tech Lesson #5 — HTTP Idempotency: Making Retries Safe

We’ve recently covered **Big-O, hash tables, database indexes, and dependency injection**. Today we’ll switch to **networking/API design**, while connecting back to database design and distributed systems.

The key question:

> **What happens if your client sends the same payment request twice?**

Estimated time: **10–15 minutes**

## The real-world problem

Imagine your checkout frontend sends:

```
httpPOST /api/orders
```

with:

```json
{
  "product_id": 123,
  "quantity": 1
}
```

Your backend successfully creates:

```
Order #9001
```

but before the response reaches the browser, the network drops.

From the frontend's perspective:

```
Request sent
     ↓
     ???
     ↓
timeout
```

It doesn't know whether the server processed the request.

A perfectly reasonable retry mechanism does:

```
POST /api/orders
        ↓
timeout
        ↓
retry POST /api/orders
```

But the backend might see:

```
Request 1 → Create Order #9001
Request 2 → Create Order #9002
```

Now the customer has **two orders**.

If this endpoint charges a card, things become much worse:

```
$99 charge
+
$99 charge
```

Nothing necessarily malfunctioned.

The server behaved correctly twice.

The problem is that our distributed system couldn't determine whether the second request represented:

> “Do this again.”

or:

> “I'm retrying the thing I already asked you to do.”

That is where **idempotency** enters the picture.

---

# What does idempotent mean?

An operation is idempotent when performing it multiple times has the same intended effect as performing it once.

Consider:

```
httpDELETE /users/123
```

First request:

```
User exists
    ↓
delete
    ↓
user gone
```

Second request:

```
User already gone
    ↓
still gone
```

The final state is the same:

```
user 123 does not exist
```

So deletion can naturally be designed as an idempotent operation.

Compare that with:

```
httpPOST /payments
```

Every execution might create another payment:

```
request #1 → $100 payment

request #2 → another $100 payment

request #3 → another $100 payment
```

That's **not naturally idempotent**.

But we can design it to behave idempotently.

---

# Enter the idempotency key

The client generates a unique identifier for the logical operation:

```
checkout_7f92ab...
```

Then sends:

```
httpPOST /payments
Idempotency-Key: checkout_7f92ab
```

The server remembers:

```
checkout_7f92ab
       ↓
Payment #8271
       ↓
$100
```

If the request arrives again:

```
httpPOST /payments
Idempotency-Key: checkout_7f92ab
```

the server recognizes:

```
I've already processed this operation.
```

Instead of creating another payment, it returns the previous result.

Conceptually:

```
Request
   │
   ▼
Idempotency key exists?
   │
 ┌─┴──────────────┐
 │                │
YES               NO
 │                │
 ▼                ▼
return         perform
previous       operation
result            │
                  ▼
              save result
```

This converts a dangerous retry into a safe retry.

---

# A simple PHP implementation

Let's simplify the idea:

```php
function createPayment(
    string $idempotencyKey,
    int $amount
): array {
    $existing = findRequest($idempotencyKey);

    if ($existing !== null) {
        return $existing['response'];
    }

    $payment = processPayment($amount);

    saveRequest(
        $idempotencyKey,
        $payment
    );

    return $payment;
}
```

Your table might look conceptually like:

```
idempotency_key       response
-------------------------------------
abc123                payment_8271
def456                payment_8272
```

Then:

```
abc123 → payment_8271
abc123 → payment_8271
abc123 → payment_8271
```

instead of:

```
request → payment_8271
retry   → payment_8272
retry   → payment_8273
```

Notice something from an earlier lesson?

We're doing:

```
key → value
```

Again.

The same lookup idea from **hash tables and database indexes** appears inside distributed API design.

---

# But our implementation has a serious bug

Consider two requests arriving almost simultaneously:

```
Request A                 Request B
    │                         │
check abc123             check abc123
    │                         │
not found                not found
    │                         │
charge $100              charge $100
```

Both checked before either saved the result.

Congratulations—we still charged twice.

This is a **race condition**.

The problem isn't visible when requests execute sequentially:

```
A → finish → B
```

It appears with concurrency:

```
A ──────────┐
            ├── overlapping
B ──────────┘
```

This is your first small introduction to **concurrency bugs**.

---

# Let the database enforce uniqueness

Suppose we create:

```sql
CREATE TABLE idempotency_requests (
    idempotency_key VARCHAR(255) NOT NULL,
    response JSON,
    created_at TIMESTAMP NOT NULL,

    UNIQUE (idempotency_key)
);
```

Now the database guarantees:

```
abc123
```

can exist only once.

Two concurrent requests cannot successfully claim the same key independently.

Conceptually:

```
Request A
    │
INSERT abc123
    │
    ▼
SUCCESS


Request B
    │
INSERT abc123
    │
    ▼
UNIQUE CONSTRAINT VIOLATION
```

Request B can then wait/retrieve the operation associated with that key, depending on the design.

This illustrates a powerful engineering principle:

> **Put critical invariants as close as possible to the system capable of enforcing them atomically.**

Application code saying:

```
PHPif (!$exists) {
    create();
}
```

doesn't automatically make the combined operation atomic.

---

# Why transactions matter

Imagine order creation requires:

```
create order
     ↓
reserve inventory
     ↓
record payment
```

Something fails halfway through.

Without careful transaction boundaries:

```
Order created ✓

Inventory reserved ✓

Payment record ✗
```

Your system now has inconsistent state.

A database transaction can make related database operations behave like one logical unit:

```
SQLBEGIN;

INSERT INTO orders (...);

UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 123;

INSERT INTO payments (...);

COMMIT;
```

If something fails:

```
SQLROLLBACK;
```

Conceptually:

```
ALL succeed
    ↓
COMMIT

or

something fails
    ↓
ROLLBACK
```

Transactions and idempotency solve different problems, but you'll often see them together.

**Transaction:** keep related state changes consistent.

**Idempotency:** prevent repeated logical requests from causing repeated side effects.

---

# Why retries exist in the first place

A beginner might ask:

> Why don't we simply disable retries?

Because distributed systems fail constantly in small ways.

Your request crosses things like:

```
Browser
   ↓
Wi-Fi
   ↓
Internet
   ↓
CDN
   ↓
Load balancer
   ↓
API
   ↓
Database
   ↓
payment provider
```

Any component can:

```
timeout
disconnect
restart
drop a packet
return temporary failure
```

Retries are therefore extremely useful.

The mature design isn't:

```
Never retry.
```

It's:

```
Make operations safe to retry.
```

This principle appears everywhere:

```
HTTP APIs
payment processing
message queues
background jobs
webhooks
microservices
cloud infrastructure
```

---

# Queue systems have exactly the same problem

Imagine:

```
Queue
  │
  ▼
SendOrderEmailJob
```

The worker processes the message:

```
send email ✓
```

but crashes before acknowledging completion.

The queue thinks:

```
Maybe it wasn't processed.
```

So it delivers the job again.

Now:

```
email #1 ✓
email #2 ✓
```

This is why many queue systems are described using concepts such as **at-least-once delivery**.

Your consumer must often assume:

> **This message may arrive more than once.**

Therefore:

```
job_id → already processed?
```

becomes another idempotency problem.

We'll revisit this when we study **queues and distributed systems**.

---

# HTTP methods and idempotency

HTTP itself includes this idea.

Methods such as:

```
GET
PUT
DELETE
```

are defined with idempotent semantics.

For example:

```
httpPUT /users/42
```

with:

```json
{
  "name": "Alice"
}
```

sending the same request repeatedly should leave the intended resource state as:

```
name = Alice
```

`POST`, however, isn't inherently idempotent.

That's why operations such as:

```
httpPOST /payments
POST /orders
POST /refunds
```

often need application-level idempotency mechanisms.

The HTTP specification discusses this explicitly because clients can automatically retry idempotent requests when communication fails before they receive a response.

---

# 🛒 E-commerce example

Suppose an AI shopping agent eventually calls:

```
httpPOST /checkout
```

Humans double-click buttons.

Mobile apps retry.

Networks fail.

Webhooks repeat.

Queues redeliver.

AI agents may retry tool calls.

Your backend cannot safely assume:

```
one request
=
one delivery
=
one execution
```

A more realistic model is:

```
one logical intention
        │
        ├── request
        ├── retry
        ├── retry
        └── retry
             │
             ▼
      ONE business effect
```

This becomes increasingly important as software—not humans—starts calling commerce APIs autonomously.

---

# ⚠️ Common misconception

### “If I use an idempotency key, duplicate operations are impossible.”

Not automatically.

This is still dangerous:

```
PHPif (!IdempotencyKey::exists($key)) {
    chargeCard();

    IdempotencyKey::create($key);
}
```

Two concurrent requests can both pass `exists()`.

You need to think about:

```
atomicity
unique constraints
transactions
locking
failure recovery
expiration
request equivalence
```

The idempotency key is the **identity mechanism**.

Correct concurrency handling makes it reliable.

---

# 🔗 Connect today's lesson to earlier concepts

Your mental map is expanding:

```
Big-O
   ↓
How does work grow?

Hash tables
   ↓
Fast key lookup

Database indexes
   ↓
Fast persistent lookup

Dependency Injection
   ↓
Separate business logic from infrastructure

Idempotency
   ↓
Make distributed operations safe to repeat
```

And today we encountered two future topics:

```
Race conditions
      ↓
Concurrency

Duplicate queue delivery
      ↓
Distributed systems
```

We'll return to both later.

---

# 🧩 Quick quiz

**1. Your client sends `POST /payments`, times out, and retries. What's the danger?**

The first request may have succeeded even though the response was lost, so the retry could create a second payment.

**2. Why isn't this safe enough?**

```
PHPif (!$repository->exists($key)) {
    $repository->create($key);
    process();
}
```

Because two concurrent requests may both observe that the key doesn't exist before either creates it: a **race condition**.

**3. Why might a queue deliver the same job twice?**

The worker may perform the operation but fail before acknowledging the message, causing the queue to redeliver it.

---

# 🛠️ Today's practical challenge

You're building:

```
httpPOST /api/orders
```

The frontend sends:

```
httpIdempotency-Key: order-f91a82
```

Your system currently does:

```
PHPpublic function createOrder(
    string $key,
    array $data
): Order {
    if ($this->requests->exists($key)) {
        return $this->requests->getOrder($key);
    }

    $order = $this->orders->create($data);

    $this->payments->charge(
        $order->total
    );

    $this->requests->save($key, $order);

    return $order;
}
```

Think about three failure points:

```
1. Two requests arrive simultaneously.

2. The payment succeeds but the process crashes
   before save($key).

3. The same key is reused with a different order body.
```

Your challenge isn't merely to rewrite the PHP.

Design the behavior.

Ask:

> **What information should be stored with the idempotency key, what should the database enforce, and what should happen when an operation is currently in progress?**

That question moves you from ordinary CRUD development toward **distributed-systems engineering**.

For deeper reading, the official HTTP Semantics specification's section on idempotent methods is the authoritative foundation. For a concrete production API design, [Stripe's documentation on idempotent requests](https://docs.stripe.com/api/idempotent_requests?utm_source=chatgpt.com) is an excellent practical example.

### 🧠 Keep one mental model today

> **A timeout means “I don't know what happened,” not “nothing happened.”**

Once you internalize that sentence, retries, idempotency, queues, transactions and distributed systems start making much more sense.
