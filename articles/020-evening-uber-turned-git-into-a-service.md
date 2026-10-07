---
title: "🌙 Evening Tech Discovery — Uber turned Git into a service"
order: 20
volume: 4
category: evening
source: Evening Tech Discovery
---

# 🌙 Evening Tech Discovery — Uber turned Git into a service

Tonight’s topic is a great example of a familiar developer tool becoming a **distributed-systems problem at scale**.

In July, [Uber Engineering](https://www.uber.com/by/en/blog/gitfarm-as-a-service/?utm_source=chatgpt.com) published **GitFarm**, an internal platform that provides **Git operations as a service through gRPC**. Uber’s automation systems execute Git commands millions of times per day, and repeatedly cloning enormous monorepos had become expensive enough that they changed the architecture entirely. [Uber](https://www.uber.com/by/en/blog/gitfarm-as-a-service/?utm_source=chatgpt.com)

The interesting lesson isn't that you should build your own Git service. It's this:

> **When setup cost becomes larger than the actual work, move the expensive setup into shared, reusable infrastructure.**

## The problem: sometimes `git clone` is the expensive operation

Imagine a CI-related service needs to answer:

```
Who owns this file?
```

It might do something like:

```
Bashgit clone repo
cd repo
git checkout abc123
cat path/to/CODEOWNERS
```

For your Laravel project, this might take a few seconds.

Uber's Go monorepo is a different story. According to Uber, cloning it takes roughly:

```
~15 minutes
~6 CPU cores
~32 GB RAM
>40 GB disk
```

And a service maintaining all of Uber's major monorepos could need around:

```
16 CPU cores
64 GB RAM
96 GB storage
```

just to maintain repository state. [Uber](https://www.uber.com/by/en/blog/gitfarm-as-a-service/?utm_source=chatgpt.com)

Now multiply that across:

```
CI services
code ownership tools
merge queues
compliance scanners
code search
release automation
developer tools
AI coding agents
```

Each service doing:

```
clone
fetch
checkout
maintain repository
```

creates enormous duplicated work.

The architecture effectively becomes:

```
Service A ── clone ──┐
Service B ── clone ──┤
Service C ── clone ──┼── Git server
Service D ── clone ──┤
Service E ── clone ──┘
```

Every client independently downloads and maintains similar repository state.

At enough scale, the Git server itself becomes a bottleneck because it repeatedly has to enumerate objects, generate packfiles, and send essentially the same repository information to many clients. [Uber](https://www.uber.com/by/en/blog/gitfarm-as-a-service/?utm_source=chatgpt.com)

---

# Why not just use shallow clones?

You might immediately think:

```
Bashgit clone --depth=1
```

That's exactly the kind of optimization you'd normally try.

It reduces:

```
network transfer
disk usage
clone time
```

but doesn't eliminate the architectural duplication.

And some Git operations require history.

For example:

```
Bashgit merge-base feature main
```

asks:

> Where did these histories diverge?

A shallow clone might not contain enough history to answer correctly.

Likewise operations such as:

```
Bashgit bisect
```

fundamentally depend on historical commits.

So there's a difference between:

```
optimizing repeated work
```

and:

```
eliminating repeated work
```

Uber chose the second.

---

# Enter GitFarm

Instead of every application owning a repository:

```
Service A → repository

Service B → repository

Service C → repository
```

Uber built something conceptually like:

```
              Services
          /      |      \
         /       |       \
        ▼        ▼        ▼
              GitFarm
                 │
                 ▼
        warm repository state
                 │
                 ▼
           Git upstream
```

A client can effectively request:

```
Run:

git merge-base feature main
```

without cloning the repository locally.

GitFarm is **not a Git hosting service**. It doesn't replace the upstream repository or become another GitHub.

Instead, Uber describes it as essentially a:

> centralized Git client in the cloud. [Uber](https://www.uber.com/by/en/blog/gitfarm-as-a-service/?utm_source=chatgpt.com)

That's a useful distinction.

---

# 🧠 Think of Git as compute instead of storage

Normally we think:

```
Git
 ↓
repository on disk
```

But many automated systems don't actually care about owning the repository.

They care about an operation:

```
give me file X at commit Y

compute merge-base

show commit history

create ref

perform cherry-pick

push ref
```

So Uber changed the abstraction from:

```
"Here is a repository.
Do whatever you need."
```

to something closer to:

```
"Tell me which Git operation
you need performed."
```

That is an important software-design transformation.

You're replacing:

```
shared implementation responsibility
```

with:

```
shared capability
```

---

# The architecture

A simplified GitFarm request looks like:

```
Client
  │
  │ gRPC
  ▼
Gateway
  │
  ▼
Backend
  │
  ▼
Ephemeral Sandbox
  │
  ▼
pre-warmed Git checkout
```

The **gateway** handles authentication, authorization and routing. It tracks which backend nodes have available repository sandboxes and sends work accordingly. [Uber](https://www.uber.com/us/en/blog/gitfarm-as-a-service/)

The backend maintains warm repository state:

```
Git upstream
      │
      │ fetch
      ▼
Bare/warm repository
      │
      ├── Sandbox A
      ├── Sandbox B
      └── Sandbox C
```

Instead of:

```
request arrives
      ↓
clone 40 GB
      ↓
run command
```

you get:

```
request arrives
      ↓
acquire warm sandbox
      ↓
run command
```

Uber says providing a sandbox with a ready repository takes **less than one second**, and clients can access a full checkout in under roughly **500 ms** in their setup. [Uber+1](https://www.uber.com/by/en/blog/gitfarm-as-a-service/?utm_source=chatgpt.com)

That's essentially **precomputation + pooling**.

You've seen the same principle elsewhere:

```
Database connection pool

instead of:
connect → authenticate → query → disconnect


Worker pool

instead of:
start process → work → kill process


Warm serverless instance

instead of:
boot runtime → load app → request


GitFarm

instead of:
clone repository → Git operation
```

Different systems.

Same optimization.

---

# 🔗 This connects directly to caching

You recently learned:

> Don't repeat expensive work when the result or state can safely be reused.

GitFarm applies that principle at infrastructure scale.

Instead of:

```
Service A → clone repository
Service B → clone repository
Service C → clone repository
```

you maintain shared warm repository state:

```
                warm state
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
       request   request   request
```

But here's the interesting twist.

A cache raises the question:

```
How stale can this be?
```

GitFarm has exactly the same problem.

---

# Repository freshness becomes a consistency problem

Suppose GitFarm has:

```
main = commit A
```

Then someone pushes:

```
main = commit B
```

The GitFarm backend hasn't fetched yet.

A client asks:

```
Bashgit log main
```

What should it see?

```
A?

or

B?
```

Uber doesn't enforce one universal freshness guarantee.

Instead, repository state is **eventually consistent** with upstream. Clients that need guaranteed freshness can explicitly request a `git fetch` before performing their operations. [Uber](https://www.uber.com/us/en/blog/gitfarm-as-a-service/)

That's a great system-design tradeoff.

Some operations might tolerate:

```
repository state from a few seconds ago
```

while something like a merge queue may require:

```
latest main
```

So:

```
fast path
   ↓
use warm repository


strict freshness path
   ↓
fetch first
   ↓
execute
```

Remember your caching lesson:

```
performance
      ↕
freshness
```

The same trade-off appears again.

---

# Why sandboxes matter

Running arbitrary Git operations on shared repository state sounds dangerous.

Imagine two clients simultaneously executing:

```
Bashgit checkout branch-a
```

and:

```
Bashgit checkout branch-b
```

against the same working tree.

Chaos.

Instead, GitFarm runs requests inside **ephemeral isolated sandboxes**, each with its own checkout and permissions scoped to the caller. [Uber](https://www.uber.com/us/en/blog/gitfarm-as-a-service/)

Conceptually:

```
Shared repository data
         │
    ┌────┴────┐
    ▼         ▼
Sandbox A   Sandbox B
branch A    branch B
```

This separates:

```
expensive shared state
```

from:

```
mutable request state
```

That's a very useful architecture pattern.

You can share the expensive immutable-ish foundation while isolating mutations.

Container systems do similar things with layered filesystems.

---

# Multi-command workflows create another interesting problem

Suppose you need:

```
BashBASE=$(git merge-base feature main)

git push origin "$BASE:refs/something"
```

The second command depends on the first.

You don't want:

```
Request 1
    ↓
Sandbox A
    ↓
merge-base


Request 2
    ↓
Sandbox B
    ↓
push
```

because the repository could change between operations.

Instead, GitFarm supports a persistent execution session using **bidirectional gRPC streaming**:

```
Client
   │
   ▼
Sandbox
   │
   ├── command 1
   │
   ├── command 2
   │
   └── command 3
```

All commands observe the same checkout.

Uber gives each session guarantees around isolation, consistent repository state, and command ordering. [Uber](https://www.uber.com/us/en/blog/gitfarm-as-a-service/)

Notice the conceptual similarity to a database transaction:

```
BEGIN

operation A
operation B
operation C

COMMIT
```

GitFarm isn't giving you an ACID database transaction, but the architectural motivation is similar:

> **Related operations need a stable execution context.**

Yesterday's transaction lesson suddenly has a cousin in developer infrastructure.

---

# The results are impressive

One Uber service scans `CODEOWNERS` files across hundreds of thousands of directories.

Before GitFarm, six hosts maintained complete local copies of every monorepo.

After removing those local checkouts, Uber reports:

```
CPU

>70 cores
   ↓
16 cores

≈77% reduction
```

and:

```
Memory

400–600 GB
    ↓
32 GB

>90% reduction
```

Startup time dropped from roughly:

```
15–20 minutes
```

to:

```
under 1 minute
```

[Uber](https://www.uber.com/by/en/blog/gitfarm-as-a-service/?utm_source=chatgpt.com)

Another compliance system processes **10,000–20,000 events per hour across about 9,000 repositories**.

Previously, running its Git work through CI produced p50 latency around:

```
110–160 seconds
```

With GitFarm:

```
20–30 seconds
```

an improvement of more than 80%, largely by removing CI scheduling, workspace setup and repeated repository synchronization. [Uber](https://www.uber.com/us/en/blog/gitfarm-as-a-service/)

---

# Why should *you* care if you're not Uber?

You probably shouldn't build GitFarm.

But you absolutely should learn the pattern.

Imagine one of your systems eventually has:

```
20 AI coding agents

10 CI workers

security scanner

deployment service

code analysis service
```

and every component does:

```
Bashgit clone ...
```

Your first instinct might be:

```
make cloning faster
```

But at sufficient scale, the better question becomes:

```
Why does every component need
its own copy of the repository?
```

That's a recurring senior-engineering question:

> **Is this expensive resource really owned by every consumer, or should it become shared infrastructure?**

---

# 🤖 This becomes especially interesting for coding agents

Imagine 100 AI agents simultaneously working against one large repository:

```
Agent 1 → clone
Agent 2 → clone
Agent 3 → clone
...
Agent 100 → clone
```

Each agent may only need:

```
inspect files
search history
compute diff
checkout commit
create patch
```

Repeatedly cloning a giant monorepo for ephemeral agents could become enormously wasteful.

An architecture inspired by GitFarm might instead look like:

```
             Coding Agents
          /       |        \
         ▼        ▼         ▼
           Repository Service
                  │
          ┌───────┴───────┐
          ▼               ▼
      warm repo       sandbox pool
```

Agents become **stateless consumers of repository capabilities**.

That fits a larger trend we've seen in recent evening reads:

```
AI agent
   ↓
ephemeral sandbox
   ↓
shared expensive infrastructure
   ↓
controlled capabilities
```

The more autonomous agents we run, the more valuable this type of infrastructure becomes.

---

# One subtle lesson: don't use CI as a universal job runner

Uber explicitly considered using systems such as CI for these operations.

But asking CI to calculate something like:

```
Bashgit merge-base
```

may require:

```
schedule worker
      ↓
initialize workspace
      ↓
sync repository
      ↓
configure environment
      ↓
run one Git command
```

That's enormous orchestration around tiny work.

This resembles something you should watch for in your own systems:

```
actual work:        20 ms

setup:            3,000 ms
```

When you see that ratio, optimization opportunities are often architectural rather than algorithmic.

Instead of optimizing:

```
20 ms → 10 ms
```

attack:

```
3,000 ms setup
```

That's where the real win is.

---

# 🔗 Connect tonight's lesson to what you've been learning

Several recent concepts unexpectedly meet here:

```
Caching
    ↓
reuse expensive state


Concurrency
    ↓
many Git operations happen simultaneously


Isolation
    ↓
each operation gets its own sandbox


Eventual consistency
    ↓
warm repositories may briefly lag upstream


Transactions
    ↓
related commands need consistent state


Load balancing
    ↓
requests routed to available sandbox pools


Security
    ↓
repository access follows caller identity
```

That's why engineering articles like this are useful.

A seemingly niche problem—

```
"Git cloning is slow at Uber"
```

—turns into a compact lesson in **distributed systems, caching, isolation, consistency, resource pooling and API design**.

## 🧠 Keep one idea tonight

When a system gets slow, don't only ask:

```
How can I make the operation faster?
```

Also ask:

```
Why are we performing this operation
so many times?
```

Uber didn't solve a 15-minute clone by making:

```
git clone
```

15× faster.

They changed the architecture so many consumers **no longer needed to clone at all**.

That's often where the biggest performance improvements hide.

The main read tonight is [Uber Engineering — GitFarm: Git as a Service for Large-Scale Monorepos](https://www.uber.com/by/en/blog/gitfarm-as-a-service/?utm_source=chatgpt.com), published **July 9, 2026**. The architecture section is particularly worth reading; it covers the gateway, warm repository pools, ephemeral sandboxes, gRPC command sessions, freshness trade-offs and production results in much more detail.
