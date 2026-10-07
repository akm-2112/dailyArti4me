---
title: "🌅 Morning Tech Lesson #7 — Race Conditions: When Correct Code Becomes Wrong Under Concurrency"
order: 13
volume: 3
category: morning
source: Evening Tech Discovery
---

# 🌅 Morning Tech Lesson #7 — Race Conditions: When Correct Code Becomes Wrong Under Concurrency

Recent lessons introduced **idempotency and queues**, and both quietly exposed the same problem: two pieces of work can overlap. Today we’ll make that problem explicit.

> **A race condition happens when correctness depends on the timing or ordering of concurrent operations.**

Estimated time: **10–15 minutes**.

## 🏦 Start with a deceptively simple example

Suppose an account has:

```
balance = $100
```

Two requests arrive almost simultaneously:

```
Request A: withdraw $80
Request B: withdraw $50
```

Your PHP code:

```php
function withdraw(int $accountId, int $amount): void
{
    $account = Account::find($accountId);

    if ($account->balance < $amount) {
        throw new InsufficientFunds();
    }

    $account->balance -= $amount;

    $account->save();
}
```

Reading it sequentially, it looks correct:

```
read balance
    ↓
check enough money
    ↓
subtract
    ↓
save
```

But production doesn't promise requests execute one after another.

They can overlap.

Imagine this sequence:

```
Request A                         Request B

READ balance = 100
                                  READ balance = 100

100 >= 80 ✓
                                  100 >= 50 ✓

new balance = 20
                                  new balance = 50

SAVE 20
                                  SAVE 50
```

Final database value:

```
balance = $50
```

But users successfully withdrew:

```
$80 + $50 = $130
```

from an account containing only $100.

Every individual line behaved correctly.

The **interleaving** made the overall operation incorrect.

---

# 🧠 Why this happens

The dangerous assumption was that:

```php
$account->balance -= $amount;
```

is one indivisible operation.

Conceptually it isn't.

It means something closer to:

```
READ balance
     ↓
COMPUTE balance - amount
     ↓
WRITE new balance
```

That's multiple operations.

Another request can execute between them.

This pattern is often called:

> **read → modify → write**

and it's one of the first things you should notice when thinking about concurrency.

---

# What does concurrency actually mean?

Concurrency doesn't necessarily mean two CPUs executing instructions at the exact same nanosecond.

For backend engineering, the useful mental model is simply:

```
multiple operations can be in progress
during overlapping periods
```

For example:

```
Request A ────────────────>

       Request B ────────────────>
```

Your PHP workers, Node processes, queue workers, database connections, containers, or servers may all operate independently.

Imagine your production architecture:

```
                 Load Balancer
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
      Server A                 Server B
          │                       │
          └───────────┬───────────┘
                      ▼
                    MySQL
```

Two requests for the same account might hit entirely different machines.

This is why something like:

```php
$processing = true;
```

inside one PHP process cannot generally protect distributed application state.

Server B has its **own memory**.

---

# 🔒 Solution #1: let the database perform an atomic update

Instead of:

```
READ
 ↓
CHECK
 ↓
MODIFY
 ↓
WRITE
```

we can sometimes express the entire state transition as one database operation:

```
SQLUPDATE accounts
SET balance = balance - 80
WHERE id = 42
AND balance >= 80;
```

Notice what changed.

We're no longer saying:

```
Application:
"Tell me the balance and I'll decide."
```

We're saying:

```
Database:
"Perform this update only if
the invariant still holds."
```

Then inspect how many rows were updated.

Conceptually:

```php
$updated = DB::table('accounts')
    ->where('id', $accountId)
    ->where('balance', '>=', $amount)
    ->decrement('balance', $amount);

if ($updated === 0) {
    throw new InsufficientFunds();
}
```

The important part isn't Laravel syntax.

It's moving:

```
check + update
```

into an operation the database can execute atomically.

---

# What does “atomic” mean?

Think of an atomic operation as:

> **Other operations cannot observe or interfere with a half-completed version of this operation.**

The word comes from the idea of something indivisible.

Conceptually:

```
NOT atomic:

read
  ← another operation can interfere here
modify
  ← or here
write


atomic:

[ check + update ]
```

Atomicity appears everywhere:

```
database transactions
CPU instructions
Redis operations
locks
file operations
distributed systems
```

You'll see this word constantly as your systems knowledge grows.

---

# 🔐 Solution #2: locking

Sometimes the business operation is too complicated for one SQL statement.

Suppose checkout needs to:

