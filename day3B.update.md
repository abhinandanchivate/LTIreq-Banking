
---

# 1️⃣ Dependency Injection (DI)

---

## Concept (What / Why / When / Where / How)

**What**
Dependency Injection means providing required objects from outside instead of creating them inside the class.

**Why**
To reduce tight coupling and increase flexibility.

**When**
When one class depends on another class to perform work.

**Where**

* Service → Notification
* Service → Repository
* Strategy injection

**How**
Pass dependency through constructor (or setter).

---

## Scenario

PaymentService needs a Notification service to send confirmation.

---

## Goal

Allow PaymentService to work with SMS or Email without modification.

---

## What Can Go Wrong

* Tight coupling
* Hard to test
* Hard to switch implementation

---

## ❌ Incorrect Example

```java
class PaymentService {

    private SmsSender sender = new SmsSender();

    public void process() {
        sender.send();
    }
}
```

---

## Why It Fails

* PaymentService directly depends on SmsSender.
* Cannot switch to EmailSender without modifying class.
* Cannot inject mock for testing.

---

## ✅ Correct Implementation

```java
interface Notification {
    void send();
}

class SmsSender implements Notification {
    public void send() {
        System.out.println("SMS sent");
    }
}

class EmailSender implements Notification {
    public void send() {
        System.out.println("Email sent");
    }
}

class PaymentService {

    private Notification sender;

    public PaymentService(Notification sender) {
        this.sender = sender;
    }

    public void process() {
        sender.send();
    }
}
```

Usage:

```java
PaymentService service = new PaymentService(new SmsSender());
service.process();
```

---

## Explanation

* PaymentService depends on abstraction (Notification).
* Implementation is injected from outside.
* Switching implementation requires no change in PaymentService.
* Improves maintainability and testability.

---

# 2️⃣ Factory Pattern

---

## Concept (What / Why / When / Where / How)

**What**
Factory centralizes object creation.

**Why**
To avoid spreading object creation logic everywhere.

**When**
When object type depends on input.

**Where**

* Payment types
* Notification types
* File parsers

**How**
Create a class responsible for returning correct object.

---

## Scenario

System must create SMS or Email sender dynamically.

---

## Goal

Avoid multiple `if-else` blocks across system.

---

## What Can Go Wrong

* Duplicate creation logic
* Hard to add new types
* Code repetition

---

## ❌ Incorrect Example

```java
Notification sender;

if(type.equals("SMS"))
    sender = new SmsSender();
else if(type.equals("EMAIL"))
    sender = new EmailSender();
```

---

## Why It Fails

* Every new type requires modification.
* Creation logic repeated in many places.

---

## ✅ Correct Implementation

```java
class NotificationFactory {

    public static Notification create(String type) {
        if(type.equals("SMS")) return new SmsSender();
        if(type.equals("EMAIL")) return new EmailSender();
        throw new IllegalArgumentException();
    }
}
```

Usage:

```java
Notification sender = NotificationFactory.create("SMS");
sender.send();
```

---

## Explanation

* Object creation logic is centralized.
* Business code does not worry about which class to instantiate.
* Only factory changes when new type is added.

---

## Key Takeaway

Separate creation from usage.

---

# 3️⃣ Builder Pattern

---

## Concept (What / Why / When / Where / How)

**What**
Build complex objects step-by-step.

**Why**
To avoid confusing long constructors.

**When**
When object has many optional parameters.

**Where**

* User profile
* Order object
* Configuration object

**How**
Use inner Builder class.

---

## Scenario

Create User object with many fields.

---

## Goal

Improve readability and reduce errors.

---

## What Can Go Wrong

* Hard to remember parameter order
* Code becomes unreadable

---

## ❌ Incorrect Example

```java
new User("Abhi", "Pune", 30, true, false);
```

---

## Why It Fails

* Parameter meaning unclear.
* Easy to swap values mistakenly.

---

