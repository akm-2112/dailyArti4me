---
title: "🌅 Morning Tech Lesson #11 — HTTP Is Stateless: Cookies, Sessions, JWTs, and Where Authentication State Actually Lives"
order: 21
volume: 4
category: morning
source: Evening Tech Discovery
---

# 🌅 Morning Tech Lesson #11 — HTTP Is Stateless: Cookies, Sessions, JWTs, and Where Authentication State Actually Lives

We’ve recently gone deep on **transactions, caching, queues, concurrency, and the Node.js event loop**. Today we’ll shift into **networking + API architecture + security**.

The central question is:

> **If HTTP forgets every request after it finishes, how does a website remember that you logged in?**

Estimated time: **10–15 minutes**.

## Start with the fundamental HTTP model

Your browser sends:

```
httpGET /profile HTTP/1.1
Host: example.com
```

The server responds:

```
httpHTTP/1.1 200 OK
Content-Type: text/html

...
```

Later:

```
httpGET /orders HTTP/1.1
Host: example.com
```

At the HTTP protocol level, these are independent requests.

Conceptually:

```
Request A
   ↓
Server
   ↓
Response A

finished


Request B
   ↓
Server
   ↓
Response B
```

HTTP itself doesn't inherently say:

```
Request B belongs to
the same logged-in person as Request A.
```

We therefore describe HTTP as **stateless**.

But applications obviously need state:

```
logged-in user
shopping cart
language preference
CSRF protection
checkout progress
```

So we build state management **on top of HTTP**.

---

# 🍪 Cookies: the browser remembers something for you

A server can respond:

```
httpSet-Cookie: session_id=abc123
```

The browser stores it.

Future requests automatically include:

```
httpCookie: session_id=abc123
```

Conceptually:

```
LOGIN

Browser
   │
   │ username/password
   ▼
Server
   │
   │ Set-Cookie:
   │ session_id=abc123
   ▼
Browser stores cookie
```

Then:

```
GET /profile

Browser
   │
   │ Cookie:
   │ session_id=abc123
   ▼
Server
```

Now the server has something linking this request to earlier state.

Important distinction:

> **A cookie is a browser storage/transport mechanism, not an authentication architecture by itself.**

A cookie could contain:

```
session ID
theme
language
tracking identifier
JWT
```

Cookies and sessions are related, but they are not the same thing.

---

# 🧠 Traditional server-side sessions

Suppose the cookie contains:

```
session_id = abc123
```

The server stores:

```
abc123
    ↓
{
    user_id: 42,
    role: "admin"
}
```

The flow becomes:

```
Browser

Cookie:
abc123
   │
   ▼
Application
   │
   ▼
Session Store
   │
   │ abc123
   ▼
user_id = 42
```

In Laravel you may use sessions constantly without thinking about this architecture.

Conceptually:

```php
$userId = session('user_id');
```

Underneath, Laravel may retrieve session state from something such as:

```
file
database
Redis
```

depending on configuration.

---

# Why storing sessions in local memory becomes a scaling problem

Imagine one server:

```
User
 ↓
Server A

session abc123 exists here
```

Fine.

Now your application grows:

```
              Load Balancer
                   │
            ┌──────┴──────┐
            ▼             ▼
         Server A       Server B
```

Login request reaches:

```
Server A
```

and Server A stores:

```
abc123 → user 42
```

Next request reaches:

```
Server B
```

Server B says:

```
abc123?

Never heard of it.
```

Your user appears logged out.

One workaround is **sticky sessions**:

```
User 42
   ↓
always route
   ↓
Server A
```

But that introduces coupling between users and particular servers.

A more common scalable architecture is:

```
              Load Balancer
                   │
          ┌────────┴────────┐
          ▼                 ▼
      Server A          Server B
          │                 │
          └────────┬────────┘
                   ▼
                 Redis
                   │
                   ▼
          shared session state
```

Now either server can resolve:

```
abc123 → user 42
```

Recognize Redis?

You recently learned it as a **cache**.

But Redis can also act as:

```
session store
queue backend
rate-limit store
distributed lock store
```

The technology is the same.

The semantics differ.

And unlike disposable cached product data, losing active session state may log users out—so don't mentally treat every Redis use as “just cache.”

---

# Enter JWT

Another architecture moves some authentication information into a cryptographically signed token.

A JWT looks roughly like:

```
xxxxx.yyyyy.zzzzz
```

It contains three Base64URL-encoded parts:

```
header.payload.signature
```

A decoded payload might contain:

```json
{
  "sub": "42",
  "role": "admin",
  "exp": 1788811200
}
```

The client sends:

```
httpAuthorization: Bearer eyJ...
```

The server can verify the signature.

Simplified:

```
Client
  │
  │ JWT
  ▼
Server
  │
  ├── verify signature
  ├── verify expiration
  └── read claims
```

The server may not need to perform:

```
session_id
    ↓
Redis lookup
```

for every authentication check.

That's why JWT-based authentication is often described as **stateless authentication**.

