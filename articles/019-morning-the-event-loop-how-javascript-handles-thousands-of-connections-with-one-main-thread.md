---
title: "🌅 Morning Tech Lesson #10 — The Event Loop: How JavaScript Handles Thousands of Connections With One Main Thread"
order: 19
volume: 4
category: morning
source: Evening Tech Discovery
---

# 🌅 Morning Tech Lesson #10 — The Event Loop: How JavaScript Handles Thousands of Connections With One Main Thread

We’ve recently spent several lessons in distributed/backend territory—**queues, race conditions, caching, transactions**. Today we’ll change layers and go downward into **runtime + operating-system concepts**, while revisiting concurrency from a different angle.

The central question:

> **If JavaScript runs one piece of code at a time, how can Node.js handle thousands of network connections concurrently?**

Estimated time: **10–15 minutes**.

## Start with a confusing example

Consider this TypeScript:

```
TypeScriptconsole.log("A");

setTimeout(() => {
    console.log("B");
}, 0);

console.log("C");
```

What prints?

Not:

```
A
B
C
```

but:

```
A
C
B
```

Even though the timeout was:

```
TypeScript0
```

Why?

Because `setTimeout()` does **not** mean:

> Execute this function in zero milliseconds.

It means roughly:

> After at least this delay, make this callback eligible to run when the JavaScript runtime can execute it.

That distinction leads us to the **event loop**.

---

# 🧠 First understand the call stack

JavaScript executes synchronous functions using a call stack.

Consider:

```typescript
function calculateTotal() {
    return addTax(100);
}

function addTax(price: number) {
    return price * 1.1;
}

calculateTotal();
```

Conceptually:

```
Call Stack

calculateTotal()
       │
       ▼
   addTax()
```

When `addTax()` finishes:

```
addTax() removed
```

Then:

```
calculateTotal() removed
```

JavaScript executes one stack frame at a time on its main execution thread.

So if you write:

```
TypeScriptwhile (true) {
}
```

you've created a serious problem.

The call stack never becomes available.

---

# Now imagine a web server

```
TypeScriptapp.get("/users/:id", async (req, res) => {
    const user = await database.findUser(
        req.params.id
    );

    res.json(user);
});
```

Suppose the database takes:

```
100 ms
```

A common beginner mental model is:

```
JavaScript thread

query database
      │
      ▼
wait 100 ms doing nothing
      │
      ▼
continue
```

If that were how Node worked, one slow database request could block every other request.

But that's not what happens for asynchronous I/O.

A better simplified model is:

```javascript
    │
    │ start DB/network operation
    ▼
Operating system / runtime
    │
    │ waiting...
    │
JavaScript continues doing other work
```

When the I/O completes:

```
OS/runtime
    │
    ▼
completion becomes ready
    │
    ▼
event loop
    │
    ▼
JavaScript continues callback/promise
```

The important idea is:

> **JavaScript does not need to occupy the CPU while waiting for the network.**

---

# CPU work and I/O work are very different

Suppose your application performs:

```
Database query      100 ms
```

During most of those 100 ms, your CPU isn't calculating the query result.

The database server is doing work somewhere else.

Likewise:

```
HTTP request        300 ms
Redis request         2 ms
file read            20 ms
```

Much of that time means:

```
waiting
```

not:

```
executing JavaScript instructions
```

Node's architecture is particularly effective for applications containing lots of:

```
network I/O
database I/O
WebSockets
API calls
filesystem I/O
```

because it can keep many operations **in flight** without dedicating one JavaScript thread to each waiting operation.

---

# Imagine 10,000 connections

A traditional simplified thread-per-connection mental model looks like:

```
Connection 1 → Thread 1
Connection 2 → Thread 2
Connection 3 → Thread 3
...
```

Threads aren't free.

Each involves resources such as:

```
stack memory
scheduler bookkeeping
context switching
```

An event-driven architecture instead looks conceptually like:

```
        10,000 connections
               │
               ▼
          OS networking
               │
               ▼
           Event Loop
               │
               ▼
       JavaScript callbacks
```

Most connections spend much of their lifetime doing:

```
waiting
```

so the runtime doesn't need 10,000 simultaneously executing JavaScript threads.

This is one reason event-driven systems can handle large numbers of mostly-I/O-bound connections efficiently.

---

# What does the operating system contribute?

This is an important detail developers sometimes miss.

Node isn't magically watching every network socket using JavaScript.

The operating system provides efficient mechanisms for notifying programs when I/O becomes ready.

On Linux you'll frequently encounter:

```
epoll
```

macOS/BSD systems have:

```
kqueue
```

Windows has mechanisms such as:

```
IOCP
```

Node uses libuv to abstract these OS differences.

Very simplified:

```
Your TypeScript
      │
      ▼
    Node.js
      │
      ▼
    libuv
      │
      ▼
Operating System
      │
      ▼
network / filesystem
```

So when you write:

```
TypeScriptawait fetch(url);
```

you're standing on top of several layers of runtime and operating-system machinery.

---

# What does `await` actually do?

