---
title: "🌅 Morning Tech Lesson #6 — Queues: Why Good Backends Don’t Do Everything During the HTTP Request"
order: 11
volume: 2
category: morning
source: Evening Tech Discovery
---

# 🌅 Morning Tech Lesson #6 — Queues: Why Good Backends Don’t Do Everything During the HTTP Request

Recent lessons covered **Big-O → hash tables → indexes → dependency injection → idempotency**. Today we move into **asynchronous architecture** and deliberately revisit idempotency from a new angle.

The key question is:

> **When your API needs to do five things, do they all really need to happen before you return the response?**

Estimated time: **10–15 minutes**.

## Start with a real checkout API

Imagine this PHP endpoint:

```
PHPpublic function checkout(Request $request)
{
    $order = $this->orders->create($request->all());

    $this->payments->charge($order);

    $this->emails->sendConfirmation($order);

    $this->analytics->trackPurchase($order);

    $this->warehouse->notify($order);

    $this->crm->syncCustomer($order);

    return response()->json($order);
}
```

The request path is effectively:

```
Browser
   ↓
Create order
   ↓
Charge payment
   ↓
Send email
   ↓
Analytics API
   ↓
Warehouse API
   ↓
CRM API
   ↓
Response
```

Suppose those operations take:

```
Create order       100 ms
Payment            600 ms
Email              800 ms
Analytics          300 ms
Warehouse           700 ms
CRM                 900 ms
-------------------------
Total             3,400 ms
```

Your customer waits **3.4 seconds**.

Worse, your endpoint now depends on five systems being healthy.

Suppose the CRM is temporarily down.

Should the customer really see:

```
http500 Internal Server Error
```

even though:

```
order created ✓
payment succeeded ✓
```

Probably not.

The fundamental problem is that we've coupled **business work** with **request latency**.

---

# Separate “must happen now” from “can happen later”

Ask this question for every operation:

> Does the user need the result of this operation before I can respond?

Payment authorization?

Probably yes.

Sending a confirmation email?

Probably no.

Analytics?

Definitely not.

CRM synchronization?

Probably not.

Warehouse processing?

Usually asynchronous.

A better architecture becomes:

```
HTTP Request
     │
     ▼
Create order
     │
     ▼
Charge payment
     │
     ▼
Publish jobs
     │
     ▼
Return response
```

Meanwhile:

```
                Queue
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
      Email   Warehouse   Analytics
      Worker    Worker      Worker
```

The customer might now wait:

```
Create order   100 ms
Payment        600 ms
Queue jobs      20 ms
---------------------
Total          720 ms
```

instead of 3.4 seconds.

---

# 🧠 What exactly is a queue?

At its simplest:

```
Producer
   │
   │ message
   ▼
QUEUE
   │
   │ message
   ▼
Consumer
```

The **producer** says:

```
"Please do this work."
```

The queue stores that request.

A **consumer/worker** processes it later.

For example:

```
Checkout API
     │
     ▼
"send confirmation for order 9281"
     │
     ▼
QUEUE
     │
     ▼
Email Worker
     │
     ▼
Email Provider
```

The API doesn't necessarily wait for the email.

It only needs confidence that the work has been handed off reliably.

---

# PHP example

In a Laravel-style application, instead of:

```php
$this->mailer->sendConfirmation($order);
```

you might dispatch:

```
PHPSendOrderConfirmation::dispatch($order->id);
```

The HTTP request ends.

Later, a worker executes something conceptually like:

```php
class SendOrderConfirmation
{
    public function __construct(
        public int $orderId
    ) {}

    public function handle(
        OrderRepository $orders,
        Mailer $mailer
    ): void {
        $order = $orders->find($this->orderId);

        $mailer->sendConfirmation($order);
    }
}
```

Notice a useful design choice:

```php
$order->id
```

was queued rather than blindly serializing an enormous application object graph.

The worker retrieves the state it needs when processing the job.

---

# TypeScript works exactly the same way

Conceptually:

```
TypeScriptawait orderQueue.add("send-confirmation", {
    orderId: order.id
});
```

Worker:

```
TypeScriptworker.process(
    "send-confirmation",
    async (job) => {
        const order =
            await orders.find(job.data.orderId);

        await mailer.sendConfirmation(order);
    }
);
```