```
read inventory
    ↓
verify quantity
    ↓
modify several records
    ↓
create reservation
```

You may use a database transaction and lock the relevant row.

Conceptually:

```
SQLBEGIN;

SELECT quantity
FROM inventory
WHERE product_id = 123
FOR UPDATE;

-- verify quantity

UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 123;

INSERT INTO reservations (...);

COMMIT;
```

`FOR UPDATE` tells the database, roughly:

> “I'm going to modify this row; prevent conflicting transactions from modifying it until I'm finished.”

Now:

```
Transaction A

LOCK product 123
      │
      ▼
read inventory
      │
update inventory
      │
COMMIT
      │
      ▼
unlock


Transaction B

tries lock product 123
      │
      ▼
    WAIT
      │
      ▼
A commits
      │
      ▼
B continues using current state
```

We've converted dangerous overlap into controlled serialization for that resource.

---

# 🛒 The classic e-commerce example

Imagine:

```
PS5 inventory = 1
```

Alice clicks:

```
BUY
```

at almost exactly the same time as Bob.

Naive code:

```typescript
const product = await products.find(id);

if (product.stock > 0) {
    product.stock--;

    await products.save(product);

    await orders.create(...);
}
```

Possible execution:

```
Alice                          Bob

stock = 1
                               stock = 1

1 > 0 ✓
                               1 > 0 ✓

stock = 0
                               stock = 0

save
                               save

order ✓                        order ✓
```

You've sold:

```
2 units
```

while owning:

```
1 unit
```

This is **overselling caused by a race condition**.

The correct solution isn't:

```
TypeScriptif (product.stock > 0) {
```

with an extra clever `if`.

You need a concurrency strategy.

Perhaps:

```
SQLUPDATE products
SET stock = stock - 1
WHERE id = ?
AND stock > 0;
```

or appropriate locking/transactions depending on the larger workflow.

---

# 🧩 Another solution: optimistic concurrency control

Locking isn't always necessary.

Suppose your row contains:

```
id        42
balance   100
version   7
```

You read:

```
version = 7
```

Then update only if nobody has changed the row since:

```
SQLUPDATE accounts
SET
    balance = 20,
    version = 8
WHERE id = 42
AND version = 7;
```

If another request already changed it:

```
version = 8
```

your update affects:

```
0 rows
```

Now you know:

> Someone modified the data after I read it.

You can retry or reject the operation.

This is **optimistic concurrency control**.

The philosophy is:

```
Don't lock immediately.

Assume conflicts are uncommon.

Detect them if they happen.
```

Compare that with pessimistic locking:

```
Assume conflict might happen.

Lock before proceeding.
```

Neither is universally better.

It depends on workload and contention.

---

# 🔄 Now reconnect idempotency

A few lessons ago we had:

```
PHPif (!$requests->exists($key)) {
    processPayment();
    $requests->save($key);
}
```

We said this was unsafe.

Now you can explain exactly why.

Two requests:

```
A                              B

exists? → NO
                               exists? → NO

charge()
                               charge()

save key
                               save key
```

That's a **check-then-act race condition**.

A database uniqueness constraint:

```
SQLUNIQUE(idempotency_key)
```

helps enforce the invariant:

```
one key
   ↓
one logical operation
```

This is why database constraints are more than validation conveniences.

They can be **concurrency correctness mechanisms**.

---

# 🔄 And reconnect queues

Suppose you run 20 workers:

```
                Queue
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
    Worker 1   Worker 2   Worker 3
```

Multiple jobs might update the same entity.

For example:

```
InventoryReserved(order 1)
InventoryReserved(order 2)
```

both affect:

```
product_id = 123
```

Even though the jobs are different, they can race over shared state.

So:

> **Putting work on a queue doesn't eliminate concurrency.**

Often it increases concurrency because you're intentionally running many workers in parallel.

---

# 🟨 What about JavaScript being “single-threaded”?

This is a very common misconception.

You may hear:

> “Node.js is single-threaded, so race conditions aren't possible.”

False.

Consider:

```typescript
const balance = await getBalance(accountId);

// another request can run here

await setBalance(
    accountId,
    balance - amount
);
```

When execution reaches:

```
TypeScriptawait getBalance(...)
```

the event loop can process other work.

Two requests can interleave:

```
Request A
read balance
    │
   await
    │
          Request B
          read balance
              │
             await
              │
Request A resumes
              │
          Request B resumes
```

And your database may also be receiving traffic from:

```
other Node processes
PHP workers
queue workers
cron jobs
other servers
```

