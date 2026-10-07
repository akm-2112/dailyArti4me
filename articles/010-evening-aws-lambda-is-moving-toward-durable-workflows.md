---
title: "🌙 Evening Tech Discovery — AWS Lambda is moving toward durable workflows"
order: 10
volume: 2
category: evening
source: Evening Tech Discovery
---

# 🌙 Evening Tech Discovery — AWS Lambda is moving toward durable workflows

Tonight’s topic is a useful bridge between ordinary backend code and distributed systems:

> **What if a serverless function could sleep for hours or days, survive restarts, and then continue from exactly where it stopped?**

On **August 3, 2026**, [AWS](https://aws.amazon.com/?utm_source=chatgpt.com) announced that its **Durable Execution SDK for .NET** reached general availability, joining existing **TypeScript, Python, and Java** support. [Amazon Web Services, Inc.](https://aws.amazon.com/blogs/developer/?utm_source=chatgpt.com)

The TypeScript support makes this especially relevant to your stack.

## First, the normal Lambda model

A typical AWS Lambda function is intentionally short-lived.

For example:

```
TypeScriptexport const handler = async (event) => {
    const order = await createOrder(event);

    await sendConfirmation(order);

    return order;
};
```

Conceptually:

```
request
   ↓
Lambda starts
   ↓
execute code
   ↓
return response
   ↓
Lambda finishes
```

That's excellent for:

```
HTTP handlers
image processing
webhooks
queue consumers
scheduled jobs
```

But imagine an e-commerce workflow:

```
Create order
     ↓
Wait for payment
     ↓
Reserve inventory
     ↓
Wait 24 hours
     ↓
Check shipping
     ↓
Send notification
```

A normal function can't simply do:

```
TypeScriptawait sleep(24 * 60 * 60 * 1000);
```

and sit there for 24 hours.

You'd waste compute resources, and ordinary Lambda execution has runtime limits anyway.

Traditionally, you solve this with orchestration infrastructure such as:

```
Lambda
Step Functions
SQS
EventBridge
database state
```

Your application becomes a state machine distributed across multiple components.

---

# Durable execution changes the programming model

With a durable execution system, you write code that looks much more like an ordinary sequential program:

```typescript
const order = await createOrder();

await waitForPayment(order.id);

await reserveInventory(order);

await sleep("24 hours");

await checkShipping(order);

await notifyCustomer(order);
```

But the runtime doesn't literally keep a process alive for 24 hours.

Instead it can conceptually do:

```
execute
   ↓
checkpoint
   ↓
stop compute
   ↓
──────── 24 hours ────────
   ↓
resume
   ↓
continue
```

That's the interesting part.

**Your code looks sequential while the infrastructure behaves asynchronously.**

---

# 🧠 Why is this difficult?

Suppose execution reaches:

```
TypeScriptawait chargeCustomer(order);
```

and successfully charges:

```
$149
```

Then the machine crashes immediately afterward.

When execution resumes, what happens?

Naïvely:

```
resume
   ↓
run chargeCustomer()
   ↓
charge another $149
```

You've just created the same problem we covered in your recent **idempotency lesson**.

Durable execution engines therefore need to distinguish between:

```
deterministic workflow logic
```

and:

```
external side effects
```

The runtime records enough history/checkpoints to reconstruct where execution was and avoid blindly replaying completed effects.

This idea appears in several durable workflow systems, not just AWS.

---

# Replay is the clever trick

Imagine this program:

```typescript
const order = await createOrder();

await sleep("1 day");

await ship(order);
```

The runtime may record something conceptually like:

```
Workflow history

1. createOrder started
2. createOrder completed → order #9281
3. timer created
4. timer completed
5. ship started
6. ship completed
```

If the worker disappears after step 4, another worker can reconstruct:

```
createOrder → already happened
timer       → already happened
```

and continue around:

```
ship(order)
```

instead of starting the business process from scratch.

Think of the workflow history almost like a **journal of execution**.

---

# This introduces deterministic code

Now suppose your workflow contains:

```typescript
const discount = Math.random();
```

During the original execution:

```
Math.random() → 0.73
```

During replay:

```
Math.random() → 0.18
```

Suddenly the reconstructed execution doesn't match its history.

Likewise:

```typescript
const now = new Date();
```

might return different values.

Durable workflow frameworks therefore usually provide controlled mechanisms for things like:

```
time
randomness
external API calls
side effects
```

because replay needs predictable behavior.

This leads to a surprisingly deep principle:

> **If execution can be replayed, the orchestration portion of your program needs deterministic behavior.**

That's closely related to ideas from functional programming that we'll study later.

---

# Why should a backend developer care?

Consider how you might currently implement order fulfillment.

You could create:

```
orders table

order_jobs table

payment_webhooks

shipping_jobs

retry counters

cron job

queue workers
```

and manually coordinate everything.

Something like:

```
PHPif ($order->payment_status === 'paid'
    && $order->inventory_reserved === false) {

    ReserveInventoryJob::dispatch($order);
}
```

Then another job:

```
PHPif ($order->inventory_reserved
    && !$order->shipping_created) {

    CreateShipmentJob::dispatch($order);
}
```

Eventually you have an implicit state machine scattered across:

```
controllers
jobs
database columns
webhooks
cron tasks
queues
```

This works—and sometimes is exactly the right design.

But reasoning about the entire workflow becomes difficult.

Durable execution lets you express the orchestration more naturally:

```
Order Workflow

create
   ↓
payment
   ↓
inventory
   ↓
shipping
   ↓
notification
```

while the runtime manages much of:

```
checkpointing
waiting
resuming
retries
workflow state
```

---

# 🤖 There's also an AI-agent connection

Long-running agents have exactly this problem.

Imagine:

```
Research competitor
       ↓
browse 50 sites
       ↓
wait for approval
       ↓
generate report
       ↓
send report
```

The agent might run for:

```
minutes
hours
days
```

You don't want a single Node.js process holding everything in RAM.

If it crashes:

```
48 minutes of agent work
        ↓
       gone
```

A durable architecture instead persists progress:

```
Agent
  ↓
step
  ↓
checkpoint
  ↓
tool call
  ↓
checkpoint
  ↓
human approval
  ↓
sleep
  ↓
resume tomorrow
```

This is one reason **durable execution is becoming important infrastructure for agents**.

Agent frameworks often talk about memory, tools, and reasoning, but production agents also need boring distributed-systems capabilities:

```
retries
timeouts
state persistence
idempotency
recovery
observability
```

---

# 🔗 Notice how your recent lessons connect

You learned:

```
Idempotency
    ↓
"Can this operation safely run twice?"
```

Now add:

```
Durable execution
    ↓
"Can this workflow survive interruption?"
```

Soon we'll add:

```
Queues
    ↓
"How do components communicate asynchronously?"

Concurrency
    ↓
"What happens when operations overlap?"

Transactions
    ↓
"Which state changes must succeed together?"
```

These concepts eventually combine into **distributed-systems engineering**.

---

# ⚠️ Don't conclude that durable workflows replace queues

They solve related but different problems.

A queue is excellent for:

```
"Someone needs to process this job."
```

A durable workflow is excellent for:

```
"This business process consists of
multiple steps over time."
```

For example:

```
Order workflow
      │
      ├── charge payment
      ├── queue inventory job
      ├── wait for result
      ├── wait 24h
      └── queue shipping job
```

You can—and often should—use both.

The workflow coordinates.

Queues distribute work.

---

# A practical mental model

Think about a normal function:

```
function
   ↓
state lives mostly in RAM
   ↓
process dies
   ↓
state disappears
```

A durable function:

```
function
   ↓
execution history/state persisted
   ↓
process can disappear
   ↓
another process reconstructs state
   ↓
execution continues
```

That changes what a "function call" can represent.

Instead of milliseconds or seconds, a logical function could represent a business process lasting **days**.

## Why this is worth watching

AWS isn't alone in moving this direction. Durable execution is becoming a broader programming model because modern applications increasingly contain long-running asynchronous workflows—payments, approvals, data pipelines, infrastructure automation and AI agents.

AWS has also recently been pushing workflow tooling further: on **August 19**, it added tooling to help coding agents build Step Functions workflows, another signal that orchestration is becoming more integrated into ordinary developer workflows. [Amazon Web Services, Inc.](https://aws.amazon.com/blogs/compute//?utm_source=chatgpt.com)

And AWS is experimenting with the runtime layer itself: on **August 15**, Lambda introduced preview runtimes beginning with **Node.js 26 and Python 3.15**, allowing developers to test upcoming language runtimes before GA. [Amazon Web Services, Inc.](https://aws.amazon.com/blogs/compute//?utm_source=chatgpt.com)

### One idea worth remembering tonight

> **Durability means the lifetime of your business process no longer has to equal the lifetime of the machine executing it.**

That's a fundamental distributed-systems idea.

A process can disappear.

A container can restart.

A server can fail.

But:

```
business workflow
       ↓
     survives
```

For deeper reading, start with the [AWS Developer Tools Blog](https://aws.amazon.com/blogs/developer/?utm_source=chatgpt.com) covering the Durable Execution SDK and recent developer releases. The [AWS Compute Blog](https://aws.amazon.com/blogs/compute//?utm_source=chatgpt.com) is also worth bookmarking; it currently covers the new Lambda runtime previews and workflow tooling.

Tomorrow, when you encounter concepts like **queues, event-driven architecture, retries, transactions, and concurrency**, keep this execution-history idea in the back of your mind—they're all pieces of the same larger problem: **making software reliable even when individual machines and requests are unreliable.**