The specific framework doesn't matter.

You could use:

```
Redis-backed queue
RabbitMQ
Amazon SQS
Google Pub/Sub
Kafka
```

The underlying architectural idea transfers.

---

# Why queues improve resilience

Suppose your email provider goes down for 15 minutes.

Without a queue:

```
API
 ↓
Email provider unavailable
 ↓
request fails
```

With a queue:

```
API
 ↓
Queue
 ↓
Worker
 ↓
Email provider unavailable
 ↓
retry later
```

The queue acts as a **buffer between systems operating at different speeds or reliability levels**.

That's one of its most important properties.

Think:

```
Producer speed ≠ Consumer speed
```

and that's okay.

---

# Queues also absorb traffic spikes

Imagine normal traffic:

```
100 orders/minute
```

Suddenly an influencer promotes your product:

```
10,000 orders/minute
```

Your email provider can safely handle only:

```
1,000 emails/minute
```

Without buffering:

```
10,000 requests
       ↓
Email system
       ↓
💥
```

With a queue:

```
10,000 jobs
       ↓
    QUEUE
       ↓
process 1,000/minute
       ↓
backlog gradually drains
```

Your queue absorbs the temporary mismatch.

This connects directly to **scalability**.

---

# But queues create a new concept: eventual consistency

Suppose checkout succeeds at:

```
10:00:00
```

The email worker processes the job at:

```
10:00:03
```

For three seconds:

```
Order exists ✓
Email sent   ✗
```

The system isn't instantly synchronized.

But eventually:

```
Order exists ✓
Email sent   ✓
```

That's a simple example of **eventual consistency**.

Distributed systems often accept temporary differences in state in exchange for better availability, scalability or decoupling.

You'll encounter this concept repeatedly.

---

# 🔁 Now revisit idempotency

Last lesson you learned:

> A timeout means “I don't know what happened,” not “nothing happened.”

Queues have exactly the same problem.

Suppose the worker receives:

```
Send email for order 9281
```

It executes:

```
send email ✓
```

Then the worker crashes **before acknowledging the message**.

The queue sees:

```
No acknowledgement.
Maybe processing failed.
```

So it sends the message again.

Another worker receives:

```
Send email for order 9281
```

Result:

```
Customer receives:

Order confirmed!
Order confirmed!
```

This is why many queue architectures provide **at-least-once delivery** semantics.

You should assume:

> **A message may be delivered more than once.**

Therefore consumers should often be **idempotent**.

---

# One simple strategy

Suppose jobs have unique IDs:

```
job_847291
```

Before performing the side effect:

```
PHPif ($processedJobs->exists($job->id)) {
    return;
}

$mailer->sendConfirmation($order);

$processedJobs->record($job->id);
```

Conceptually:

```
job_847291
    ↓
already processed?
    │
 ┌──┴──┐
YES    NO
 │      │
skip   process
```

But remember your previous lesson.

This implementation can still have a **race condition** if two workers process the same message simultaneously.

You may need:

```
unique constraints
transactions
atomic operations
locks
```

depending on the system.

Your lessons are starting to overlap intentionally.

---

# Another important concept: retries

Suppose the worker calls:

```
Email API
```

and receives:

```
http503 Service Unavailable
```

Retrying immediately 100 times is usually terrible.

Instead:

```
attempt 1
   ↓
fail

wait 1 second

attempt 2
   ↓
fail

wait 2 seconds

attempt 3
   ↓
fail

wait 4 seconds
```

This is **exponential backoff**.

Conceptually:

```
delay ≈ base × 2^attempt
```

Often some randomness called **jitter** is added so thousands of workers don't all retry simultaneously.

Without jitter:

```
10,000 workers fail
        ↓
wait 5 seconds
        ↓
10,000 workers retry together
        ↓
💥 downstream service again
```

With jitter:

```
worker A → 4.7s
worker B → 5.3s
worker C → 6.1s
...
```

The retry load spreads out.

This connects queues to **distributed-systems reliability**.

---

# ☠️ What happens when a job keeps failing?

Imagine:

```
SendOrderJob
```

fails:

```
attempt 1 ✗
attempt 2 ✗
attempt 3 ✗
attempt 4 ✗
attempt 5 ✗
```

Maybe the problem isn't temporary.

Perhaps the payload is invalid.

