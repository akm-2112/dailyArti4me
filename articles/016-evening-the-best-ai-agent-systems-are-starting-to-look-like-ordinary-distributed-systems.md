---
title: "🌙 Evening Tech Discovery — The best AI-agent systems are starting to look like ordinary distributed systems"
order: 16
volume: 3
category: evening
source: Evening Tech Discovery
---

# 🌙 Evening Tech Discovery — The best AI-agent systems are starting to look like ordinary distributed systems

A useful article landed on **September 2, 2026** from [Google Developers](https://developers.googleblog.com/4-engineering-patterns-behind-the-strongest-ai-agents-challenge-submissions/?utm_source=chatgpt.com). Google reviewed thousands of submissions to its AI Agents Challenge and noticed that the strongest systems repeatedly converged on four engineering patterns.

The interesting part is that none of them are really "AI tricks."

They're classic software-engineering ideas:

```
clear interfaces
event-driven concurrency
shared validation
cheap routing before expensive computation
```

That's an important signal for working developers: building production agents is becoming less about clever prompting and increasingly about **systems engineering**. [Google Developers Blog](https://developers.googleblog.com/4-engineering-patterns-behind-the-strongest-ai-agents-challenge-submissions/)

## 1. An agent can be both a client and a server

You've probably encountered Model Context Protocol.

The normal architecture looks like:

```
Agent
  │
  │ MCP
  ▼
Tool Server
  │
  ├── database
  ├── GitHub
  ├── filesystem
  └── internal API
```

The agent is the **client**.

It asks tools to do things.

Google saw a more interesting architecture among the stronger submissions:

```
                MCP
                 │
                 ▼
            Performance Agent
              /          \
             ▼            ▼
      telemetry DB     internal tools

                 ▲
                 │ MCP
                 │
          Coding Agent
```

The performance agent itself becomes an **MCP server**.

Another agent can therefore ask:

```
"Why did job 8291 become slow?"
```

instead of directly querying the telemetry database.

The performance agent can:

```
inspect execution plan
      ↓
retrieve relevant traces
      ↓
analyze bottleneck
      ↓
return bounded answer
```

Google calls this **bidirectional MCP**: an agent consumes tools internally while exposing useful capabilities externally. [Google Developers Blog](https://developers.googleblog.com/4-engineering-patterns-behind-the-strongest-ai-agents-challenge-submissions/)

This is more important than it initially sounds.

Suppose you build an e-commerce agent:

```
Order Agent
    │
    ├── order database
    ├── inventory API
    └── shipping API
```

Eventually another agent needs order information.

The naive architecture might give it all the same low-level access:

```
Customer Agent
      │
      ├── orders DB
      ├── inventory API
      └── shipping API
```

Now knowledge about your order system is duplicated.

Instead:

```
Customer Agent
      │
      │ MCP / agent protocol
      ▼
Order Agent
      │
      ├── orders DB
      ├── inventory
      └── shipping
```

The Order Agent becomes a **capability boundary**.

This should remind you of your dependency-injection lesson:

```
OrderService
      │
      ▼
PaymentGateway
```

The caller doesn't need to understand Stripe.

It needs:

```
charge(amount)
```

Likewise, another agent doesn't necessarily need SQL access.

It needs:

```
getOrderStatus(orderId)
```

Same architectural principle, different layer.

Google points out another benefit: instead of dumping an entire telemetry table into an LLM context, tools can return a specific execution plan or stack trace. That reduces token usage and creates a safer interface than handing an external caller raw database access. [Google Developers Blog](https://developers.googleblog.com/4-engineering-patterns-behind-the-strongest-ai-agents-challenge-submissions/)

---

# 2. Real multi-agent systems shouldn't necessarily form a call chain

This was probably the most interesting pattern.

Google found many submissions calling themselves:

```
multi-agent systems
```

that were essentially:

```
Agent A
   ↓
Agent B
   ↓
Agent C
   ↓
Agent D
```

That's technically multiple agents.

Architecturally, though, it's still a synchronous pipeline.

Suppose:

```
Agent A = 2 sec
Agent B = 4 sec
Agent C = 3 sec
Agent D = 5 sec
```

Total latency becomes roughly:

```
2 + 4 + 3 + 5

= 14 seconds
```

One strong challenge submission instead used an **event bus**.

Google describes agents consuming separate asynchronous queues and subscribing to typed events. [Google Developers Blog](https://developers.googleblog.com/4-engineering-patterns-behind-the-strongest-ai-agents-challenge-submissions/)

Conceptually:

```
                    EVENT BUS
                        │
         ┌──────────────┼──────────────┐
         ▼              ▼              ▼
   Compliance       Messaging       Dispatch
     Agent            Agent           Agent
```

Something happens:

```
ANOMALY_DETECTED
```

and multiple interested agents can react independently.

Instead of:

```
Agent A
   ↓
wait
   ↓
Agent B
   ↓
wait
   ↓
Agent C
```

you can have:

```
               EVENT
                 │
        ┌────────┼────────┐
        ▼        ▼        ▼
      Agent A  Agent B  Agent C
        │        │        │
        └────────┼────────┘
                 ▼
              results
```

Now independent work happens concurrently.

Google makes the distinction nicely: in a call chain, latency becomes additive; on an event bus, agents that don't depend on one another can run simultaneously. [Google Developers Blog](https://developers.googleblog.com/4-engineering-patterns-behind-the-strongest-ai-agents-challenge-submissions/)

Notice what happened.

Your recent lesson about **queues** suddenly applies directly to AI agents.

You learned:

```
Producer
   ↓
Queue
   ↓
Consumer
```

Now simply generalize it:

```
Agent
   ↓
Event Bus
   ↓
Agent(s)
```

Agents aren't exempt from distributed-systems architecture.

They're participants in it.

---

# 3. Fallback models must pass the same validation

This pattern is particularly practical.

One challenge system used a powerful model for clinical reasoning.

Under load:

```
Primary model
     ↓
503
```

Instead of simply retrying forever, the system fell back to a cheaper/faster model.

Conceptually:

```
Request
   │
   ▼
Powerful Model
   │
  503
   │
   ▼
Fallback Model
```

Reasonable.

But there's a subtle danger.

Imagine:

```
PRIMARY PATH

model
  ↓
citation validation
  ↓
schema validation
  ↓
response
```

while the fallback does:

```
FALLBACK PATH

model
  ↓
response
```

You've accidentally created:

```
high-quality product

and

lower-quality emergency product
```

depending on infrastructure conditions.

The stronger submission instead used:

```
Primary Model ────┐
                  │
                  ▼
              VALIDATOR
                  │
                  ▼
                output
                  ▲
                  │
Fallback Model ───┘
```

Both outputs passed through exactly the same validation function. [Google Developers Blog](https://developers.googleblog.com/4-engineering-patterns-behind-the-strongest-ai-agents-challenge-submissions/)

That's classic software design.

Don't write:

```
TypeScriptif (primaryWorked) {
    validatePrimary(result);
} else {
    validateFallback(result);
}
```

Prefer conceptually:

```typescript
const result =
    await primary()
        .catch(() => fallback());

return validate(result);
```

The important idea isn't the TypeScript syntax.

It's the invariant:

> **Every result must satisfy the same acceptance criteria regardless of which implementation produced it.**

This is remarkably similar to interface contracts.

You might have:

```
PaymentGateway

   ├── StripeGateway
   └── PayPalGateway
```

Your business invariant doesn't become weaker because one implementation was unavailable.

Likewise:

```
ReasoningModel

   ├── ExpensiveModel
   └── CheapFallbackModel
```

shouldn't mean:

```
different correctness standards
```

---

# 4. The cheapest AI call may be no AI call at all

This one is extremely useful if you ever build commercial AI products.

One challenge team analyzed where its inference budget was going.

You might expect difficult questions:

```
"Analyze the interaction between these
three medical conditions..."
```

to dominate.

Instead, simple requests were wasting expensive inference:

```
"Where's my order?"

"Cancel my appointment."
```

Their architecture became:

```
User message
     │
     ▼
Regex / deterministic rules
     │
     ├── obvious? → handle
     │
     ▼
Cheap classifier model
     │
     ├── clear? → route
     │
     ▼
Expensive reasoning model
```

The first deterministic layer alone handled **more than 40% of incoming messages** in that team's measured traffic. [Google Developers Blog](https://developers.googleblog.com/4-engineering-patterns-behind-the-strongest-ai-agents-challenge-submissions/)

Think about that economically.

Imagine:

```
1,000,000 requests/month
```

Without routing:

```
1,000,000
      ↓
expensive model
```

With the architecture above:

```
1,000,000
     │
     ├── 400,000 deterministic
     │
     ├── 300,000 cheap model
     │
     └── 300,000 expensive model
```

You haven't improved your prompt.

You've improved your **architecture**.

---

# 🛒 This matters enormously for commerce agents

Imagine you eventually build an AI commerce backend.

Users say:

```
"Where is order 123?"

"Cancel order 123."

"What's your return policy?"

"Find me a waterproof hiking jacket
under $150 suitable for Iceland."

"Compare these products and determine
which is better for my use case."
```

Those requests don't deserve identical computational treatment.

You could design:

```
                 Request
                    │
                    ▼
               Router
              /   |    \
             /    |     \
            ▼     ▼      ▼
        deterministic  cheap    reasoning
           logic      model      model
            │           │          │
            ▼           ▼          ▼
        Order API     intent     Shopping
                    classifier    Agent
```

For:

```
"Where is order 123?"
```

you may not need generative AI at all.

Perhaps:

```
TypeScriptif (isOrderStatusRequest(message)) {
    return orderService.getStatus(orderId);
}
```

For:

```
"Help me decide between these seven products
based on my hiking requirements."
```

Now reasoning adds genuine value.

This is one of the most important economics lessons in AI engineering:

> **Use intelligence where intelligence actually creates value.**

---

# 🤖 So what actually makes something multi-agent?

Google makes an interesting observation from the competition: many supposed multi-agent systems were really a single model going through a sequence of prompts with different agent names attached. [Google Developers Blog](https://developers.googleblog.com/4-engineering-patterns-behind-the-strongest-ai-agents-challenge-submissions/)

For example:

```
"You are Research Agent"

        ↓

"You are Analysis Agent"

        ↓

"You are Writer Agent"
```

doesn't automatically give you meaningful system architecture.

A stronger reason for separate agents is that they have genuinely different:

```
capabilities
permissions
tools
lifecycles
models
data access
scaling characteristics
failure boundaries
```

For example:

```
Shopping Agent
     │
     ├── Catalog Agent
     │      └── product search
     │
     ├── Pricing Agent
     │      └── pricing database
     │
     ├── Inventory Agent
     │      └── warehouse system
     │
     └── Checkout Agent
            └── payment capability
```

Now the boundaries mean something.

The Checkout Agent might have permission to:

```
create payment
```

while the Catalog Agent absolutely should not.

That separation gives you:

```
security boundaries
ownership boundaries
independent scaling
specialized context
isolated failure
```

Those are software-engineering reasons for decomposition.

Not:

```
"Four agents sounds cooler than one."
```

---

# There's another fresh development supporting this architecture

Google Cloud's Agent Registry became generally available in June, and its recent documentation now supports discovering agents, MCP servers and skills through keyword, prefix and **semantic search**. [Google Cloud Documentation+1](https://docs.cloud.google.com/agent-registry/search-agents-and-tools?utm_source=chatgpt.com)

Think:

```
Agent ecosystem

 ├── Order Agent
 ├── Inventory Agent
 ├── Shipping Agent
 ├── Fraud Agent
 ├── Support Agent
 ├── Database MCP
 └── Search MCP

          ↓

     Agent Registry

          ↓

"What capability can
handle shipment tracking?"

          ↓

     Shipping Agent
```

That's starting to look suspiciously like something backend engineers already know:

```
service registry
```

Microservice architectures needed ways to answer:

```
Where is service X?
```

Agent architectures increasingly need to answer:

```
Which agent/tool provides capability X?
```

Different technology.

Very familiar systems problem.

---

# 🔗 Connect this to what you've been learning

Your recent lessons now form a surprisingly useful foundation:

```
Dependency Injection
       ↓
Depend on capabilities,
not implementations


Queues
       ↓
Decouple producers
from consumers


Idempotency
       ↓
Make repeated operations safe


Race Conditions
       ↓
Handle concurrent execution


Caching
       ↓
Avoid unnecessary expensive work
```

Now map those directly into AI systems:

```
MCP / A2A
       ↓
capability interfaces


Agent event bus
       ↓
async decoupling


Agent tool calls
       ↓
idempotent operations


Parallel agents
       ↓
concurrency control


Tiered routing
       ↓
avoid unnecessary inference
```

That's the larger lesson tonight.

AI engineering isn't replacing software engineering.

It's creating another environment where **good software engineering becomes extremely valuable**.

## One idea worth remembering tonight

When someone shows you a sophisticated diagram containing:

```
17 agents
23 prompts
6 LLMs
```

don't immediately ask:

```
Which model are you using?
```

Ask:

```
Why are these separate components?

Which work can run concurrently?

What happens when one fails?

What happens when a request repeats?

Where is validation enforced?

Which component owns which permissions?

Why does this request need an LLM at all?
```

Those are ordinary engineering questions.

And according to Google's analysis of thousands of real agent projects, they're increasingly the questions separating impressive demos from stronger systems. [Google Developers Blog](https://developers.googleblog.com/4-engineering-patterns-behind-the-strongest-ai-agents-challenge-submissions/)

The best read tonight is Google's fresh September 2 article, [4 engineering patterns behind the strongest AI Agents Challenge submissions](https://developers.googleblog.com/4-engineering-patterns-behind-the-strongest-ai-agents-challenge-submissions/?utm_source=chatgpt.com). For the emerging discovery layer, the new [Google Cloud Agent Registry documentation](https://docs.cloud.google.com/agent-registry/search-agents-and-tools?utm_source=chatgpt.com) is worth skimming. And if you want the broader architecture showing how MCP and agent-to-agent communication fit together, Google's [A2A developer toolkit overview](https://cloud.google.com/blog/products/ai-machine-learning/agent2agent-protocol-is-getting-an-upgrade?utm_source=chatgpt.com) provides useful background.
