
---

# 🧨 BAD PAYMENT MODULE (Starting Point)

```java
class PaymentProcessor {

    public void process(String type, double amount) {

        if (amount <= 0) {
            throw new IllegalArgumentException("Invalid amount");
        }

        if (type.equals("CARD")) {
            System.out.println("Processing card payment");
        } else if (type.equals("UPI")) {
            System.out.println("Processing UPI payment");
        }

        System.out.println("Saving to database");
        System.out.println("Generating receipt");
        System.out.println("Sending SMS");
    }
}
```

Problems exist.
Now we identify and fix them one by one.

---

# 1️⃣ SRP – Single Responsibility Principle

---

## Concept (What / Why / When / Where / How)

**What**
One class → one job.

**Why**
If a class has many jobs, small changes break many things.

**When**
When validation, logging, DB, payment logic are mixed.

**Where (Banking Example)**
When customer pays ₹2,000 using card:

* validate
* process payment
* save transaction
* generate receipt
* send SMS

All inside one class → risky.

**How**
Split responsibilities into separate classes.

---

## Scenario

Bank wants to change SMS provider.

Current design forces editing PaymentProcessor.

---

## Goal

Make SMS change without touching payment logic.

---

## What Can Go Wrong

* Change in receipt breaks payment.
* Change in DB breaks payment.
* Hard to test.
* Hard to maintain.

---

## ❌ Incorrect Example (Already Seen Above)

One class doing everything.

---

## Why It Fails

It has 5 responsibilities:

* validation
* payment routing
* persistence
* receipt
* notification

Many reasons to change.

---

## ✅ Correct Implementation

```java
class PaymentValidator {
    void validate(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Invalid amount");
        }
    }
}

class ReceiptService {
    void generate() {
        System.out.println("Receipt generated");
    }
}

class NotificationService {
    void sendSMS() {
        System.out.println("SMS sent");
    }
}

class PaymentService {

    private PaymentValidator validator = new PaymentValidator();
    private ReceiptService receipt = new ReceiptService();
    private NotificationService notification = new NotificationService();

    public void process(String type, double amount) {
        validator.validate(amount);
        System.out.println("Processing payment");
        receipt.generate();
        notification.sendSMS();
    }
}
```

---

## Explanation

Now:

* Change SMS → modify NotificationService only.
* Change receipt → modify ReceiptService only.

Each class has one job.

---

## Common Mistakes

* Making one big “Service” class.
* Mixing business logic with infrastructure logic.

---

## Best Practices

* Separate validation.
* Separate communication.
* Keep service as coordinator only.

---

## Key Takeaways

One class → one responsibility.

---

# 2️⃣ OCP – Open Closed Principle

---

## Concept

You should add new features **without modifying old code**.

---

## Scenario

Bank introduces new payment type: WALLET.

---

## Goal

Add Wallet payment without editing PaymentService.

---

## What Can Go Wrong

If you keep using:

```java
if(type.equals("CARD")) ...
```

You must edit this every time.

Risk:

* Break existing logic
* Increase bugs

---

## ❌ Incorrect Example

```java
if (type.equals("CARD")) {
    System.out.println("Card payment");
} else if (type.equals("UPI")) {
    System.out.println("UPI payment");
} else if (type.equals("WALLET")) {
    System.out.println("Wallet payment");
}
```

---

## Why It Fails

Every new payment → modify this block.

---

## ✅ Correct Implementation

```java
interface PaymentMethod {
    void pay(double amount);
}

class CardPayment implements PaymentMethod {
    public void pay(double amount) {
        System.out.println("Card payment");
    }
}

class WalletPayment implements PaymentMethod {
    public void pay(double amount) {
        System.out.println("Wallet payment");
    }
}

class PaymentService {
    public void process(PaymentMethod method, double amount) {
        method.pay(amount);
    }
}
```

---

## Explanation

To add new payment:

* Create new class.
* Do not modify existing code.

---

## Key Takeaways

Avoid large if-else chains.

---

# 3️⃣ LSP – Liskov Substitution Principle

---

## Concept

Child class must behave like parent.

---

## Scenario

We expect every PaymentMethod to process payment.

---

## ❌ Incorrect Example

```java
class FreePayment implements PaymentMethod {
    public void pay(double amount) {
        throw new UnsupportedOperationException();
    }
}
```

---

## Why It Fails

Caller expects pay() to work.

But this breaks behavior.

In banking, this could:

* Break transaction flow
* Skip debit step

---

## ✅ Correct Approach

Only create subclasses that truly support payment.

Or create proper abstraction.

---

## Key Takeaway

If it cannot behave like parent, don’t inherit.

---

# 4️⃣ ISP – Interface Segregation Principle

---

## Concept

Do not force classes to implement unused methods.

---

## ❌ Incorrect Example

```java
interface PaymentOperations {
    void pay();
    void refund();
    void generateReport();
}
```

Some payment types may not support refund.

---

## Why It Fails

They will write:

```java
throw new UnsupportedOperationException();
```

Bad design.

---

## ✅ Correct Implementation

```java
interface Payable {
    void pay();
}

interface Refundable {
    void refund();
}
```

Small, focused interfaces.

---

## Key Takeaway

Keep interfaces small and specific.

---

# 5️⃣ DIP – Dependency Inversion Principle

---

## Concept

High-level class should not depend on concrete class.

---

## ❌ Incorrect Example

```java
class PaymentService {
    private CardPayment card = new CardPayment();

    public void process(double amount) {
        card.pay(amount);
    }
}
```

---

## Why It Fails

* Cannot switch gateway
* Hard to test
* Tightly coupled

---

## ✅ Correct Implementation

```java
class PaymentService {

    private final PaymentMethod method;

    PaymentService(PaymentMethod method) {
        this.method = method;
    }

    public void process(double amount) {
        method.pay(amount);
    }
}
```

Now you can inject:

* CardPayment
* WalletPayment
* MockPayment

---

## Key Takeaway

Depend on interface, not concrete class.

---