Retrying forever creates a **poison message**.

A common architecture moves repeatedly failing messages into a:

**Dead-Letter Queue (DLQ)**

```
Main Queue
    │
    ▼
 Worker
    │
    ▼
 failure
    │
    ▼
 retry
    │
    ▼
 failure
    │
    ▼
 retry limit reached
    │
    ▼
Dead-Letter Queue
```

Now engineers can inspect:

```
Why did this message fail?
Can it be fixed?
Should we replay it?
```

A DLQ prevents one broken job from cycling forever.

---

# ⚠️ Common misconception

A common assumption is:

> “Once I put something into a queue, it's guaranteed to happen exactly once.”

Usually, don't build your business logic around that assumption.

Real systems experience:

```
worker crashes
network failures
acknowledgement failures
timeouts
redelivery
duplicate messages
```

“Exactly once” is much harder than it sounds, especially when external side effects are involved.

A safer engineering mindset is:

```
Messages may repeat.
Workers may crash.
Networks may fail.

Design consumers accordingly.
```

That mindset is far more valuable than memorizing a particular queue product.

---

# 🤖 Why this matters for AI agents

Imagine an AI commerce agent decides:

```
Customer wants refund
```

The system might create:

```
RefundRequested
      │
      ▼
QUEUE
      │
      ├── Payment worker
      ├── Inventory worker
      ├── Notification worker
      └── Analytics worker
```

Or a long-running agent might perform:

```
Research task
     ↓
Queue browser work
     ↓
Queue analysis
     ↓
Wait for human approval
     ↓
Queue final action
```

Agents don't eliminate distributed-systems problems.

They often **increase the need for queues, retries, idempotency and durable state**, because autonomous workflows may run for minutes, hours or days.

---

# 🔗 Your growing mental map

You've now encountered:

```
Big-O
   ↓
How does work grow?

Hash tables
   ↓
How can lookup become fast?

Indexes
   ↓
How can databases locate rows efficiently?

Dependency Injection
   ↓
How do we separate capabilities
from implementations?

Idempotency
   ↓
How do we safely repeat operations?

Queues
   ↓
How do we move work outside
the synchronous request path?
```

And queues introduced three concepts we'll revisit later:

```
Eventual consistency
Retry/backoff
Dead-letter queues
```

These lead naturally toward:

```
event-driven architecture
message brokers
Kafka
distributed systems
observability
scalability
```

## 🧩 Quick quiz

**1. Why might sending an email directly inside checkout be undesirable?**

Because checkout latency and reliability now depend on the email provider, even though sending the email doesn't need to complete before checkout returns.

**2. A worker successfully processes a job but crashes before acknowledging it. What might happen?**

The queue may redeliver the message. The consumer therefore needs to tolerate duplicate execution.

**3. Why use exponential backoff instead of continuously retrying?**

It gives the failing downstream service time to recover and reduces the chance of overwhelming it with retries.

---

# 🛠️ Today's practical challenge

Imagine this endpoint:

```
PHPpublic function createOrder(Request $request)
{
    $order = $this->orders->create(
        $request->validated()
    );

    $this->payment->charge($order);

    $this->inventory->reserve($order);

    $this->mailer->sendConfirmation($order);

    $this->analytics->track($order);

    $this->crm->sync($order);

    return $order;
}
```

Your challenge is to classify each operation into:

```
Must happen before HTTP response

or

Can happen asynchronously
```

Then design:

```
HTTP Request
     │
     ▼
synchronous work
     │
     ▼
Queue(s)
     │
     ▼
workers
```

For each queued operation, answer three questions:

```
What happens if this runs twice?

What failures should be retried?

What happens after all retries fail?
```

Those three questions alone will dramatically improve many background-job designs.

For deeper reading, Laravel's official [Queue documentation](https://laravel.com/docs/queues?utm_source=chatgpt.com) is immediately useful for PHP work, while [Amazon SQS's explanation of standard queue delivery behavior](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html?utm_source=chatgpt.com) is a good short explanation of why duplicate delivery happens.

### 🧠 Keep one mental model today

> **A queue lets the producer say “this work needs to happen” without requiring “this work must happen right now, inside my request.”**

And when you introduce a queue, immediately remember yesterday's lesson:

> **Assume the work might happen more than once.**
