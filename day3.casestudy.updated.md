
---

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

## 🔎 Explanation

The system worked when scale was small.

But growth exposed structural problems:

* Tight coupling
* Rigid design
* Contract violations
* Too many responsibilities in one place

The failure was not about syntax.
It was about **design stability under change**.

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

### 🔎 Explanation

Everything revolves around a few files.
Processor controls everything.
Database is tightly coupled.
PaymentService forces behavior.

---

## 🔴 PaymentService.java (Fat Interface – ISP Violation)

```java
public interface PaymentService {
    void pay(double amount);
    void refund(double amount);
    void generateReport();
}
```

### 🔎 Problem Explanation

* Every payment must implement refund.
* Every payment must implement report.
* Even if they don’t logically support them.

Violates **Interface Segregation Principle (ISP)**.

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

### 🔎 Problem Explanation

* Interface promises refund.
* Implementation breaks it.

Violates **Liskov Substitution Principle (LSP)**.

If used polymorphically, it crashes at runtime.

---

## 🔴 MySQLConnection.java (DIP Violation)

```java
public class MySQLConnection {

    public void save(String type, double amount) {
        System.out.println("Saving to MySQL database");
    }
}
```

### 🔎 Problem Explanation

High-level logic directly depends on MySQL.

Violates **Dependency Inversion Principle (DIP)**.

If DB changes → processor must change.

---

## 🔴 PaymentProcessor.java (Multiple Violations)

```java
public class PaymentProcessor {

    public void process(String paymentType, double amount) {

        if (amount <= 0) {
            throw new RuntimeException("Invalid amount");
        }

        PaymentService service;

        if (paymentType.equals("CARD")) {
            service = new CardPayment();
        } else if (paymentType.equals("UPI")) {
            service = new UpiPayment();
        } else {
            throw new RuntimeException("Unsupported type");
        }

        service.pay(amount);

        if (amount > 100000) {
            System.out.println("Manual fraud review required");
        }

        MySQLConnection db = new MySQLConnection();
        db.save(paymentType, amount);

        System.out.println("Sending SMS");
    }
}
```

---

## 🔎 What Went Wrong

| Principle | Violation                                   |
| --------- | ------------------------------------------- |
| SRP       | Handles validation, fraud, DB, notification |
| OCP       | Adding Wallet requires modifying processor  |
| LSP       | UPI refund crashes                          |
| ISP       | Fat interface                               |
| DIP       | Direct DB dependency                        |

---

# 🛠 PART 2 — Refactored Architecture

---

## 📁 Folder Structure (After Refactoring)

```text
payment-app/
 ├── processor/
 ├── method/
 ├── validation/
 ├── fraud/
 ├── repository/
 ├── notification/
 └── Main.java
```

### 🔎 Explanation

Now responsibilities are clearly separated:

* Validation
* Fraud
* Payment method
* Repository
* Notification
* Processor as orchestrator

---

## ✅ PaymentMethod (ISP Fixed)

```java
public interface PaymentMethod {
    void pay(double amount);
}
```

✔ Only essential behavior.

---

## ✅ Refundable (Optional Capability)

```java
public interface Refundable {
    void refund(double amount);
}
```

✔ Refund is optional.

---

## ✅ CardPayment

```java
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

✔ Contract honored.

---

## ✅ UpiPayment

```java
public class UpiPayment implements PaymentMethod {

    @Override
    public void pay(double amount) {
        System.out.println("Processing UPI Payment: " + amount);
    }
}
```

✔ No broken refund.

---

## ✅ WalletPayment

```java
public class WalletPayment implements PaymentMethod, Refundable {
```

✔ Extension without modifying processor.

---

## ✅ PaymentValidator

```java
public class PaymentValidator {
    public void validate(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Invalid amount");
        }
    }
}
```

✔ SRP applied.

---

## ✅ FraudChecker

```java
public class FraudChecker {
    public void check(double amount) {
        if (amount > 100000) {
            System.out.println("Manual fraud review required");
        }
    }
}
```

✔ Separate business rule.

---

## ✅ Repository Abstraction (DIP)

```java
public interface PaymentRepository {
    void save(String type, double amount);
}
```

```java
public class MySQLPaymentRepository implements PaymentRepository {
```

✔ DB can change without processor change.

---

## ✅ Notification Abstraction

```java
public interface NotificationService {
    void notifyUser(String message);
}
```

✔ SMS → Email → Push easily replaceable.

---

## ✅ Final PaymentProcessor

```java
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

# ➕ Additional Example 1 — Stronger OCP (Add CryptoPayment)

```java
public class CryptoPayment implements PaymentMethod {

    @Override
    public void pay(double amount) {
        System.out.println("Processing Crypto Payment: " + amount);
    }
}
```

✔ No change in processor.
✔ Open for extension.
✔ Closed for modification.

---

# ➕ Additional Example 2 — Stronger SRP (Reporting Separation)

```java
public class ReportService {

    public void generateDailyReport() {
        System.out.println("Generating daily report");
    }
}
```

✔ Reporting separated from payment logic.

---

# ➕ Additional Example 3 — Stronger DIP (Notification Swap)

```java
public class EmailNotificationService implements NotificationService {

    @Override
    public void notifyUser(String message) {
        System.out.println("Sending Email: " + message);
    }
}
```

✔ Processor unchanged.
✔ Infrastructure swapped safely.

---

# 🎯 Final SOLID Coverage

| Principle | Now Covered Fully |
| --------- | ----------------- |
| SRP       | Yes               |
| OCP       | Yes               |
| LSP       | Yes               |
| ISP       | Yes               |
| DIP       | Yes               |

---

