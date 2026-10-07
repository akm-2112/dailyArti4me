---
title: "🌅 Morning Tech Lesson #4 — Dependency Injection: Stop Letting Classes Build Their Own Dependencies"
order: 7
volume: 2
category: morning
source: Evening Tech Discovery
---

# 🌅 Morning Tech Lesson #4 — Dependency Injection: Stop Letting Classes Build Their Own Dependencies

So far we've focused on **performance fundamentals**:

**Big-O → hash tables → database indexes**

Today we're deliberately switching tracks to **software design**. We'll return to those performance concepts later through spaced repetition.

Today's idea is one you'll encounter constantly in PHP frameworks and TypeScript backend applications:

> **Dependency Injection (DI)**

Estimated time: **10–15 minutes**

---

## 🧠 Start with a real example

Imagine an e-commerce application.

You need a service that creates an order and sends a confirmation email:

```php
class OrderService
{
    public function createOrder(array $data): void
    {
        // save order...

        $mailer = new EmailService();

        $mailer->send(
            $data['email'],
            'Order confirmed'
        );
    }
}
```

Looks perfectly reasonable.

But there's a hidden architectural problem:

```
OrderService
     │
     │ creates
     ▼
EmailService
```

`OrderService` has decided:

> “I know exactly which email implementation I need, and I'll construct it myself.”

That creates **tight coupling**.

Let's see why that matters.

---

# 🧪 Problem #1 — Testing

Suppose you want to unit-test:

```php
$orderService->createOrder($data);
```

Your test might accidentally send:

```
REAL EMAIL
   ↓
customer@example.com
```

That's obviously undesirable.

Maybe you try:

```
PHPif (app()->environment('testing')) {
    // don't send
}
```

Now production concerns and testing concerns are leaking into your business logic.

Or perhaps `EmailService` calls an external API.

Your test now depends on:

```
Internet
   ↓
email provider
   ↓
API credentials
```

Your "unit test" isn't really isolated anymore.

---

# 🔧 Dependency Injection changes ownership

Instead of:

```php
class OrderService
{
    public function __construct()
    {
        $this->mailer = new EmailService();
    }
}
```

we do:

```php
class OrderService
{
    public function __construct(
        private EmailService $mailer
    ) {}

    public function createOrder(array $data): void
    {
        // save order...

        $this->mailer->send(
            $data['email'],
            'Order confirmed'
        );
    }
}
```

Something outside `OrderService` provides the dependency:

```php
$mailer = new EmailService();

$orderService = new OrderService($mailer);
```

That's **dependency injection**.

Instead of:

```
OrderService
     │
     ▼
construct EmailService
```

we have:

```
Application
    │
    ├── creates EmailService
    │
    ▼
injects it
    │
    ▼
OrderService
```

The class **uses** the dependency but doesn't decide how to construct it.

---

# 🧠 The simplest definition

Dependency Injection means:

> **Give an object the things it needs instead of making the object create those things itself.**

That's really the core idea.

The word *injection* makes it sound more complicated than it is.

---

# But we can improve the design further

Our class still depends directly on:

```
PHPEmailService
```

What happens next year when the business says:

> “Some customers want SMS confirmations.”

You could start adding:

```
PHPif ($customer->prefersSms()) {
    $sms = new SmsService();
} else {
    $email = new EmailService();
}
```

Eventually:

```
OrderService
   │
   ├── EmailService
   ├── SmsService
   ├── WhatsAppService
   ├── PushNotificationService
   └── ...
```

Your order logic starts knowing far too much about notification infrastructure.

Instead, define the behavior the order service actually needs:

```php
interface OrderNotifier
{
    public function sendConfirmation(
        string $recipient
    ): void;
}
```

Then:

```php
class EmailOrderNotifier implements OrderNotifier
{
    public function sendConfirmation(
        string $recipient
    ): void {
        // send email
    }
}
```

And:

```php
class OrderService
{
    public function __construct(
        private OrderNotifier $notifier
    ) {}

    public function createOrder(array $data): void
    {
        // create order...

        $this->notifier->sendConfirmation(
            $data['email']
        );
    }
}
```

Now `OrderService` knows:

```
"I need something capable of
sending an order confirmation."
```

It doesn't care whether that something is:

```
Email
SMS
Push notification
fake test notifier
future technology
```

---

# 🧩 This connects to SOLID

You've probably heard:

```
S O L I D
```

The **D** stands for:

**Dependency Inversion Principle**

A simplified version is:

> High-level business logic shouldn't depend directly on low-level implementation details. Both should depend on abstractions.

Our high-level business logic is:

```
OrderService
```

The low-level implementation detail is:

```
Email provider
```

Instead of:

```
OrderService
      │
      ▼
EmailService
```

we introduce:

```
OrderService
      │
      ▼
OrderNotifier
      ▲
      │
EmailOrderNotifier
```

Notice something important.

The business concept defines what it needs.

The infrastructure adapts to that requirement.

---

# 🐘 Why Laravel seems magical

If you've used Laravel, you've probably written something like:

```php
class OrderController
{
    public function __construct(
        private OrderService $orders
    ) {}
}
```

But you never wrote:

```
PHPnew OrderService(...)
```

So who created it?

Laravel's **service container**.

Conceptually:

```
HTTP request
     │
     ▼
Laravel Router
     │
     ▼
Need OrderController
     │
     ▼
inspect constructor
     │
     └── needs OrderService
              │
              ▼
         create OrderService
              │
              └── needs OrderNotifier
                       │
                       ▼
              resolve implementation
```

The framework constructs the **object graph**.

You might configure:

```php
$this->app->bind(
    OrderNotifier::class,
    EmailOrderNotifier::class
);
```

Now when Laravel sees:

```
PHPpublic function __construct(
    OrderNotifier $notifier
)
```

the container knows:

```
OrderNotifier
      ↓
EmailOrderNotifier
```

This mechanism is called **dependency resolution**.

The container isn't DI itself.

It's a tool that **automates dependency injection**.

That's an important distinction.

---

# 🟦 The same idea in TypeScript

Imagine:

```typescript
class OrderService {
    private stripe = new StripeClient();

    async createOrder() {
        await this.stripe.charge();
    }
}
```

Again:

```
OrderService
     │
     ▼
constructs StripeClient
```

Instead:

```typescript
interface PaymentGateway {
    charge(amount: number): Promise<void>;
}

class OrderService {
    constructor(
        private paymentGateway: PaymentGateway
    ) {}

    async createOrder(amount: number) {
        await this.paymentGateway.charge(amount);
    }
}
```

Production:

```typescript
const service = new OrderService(
    new StripePaymentGateway()
);
```

Testing:

```typescript
class FakePaymentGateway
    implements PaymentGateway {

    async charge(amount: number) {
        // record call instead of charging money
    }
}

const service = new OrderService(
    new FakePaymentGateway()
);
```

Same architectural idea.

No Laravel required.

No framework required.

That's why understanding the concept matters more than memorizing a framework's container syntax.

---

# 🧪 Now testing becomes easy

Suppose we make:

```php
class FakeOrderNotifier implements OrderNotifier
{
    public array $recipients = [];

    public function sendConfirmation(
        string $recipient
    ): void {
        $this->recipients[] = $recipient;
    }
}
```

Our test:

```php
$notifier = new FakeOrderNotifier();

$service = new OrderService($notifier);

$service->createOrder([
    'email' => 'alice@example.com',
]);

assert(
    $notifier->recipients === [
        'alice@example.com'
    ]
);
```

No:

```
SMTP
SendGrid
internet
API key
email inbox
```

We're testing the **business behavior**.

This is one of DI's biggest practical benefits.

---

# ⚠️ Common misconception

A common belief is:

> **“Every class should have an interface because DI is good.”**

No.

This can create meaningless architecture:

```
UserService
IUserService

ProductService
IProductService

OrderService
IOrderService

ReportService
IReportService
```

