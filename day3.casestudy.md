

# 🏦 CASE STUDY

# “Payment Module Collapse at QuickPay Systems”

---

## 📖 Background Story

QuickPay Systems is a mid-sized fintech company handling:

* Card payments
* UPI transfers
* Wallet transactions

Initially, the engineering team built a simple payment module. It worked fine when:

* Only 2 payment types existed
* Refunds were rare
* Fraud rules were basic
* Database was fixed

But within 1 year:

* Wallet payment added
* Refund feature expanded
* Fraud rules changed
* Database migration required
* Email notifications introduced

Suddenly:

* Every change required modifying the same file
* Refund started failing for some methods
* Tests became hard to write
* Production bugs increased

A refactoring initiative was launched.

---

# 🔴 PART 1 — ORIGINAL DESIGN (Problem Version)

---

## 📁 Folder Structure (Before Refactoring)

```text
payment-app/
 ├── PaymentProcessor.java
 ├── PaymentService.java
 ├── MySQLConnection.java
 ├── CardPayment.java
 ├── UpiPayment.java
 └── Main.java
```

---

## 🔴 PaymentService.java (Fat Interface – ISP Violation)

```java
public interface PaymentService {
    void pay(double amount);
    void refund(double amount);
    void generateReport();
}
```

Problem:

* Every payment must implement refund.
* Every payment must implement report.
* Even if not supported.

---

## 🔴 UpiPayment.java (LSP Violation)

```java
public class UpiPayment implements PaymentService {

    @Override
    public void pay(double amount) {
        System.out.println("Processing UPI Payment");
    }

    @Override
    public void refund(double amount) {
        throw new UnsupportedOperationException("Refund not supported");
    }

    @Override
    public void generateReport() { }
}
```

Issue:

* Base contract promises refund.
* Subclass breaks behavior.
* Production bug: refund crashes.

---

## 🔴 MySQLConnection.java (DIP Violation)

```java
public class MySQLConnection {

    public void save(String type, double amount) {
        System.out.println("Saving to MySQL database");
    }
}
```

---

## 🔴 PaymentProcessor.java (SRP + OCP + DIP Violations)

```java
public class PaymentProcessor {

    public void process(String paymentType, double amount) {

        // validation
        if (amount <= 0) {
            throw new RuntimeException("Invalid amount");
        }

        // payment selection (OCP violation)
        PaymentService service;

        if (paymentType.equals("CARD")) {
            service = new CardPayment();
        } else if (paymentType.equals("UPI")) {
            service = new UpiPayment();
        } else {
            throw new RuntimeException("Unsupported type");
        }

        service.pay(amount);

        // fraud rule
        if (amount > 100000) {
            System.out.println("Manual fraud review required");
        }

        // direct DB dependency
        MySQLConnection db = new MySQLConnection();
        db.save(paymentType, amount);

        // notification
        System.out.println("Sending SMS");
    }
}
```

---

# ❌ What Went Wrong (Principle Breakdown)

| Principle | Violation                                              |
| --------- | ------------------------------------------------------ |
| SRP       | Processor handling validation, fraud, DB, notification |
| OCP       | Adding Wallet requires modifying if/else               |
| LSP       | UPI refund throws exception                            |
| ISP       | Fat PaymentService                                     |
| DIP       | Direct MySQL dependency                                |

---

# 🛠 PART 2 — Refactored Architecture

The team decided:

* Separate responsibilities
* Make refund optional
* Depend on abstractions
* Make adding new payment types easy
* Remove tight coupling

---

# 📁 Folder Structure (After Refactoring)

```text
payment-app/
 ├── processor/
 │     └── PaymentProcessor.java
 │
 ├── method/
 │     ├── PaymentMethod.java
 │     ├── Refundable.java
 │     ├── CardPayment.java
 │     ├── UpiPayment.java
 │     └── WalletPayment.java
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

# ✅ PaymentMethod (ISP Fixed)

```java
package method;

public interface PaymentMethod {
    void pay(double amount);
}
```

---

# ✅ Refundable (Optional Capability)

```java
package method;

public interface Refundable {
    void refund(double amount);
}
```

Now refund is not forced.

---

# ✅ CardPayment (Supports Refund)

```java
package method;

public class CardPayment implements PaymentMethod, Refundable {

    @Override
    public void pay(double amount) {
        System.out.println("Processing Card Payment: " + amount);
    }

    @Override
    public void refund(double amount) {
        System.out.println("Refunding Card Payment: " + amount);
    }
}
```

---

# ✅ UpiPayment (Clean LSP)

```java
package method;

public class UpiPayment implements PaymentMethod {

    @Override
    public void pay(double amount) {
        System.out.println("Processing UPI Payment: " + amount);
    }
}
```

No exception. No broken contract.

---

# ✅ WalletPayment

```java
package method;

public class WalletPayment implements PaymentMethod, Refundable {

    @Override
    public void pay(double amount) {
        System.out.println("Processing Wallet Payment: " + amount);
    }

    @Override
    public void refund(double amount) {
        System.out.println("Refunding Wallet Payment: " + amount);
    }
}
```

---

# ✅ PaymentValidator (SRP)

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

---

# ✅ FraudChecker (SRP)

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

# ✅ Repository Abstraction (DIP)

```java
package repository;

public interface PaymentRepository {
    void save(String type, double amount);
}
```

```java
package repository;

public class MySQLPaymentRepository implements PaymentRepository {

    @Override
    public void save(String type, double amount) {
        System.out.println("Saving payment to MySQL");
    }
}
```

---

# ✅ Notification Abstraction

```java
package notification;

public interface NotificationService {
    void notifyUser(String message);
}
```

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

# ✅ Final PaymentProcessor (Clean Version)

```java
package processor;

import fraud.FraudChecker;
import method.PaymentMethod;
import method.Refundable;
import notification.NotificationService;
import repository.PaymentRepository;
import validation.PaymentValidator;

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

    public void refund(PaymentMethod method, double amount) {

        if (method instanceof Refundable refundable) {
            refundable.refund(amount);
            notificationService.notifyUser("Refund successful");
        } else {
            System.out.println("Refund not supported for this payment type");
        }
    }
}
```

---

# 🎯 Business Impact After Refactoring

Before:

* Adding Wallet → modify processor
* Refund bug in UPI → production crash
* Database migration → change processor
* Hard to unit test

After:

* Add new payment → create new class
* Refund optional and safe
* DB swap → change implementation only
* Processor untouched

---

# 🧠 Key Learning from Story

The issue was never syntax.
The issue was design stability under change.

SOLID helps when:

* Business grows
* Features expand
* Teams scale
* Systems evolve

---


