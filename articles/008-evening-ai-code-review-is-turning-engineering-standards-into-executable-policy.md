---
title: "🌙 Evening Tech Discovery — AI code review is turning engineering standards into executable policy"
order: 8
volume: 2
category: evening
source: Evening Tech Discovery
---

# 🌙 Evening Tech Discovery — AI code review is turning engineering standards into executable policy

Tonight’s topic is less about “AI writes code” and more about a potentially bigger change:

> **What if your team's engineering knowledge could automatically review every pull request, design document, and incident report?**

On **August 4, 2026**, [Cloudflare](https://www.cloudflare.com/?utm_source=chatgpt.com) published a useful engineering write-up about a system it calls the **Engineering Codex**. Over roughly four months, its AI code reviewer flagged close to **230,000 violations of internal engineering standards**, while its spec-review agent evaluated nearly **600 technical designs**. Cloudflare says these systems have already blocked **16,000 merges**. [Cloudflare Blog](https://blog.cloudflare.com/engineering-standards-enforcement/?utm_source=chatgpt.com)

The interesting part isn't the LLM.

It's how they organized the **knowledge around the LLM**.

## The problem: engineering knowledge is usually scattered

Imagine joining a mature engineering team.

Important knowledge might exist in:

```
README.md

Slack conversations

Confluence

old pull requests

senior engineer's memory

incident postmortems

architecture documents

random comments in code
```

Then you submit:

```
PR #1842
```

A senior developer says:

> “We don't do retries like this because three years ago this caused cascading failures.”

That's useful knowledge.

But the organization has a problem:

```
Senior engineer's brain
        ↓
only available when
someone remembers to ask
```

Now imagine that engineer leaves.

Some of that knowledge disappears.

Cloudflare's idea was to turn those lessons into a **governed repository of engineering standards**. [Cloudflare Blog](https://blog.cloudflare.com/engineering-standards-enforcement/?utm_source=chatgpt.com)

---

# The interesting architecture

Cloudflare calls this repository the **Codex**.

Standards are written as RFCs—Requests for Comments.

They use familiar RFC terminology:

```
MUST

SHOULD
```

For example, conceptually:

```
MUST validate external input.

MUST NOT log authentication tokens.

SHOULD use structured logging.

SHOULD use dependency X for retries.
```

But these aren't just documents humans might read someday.

They're structured so **agents can consume them**.

The architecture becomes roughly:

```
Domain experts
      │
      ▼
Engineering RFCs
      │
      ▼
Engineering Codex
      │
      ├───────────────┐
      ▼               ▼
AI Code Reviewer   Spec Reviewer
      │               │
      ▼               ▼
Pull Requests     Design Documents
```

Cloudflare says its agents also use the same standards when reviewing **incident reports**. [Cloudflare Blog](https://blog.cloudflare.com/engineering-standards-enforcement/?utm_source=chatgpt.com)

That's important.

The AI isn't inventing what “good engineering” means.

Humans define the standards.

The AI applies them repeatedly.

---

# 🤖 What happens during code review?

Cloudflare's broader internal AI engineering architecture gives us more detail.

Every merge request on its standard CI pipeline gets an AI review. A coordinator first determines the change's risk level and can delegate to specialized reviewers covering areas such as:

```
Code quality
Security
Codex compliance
Documentation
Performance
Release impact
```

The reviewers can read the repository's `AGENTS.md`, retrieve relevant engineering standards, and post structured comments back to the merge request. Cloudflare also separates model configuration from the CI template, allowing it to change which models perform particular review tasks without modifying every repository. [Cloudflare Blog](https://blog.cloudflare.com/internal-ai-engineering-stack/?utm_source=chatgpt.com)

Conceptually:

```
Developer opens PR
        │
        ▼
       CI
        │
        ▼
AI review coordinator
        │
        ├── Security agent
        ├── Performance agent
        ├── Code-quality agent
        └── Standards agent
                  │
                  ▼
          Engineering Codex
                  │
                  ▼
             PR feedback
```

This is much more interesting than:

```
"Claude/Copilot/ChatGPT, review my code."
```

because the agent receives **organization-specific context**.

---

# Here's the deeper idea: AI needs context more than clever prompts

Suppose you ask an LLM:

```
Review this TypeScript API.
```

It knows general TypeScript practices.

But it doesn't automatically know that **your company** requires:

```
All money represented as integer cents.

Every external request must have a timeout.

Controllers cannot access the database directly.

All payment operations require idempotency keys.

Domain logic cannot import infrastructure code.

PII must never appear in logs.
```

So instead of writing increasingly gigantic prompts, you create **persistent engineering context**.

Something like:

```
Markdown# AGENTS.md

## Architecture

Controllers MUST call application services.
Controllers MUST NOT query repositories directly.

## Payments

All payment mutations MUST be idempotent.

## Logging

Never log access tokens, passwords,
payment data, or customer PII.

## TypeScript

Use strict TypeScript.
Avoid `any` unless explicitly justified.
```

Now agents working on the repository have an architectural map.

This approach isn't limited to Cloudflare.

[GitHub's current Copilot documentation](https://docs.github.com/en/copilot/concepts/agents/code-review?utm_source=chatgpt.com) describes several layers of repository context, including:

```
.github/copilot-instructions.md

.github/instructions/*.instructions.md

AGENTS.md

agent skills
```

and says Copilot code review can use repository instructions, skills and relevant MCP context during reviews. [GitHub Docs](https://docs.github.com/en/copilot/concepts/agents/code-review?utm_source=chatgpt.com)

So this pattern is becoming part of mainstream developer tooling.

---

# 🧠 Why this matters for a PHP / TypeScript developer

Imagine you're building a Laravel backend.

Without explicit architecture rules, an AI coding agent might happily generate:

```php
class OrderController
{
    public function store(Request $request)
    {
        $order = Order::create(...);

        Stripe::charge(...);

        Mail::send(...);

        Redis::set(...);

        return response()->json($order);
    }
}
```

Technically?

It might work.

Architecturally?

Perhaps your project requires:

```
Controller
    ↓
Application Service
    ↓
Domain
    ↓
Interfaces
    ↓
Infrastructure
```

Your instructions could state:

```
Controllers MUST contain no business logic.

Payment providers MUST be accessed
through PaymentGateway.

External API calls MUST define
timeouts and retry policies.

Order creation MUST be idempotent.
```

Now when an agent generates the previous controller, another agent—or the same coding environment—can recognize:

```
❌ Business logic in controller

❌ Direct Stripe dependency

❌ Missing timeout/retry policy

❌ Missing idempotency handling
```

That's a very different level of AI-assisted development.

---

# 🔄 The really clever part: incidents can improve future code

This is probably the most useful idea in Cloudflare's approach.

Imagine production fails because:

```
Service A assumes
Service B is healthy

        ↓

Service B fails

        ↓

Service A continues processing

        ↓

cascading failure
```

Traditional incident lifecycle:

```
Outage
  ↓
Postmortem
  ↓
Document written
  ↓
Everyone promises to remember
  ↓
6 months pass
  ↓
someone repeats mistake
```

A stronger loop is:

```
Production incident
        │
        ▼
Postmortem
        │
        ▼
New engineering rule
        │
        ▼
Codex / AGENTS.md / lint rule
        │
        ▼
AI + CI enforcement
        │
        ▼
Future PR violating rule
        │
        ▼
BLOCKED
```

Cloudflare describes this explicitly as a flywheel: incidents reveal gaps, gaps become RFCs, RFCs become rules, and those rules feed future reviews. [Cloudflare Blog](https://blog.cloudflare.com/code-orange-fail-small-complete/?utm_source=chatgpt.com)

That's **organizational learning encoded into the software-development lifecycle**.

---

# Some rules shouldn't need AI at all

There's an important engineering distinction here.

Suppose your rule is:

```
Never commit .env files.
```

Don't ask an LLM.

Use deterministic tooling.

Likewise:

```
formatting → formatter

type errors → TypeScript compiler

code patterns → linter

unit behavior → tests

known vulnerabilities → dependency scanner
```

AI becomes interesting for rules such as:

```
"Does this change violate our
service ownership boundaries?"

"Does this retry strategy risk
amplifying downstream failure?"

"Does this design follow our
authentication architecture?"

"Is this API inconsistent with
our pagination standard?"
```

Those require more **semantic reasoning**.

So a mature pipeline might become:

```
                 PR
                  │
        ┌─────────┴─────────┐
        ▼                   ▼
Deterministic checks     AI review
        │                   │
        ├── tests            ├── architecture
        ├── types            ├── security reasoning
        ├── lint             ├── standards
        └── formatting       └── design issues
                  │
                  ▼
             Human review
```

AI doesn't replace your existing engineering tools.

It fills some of the gaps between them.

---

# ⚠️ The dangerous version of this idea

There's an obvious failure mode:

```
AI says violation
      ↓
automatically block everything
```

LLMs can misunderstand code and produce false positives.

Cloudflare handles this partly through **rule maturity and severity**. Approved RFC findings can initially be recommendations; enforced `MUST` requirements can become blocking conditions. [Cloudflare Blog](https://blog.cloudflare.com/engineering-standards-enforcement/?utm_source=chatgpt.com)

That gives you a useful adoption pattern:

```
Rule created
    ↓
AI warns
    ↓
team observes accuracy
    ↓
rule improves
    ↓
confidence increases
    ↓
selected rules become enforced
```

That's much safer than immediately granting an AI reviewer absolute authority.

---

# 💡 Something you could start doing now

Even without building Cloudflare's infrastructure, there's a useful habit here.

Create an:

```
AGENTS.md
```

in an active repository.

Don't fill it with generic advice like:

```
Write clean code.
Use best practices.
```

That's almost useless.

Capture **decisions specific to the system**:

```
Architecture

Controllers only handle HTTP concerns.
Business logic belongs in application services.
Repositories own database access.


Database

Every new query expected to operate on
large tables must have its query plan checked.

Never perform database queries inside loops.


External APIs

All calls require explicit timeouts.
Retry only idempotent operations.


Security

Never log credentials or PII.


Testing

Business rules require unit tests.
External services must be mocked/faked.
```

Notice something interesting?

Your recent morning lessons are already giving you concepts that could eventually become rules:

```
Database indexes
        ↓
"Check query plans for large-table queries"

Dependency Injection
        ↓
"Business services depend on interfaces,
not concrete infrastructure"

Big-O
        ↓
"Avoid repeated O(n) lookup inside loops"
```

Your theoretical knowledge starts turning into **engineering standards**.

---

# 🔭 The bigger shift

The first generation of coding AI was mostly:

```
Developer
    ↓
prompt
    ↓
AI
    ↓
code
```

The emerging model looks more like:

```
               Engineering knowledge
                       │
                       ▼
Developer ───────→ Coding Agent
                       │
                       ▼
                      PR
                       │
               ┌───────┴───────┐
               ▼               ▼
             Tests         Review Agents
                               │
                               ▼
                       Engineering Standards
                               │
                               ▼
                          Human Review
```

The competitive advantage may therefore become less about:

> **“Which company has access to the smartest coding model?”**

and increasingly about:

> **“How well has the company encoded its engineering knowledge so agents can use it?”**

The model can be replaced.

Your architecture decisions, incident history, security requirements, domain knowledge and engineering standards are the valuable context.

### One idea worth remembering tonight

> **Don't just give AI your code. Give it the rules that explain what good code means inside your system.**

That turns an AI coding assistant from a generic programmer into something closer to a developer who understands your team's architecture.

Tonight's main read is Cloudflare's **August 4, 2026** engineering article, [How Cloudflare enforces engineering standards using AI](https://blog.cloudflare.com/engineering-standards-enforcement/?utm_source=chatgpt.com). For the broader architecture, read [The AI engineering stack we built internally](https://blog.cloudflare.com/internal-ai-engineering-stack/?utm_source=chatgpt.com). And for something you can apply directly in repositories today, [GitHub's guide to customizing Copilot code review with repository instructions and AGENTS.md](https://docs.github.com/en/enterprise-cloud%40latest/copilot/tutorials/customize-code-review?utm_source=chatgpt.com) is a useful practical follow-up.
