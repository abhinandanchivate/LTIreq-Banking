
---

# 🔴 Starting Point – Poorly Designed Payment Module

```java
class PaymentProcessor {

    public void process(String type, double amount) {

        if (amount <= 0) {
            throw new IllegalArgumentException("Invalid amount");
        }

        if (type.equals("CARD")) {
            System.out.println("Processing card");
        } else if (type.equals("UPI")) {
            System.out.println("Processing UPI");
        }

        System.out.println("Saving transaction");
        System.out.println("Generating receipt");
        System.out.println("Sending notification");
    }
}
```

Now we refactor step by step.

---

# 1️⃣ SRP – Single Responsibility Principle

---

## Concept (What / Why / When / Where / How)

**What**
One class should have only one job.

**Why**
If one class handles many responsibilities, small changes affect unrelated logic.

**When**
When validation, business logic, persistence, and notification are mixed.

**Where**
Any service class doing too many things.

**How**
Split responsibilities into separate classes.

---

## Scenario

Receipt format needs to change.

---

## Goal

Change receipt logic without touching payment logic.

---

## What Can Go Wrong

* Changing receipt breaks payment flow.
* Hard to test.
* Code becomes large and messy.

---

## ❌ Incorrect Example

All logic inside `PaymentProcessor`.

---

## Why It Fails

It does:

* Validation
* Payment handling
* Saving
* Receipt
* Notification

Many reasons to change.

---

## ✅ Correct Implementation

```java
class PaymentValidator {
    void validate(double amount) {
        if (amount <= 0)
            throw new IllegalArgumentException("Invalid amount");
    }
}

class ReceiptService {
    void generate() {
        System.out.println("Receipt generated");
    }
}

class NotificationService {
    void notifyUser() {
        System.out.println("Notification sent");
    }
}

class PaymentService {

    private PaymentValidator validator = new PaymentValidator();
    private ReceiptService receipt = new ReceiptService();
    private NotificationService notification = new NotificationService();

    public void process(double amount) {
        validator.validate(amount);
        System.out.println("Processing payment");
        receipt.generate();
        notification.notifyUser();
    }
}
```

---

## Explanation

Now:

* Change receipt → modify `ReceiptService`
* Change notification → modify `NotificationService`
* Payment logic remains untouched

---

## Common Mistakes

* One large “Service” class.
* Mixing business and infrastructure code.

---

## Best Practices

* Extract collaborators.
* Keep service as coordinator only.

---

## Key Takeaways

One class → one responsibility.

---

# 2️⃣ OCP – Open Closed Principle

---

## Concept

Open for extension, closed for modification.

---

## Scenario

Add new payment type: WALLET.

---

## Goal

Add new type without editing existing code.

---

## What Can Go Wrong

If using `if-else`, every new type modifies existing code.

---

## ❌ Incorrect Example

```java
if(type.equals("CARD")) ...
else if(type.equals("UPI")) ...
else if(type.equals("WALLET")) ...
```

---

## Why It Fails

Every new feature changes old code.

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

To add new type:

* Create new class.
* No modification needed.

---

## Common Mistakes

* Large conditional blocks.
* Modifying old tested classes.

---

## Best Practices

* Use interfaces.
* Prefer polymorphism.

---

## Key Takeaways

Extend by adding new classes.

---

# 3️⃣ LSP – Liskov Substitution Principle

---

## Concept

Child class must behave like parent.

---

## Scenario

Every payment method must process payment.

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

But this breaks contract.

---

## ✅ Correct Implementation

Only create valid implementations.

Or redesign abstraction if behavior differs.

---

## Explanation

If a class cannot follow base contract, it should not implement that interface.

---

## Common Mistakes

* Forcing inheritance.
* Throwing unsupported exceptions in subclass.

---

## Best Practices

* Ensure subclass honors parent behavior.
* Design clear contracts.

---

## Key Takeaways

Subtypes must be substitutable.

---

# 4️⃣ ISP – Interface Segregation Principle

---

## Concept

Clients should not depend on methods they don’t use.

---

## Scenario

Some payment types support refund, some don’t.

---

## ❌ Incorrect Example

```java
interface PaymentOperations {
    void pay();
    void refund();
}
```

Some classes may not support refund.

---

## Why It Fails

They must implement unused methods.

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

---

## Explanation

Classes implement only what they need.

---

## Common Mistakes

* Creating large interfaces.
* Mixing unrelated methods.

---

## Best Practices

* Keep interfaces small.
* Focus on single capability.

---

## Key Takeaways

Small interfaces are better.

---

# 5️⃣ DIP – Dependency Inversion Principle

---

## Concept

High-level modules should depend on abstraction, not concrete classes.

---

## Scenario

PaymentService directly creates CardPayment.

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

* Hard to test.
* Cannot switch implementation.
* Tight coupling.

---

## ✅ Correct Implementation

```java
class PaymentService {

    private PaymentMethod method;

    PaymentService(PaymentMethod method) {
        this.method = method;
    }

    public void process(double amount) {
        method.pay(amount);
    }
}
```

---

## Explanation

Now:

* Can inject CardPayment.
* Can inject WalletPayment.
* Can inject MockPayment for testing.

---

## Common Mistakes

* Using `new` inside business classes.
* Not using abstraction.

---

## Best Practices

* Depend on interfaces.
* Inject dependencies via constructor.

---

# 🏁 Final Summary

| Principle | Problem                   | Solution               |
| --------- | ------------------------- | ---------------------- |
| SRP       | One class doing many jobs | Split responsibilities |
| OCP       | if-else modification      | Use polymorphism       |
| LSP       | Subclass breaks contract  | Respect behavior       |
| ISP       | Large interface           | Split into smaller     |
| DIP       | Concrete dependency       | Depend on abstraction  |

---