Consider:

```
TypeScriptconsole.log("start");

const user = await getUser();

console.log(user);

console.log("end");
```

`await` does **not** mean:

```
freeze the entire Node process
until getUser() completes
```

It pauses continuation of that **async function**.

Other work can run.

Conceptually:

```
getUser()
   │
   ▼
Promise pending

current async function pauses
   │
   ▼
event loop handles other work

...

Promise resolves
   │
   ▼
function continuation scheduled
   │
   ▼
console.log(user)
```

That's why:

```
TypeScriptawait database.query(...)
```

is normally fine.

But there's an important exception.

---

# 🔥 CPU-bound JavaScript can block everything

Imagine:

```
TypeScriptapp.get("/report", (req, res) => {
    let result = 0;

    for (let i = 0; i < 10_000_000_000; i++) {
        result += i;
    }

    res.json({ result });
});
```

Unlike network I/O, this loop needs the JavaScript CPU continuously.

The event loop cannot simply say:

```
I'll come back when the CPU finishes.
```

JavaScript **is the thing doing the CPU work**.

While that loop executes:

```
/report
   │
   ▼
CPU loop
   │
   │
   │
   │
```

other callbacks cannot use the same main JS thread.

Requests can pile up:

```
request A ──────┐
request B ──────┤ WAIT
request C ──────┤ WAIT
WebSocket ──────┤ WAIT
timer ──────────┘ WAIT
```

This is called **blocking the event loop**.

---

# Practical example: image processing

Suppose your API receives:

```
POST /avatar
```

and performs heavy image manipulation synchronously.

If the processing consumes:

```
800 ms CPU
```

your server may become unresponsive during that period.

A better architecture might use:

```
HTTP API
   │
   ▼
Queue
   │
   ▼
Image Worker
```

Remember your queue lesson?

Now you have another reason for queues.

Previously:

```
Queues move non-urgent work
outside the HTTP request.
```

Today add:

```
Queues can also move expensive CPU work
away from latency-sensitive processes.
```

Alternatively, Node supports mechanisms such as worker threads for CPU-intensive JavaScript.

Different tool, same principle:

> **Don't monopolize the event loop with expensive CPU work.**

---

# 🟨 PHP gives you an interesting comparison

Traditional PHP-FPM architectures often work differently.

Simplified:

```
Request 1 → PHP worker 1

Request 2 → PHP worker 2

Request 3 → PHP worker 3
```

If worker 1 does:

```php
$response = Http::get($slowApi);
```

and waits 2 seconds, that worker may remain occupied.

But other workers can still handle other requests.

Node often approaches concurrency more like:

```
many requests
     │
     ▼
event-driven process
```

Neither model is universally superior.

They have different:

```
runtime models
resource trade-offs
failure characteristics
programming styles
```

This is useful because knowing both PHP and JavaScript exposes you to different concurrency models.

Frameworks and newer PHP runtimes can also introduce asynchronous/event-driven approaches, so don't equate:

```
PHP = always synchronous
```

or:

```
JavaScript = always asynchronous
```

with language fundamentals.

Architecture and runtime matter.

---

# Promise concurrency: sequential vs parallel waiting

Consider:

```typescript
const user = await getUser();

const orders = await getOrders();

const recommendations =
    await getRecommendations();
```

Suppose each operation takes:

```
getUser()             100 ms
getOrders()           200 ms
getRecommendations()  300 ms
```

Sequentially:

```
100 + 200 + 300
≈ 600 ms
```

But if these operations are independent:

```typescript
const [
    user,
    orders,
    recommendations
] = await Promise.all([
    getUser(),
    getOrders(),
    getRecommendations()
]);
```

they can overlap:

```
User            ├────100────┤

Orders          ├────────200────────┤

Recommendations ├────────────300────────────┤
```

Total becomes closer to:

```
max(100, 200, 300)

≈ 300 ms
```

rather than:

```
sum(...)
```

This is a very practical performance technique.

But don't blindly `Promise.all()` everything.

---

# 🚨 Concurrency needs limits

Imagine:

```
TypeScriptawait Promise.all(
    users.map(user =>
        sendEmail(user)
    )
);
```

and:

```
users.length = 1,000,000
```

You've potentially attempted to create an enormous amount of concurrent work.

That can overwhelm:

```
memory
network connections
database pools
external APIs
rate limits
```

This connects back to queues and backpressure.

Concurrency is useful.

**Unbounded concurrency is dangerous.**

Real systems often use bounded concurrency:

```
1,000,000 jobs
       │
       ▼
Queue
       │
       ▼
50 workers
```

or a concurrency limiter:

```
process at most 20
simultaneously
```

Now your system controls resource pressure.

---

# 🔄 Microtasks: one level deeper

Let's increase the difficulty slightly.

What prints?

```
TypeScriptconsole.log("A");

setTimeout(() => {
    console.log("B");
}, 0);

Promise.resolve().then(() => {
    console.log("C");
});

console.log("D");
```

Answer:

```
A
D
C
B
```

Why?

