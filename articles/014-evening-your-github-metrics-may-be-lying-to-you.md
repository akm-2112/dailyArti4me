---
title: "🌙 Evening Tech Discovery — Your GitHub metrics may be lying to you"
order: 14
volume: 3
category: evening
source: Evening Tech Discovery
---

# 🌙 Evening Tech Discovery — Your GitHub metrics may be lying to you

Tonight’s topic is **data engineering and observability**, prompted by a fresh September 3 article from [Google Open Source](https://opensource.googleblog.com/2026/09/?utm_source=chatgpt.com).

The surprising finding: a widely used public dataset of [GitHub](https://github.com/?utm_source=chatgpt.com) activity may now capture only a fraction of what is actually happening.

That sounds like an open-source analytics problem. But the underlying engineering lesson applies directly to **production dashboards, event pipelines, AI datasets, analytics, logs, and business metrics**.

> **A perfectly correct query over an incomplete dataset still produces a wrong conclusion.**

## The interesting discovery

GH Archive has collected public GitHub events since 2011.

Think:

```
GitHub
   │
   ├── PushEvent
   ├── PullRequestEvent
   ├── IssuesEvent
   ├── IssueCommentEvent
   └── ...
        │
        ▼
    GH Archive
        │
        ▼
historical dataset
```

Researchers and developers can then ask questions such as:

```
How much is GitHub activity growing?

Which languages are becoming popular?

How many PRs are being created?

Which projects have active communities?
```

Historically, this has been an extremely useful dataset.

But Google Open Source and Ecosyste.ms reported yesterday that its coverage has degraded substantially. They estimate that since 2025, retention has fallen to roughly **50% overall**, and in 2026 some event types may have coverage as low as **20%**. [Google Open Source Blog](https://opensource.googleblog.com/2026/09/?utm_source=chatgpt.com)

Here's the particularly deceptive part.

GH Archive recorded **14% fewer events in 2025 than in 2024**, even while GitHub itself continued growing. [Google Open Source Blog](https://opensource.googleblog.com/2026/09/?utm_source=chatgpt.com)

You could look at that graph and conclude:

```
GitHub activity ↓ 14%

Therefore:

open-source activity is declining.
```

But the real explanation may instead be:

```
GitHub activity ↑

collector coverage ↓↓

observed events ↓
```

The measurement changed.

Not necessarily the system being measured.

---

# 🧠 Why would a crawler start losing data?

The basic collector sounds straightforward:

```
repeat forever:

    request GitHub events
    save events
```

But APIs impose constraints.

Suppose GitHub produces:

```
1,000 events/sec
```

and your collector can reliably ingest:

```
1,000 events/sec
```

Everything works.

Then GitHub grows:

```
10,000 events/sec
```

while your collection architecture still handles roughly the old workload.

Now:

```
producer rate
     ↓
10,000/sec

collector capacity
     ↓
1,000/sec
```

Depending on the API, pagination, retention window and rate limits, the collector may never catch up.

Google notes that API call limits and limits on how many events are exposed can cause the crawler to miss activity during spikes. GitHub itself has gone from about **2 million public repositories in 2011 to more than 400 million by 2026**, while automation has also dramatically increased activity. [Google Open Source Blog](https://opensource.googleblog.com/2026/09/?utm_source=chatgpt.com)

This should sound familiar from our recent queue lesson:

```
Producer faster than consumer
        ↓
backlog
```

Except there is an important difference.

With a durable queue:

```
Producer
   ↓
QUEUE
   ↓
Consumer

slow consumer
   ↓
messages remain queued
```

But imagine an API exposing only a moving window:

```
event 100
event 101
event 102
...
event 999

NEW EVENTS ARRIVE

event 100 disappears
```

If your crawler didn't fetch `event 100` before it disappeared:

```
it's gone.
```

There may be no retry.

Your historical dataset now has a permanent hole.

---

# This is really an observability problem

Imagine your production API handles:

```
1,000,000 requests/day
```

Your logging pipeline records:

```
920,000
```

Then your dashboard says:

```
Error rate:

errors / logged requests
```

Everything might look professional:

```
Grafana dashboard ✓
Prometheus ✓
SQL queries ✓
beautiful charts ✓
```

But you're missing:

```
80,000 requests
```

Now ask the critical question:

> Are those 80,000 missing requests random?

Maybe not.

Suppose your logging pipeline drops events specifically when traffic spikes.

Then:

```
normal traffic
    ↓
logs complete


traffic spike
    ↓
system overloaded
    ↓
logs dropped
```

Exactly when your production system becomes interesting...

**your observability becomes least reliable.**

That's dangerous.

---

# Missing data is often biased

Suppose your metrics contain:

```
95% successful requests

5% failures
```

But your logging system has trouble sending logs during crashes.

Perhaps:

```
Successful request
      ↓
log successfully delivered


Application crashes
      ↓
log never delivered
```

Now your dataset might report:

```
98% success
```

even though reality is:

```
95% success
```

This isn't merely missing data.

It's **systematically missing data**.

That's much worse.

---

# 🛒 Practical e-commerce example

Imagine your checkout analytics pipeline:

```
Browser
   ↓
POST /checkout
   ↓
Order Service
   ↓
Analytics Event

"checkout_completed"
```

Your dashboard says:

```
Checkout conversion = 72%
```

Great.

Then someone optimizes the system and reports:

```
Conversion increased to 81%!
```

But perhaps an unrelated frontend deployment broke:

```
TypeScriptanalytics.track("checkout_started");
```

for Safari users.

Now your denominator shrinks.

Imagine reality:

```
1,000 checkout attempts

720 purchases

conversion = 72%
```

But analytics misses 110 checkout-start events:

```
890 observed attempts

720 purchases

observed conversion ≈ 81%
```

Your dashboard says:

```
🎉 HUGE IMPROVEMENT
```

Reality:

```
nothing changed
```

The metric calculation is mathematically correct.

The **data pipeline is wrong**.

---

# A very useful distinction

Developers often think data quality means:

```
Is the value valid?
```

For example:

```
email valid?
timestamp valid?
price numeric?
```

But mature data systems care about several dimensions.

One is **validity**:

```
Does this record make sense?
```

Another is **completeness**:

```
Did we receive all expected records?
```

Another is **freshness**:

```
Is the data arriving on time?
```

Another is **uniqueness**:

```
Did we accidentally process something twice?
```

Notice something interesting.

Your recent lessons map directly onto these problems:

```
Queues
   ↓
Could events be delayed?


Idempotency
   ↓
Could events appear twice?


Race conditions
   ↓
Could concurrent updates corrupt counts?


Today's lesson
   ↓
Could events disappear entirely?
```

Distributed systems keep attacking your data from different directions.

---

# How would you detect missing events?

A very useful technique is a **sequence number**.

Imagine your producer emits:

```json
{ "sequence": 1001, "event": "order_created" }
{ "sequence": 1002, "event": "payment_completed" }
{ "sequence": 1003, "event": "order_shipped" }
```

Your consumer receives:

```
1001
1002
1003
```

Good.

But suppose it receives:

```
1001
1003
```

Immediately:

```
expected 1002
received 1003

⚠️ GAP DETECTED
```

Without sequence numbers, these events:

```
order_created
order_shipped
```

might look completely legitimate.

You wouldn't know something was missing.

This is one reason distributed event systems often use concepts such as:

```
offset
sequence number
cursor
checkpoint
```

For example, an event consumer might remember:

```
last processed offset = 8,193,201
```

and resume from there.

---

# Another technique: reconciliation

Sometimes you can't guarantee perfect event delivery.

Instead, periodically compare your derived system against the authoritative source.

Imagine:

```
Orders database

1,000 orders today
```

but:

```
Analytics pipeline

982 order_created events
```

Your reconciliation job detects:

```
Expected: 1,000
Observed:   982
Difference: 18
```

Now you know your event pipeline lost data.

This leads to a valuable architecture:

```
Operational Database
       │
       │ source of truth
       ▼
Event Pipeline
       │
       ▼
Analytics


Periodic reconciliation
       │
       └──────── compare ────────┐
                                 ▼
                         detect missing data
```

Financial systems use this kind of thinking constantly.

You shouldn't blindly assume:

```
event sent
=
event received
=
event stored
=
event processed
```

Those are four different things.

---

# There's also a database lesson here

Google's article points out another messy reality when combining open-source datasets:

```
repository names
package names
versions
tags
licenses
URLs
```

may be represented differently across platforms. Some names are case-sensitive, duplicates exist, and repositories can be renamed. [Google Open Source Blog](https://opensource.googleblog.com/2026/09/?utm_source=chatgpt.com)

Suppose one source says:

```
github.com/OpenAI/project
```

another:

```
github.com/openai/project
```

another:

```
project
```

and another stores an old repository name.

Now you perform:

```
SQLJOIN repositories
ON source_a.name = source_b.name
```

and silently lose matches.

Your SQL executes successfully.

No exception.

No red error screen.

Just:

```
wrong data
```

Those are often the nastiest data bugs.

---

# 🤖 And this matters enormously for AI

There's a particularly timely connection.

We're increasingly doing:

```
Dataset
   ↓
LLM / embedding / classifier
   ↓
AI result
```

Developers naturally spend time asking:

```
Which model?

Which embedding model?

Which vector database?

Which prompt?
```

But Google's article makes the more fundamental point:

> All AI systems ultimately depend on data. [Google Open Source Blog](https://opensource.googleblog.com/2026/09/?utm_source=chatgpt.com)

Imagine training an AI to predict:

```
Which open-source projects
are healthy?
```

using incomplete GitHub event data.

Your model might learn:

```
Project activity ↓
```

when the actual cause is:

```
collector coverage ↓
```

No amount of better prompting fixes that.

You have a **data-generating-process problem**.

---

# This gives you a powerful debugging question

Whenever a dashboard looks surprising, developers often immediately investigate:

```
application behavior
```

Instead, ask two independent questions:

```
1. Did the system change?

2. Did our ability to observe the system change?
```

Those are not the same question.

Suppose CPU suddenly falls:

```
80%
 ↓
25%
```

Maybe:

```
great optimization!
```

Or perhaps:

```
half the metrics agents stopped reporting.
```

Suppose errors disappear:

```
2.4%
 ↓
0.1%
```

Maybe you fixed the service.

Or perhaps your error logger broke.

An observability system must itself be **observable**.

---

# What should you monitor about your monitoring?

This sounds recursive, but it's important.

For an event pipeline, useful meta-metrics include:

```
events produced/sec

events consumed/sec

queue depth

consumer lag

dropped events

ingestion failures

last successful ingestion timestamp

expected vs observed counts
```

Now instead of merely:

```
Dashboard:
12,482 events
```

you can also know:

```
Expected ≈ 12,500

Received = 12,482

Completeness ≈ 99.86%
```

That's a much more trustworthy metric.

---

# 🔗 The larger engineering connection

Your recent lessons are starting to form a surprisingly coherent distributed-systems picture:

```
Idempotency
     ↓
"What if I receive something twice?"


Queues
     ↓
"What if processing happens later?"


Race conditions
     ↓
"What if two things happen simultaneously?"


Data completeness
     ↓
"What if I never receive something?"
```

Together:

```
duplicate
delayed
concurrent
missing
```

Those four words describe a huge percentage of real production problems.

And notice how different this is from local development.

On your laptop you often imagine:

```
request
   ↓
function
   ↓
database
   ↓
response
```

Production looks more like:

```
          ┌── delayed
          │
event ────┼── duplicated
          │
          ├── reordered
          │
          └── possibly lost
```

Becoming a stronger backend engineer means gradually replacing the first mental model with the second.

## One idea worth remembering tonight

> **Before trusting a metric, understand how the data that created it was collected.**

Don't only ask:

```
What does the graph say?
```

Ask:

```
Where did this data come from?

What can the collector miss?

What happens during traffic spikes?

Are there rate limits?

Can events expire before collection?

Can events duplicate?

Can identifiers change?

How would I know if ingestion broke?
```

That's the difference between **using data** and **engineering trustworthy data systems**.

The main read tonight is Google's September 3 article, [How much should you trust your OSS data?](https://opensource.googleblog.com/2026/09/?utm_source=chatgpt.com). It's worth reading because it uses real GitHub-scale failures to explain problems you'll encounter in much smaller production systems too. For the wider context, [Google Open Source's 2026 archive](https://opensource.googleblog.com/2026/?utm_source=chatgpt.com) also covers how the open-source ecosystem is changing as AI and automation increase activity.