“Single-threaded JavaScript” does **not** mean “single-user system.”

---

# ⚠️ Common misconception

### “Transactions automatically prevent race conditions.”

Not necessarily.

This can still be problematic depending on isolation and operations:

```
SQLBEGIN;

SELECT balance FROM accounts WHERE id = 42;

-- application checks balance

UPDATE accounts SET balance = ...;

COMMIT;
```

Simply surrounding code with:

```
BEGIN
...
COMMIT
```

doesn't magically serialize every possible conflict.

You need to understand:

```
what data is locked?

when is it locked?

what isolation level applies?

can another transaction modify it?

what invariant are we protecting?
```

We'll eventually study **transaction isolation levels** and anomalies such as:

```
lost updates
dirty reads
non-repeatable reads
phantom reads
```

Today's goal is simply recognizing the underlying problem.

---

# 🧠 A useful engineering habit

Whenever you see:

```
READ
 ↓
make decision
 ↓
WRITE
```

ask:

> **What happens if another request changes this data between my read and write?**

Whenever you see:

```
IF something doesn't exist
    CREATE it
```

ask:

> **What happens if two requests perform the check simultaneously?**

Whenever you see:

```
increment
decrement
reserve
claim
assign
withdraw
purchase
redeem
```

your concurrency radar should turn on.

---

# 🔗 Your mental map is becoming a system

Notice how several lessons now connect:

```
Database indexes
       │
       ▼
Find state efficiently

Transactions
       │
       ▼
Change related state safely

Idempotency
       │
       ▼
Handle repeated operations

Queues
       │
       ▼
Process work asynchronously

Concurrency
       │
       ▼
Handle overlapping operations

Race conditions
       │
       ▼
Protect invariants when operations overlap
```

The central word here is:

## **Invariant**

An invariant is something your system promises must remain true.

Examples:

```
balance >= 0

stock >= 0

one payment per idempotency key

email address unique per account

one active reservation per seat
```

Strong backend engineering is often about identifying invariants and choosing the correct layer to enforce them.

Sometimes that's:

```
application code
```

Sometimes:

```
database constraint
```

Sometimes:

```
transaction / lock
```

Sometimes:

```
atomic operation
```

The important thing is not accidentally relying on timing.

---

# 🧩 Quick quiz

**1. Why is this unsafe under concurrency?**

```
PHPif ($product->stock > 0) {
    $product->stock--;
    $product->save();
}
```

Because multiple requests can read the same stock value before either saves its update.

**2. Does single-threaded Node.js eliminate race conditions?**

No. Async operations can interleave, multiple Node processes can run, and external systems such as databases are shared concurrently.

**3. What's the difference between pessimistic and optimistic concurrency?**

Pessimistic concurrency prevents conflicts by locking first.

Optimistic concurrency allows work to proceed and detects whether conflicting modification happened before committing the change.

---

# 🛠️ Today's practical challenge

You have a coupon:

```
coupon: WELCOME100
remaining_uses: 1
```

Your current PHP code:

```php
$coupon = Coupon::where(
    'code',
    'WELCOME100'
)->first();

if ($coupon->remaining_uses > 0) {
    $coupon->remaining_uses--;

    $coupon->save();

    $this->orders->applyDiscount(
        $order,
        $coupon
    );
}
```

Two customers submit the coupon simultaneously.

Your task is to design a solution where this invariant always holds:

```
remaining_uses >= 0
```

Start by considering whether you can make the claim atomic:

```
SQLUPDATE coupons
SET remaining_uses = remaining_uses - 1
WHERE code = ?
AND remaining_uses > 0;
```

Then think one step deeper:

```
What happens if decrement succeeds,
but applying the discount fails?
```

Now you're naturally being pushed toward **transactions**.

And if applying the discount involves an external service that can't participate in your database transaction?

That's where things get really interesting—and eventually leads us toward patterns such as **outbox, sagas, and distributed transactions**.

For deeper reading, the official [PostgreSQL documentation on explicit locking](https://www.postgresql.org/docs/current/explicit-locking.html?utm_source=chatgpt.com) is an excellent reference. If you're primarily using MySQL, [MySQL's InnoDB locking documentation](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html?utm_source=chatgpt.com) is worth bookmarking.

### 🧠 Keep one mental model today

> **Correct when executed once does not necessarily mean correct when executed concurrently.**

Whenever your code reads shared state, makes a decision, and writes new state, imagine **two copies of that code running side by side**.

That simple habit catches an enormous class of production bugs before they happen.