But that phrase needs care.

---

# 🔐 Signing is NOT encryption

This is one of the most important JWT misconceptions.

Suppose your JWT payload contains:

```json
{
  "user_id": 42,
  "salary": 150000
}
```

A normal signed JWT does **not** automatically hide that information.

Base64URL encoding is reversible.

The signature provides something closer to:

> **Someone without the signing key should not be able to modify these claims and produce a valid signature.**

It does not mean:

> Nobody can read the payload.

So don't put secrets in a normal signed JWT merely because the string looks unreadable.

Think:

```
Encoding
    ≠
Encryption
```

and:

```
Signing
    ≠
Encryption
```

Signing protects **integrity/authenticity**.

Encryption protects **confidentiality**.

Different security properties.

---

# Why signatures are useful

Imagine a naive client-controlled token:

```json
{
  "user_id": 42,
  "role": "user"
}
```

An attacker changes:

```json
{
  "user_id": 42,
  "role": "admin"
}
```

If the server blindly trusts it:

```
💥 privilege escalation
```

With a properly signed token, changing:

```
user → admin
```

invalidates the signature.

Conceptually:

```
payload
   +
secret/private key
   ↓
signature
```

The server verifies:

```
Does this signature match
this exact token content?
```

If the payload changes, verification fails.

---

# 🧩 Sessions vs JWTs

Developers sometimes ask:

> Which one is better?

That's usually the wrong question.

They have different trade-offs.

A server-side session:

```
Cookie
  ↓
session ID
  ↓
server-side state
```

gives you easy central control.

Want to log someone out immediately?

Delete:

```
session abc123
```

Future requests fail authentication.

JWT:

```
Client
  ↓
signed token
  ↓
server verifies locally
```

can reduce centralized session lookups.

But now imagine the token says:

```
valid until 12:00
```

At:

```
10:00
```

you discover the account is compromised.

How do you immediately invalidate the token?

If every server simply verifies:

```
signature ✓
expiration ✓
```

then the token may remain valid until expiration.

Possible strategies include:

```
short-lived access tokens
refresh tokens
revocation lists
token versioning
key rotation
```

But notice what happened.

You started with:

```
"JWT means no server state!"
```

and eventually introduced:

```
revocation database
refresh-token storage
```

State has returned.

Distributed systems have a habit of doing that.

---

# A common practical architecture

Many systems use:

```
Access Token

short lifetime
e.g. 10–15 minutes
```

plus:

```
Refresh Token

longer lifetime
stronger storage/revocation controls
```

Conceptually:

```
Login
  │
  ▼
Access Token + Refresh Token
  │
  ▼

API request
  │
  │ Access Token
  ▼
API
```

When access token expires:

```
Client
  │
  │ Refresh Token
  ▼
Auth Server
  │
  ▼
new Access Token
```

If a refresh token is compromised or revoked, the server can reject future refresh attempts.

The exact design depends heavily on whether you're building:

```
browser app
mobile app
machine-to-machine API
third-party OAuth integration
internal service
```

Authentication architecture should follow the threat model.

---

# 🍪 Where should a browser store tokens?

A common beginner implementation is:

```
TypeScriptlocalStorage.setItem(
    "token",
    accessToken
);
```

Then:

```
TypeScriptfetch("/api/orders", {
    headers: {
        Authorization:
            `Bearer ${localStorage.getItem("token")}`
    }
});
```

This works mechanically.

But consider an XSS vulnerability:

```javascript
const token =
    localStorage.getItem("token");

sendToAttacker(token);
```

JavaScript running in the page can access `localStorage`.

An `HttpOnly` cookie, however, can be configured so browser JavaScript cannot directly read it:

```
httpSet-Cookie:
session=abc123;
HttpOnly;
Secure;
SameSite=Lax
```

`HttpOnly` helps reduce token theft through JavaScript.

`Secure` tells the browser to send it only over HTTPS.

`SameSite` helps control cross-site cookie behavior and is relevant to CSRF defenses.

This doesn't mean:

```
cookies = automatically secure
localStorage = always wrong
```

Authentication security depends on the entire architecture.

But you should understand what each storage mechanism exposes.

---

# 🚨 Authentication vs authorization

Another distinction worth locking in:

**Authentication** asks:

```
Who are you?
```

**Authorization** asks:

```
Are you allowed to do this?
```

Suppose JWT says:

```json
{
  "sub": "42"
}
```

Great.

You authenticated:

```
user 42
```

Now the request is:

```
httpDELETE /users/99
```

You still need authorization:

```
Can user 42 delete user 99?
```

Authentication does not imply permission.

This becomes especially important in APIs.

Never assume:

```
valid token
=
allowed action
```

---

# 🛒 E-commerce example

Suppose your API receives:

```
httpGET /orders/9281
Authorization: Bearer ...
```

The token proves:

```
user_id = 42
```

Your controller does:

```php
$order = Order::findOrFail(9281);

return $order;
```

Security bug.

You've authenticated the user, but didn't authorize access to this order.

