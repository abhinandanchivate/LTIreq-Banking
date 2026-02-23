Perfect Abhi 👍
Below are **10 enterprise-grade assignments WITH SOLUTIONS + Java code snippets**, aligned to your BFSI Payment Workflow training model.

All examples are Java 17 compatible (plain Java, no Spring).

---

# 🏦 ASSIGNMENT 1 – Factory Pattern

## 🎯 Assignment

Remove `if-else` payment creation logic using Factory.

### Requirements

* Support: UPI, CARD, WALLET
* Adding new type should not change main class

---

## ✅ Solution

### Step 1 – Common Interface

```java
interface PaymentProcessor {
    void process(double amount);
}
```

### Step 2 – Implementations

```java
class UPIPayment implements PaymentProcessor {
    public void process(double amount) {
        System.out.println("UPI Payment processed: ₹" + amount);
    }
}

class CardPayment implements PaymentProcessor {
    public void process(double amount) {
        System.out.println("Card Payment processed: ₹" + amount);
    }
}

class WalletPayment implements PaymentProcessor {
    public void process(double amount) {
        System.out.println("Wallet Payment processed: ₹" + amount);
    }
}
```

### Step 3 – Factory

```java
class PaymentFactory {

    public static PaymentProcessor create(String type) {
        return switch (type) {
            case "UPI" -> new UPIPayment();
            case "CARD" -> new CardPayment();
            case "WALLET" -> new WalletPayment();
            default -> throw new IllegalArgumentException("Invalid payment type");
        };
    }
}
```

### Step 4 – Usage

```java
public class Main {
    public static void main(String[] args) {
        PaymentProcessor processor = PaymentFactory.create("CARD");
        processor.process(5000);
    }
}
```

---

# 🏦 ASSIGNMENT 2 – Builder Pattern

## 🎯 Assignment

Create immutable PaymentTransaction with mandatory and optional fields.

---

## ✅ Solution

```java
import java.time.LocalDateTime;

class PaymentTransaction {

    private final String txnId;
    private final double amount;
    private final String currency;
    private final String customerId;
    private final LocalDateTime timestamp;
    private final int riskScore;

    private PaymentTransaction(Builder builder) {
        this.txnId = builder.txnId;
        this.amount = builder.amount;
        this.currency = builder.currency;
        this.customerId = builder.customerId;
        this.timestamp = builder.timestamp == null
                ? LocalDateTime.now()
                : builder.timestamp;
        this.riskScore = builder.riskScore;
    }

    public static class Builder {
        private String txnId;
        private double amount;
        private String currency;
        private String customerId;
        private LocalDateTime timestamp;
        private int riskScore;

        public Builder txnId(String txnId) {
            this.txnId = txnId;
            return this;
        }

        public Builder amount(double amount) {
            if (amount <= 0) throw new IllegalArgumentException("Invalid amount");
            this.amount = amount;
            return this;
        }

        public Builder currency(String currency) {
            this.currency = currency;
            return this;
        }

        public Builder customerId(String customerId) {
            this.customerId = customerId;
            return this;
        }

        public Builder riskScore(int riskScore) {
            this.riskScore = riskScore;
            return this;
        }

        public PaymentTransaction build() {
            if (txnId == null || currency == null)
                throw new IllegalStateException("Mandatory fields missing");
            return new PaymentTransaction(this);
        }
    }
}
```

Usage:

```java
PaymentTransaction txn = new PaymentTransaction.Builder()
        .txnId("TXN123")
        .amount(1000)
        .currency("INR")
        .customerId("C001")
        .riskScore(10)
        .build();
```

---

# 🏦 ASSIGNMENT 3 – Strategy Pattern (Fee Calculation)

## 🎯 Assignment

Implement dynamic fee strategies.

---

## ✅ Solution

```java
interface FeeStrategy {
    double calculateFee(double amount);
}

class CardFeeStrategy implements FeeStrategy {
    public double calculateFee(double amount) {
        return amount * 0.02;
    }
}

class UPIFeeStrategy implements FeeStrategy {
    public double calculateFee(double amount) {
        return 0;
    }
}

class WalletFeeStrategy implements FeeStrategy {
    public double calculateFee(double amount) {
        return 10;
    }
}

class PaymentContext {
    private FeeStrategy strategy;

    public PaymentContext(FeeStrategy strategy) {
        this.strategy = strategy;
    }

    public void setStrategy(FeeStrategy strategy) {
        this.strategy = strategy;
    }

    public void process(double amount) {
        double fee = strategy.calculateFee(amount);
        System.out.println("Total Payable: ₹" + (amount + fee));
    }
}
```