where every interface has exactly one implementation and no useful abstraction boundary.

Interfaces are valuable when they represent a meaningful **contract or variation point**.

For example:

```
PaymentGateway
    ├── StripeGateway
    ├── PayPalGateway
    └── FakePaymentGateway
```

makes sense.

Likewise:

```
Storage
    ├── S3Storage
    ├── LocalStorage
    └── FakeStorage
```

The goal isn't:

> “Use as many interfaces as possible.”

The goal is:

> **Keep important business logic from becoming unnecessarily coupled to infrastructure details.**

---

# 🔗 Here's the bigger architecture connection

Imagine an AI-commerce application:

```
CheckoutService
      │
      ├── PaymentGateway
      │
      ├── InventoryRepository
      │
      ├── ShippingProvider
      │
      └── NotificationService
```

Those implementations might eventually become:

```
PaymentGateway
   ├── Stripe
   └── another provider

InventoryRepository
   ├── MySQL
   └── remote commerce API

ShippingProvider
   ├── FedEx
   └── UPS

NotificationService
   ├── Email
   └── SMS
```

And perhaps an AI agent becomes another caller:

```
Web Controller ──┐
                 │
Mobile API ──────┼──→ CheckoutService
                 │
AI Agent ────────┘
```

If your business logic is cleanly separated from infrastructure, adding new entry points and implementations becomes much easier.

This is where DI stops being a testing trick and becomes an **architectural boundary tool**.

---

# 🧩 Quick quiz

### 1. What's wrong with this?

```php
class CheckoutService
{
    public function checkout()
    {
        $stripe = new StripeClient();

        $stripe->charge(...);
    }
}
```

The class is tightly coupled to a concrete payment implementation and controls its construction.

A better boundary might be:

```
PHPPaymentGateway
```

injected through the constructor.

---

### 2. Is this dependency injection?

```php
$gateway = new StripeGateway();

$checkout = new CheckoutService(
    $gateway
);
```

**Yes.**

You don't need a DI container.

The dependency was created externally and injected into the object.

---

### 3. What's the difference between DI and a DI container?

**Dependency Injection** is the design technique:

```
give objects their dependencies
```

A **DI container** is infrastructure that automates:

```
constructing
resolving
wiring
injecting
```

those dependencies.

---

# 🛠️ Today's practical challenge

You're reviewing this PHP code:

```php
class ReportService
{
    public function generate(
        int $customerId
    ): void {
        $database = new MySqlDatabase();

        $storage = new S3Storage();

        $mailer = new SendGridMailer();

        $report = $database
            ->loadCustomerData($customerId);

        $path = $storage
            ->save($report);

        $mailer
            ->sendReport($customerId, $path);
    }
}
```

Don't write code immediately.

First identify the architecture.

`ReportService` currently controls:

```
MySQL
S3
SendGrid
```

Your challenge is to redesign it so the service thinks instead in terms of capabilities:

```
CustomerDataSource

ReportStorage

ReportNotifier
```

Then imagine testing it using:

```
FakeCustomerDataSource

InMemoryStorage

FakeNotifier
```

Ask yourself:

> If tomorrow S3 becomes Google Cloud Storage, how much should `ReportService` need to change?

A strong design aims for the answer:

**Ideally, not at all.**

For deeper reading, Laravel's official [Service Container documentation](https://laravel.com/docs/container?utm_source=chatgpt.com) is worth reading now that you know what problem the container is actually solving. PHP's [Interfaces documentation](https://www.php.net/manual/en/language.oop5.interfaces.php?utm_source=chatgpt.com) is also a useful language-level reference.

### 🧠 Keep one mental model today

> **An object should usually receive important external collaborators rather than secretly constructing them itself.**

Or even shorter:

```
Don't ask:

"How do I create Stripe?"

Ask:

"What capability does my business logic need?"
```

That shift—from **implementation** to **capability**—is one of the foundations of maintainable software architecture.
