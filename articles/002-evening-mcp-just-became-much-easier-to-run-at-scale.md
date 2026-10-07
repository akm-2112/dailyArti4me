---
title: "🌙 Evening Tech Discovery — MCP just became much easier to run at scale"
order: 2
volume: 1
category: evening
source: Evening Tech Discovery
---

# 🌙 Evening Tech Discovery — MCP just became much easier to run at scale

Tonight’s topic is especially relevant if you’re interested in **AI agents and backend architecture**: Model Context Protocol has undergone a major architectural change.

On **July 28, 2026**, MCP released specification `2026-07-28`. The headline change is that the protocol's core is now **stateless**. [Model Context Protocol Blog+1](https://blog.modelcontextprotocol.io/posts/2026-07-28/?utm_source=chatgpt.com)

### First: what problem is MCP solving?

Think about an AI agent that needs access to:

```
AI Agent
   │
   ├── GitHub
   ├── Database
   ├── Stripe
   ├── CRM
   ├── Internal API
   └── Cloud infrastructure
```

Without a standard protocol, every integration tends to become custom glue code.

MCP provides a common interface:

```
LLM / Agent
     │
     │ MCP
     ▼
MCP Server
     │
     ├── tool: search_orders
     ├── tool: refund_order
     └── tool: get_customer
```

The agent discovers the available capabilities and calls them through the same general protocol.

That's why MCP is sometimes loosely compared to **USB for AI tools**: different systems can expose capabilities through a common interface.

## The interesting change: stateless MCP

Earlier remote MCP designs could maintain a session between client and server.

Conceptually:

```
Agent ───── persistent session ───── MCP Server
```

That introduces infrastructure problems.

Imagine you have three MCP servers:

```
              Load Balancer
             /      |      \
          MCP-1   MCP-2   MCP-3
```

Request #1 might reach `MCP-1`.

If `MCP-1` owns the session state, request #2 can't simply land on `MCP-3`.

You may need sticky sessions, shared state or additional coordination infrastructure.

The new MCP core instead makes requests **self-describing and independent**. Any request can therefore be handled by an appropriate server instance behind an ordinary round-robin load balancer. [Model Context Protocol Blog](https://blog.modelcontextprotocol.io/posts/2026-07-28/?utm_source=chatgpt.com)

Conceptually:

```
Agent
  │
  ▼
Load Balancer
  │
  ├── Request 1 → MCP-1
  ├── Request 2 → MCP-3
  ├── Request 3 → MCP-2
  └── Request 4 → MCP-1
```

That's a big infrastructure simplification.

### Why should you care as a backend developer?

Notice that this isn't really an “AI-only” concept.

You've probably encountered the same principle when designing APIs.

A stateless REST API can often scale horizontally:

```
                Load Balancer
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
       PHP API    PHP API    PHP API
```

Instead of storing important user state inside one PHP process:

```php
$serverMemory[$userId] = $conversation;
```

you put durable state somewhere appropriate:

```
PostgreSQL
Redis
object storage
etc.
```

Then application instances become replaceable.

MCP is moving toward that same cloud-native architectural principle.

**Stateless compute → easier horizontal scaling.**

That connection is worth remembering.

---

## There's another interesting development: Code Mode

[Cloudflare's Code Mode documentation](https://developers.cloudflare.com/agents/model-context-protocol/codemode/?utm_source=chatgpt.com) describes another problem emerging with agents.

Imagine an MCP server exposes **2,594 tools**.

Every tool has a schema describing parameters and responses.

Giving all those definitions to an LLM consumes an enormous amount of context before the agent has accomplished anything.

Cloudflare reports an eye-catching example for its own API:

```
Native MCP
2,594 tools
≈ 1,170,000 tokens of schemas

Required parameters only
≈ 244,000 tokens

Code Mode
2 tools
≈ 1,000 tokens
```

[Cloudflare Docs](https://developers.cloudflare.com/agents/model-context-protocol/cloudflare/servers-for-cloudflare/?utm_source=chatgpt.com)

Instead of exposing thousands of individual tools, Code Mode can expose essentially:

```
search
execute
```

The model searches for the API capability it needs and then writes code that performs the operation.

For example, instead of the model individually calling:

```
getUsers()
getOrders()
getProducts()
filterOrders()
calculateRevenue()
```

it can generate code conceptually like:

```typescript
const orders = await api.getOrders();

const completed = orders.filter(
    order => order.status === "completed"
);

return completed.reduce(
    (total, order) => total + order.amount,
    0
);
```

Intermediate data can stay inside the execution environment rather than repeatedly traveling through the model's context.

Cloudflare's implementation runs generated JavaScript inside an isolated Worker and keeps credentials in the host rather than handing them to generated code. [Cloudflare Docs](https://developers.cloudflare.com/agents/model-context-protocol/codemode/?utm_source=chatgpt.com)

Code Mode is currently marked **experimental**, so it isn't something I'd blindly base critical production architecture on yet. [Cloudflare Docs](https://developers.cloudflare.com/agents/model-context-protocol/guides/build-codemode-mcp-server/?utm_source=chatgpt.com)

But the architectural idea is important.

## 🧠 The bigger lesson

We started AI agents with something resembling:

```
LLM
 ├── tool1
 ├── tool2
 ├── tool3
 ├── ...
 └── tool500
```

We're increasingly moving toward:

```
LLM
 │
 ▼
Discover capability
 │
 ▼
Write/orchestrate code
 │
 ▼
Sandbox
 │
 ▼
APIs / MCP / services
```

That looks much more like **software execution** than traditional chatbot function calling.

For a PHP + TypeScript developer, this is an interesting opportunity. You already know many of the underlying skills needed to build these systems:

```
HTTP APIs
OAuth
TypeScript
databases
queues
permissions
webhooks
distributed systems
API design
```

AI doesn't replace those concepts. **Agents make them more important**, because now software may be calling your APIs autonomously rather than a human clicking buttons.

### One idea to keep tonight

> **Agent engineering is increasingly becoming ordinary distributed-systems engineering with an LLM making some of the decisions.**

The AI model gets attention, but reliable agents still require authentication, authorization, APIs, state management, retries, idempotency, observability, sandboxing and scalable infrastructure.

For your deeper read tonight, start with the official [MCP 2026-07-28 release explanation](https://blog.modelcontextprotocol.io/posts/2026-07-28/?utm_source=chatgpt.com). Then, if Code Mode interests you, read [Cloudflare's Code Mode architecture guide](https://developers.cloudflare.com/agents/model-context-protocol/codemode/?utm_source=chatgpt.com) and the practical [MCP + Code Mode guide](https://developers.cloudflare.com/agents/tools/codemode/mcp/?utm_source=chatgpt.com).