Promise continuations are processed through the **microtask queue**, while timers are handled through later event-loop phases/tasks.

A simplified mental model:

```
synchronous code
      ↓
microtasks
      ↓
next event-loop work/timers
```

So:

```
A
D
```

comes from synchronous execution.

Then:

```
C
```

from the Promise continuation.

Then:

```
B
```

from the timer.

The actual Node event loop has more phases and details, but this mental model is sufficient for most application reasoning.

---

# ⚠️ Common misconception

### “Node.js is single-threaded.”

That's incomplete.

Your **JavaScript execution** typically has one main event-loop thread.

But the Node runtime itself may use additional threads and operating-system facilities.

For some operations, libuv has a worker pool.

Node can also use:

```
worker_threads
child processes
multiple Node processes
```

And your application talks to systems running on completely different machines:

```
PostgreSQL
Redis
S3
external APIs
```

A better statement is:

> **Node normally executes your JavaScript callbacks on one main event-loop thread, while asynchronous work may be handled elsewhere.**

That's much more accurate.

---

# 🤖 Why this matters for AI applications

Imagine an agent backend:

```
User
 ↓
Agent API
 ↓
LLM
 ↓
tool call
 ↓
database
 ↓
another API
 ↓
LLM
```

Most of that workflow is:

```
network waiting
```

which fits asynchronous I/O extremely well.

A TypeScript agent server can have many workflows in flight:

```
Agent A → waiting for LLM

Agent B → waiting for database

Agent C → waiting for browser

Agent D → waiting for MCP tool
```

while the event loop keeps coordinating completions.

But suppose your agent performs:

```
huge local embedding computation
video processing
large compression
CPU-heavy parsing
cryptographic computation
```

directly on the event loop.

Now:

```
all other agents
      ↓
WAIT
```

The same runtime principles still apply even though the application is “AI-powered.”

---

# 🔗 Connect today's lesson to your earlier ones

Your mental map now looks like:

```
Race conditions
      ↓
Operations can overlap


Event loop
      ↓
How one runtime coordinates
many overlapping I/O operations


Queues
      ↓
How systems control and distribute
asynchronous work


Caching
      ↓
Reduce repeated expensive work


Transactions
      ↓
Protect shared state while
concurrent work happens
```

Notice the distinction:

```
Concurrency
≠
Parallelism
```

**Concurrency**:

```
multiple tasks are in progress
```

**Parallelism**:

```
multiple tasks are literally executing
at the same instant
```

The event loop gives you enormous amounts of concurrency even though your JavaScript may mostly execute on one main thread.

That's a concept worth keeping.

## 🧩 Quick quiz

**1. Why doesn't this block Node for two seconds?**

```
TypeScriptawait fetch("https://api.example.com");
```

Because the JavaScript thread doesn't need to actively execute while waiting for network I/O. The async function pauses and other event-loop work can proceed.

**2. Why can this block other requests?**

```
TypeScriptfor (let i = 0; i < 10_000_000_000; i++) {
    calculate(i);
}
```

Because it's CPU-bound JavaScript occupying the main execution thread.

**3. What's the difference between these?**

```typescript
const a = await getA();
const b = await getB();
```

versus:

```typescript
const [a, b] = await Promise.all([
    getA(),
    getB()
]);
```

The first waits sequentially. The second allows independent asynchronous operations to overlap, potentially reducing total latency.

---

# 🛠️ Today's practical challenge

You're building:

```
GET /dashboard
```

It needs:

```
user profile      80 ms
recent orders    180 ms
notifications    120 ms
recommendations  350 ms
```

Your current code:

```typescript
const user =
    await getUser(userId);

const orders =
    await getOrders(userId);

const notifications =
    await getNotifications(userId);

const recommendations =
    await getRecommendations(userId);
```

First, estimate the approximate latency if everything runs sequentially.

Then redesign it using concurrent asynchronous I/O where dependencies allow.

But add one complication:

```
recommendations requires
the user's preferred category
from getUser()
```

So you **cannot** simply throw all four operations into one `Promise.all()`.

Think in dependency groups:

```
             getUser
                │
          ┌─────┼─────────────┐
          │     │             ▼
          │     │     recommendations
          │     │
       orders notifications
```

Your job is to find the shortest valid execution path without creating unnecessary concurrency.

For deeper reading, the official [Node.js event-loop guide](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick?utm_source=chatgpt.com) explains the event-loop phases. The [Node.js guide on not blocking the event loop](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop?utm_source=chatgpt.com) is especially practical for backend developers, and [libuv's design overview](https://docs.libuv.org/en/v1.x/design.html?utm_source=chatgpt.com) is worth reading when you want to see the layer underneath Node.

### 🧠 Keep one mental model today

> **Asynchronous I/O lets your program do useful work while the outside world is making it wait.**

And whenever you're writing Node.js, ask:

```
Am I waiting for something?

or

Am I making the CPU work?
```

If you're **waiting**, the event loop is your friend.

If you're doing heavy **CPU work**, remember that everyone else sharing that event loop may be waiting for *you*.
