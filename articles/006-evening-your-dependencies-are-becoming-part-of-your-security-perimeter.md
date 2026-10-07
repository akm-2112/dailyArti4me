---
title: "🌙 Evening Tech Discovery — Your dependencies are becoming part of your security perimeter"
order: 6
volume: 1
category: evening
source: Evening Tech Discovery
---

# 🌙 Evening Tech Discovery — Your dependencies are becoming part of your security perimeter

Tonight's topic is **software supply-chain security**, and it's particularly relevant because it affects both sides of your stack: **Composer/PHP and npm/TypeScript**.

On **July 28, 2026**, [GitHub](https://github.com/?utm_source=chatgpt.com) expanded Dependabot malware alerts beyond npm. Its advisory pipeline now consumes malware reports from the OpenSSF malicious-packages project, giving coverage across eight major ecosystems—including **npm and PHP Composer**. [The GitHub Blog+1](https://github.blog/changelog/2026-07-28-dependabot-alerts-on-malicious-packages-across-more-ecosystems/?utm_source=chatgpt.com)

This matters because there's an important difference between a **vulnerable package** and a **malicious package**.

## Vulnerability vs malware

Imagine you install:

```
Bashnpm install some-package
```

A normal vulnerability might look like:

```
Legitimate package
       │
       ├── useful code
       │
       └── accidental security bug
```

For example, perhaps malformed input can bypass validation.

The maintainer didn't intend that.

A malicious package is fundamentally different:

```
Package
   │
   ├── looks useful
   │
   └── intentionally malicious code
           │
           ├── steal environment variables
           ├── steal npm/GitHub tokens
           ├── steal cloud credentials
           ├── modify files
           └── install malware
```

There's nothing to "patch" in the traditional sense.

The package **is the attack**.

GitHub's Advisory Database contains real advisories where the guidance is effectively to consider machines that ran the package compromised and rotate secrets from another machine. [GitHub](https://github.com/advisories/GHSA-w4vx-v8m8-c885?utm_source=chatgpt.com)

---

# Why package managers create an interesting security problem

Modern developers don't write applications entirely themselves.

A TypeScript project might look conceptually like:

```
Your application
      │
      ├── React
      ├── Axios
      ├── Zod
      ├── date library
      └── 30 direct dependencies
               │
               ▼
        hundreds/thousands
        of transitive dependencies
```

You might install **30 packages**, but those packages install packages, which install other packages.

Your effective dependency tree can become enormous.

The same applies to:

```
Bashcomposer install
```

A Laravel application could have:

```
Your code
   ↓
Laravel
   ↓
Symfony components
   ↓
PSR packages
   ↓
other libraries
```

Every dependency is code you're allowing into your system.

And sometimes installation itself executes code.

That's why:

```
Bashnpm install
```

isn't conceptually just:

> "Download some files."

It's closer to:

> **"Download code from external publishers and potentially allow parts of that ecosystem to execute within my development/build environment."**

That's a very different security mindset.

---

# The attack developers often underestimate

Suppose an attacker compromises the publishing credentials of a popular package maintainer.

The Git repository might still look completely normal:

```
GitHub repository
     │
     └── clean source code
```

But the attacker publishes:

```
npm registry

package@4.2.0     ✓ legitimate

package@4.2.1     ☠ malicious
```

Your automated dependency update sees:

```
New version available!
```

CI runs:

```
Bashnpm install
```

and now the malicious package is executing somewhere extremely interesting:

```
CI environment
     │
     ├── GitHub token
     ├── AWS credentials
     ├── npm token
     ├── deployment credentials
     ├── database secrets
     └── source code
```

You've effectively delivered the attacker directly into one of the most privileged parts of your engineering infrastructure.

This is why **CI/CD security and dependency security are increasingly the same problem**.

---

# GitHub's interesting response: don't update immediately

Here's a particularly useful development from last month.

On **July 14, 2026**, GitHub changed Dependabot's default behavior so normal version-update PRs wait until a newly released package has existed for at least **three days**. Security updates are not delayed. [The GitHub Blog](https://github.blog/changelog/2026-07-14-dependabot-version-updates-introduce-default-package-cooldown/?utm_source=chatgpt.com)

Why would waiting improve security?

Imagine:

```
Monday 09:00
malicious version published

Monday 09:02
automated bots update thousands of projects

Monday 11:00
security researchers discover malware
```

Automation worked perfectly.

And that's precisely the problem.

You were **too fast**.

Instead:

```
Package released
      │
      ▼
  wait 3 days
      │
      ├── community installs it
      ├── scanners inspect it
      ├── researchers investigate
      └── maintainers receive reports
      │
      ▼
Dependabot update
```

This introduces a deliberate **cooldown period**.

It's a fascinating engineering principle:

> Sometimes reliability improves when automation intentionally slows down.

Not everything should happen immediately.

---

# npm is attacking another problem: long-lived secrets

Imagine your CI configuration contains:

```
NPM_TOKEN=abc123...
```

Your pipeline uses it every time it publishes:

```
GitHub Actions
      │
      │ NPM_TOKEN
      ▼
     npm
```

That token may exist for months.

If someone steals it:

```
attacker
   │
   └── stolen NPM_TOKEN
           │
           ▼
       npm publish
```

they could potentially publish a malicious package.

A better model is **OIDC trusted publishing**.

npm now supports publishing from CI using OpenID Connect rather than requiring a permanent npm publishing token. npm's documentation recommends trusted publishing when available because the credentials are short-lived and workflow-specific rather than long-lived secrets that require manual storage and rotation. [npm Docs](https://docs.npmjs.com/trusted-publishers/?utm_source=chatgpt.com)

Conceptually:

```
OLD

CI
 │
 │ permanent secret
 ▼
npm


NEW

CI
 │
 │ prove identity through OIDC
 ▼
npm
 │
 ▼
short-lived authorization
```

Your CI says:

> "I'm this specific authorized workflow from this repository."

npm verifies that identity and grants temporary authorization.

Then it expires.

There's nothing permanent sitting around for an attacker to steal and reuse.

---

# 🧠 This pattern goes much further than npm

This is becoming an important cloud-security architecture:

```
Old architecture

Application
   │
   ├── AWS_ACCESS_KEY
   ├── DATABASE_PASSWORD
   ├── NPM_TOKEN
   ├── GITHUB_TOKEN
   └── OTHER_SECRET
```

versus:

```
Modern architecture

Workload
   │
   ▼
Identity provider
   │
   ▼
short-lived credential
   │
   ▼
resource
```

You'll encounter the same general philosophy with cloud workload identities and temporary credentials.

Instead of asking:

> **"Where should I safely store this permanent secret?"**

the better architectural question can be:

> **"Can I avoid having a permanent secret at all?"**

That's a very powerful security mindset.

---

# Another npm change worth knowing

In May, npm also made **staged publishing** generally available.

Instead of:

```
npm publish
     │
     ▼
millions of developers
```

you can have:

```
npm publish
     │
     ▼
staging area
     │
     ▼
maintainer approval
     │
     ▼
public registry
```

npm CLI 11.15+ also added more install-source controls. [The GitHub Blog](https://github.blog/changelog/2026-05-22-staged-publishing-and-new-install-time-controls-for-npm/?utm_source=chatgpt.com)

Again, notice the pattern.

We're introducing **boundaries around automation**.

---

# What this means for your PHP + TypeScript projects

There are several habits worth developing.

**Lock your dependencies.**

For npm:

```
package-lock.json
```

For Composer:

```
composer.lock
```

These aren't annoying files to delete whenever dependency resolution becomes inconvenient.

They make installations reproducible.

Your production deployment should generally not unexpectedly decide:

```
"There's a newer dependency today.
I'll use that."
```

You want:

```
Developer
    │
    ▼
review dependency update
    │
    ▼
lock exact resolved versions
    │
    ▼
CI tests
    │
    ▼
production
```

Second, treat dependency-update PRs like **code changes**.

Don't mentally categorize:

```
Dependabot PR
```

as:

```
"robot stuff — merge"
```

It's actually:

```
"Third-party code changed
inside my application."
```

Third, enable malware detection where available. GitHub says malware alerts can be enabled under the repository or organization Dependabot security settings; existing users with malware alerting enabled automatically receive the broader advisory coverage. [The GitHub Blog](https://github.blog/changelog/2026-07-28-dependabot-alerts-on-malicious-packages-across-more-ecosystems/?utm_source=chatgpt.com)

---

# 🤖 There's an AI-agent connection too

Something else changed this year.

GitHub now allows some Dependabot vulnerability alerts to be assigned to coding agents—including Copilot, Claude and Codex—which can analyze the advisory and create a draft remediation PR. [The GitHub Blog](https://github.blog/changelog/2026-04-07-dependabot-alerts-are-now-assignable-to-ai-agents-for-remediation/?utm_source=chatgpt.com)

So the emerging workflow looks like:

```
Malicious/vulnerable dependency detected
              │
              ▼
         Dependabot
              │
              ▼
        coding agent
              │
              ▼
        proposed patch
              │
              ▼
          CI tests
              │
              ▼
        human review
```

That's a much more interesting use of AI agents than simply:

> "Write this function for me."

The agent participates in an **operational software-engineering workflow**.

But notice something important:

```
AI agent ≠ security authority
```

The agent proposes.

Your security controls, tests, permissions and humans still establish the boundary.

---

# 🔗 Connect this to what you've been learning

There's a bigger picture emerging from your recent lessons:

```
Big-O
  ↓
Understand computational cost

Hash tables
  ↓
Understand data structures

Database indexes
  ↓
Understand storage performance

Supply-chain security
  ↓
Understand where your code actually comes from

CI/CD
  ↓
Understand how that code reaches production
```

Being a strong developer isn't just knowing how to write:

```
PHPpublic function createOrder() {}
```

or:

```
TypeScriptasync function createOrder() {}
```

It's understanding the **entire lifecycle of that software**:

```
source
  ↓
dependencies
  ↓
build
  ↓
tests
  ↓
credentials
  ↓
deployment
  ↓
runtime
  ↓
monitoring
```

Every arrow represents a potential engineering and security boundary.

## One idea worth remembering tonight

> **A dependency is code you didn't write but agreed to run.**

Package managers make that trust incredibly convenient—and convenience can make the trust almost invisible.

A mature engineering mindset therefore isn't:

```
"Never use dependencies."
```

That's unrealistic.

It's:

```
Use dependencies
      +
pin versions
      +
minimize unnecessary packages
      +
scan them
      +
delay risky automatic upgrades
      +
protect publishing credentials
      +
test updates
      +
review what reaches production
```

For tonight's deeper reading, the best starting point is GitHub's August 6 engineering article, [How we took malware advisories beyond npm](https://github.blog/security/supply-chain-security/how-we-took-malware-advisories-beyond-npm/?utm_source=chatgpt.com). Then read the short [Dependabot three-day package cooldown announcement](https://github.blog/changelog/2026-07-14-dependabot-version-updates-introduce-default-package-cooldown/?utm_source=chatgpt.com). If you ever publish npm packages yourself, bookmark the official [npm Trusted Publishing documentation](https://docs.npmjs.com/trusted-publishers/?utm_source=chatgpt.com)—it explains the OIDC model and why eliminating long-lived publishing tokens improves supply-chain security.