User 42 could request:

```
/orders/9282
/orders/9283
/orders/9284
```

until they find someone else's order.

The query needs something like:

```php
$order = Order::where('id', $orderId)
    ->where('user_id', $authenticatedUserId)
    ->firstOrFail();
```

or an authorization policy:

```php
$this->authorize('view', $order);
```

This class of vulnerability is often called **Broken Object Level Authorization (BOLA)** in API security discussions.

Notice the invariant:

```
Authenticated user
may only access resources
they are authorized to access.
```

Again, strong engineering often comes back to identifying invariants.

---

# 🤖 Authentication becomes even more interesting with AI agents

Traditional request:

```
Human
  ↓
API
```

Agentic system:

```
Human
  ↓
AI Agent
  ↓
Tool
  ↓
API
```

Suppose the user says:

```
"Find my last order."
```

The agent might have permission:

```
orders:read
```

Then:

```
"Refund it."
```

requires:

```
orders:refund
```

Should every agent automatically inherit every permission the human has?

Probably not.

A safer architecture follows **least privilege**:

```
Agent
  │
  ├── read orders ✓
  ├── search catalog ✓
  ├── issue refund ?
  └── transfer money ✗
```

High-impact operations might require:

```
additional authorization
user confirmation
short-lived delegated credentials
```

So the old authentication lessons become even more important in agent systems.

---

# 🔄 Connect this to your previous lessons

Your mental map is expanding:

```
HTTP
   ↓
stateless request/response


Cookies
   ↓
client carries state identifier


Sessions
   ↓
state lives server-side


Redis
   ↓
shared session state across servers


JWT
   ↓
signed claims carried by client


Load balancing
   ↓
requests can reach different servers


Caching
   ↓
shared fast storage, but different semantics


Security
   ↓
authentication ≠ authorization
```

And notice the scalability progression:

```
One server
    ↓
local sessions work


Multiple servers
    ↓
shared sessions / sticky routing


Distributed APIs
    ↓
tokens become attractive


Token revocation
    ↓
state may reappear
```

There is rarely a free architectural lunch.

---

## ⚠️ Common misconception

The big one today:

> **“JWT is more modern, therefore JWT is better than sessions.”**

No.

For many traditional web applications, a secure server-side session stored in an `HttpOnly`, `Secure`, appropriately configured cookie is simple and excellent.

JWTs become particularly useful when you genuinely benefit from properties such as:

```
delegation
cross-service verification
OAuth/OIDC ecosystems
machine-to-machine APIs
signed portable claims
```

Don't choose authentication architecture because one technology sounds more scalable.

Choose it based on:

```
clients
trust boundaries
revocation requirements
scaling architecture
threat model
```

---

## 🧩 Quick quiz

**1. HTTP is stateless. How can a server recognize two requests as belonging to the same logged-in browser session?**

The browser can send a cookie containing a session identifier, which the server maps to server-side session state.

**2. Can users read the contents of a normally signed JWT?**

Yes. Signing prevents undetected modification; it doesn't normally encrypt the payload.

**3. A request has a completely valid authentication token. Does that prove the user may access `/orders/9281`?**

No. Authentication establishes identity; authorization determines whether that identity can perform the requested operation on that resource.

---

# 🛠️ Today's practical challenge

Imagine you're designing:

```
shop.example.com
```

with:

```
Laravel API

React/TypeScript frontend

3 application servers

Redis

MySQL
```

Users need to:

```
login
view profile
view their orders
checkout
logout
```

Design the authentication architecture.

Consider:

```
Session or JWT?

Where does authentication state live?

What does the browser store?

How does Server B recognize a user
who originally logged in through Server A?

How does logout invalidate authentication?

How do you prevent user 42 from
accessing user 43's orders?
```

Then add an AI shopping agent that needs:

```
catalog:read
orders:read
```

but **not**:

```
refund:create
payment-method:delete
```

Think about whether the agent should receive the user's full credential or a narrower delegated credential.

That question will lead us later into:

```
OAuth 2.0
scopes
delegation
capability security
agent authorization
```

For deeper reading, the official [MDN HTTP cookies guide](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Cookies?utm_source=chatgpt.com) is an excellent practical reference. For token-based systems, [RFC 7519 — JSON Web Token (JWT)](https://www.rfc-editor.org/rfc/rfc7519?utm_source=chatgpt.com) is the actual standard, and [OWASP's Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html?utm_source=chatgpt.com) is worth bookmarking for production security.

### 🧠 Keep one mental model today

> **HTTP forgets. Your authentication architecture decides where the memory lives.**

Whenever you design authentication, draw this:

```
Client
  │
  │ credential / identifier
  ▼
Server
  │
  ▼
Identity
  │
  ▼
Authorization decision
```

Then ask separately:

```
How is identity proven?

Where is state stored?

How is the credential protected?

How is it revoked?

What is this identity actually
allowed to do?
```

Keeping those questions separate prevents a surprising number of authentication and API-security mistakes.
