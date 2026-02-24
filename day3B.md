
# 1️⃣ Factory Pattern

---

## Concept (What / Why / When / Where / How)

**What**
Factory creates objects.

**Why**
So object creation logic is not spread everywhere.

**When**
When object type depends on input.

**Where**

* Payment types (Card, UPI)
* Notification types (SMS, Email)
* File parsers (CSV, JSON)

**How**
Create a central class that returns the correct object.

---

## Scenario

System needs to create different notification senders.

---

## Goal

Avoid writing `if-else` everywhere.

---

## What Can Go Wrong

* Multiple `if(type)` blocks
* Hard to add new types
* Duplicate creation logic

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

Every new type → modify this code everywhere.

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

class NotificationFactory {
    static Notification create(String type) {
        if(type.equals("SMS")) return new SmsSender();
        if(type.equals("EMAIL")) return new EmailSender();
        throw new IllegalArgumentException();
    }
}
```

---

## Explanation

Now object creation is centralized.

---

## Key Takeaway

Factory isolates object creation.

---

# 2️⃣ Builder Pattern

---

## Concept

**What**
Build complex objects step-by-step.

**Why**
Avoid long constructors.

**When**
When object has many optional fields.

**Where**

* User profile
* Order details
* Configuration objects

---

## Scenario

Create a User object with many optional fields.

---

## ❌ Incorrect Example

```java
new User("Abhi", "Pune", 30, true, false, "Trainer");
```

Hard to understand parameters.

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

## Key Takeaway

Builder improves readability.

---

# 3️⃣ Strategy Pattern

---

## Concept

**What**
Multiple algorithms, same interface.

**Why**
Avoid `if-else` for behavior change.

**Where**

* Sorting strategies
* Discount calculation
* Payment fee calculation

---

## Scenario

Different discount rules.

---

## ❌ Incorrect Example

```java
if(type.equals("NEW"))
    discount = amount * 0.10;
else if(type.equals("OLD"))
    discount = amount * 0.05;
```

---

## ✅ Correct Implementation

```java
interface DiscountStrategy {
    double apply(double amount);
}

class NewUserDiscount implements DiscountStrategy {
    public double apply(double amount) {
        return amount * 0.10;
    }
}

class OldUserDiscount implements DiscountStrategy {
    public double apply(double amount) {
        return amount * 0.05;
    }
}
```

Usage:

```java
DiscountStrategy strategy = new NewUserDiscount();
double discount = strategy.apply(1000);
```

---

## Key Takeaway

Strategy removes conditional logic.

---

# 4️⃣ Chain of Responsibility

---

## Concept

**What**
Pass request through chain of handlers.

**Where**

* Validation pipeline
* Authentication filters
* Approval workflows

---

## Scenario

Request must pass through:

1. Authentication
2. Authorization
3. Validation

---

## ✅ Implementation

```java
abstract class Handler {

    protected Handler next;

    public Handler linkWith(Handler next) {
        this.next = next;
        return next;
    }

    public abstract void handle();
}

class AuthHandler extends Handler {
    public void handle() {
        System.out.println("Auth OK");
        if(next != null) next.handle();
    }
}
```

---

## Key Takeaway

Each step is independent.

---

# 5️⃣ Observer Pattern

---

## Concept

**What**
Notify multiple listeners when event occurs.

**Where**

* Order placed
* File uploaded
* Payment completed

---

## Example

```java
interface Listener {
    void update();
}

class SmsListener implements Listener {
    public void update() {
        System.out.println("SMS sent");
    }
}
```

---

## Key Takeaway

Loose coupling between event and listeners.

---

# 6️⃣ Decorator Pattern

---

## Concept

**What**
Add extra behavior without modifying class.

**Where**

* Add logging
* Add encryption
* Add caching

---

## Example

```java
class LoggingDecorator implements Runnable {

    private Runnable task;

    LoggingDecorator(Runnable task) {
        this.task = task;
    }

    public void run() {
        System.out.println("Start");
        task.run();
        System.out.println("End");
    }
}
```

---

## Key Takeaway

Extend behavior dynamically.

---

# 7️⃣ Adapter Pattern

---

## Concept

**What**
Convert one interface into another.

**Where**

* Integrating legacy system
* Third-party APIs

---

## Example

```java
class OldPrinter {
    void printData(String msg) {
        System.out.println(msg);
    }
}

interface Printer {
    void print(String msg);
}

class PrinterAdapter implements Printer {

    private OldPrinter oldPrinter = new OldPrinter();

    public void print(String msg) {
        oldPrinter.printData(msg);
    }
}
```

---

# 🏁 Final Summary

| Pattern   | Use Case                         |
| --------- | -------------------------------- |
| Factory   | Centralized object creation      |
| Builder   | Complex object construction      |
| Strategy  | Change algorithm at runtime      |
| Chain     | Sequential processing            |
| Observer  | Event notification               |
| Decorator | Add behavior dynamically         |
| Adapter   | Integrate incompatible interface |

---