## ✅ Correct Implementation

```java
class User {

    private String name;
    private String city;
    private int age;

    private User() {}

    public static class Builder {

        private User user = new User();

        public Builder name(String name) {
            user.name = name;
            return this;
        }

        public Builder city(String city) {
            user.city = city;
            return this;
        }

        public Builder age(int age) {
            user.age = age;
            return this;
        }

        public User build() {
            return user;
        }
    }
}
```

Usage:

```java
User user = new User.Builder()
        .name("Abhi")
        .city("Pune")
        .age(30)
        .build();
```

---

## Explanation

* Each property is clearly set.
* Code is readable.
* No confusion in parameter order.

---

## Key Takeaway

Builder improves clarity and safety.

---

# 4️⃣ Strategy Pattern

---

## Concept (What / Why / When / Where / How)

**What**
Encapsulate algorithms into separate classes.

**Why**
To remove conditional logic.

**When**
When behavior varies based on type.

**Where**

* Payment fee calculation
* Discount rules
* Sorting logic

**How**
Use interface and multiple implementations.

---

## Scenario

Different fee calculation for Card and UPI.

---

## Goal

Add new payment type without modifying existing code.

---

## What Can Go Wrong

* Large if-else blocks
* Hard to maintain

---

## ❌ Incorrect Example

```java
if(type.equals("CARD"))
    fee = amount * 0.02;
else if(type.equals("UPI"))
    fee = amount * 0.01;
```

---

## Why It Fails

* Every new type modifies existing code.
* Violates open/closed principle.

---

## ✅ Correct Implementation

```java
interface PaymentStrategy {
    double calculateFee(double amount);
}

class CardStrategy implements PaymentStrategy {
    public double calculateFee(double amount) {
        return amount * 0.02;
    }
}

class UpiStrategy implements PaymentStrategy {
    public double calculateFee(double amount) {
        return amount * 0.01;
    }
}
```

Usage:

```java
PaymentStrategy strategy = new CardStrategy();
double fee = strategy.calculateFee(1000);
```

---

## Explanation

* Each algorithm is isolated.
* Easy to extend.
* Clean separation of logic.

---

## Key Takeaway

Use polymorphism instead of conditionals.

---

# 5️⃣ Template Method Pattern

---

## Concept (What / Why / When / Where / How)

**What**
Define fixed workflow and allow subclasses to customize parts.

**Why**
To avoid duplicating process flow.

**When**
When structure remains same but steps differ.

**Where**

* Payment pipeline
* Order processing
* Data import

**How**
Use abstract class and final template method.

---

## Scenario

All payments follow validate → deduct → receipt.

---

## Goal

Reuse process structure.

---

## What Can Go Wrong

* Duplicate code in subclasses
* Inconsistent process flow

---

## ❌ Incorrect Example

Each payment type duplicates full process.

---

## Why It Fails

* Code repetition.
* Hard to maintain.

---

## ✅ Correct Implementation

```java
abstract class PaymentProcessor {

    public final void process(double amount) {
        validate(amount);
        deduct(amount);
        receipt();
    }

    private void validate(double amount) {
        if(amount <= 0)
            throw new IllegalArgumentException();
    }

    protected abstract void deduct(double amount);

    private void receipt() {
        System.out.println("Receipt generated");
    }
}
```

---

## Explanation

* Flow is fixed.
* Only deduction changes.
* Prevents duplication.

---

## Key Takeaway

Control algorithm structure centrally.

---

Refactoring Guru (Clear explanations + diagrams)
https://refactoring.guru/design-patterns

Gang of Four – Design Patterns Book
Design Patterns: Elements of Reusable Object-Oriented Software
Authors: Erich Gamma, Richard Helm, Ralph Johnson, John Vlissides

Head First Design Patterns – O’Reilly
Beginner-friendly but conceptually strong

Martin Fowler – Patterns of Enterprise Application Architecture
https://martinfowler.com