---

# 🏦 ASSIGNMENT 4 – Strategy for Fraud Engine

## 🎯 Assignment

Plug multiple fraud detection algorithms.

---

## ✅ Solution

```java
interface FraudStrategy {
    boolean check(double amount);
}

class RuleBasedFraud implements FraudStrategy {
    public boolean check(double amount) {
        return amount < 100000;
    }
}

class MLBasedFraud implements FraudStrategy {
    public boolean check(double amount) {
        return Math.random() > 0.2;
    }
}
```

Usage:

```java
FraudStrategy fraud = new RuleBasedFraud();
System.out.println("Fraud Check: " + fraud.check(5000));
```

---

# 🏦 ASSIGNMENT 5 – Chain of Responsibility

## 🎯 Assignment

Create validation pipeline.

---

## ✅ Solution

```java
abstract class PaymentHandler {
    protected PaymentHandler next;

    public PaymentHandler setNext(PaymentHandler next) {
        this.next = next;
        return next;
    }

    public abstract void handle(double amount);
}

class BalanceCheck extends PaymentHandler {
    public void handle(double amount) {
        if (amount > 10000) {
            System.out.println("Insufficient Balance");
            return;
        }
        if (next != null) next.handle(amount);
    }
}

class FraudCheck extends PaymentHandler {
    public void handle(double amount) {
        System.out.println("Fraud Check Passed");
        if (next != null) next.handle(amount);
    }
}
```

Usage:

```java
PaymentHandler balance = new BalanceCheck();
PaymentHandler fraud = new FraudCheck();

balance.setNext(fraud);
balance.handle(5000);
```

---

# 🏦 ASSIGNMENT 6 – Observer Pattern

## 🎯 Assignment

Notify multiple systems after payment success.

---

## ✅ Solution

```java
import java.util.*;

interface Observer {
    void update(String message);
}

class SmsService implements Observer {
    public void update(String message) {
        System.out.println("SMS: " + message);
    }
}

class EmailService implements Observer {
    public void update(String message) {
        System.out.println("Email: " + message);
    }
}

class PaymentEvent {
    private List<Observer> observers = new ArrayList<>();

    public void addObserver(Observer o) {
        observers.add(o);
    }

    public void notifyObservers(String msg) {
        for (Observer o : observers) {
            o.update(msg);
        }
    }
}
```

---

# 🏦 ASSIGNMENT 7 – Decorator Pattern

## 🎯 Assignment

Add logging dynamically.

---

## ✅ Solution

```java
interface Payment {
    void pay();
}

class BasicPayment implements Payment {
    public void pay() {
        System.out.println("Payment processed");
    }
}

class LoggingDecorator implements Payment {
    private Payment payment;

    public LoggingDecorator(Payment payment) {
        this.payment = payment;
    }

    public void pay() {
        System.out.println("Logging...");
        payment.pay();
    }
}
```

Usage:

```java
Payment payment = new LoggingDecorator(new BasicPayment());
payment.pay();
```

---

# 🏦 ASSIGNMENT 8 – Adapter Pattern

## 🎯 Assignment

Integrate legacy bank API.

---

## ✅ Solution

```java
class LegacyBankAPI {
    public void makeTxn(double amount) {
        System.out.println("Legacy Bank processed ₹" + amount);
    }
}

interface BankProcessor {
    void process(double amount);
}

class BankAdapter implements BankProcessor {

    private LegacyBankAPI legacy;

    public BankAdapter(LegacyBankAPI legacy) {
        this.legacy = legacy;
    }

    public void process(double amount) {
        legacy.makeTxn(amount);
    }
}
```

---

# 🏦 ASSIGNMENT 9 – Combine Patterns

Flow:

1. Build transaction
2. Create processor
3. Validate
4. Apply fee
5. Notify observers

(Students integrate previous solutions)

---

# 🏦 ASSIGNMENT 10 – Refactoring Challenge

Given:

```java
public void process(String type, double amount) {
    if(type.equals("UPI")) {
        if(amount < 10000) {
            System.out.println("UPI processed");
            System.out.println("SMS sent");
        }
    }
}
```

Refactor using:

* Factory
* Strategy
* Chain
* Observer

---

# 🔥 If You Want Next

I can now provide:

* Complete consolidated runnable Payment Engine
* UML diagrams
* TDD test cases
* 10 refactoring case solutions
* Architect-level explanation
* Interview Q&A
* Concurrency integrated version

Tell me next depth level 🚀
