# 🧨 CASE STUDY: Poorly Designed Payment System (Version 1 - Bad Design)

## 📁 Folder Structure (Before Refactoring)

```
payment-app/
 ├── PaymentProcessor.java
 ├── PaymentService.java
 ├── MySQLConnection.java
 └── Main.java
```

---

# 🔴 1️⃣ PaymentProcessor.java (Major Problems)

```java
public class PaymentProcessor {

    public void process(String paymentType, double amount) {

        // Validation
        if (amount <= 0) {
            throw new RuntimeException("Invalid amount");
        }

        // Business Logic
        if (paymentType.equals("CARD")) {
            System.out.println("Processing Card Payment");
        } else if (paymentType.equals("UPI")) {
            System.out.println("Processing UPI Payment");
        } else if (paymentType.equals("NETBANKING")) {
            System.out.println("Processing Net Banking Payment");
        }

        // Fraud Rule
        if (amount > 100000) {
            System.out.println("Manual fraud review required");
        }

        // Save to DB
        MySQLConnection db = new MySQLConnection();
        db.save(paymentType, amount);

        // Send Notification
        System.out.println("Sending SMS notification");

        // Logging
        System.out.println("Payment completed successfully");
    }
}
```

---

# 🔴 2️⃣ MySQLConnection.java

```java
public class MySQLConnection {

    public void save(String type, double amount) {
        System.out.println("Connecting to MySQL DB...");
        System.out.println("Saving payment record...");
    }
}
```

---

# 🔴 3️⃣ PaymentService.java

```java
public interface PaymentService {
    void pay(double amount);
    void refund(double amount);
    void generateReport();
}
```

---

# 🔎 Now Let’s Identify Violations Properly

---

# ❌ SRP Violation

`PaymentProcessor` is doing:

* Validation
* Business logic
* Fraud check
* DB access
* Notification
* Logging

👉 6 responsibilities in one class.

If fraud logic changes → modify class
If DB changes → modify class
If notification changes → modify class

Too much coupling.

---

# ❌ OCP Violation

```java
if (paymentType.equals("CARD")) { ... }
```

New method like `WALLET`?

You must edit existing code.

Not extensible.

---

# ❌ DIP Violation

```java
MySQLConnection db = new MySQLConnection();
```

High-level logic directly depends on concrete DB.

Cannot switch to PostgreSQL easily.

---

# ❌ ISP Violation

```java
public interface PaymentService {
    void pay();
    void refund();
    void generateReport();
}
```

UPI may not support refund.
But forced to implement it.

Fat interface.

---

# ❌ LSP Violation Example

```java
public class UpiPayment implements PaymentService {

    @Override
    public void pay(double amount) {
        System.out.println("UPI Payment");
    }

    @Override
    public void refund(double amount) {
        throw new UnsupportedOperationException("UPI refund not supported");
    }

    @Override
    public void generateReport() {}
}
```

Subclass breaks expected behavior.
Base interface promises refund.
Subclass throws exception.

LSP broken.

---

---

# 🛠 REFACTORING — VERSION 2 (Clean Design)

We now redesign properly.

---

# 📁 Folder Structure (After Refactoring)

```
payment-app/
 ├── processor/
 │     └── PaymentProcessor.java
 │
 ├── method/
 │     ├── PaymentMethod.java
 │     ├── CardPayment.java
 │     └── UpiPayment.java
 │
 ├── validation/
 │     └── PaymentValidator.java
 │
 ├── fraud/
 │     └── FraudChecker.java
 │
 ├── repository/
 │     ├── PaymentRepository.java
 │     └── MySQLPaymentRepository.java
 │
 ├── notification/
 │     ├── NotificationService.java
 │     └── SmsNotificationService.java
 │
 └── Main.java
```

---

# ✅ Step 1 — Fix SRP

### validation/PaymentValidator.java

```java
package validation;

public class PaymentValidator {

    public void validate(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Invalid amount");
        }
    }
}
```

Now validation is separate.

---

# ✅ Step 2 — Fix OCP Using Strategy Pattern

### method/PaymentMethod.java

```java
package method;

public interface PaymentMethod {
    void pay(double amount);
}
```

### method/CardPayment.java

```java
package method;

public class CardPayment implements PaymentMethod {

    @Override
    public void pay(double amount) {
        System.out.println("Processing Card Payment: " + amount);
    }
}
```

### method/UpiPayment.java

```java
package method;

public class UpiPayment implements PaymentMethod {

    @Override
    public void pay(double amount) {
        System.out.println("Processing UPI Payment: " + amount);
    }
}
```

Now adding `WalletPayment` requires new class only.

No modification needed.

OCP fixed.

---

# ✅ Step 3 — Fix DIP

### repository/PaymentRepository.java

```java
package repository;

public interface PaymentRepository {
    void save(String type, double amount);
}
```

### repository/MySQLPaymentRepository.java

```java
package repository;

public class MySQLPaymentRepository implements PaymentRepository {

    @Override
    public void save(String type, double amount) {
        System.out.println("Saving to MySQL database");
    }
}
```

Now processor depends on abstraction.

DIP fixed.

---

# ✅ Step 4 — Fix ISP

Instead of fat interface:

### Separate Interfaces

```java
public interface Payment {
    void pay(double amount);
}

public interface Refundable {
    void refund(double amount);
}

public interface Reportable {
    void generateReport();
}
```

Classes implement only what they support.

ISP fixed.

---

# ✅ Step 5 — Fraud Checker

### fraud/FraudChecker.java

```java
package fraud;

public class FraudChecker {

    public void check(double amount) {
        if (amount > 100000) {
            System.out.println("Manual fraud review required");
        }
    }
}
```

---

# ✅ Step 6 — Notification Service

### notification/NotificationService.java

```java
package notification;

public interface NotificationService {
    void notifyUser(String message);
}
```

### notification/SmsNotificationService.java

```java
package notification;

public class SmsNotificationService implements NotificationService {

    @Override
    public void notifyUser(String message) {
        System.out.println("Sending SMS: " + message);
    }
}
```

---

# ✅ Final Clean Processor

### processor/PaymentProcessor.java

```java
package processor;

import fraud.FraudChecker;
import method.PaymentMethod;
import repository.PaymentRepository;
import validation.PaymentValidator;
import notification.NotificationService;

public class PaymentProcessor {

    private final PaymentValidator validator;
    private final FraudChecker fraudChecker;
    private final PaymentRepository repository;
    private final NotificationService notificationService;

    public PaymentProcessor(
            PaymentValidator validator,
            FraudChecker fraudChecker,
            PaymentRepository repository,
            NotificationService notificationService) {

        this.validator = validator;
        this.fraudChecker = fraudChecker;
        this.repository = repository;
        this.notificationService = notificationService;
    }

    public void process(PaymentMethod method, String type, double amount) {

        validator.validate(amount);

        fraudChecker.check(amount);

        method.pay(amount);

        repository.save(type, amount);

        notificationService.notifyUser("Payment successful");
    }
}
```

---

# 🧠 What Changed Architecturally?

Before:

* Tight coupling
* Hard to test
* Hard to extend
* Violates all SOLID

After:

* Loosely coupled
* Testable (mock repository easily)
* Extensible
* Clean responsibilities

---


