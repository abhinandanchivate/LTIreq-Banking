Excellent Abhi 🔥
Now we’ll do **serious enterprise-grade refactoring**.

Below are **7 deep refactoring scenarios**, each mapped to:

* Factory (Creational)
* Builder (Creational)
* Strategy (Behavioral)
* Chain of Responsibility (Behavioral)
* Observer (Behavioral)
* Decorator (Structural)
* Adapter (Structural)

All applied to **real BFSI Payment Workflows**.

Each section includes:

* ❌ Realistic Bad Code
* 🔍 Why It's Dangerous in BFSI
* 🎯 Refactoring Goal
* ✅ Refactored Design
* 🧠 Architecture Thinking

---

# 🏦 REFACTORING 1 – Factory (Remove Payment Type Explosion)

---

## ❌ BAD CODE – Real Production Smell

```java
public class PaymentService {

    public void process(String type, double amount) {

        if (type.equals("CARD")) {
            System.out.println("Processing via Visa gateway");
        } else if (type.equals("UPI")) {
            System.out.println("Processing via NPCI");
        } else if (type.equals("WALLET")) {
            System.out.println("Processing via Wallet API");
        } else if (type.equals("NETBANKING")) {
            System.out.println("Processing via Bank API");
        }

        System.out.println("Payment Completed");
    }
}
```

---

## 🔍 Problem

* Violates Open/Closed Principle
* Adding payment type modifies service
* Not testable
* Business logic mixed with instantiation

In BFSI → Payment types grow constantly.

---

## 🎯 Refactoring Goal

* Centralize creation
* Remove if-else
* Make engine extensible

---

## ✅ Refactored (Factory Pattern)

### Step 1 – Interface

```java
interface PaymentProcessor {
    void process(double amount);
}
```

### Step 2 – Implementations

```java
class CardProcessor implements PaymentProcessor {
    public void process(double amount) {
        System.out.println("Card gateway processing ₹" + amount);
    }
}

class UPIProcessor implements PaymentProcessor {
    public void process(double amount) {
        System.out.println("UPI gateway processing ₹" + amount);
    }
}
```

### Step 3 – Factory

```java
class PaymentFactory {

    public static PaymentProcessor create(String type) {

        return switch (type) {
            case "CARD" -> new CardProcessor();
            case "UPI" -> new UPIProcessor();
            default -> throw new IllegalArgumentException("Invalid Type");
        };
    }
}
```

### Step 4 – Clean Service

```java
public class PaymentService {

    public void process(String type, double amount) {

        PaymentProcessor processor = PaymentFactory.create(type);
        processor.process(amount);

        System.out.println("Payment Completed");
    }
}
```

---

## 🧠 Architecture Thinking

Now:

* Adding new processor = create new class
* Factory extended only
* Service untouched

Enterprise-safe.

---

# 🏦 REFACTORING 2 – Builder (Fix 12-Parameter Constructor)

---

## ❌ BAD CODE

```java
PaymentTransaction txn =
    new PaymentTransaction(
        "TXN1", 5000, "INR", "CUST1",
        "DEVICE1", true, false, 10,
        "MOBILE", "INDIA", true, false
    );
```

---

## 🔍 Problem

* Parameter confusion
* Optional values unclear
* Prone to production bugs

In BFSI → Regulatory fields mandatory.

---

## 🎯 Refactor Goal

* Immutable
* Self-documenting
* Validation inside build()

---

## ✅ Refactored Builder

```java
public class PaymentTransaction {

    private final String txnId;
    private final double amount;
    private final String currency;
    private final String customerId;
    private final int riskScore;

    private PaymentTransaction(Builder builder) {
        this.txnId = builder.txnId;
        this.amount = builder.amount;
        this.currency = builder.currency;
        this.customerId = builder.customerId;
        this.riskScore = builder.riskScore;
    }

    public static class Builder {

        private String txnId;
        private double amount;
        private String currency;
        private String customerId;
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
                throw new IllegalStateException("Mandatory missing");
            return new PaymentTransaction(this);
        }
    }
}
```

---

# 🏦 REFACTORING 3 – Strategy (Remove Hardcoded Fraud Logic)

---

## ❌ BAD CODE

```java
if (amount > 100000) {
    System.out.println("Fraud Risk");
}

if (country.equals("INTL") && amount > 50000) {
    System.out.println("Geo Risk");
}
```

---

## 🔍 Problem

* Fraud logic scattered
* Impossible to plug ML engine
* High compliance risk

---

## 🎯 Refactor Goal

Make fraud pluggable.

---

## ✅ Strategy

```java
interface FraudStrategy {
    boolean check(PaymentTransaction txn);
}

class RuleBasedFraud implements FraudStrategy {
    public boolean check(PaymentTransaction txn) {
        return txn.getAmount() < 100000;
    }
}

class GeoFraudStrategy implements FraudStrategy {
    public boolean check(PaymentTransaction txn) {
        return txn.getAmount() < 50000;
    }
}
```

Usage:

```java
FraudStrategy fraud = new RuleBasedFraud();
if (!fraud.check(txn)) {
    throw new RuntimeException("Fraud Detected");
}
```

---

# 🏦 REFACTORING 4 – Chain (Fix Validation Mess)

---

## ❌ BAD CODE

```java
if (!kycVerified) return;
if (!hasBalance) return;
if (!limitCheck) return;
if (!fraudCheck) return;
```

---

## 🎯 Refactor Goal

Create dynamic validation pipeline.

---

## ✅ Chain Implementation

```java
abstract class Validator {

    protected Validator next;

    public Validator setNext(Validator next) {
        this.next = next;
        return next;
    }

    public abstract void validate(PaymentTransaction txn);
}

class KYCValidator extends Validator {
    public void validate(PaymentTransaction txn) {
        System.out.println("KYC Verified");
        if (next != null) next.validate(txn);
    }
}
```

Chain Setup:

```java
Validator kyc = new KYCValidator();
Validator fraud = new FraudValidator();

kyc.setNext(fraud);
kyc.validate(txn);
```

---

# 🏦 REFACTORING 5 – Observer (Decouple Notifications)

---

## ❌ BAD CODE

```java
System.out.println("SMS Sent");
System.out.println("Email Sent");
System.out.println("Ledger Updated");
```

---

## ✅ Observer

```java
interface PaymentObserver {
    void update(String message);
}

class SmsObserver implements PaymentObserver {
    public void update(String message) {
        System.out.println("SMS: " + message);
    }
}
```

Subject:

```java
class PaymentEvent {

    private List<PaymentObserver> observers = new ArrayList<>();

    public void notifyObservers(String message) {
        observers.forEach(o -> o.update(message));
    }
}
```

---

# 🏦 REFACTORING 6 – Decorator (Remove Logging Duplication)

---

## ❌ BAD CODE

```java
System.out.println("Start");
processPayment();
System.out.println("End");
```

Repeated everywhere.

---

## ✅ Decorator

```java
interface Payment {
    void execute();
}

class CorePayment implements Payment {
    public void execute() {
        System.out.println("Payment Executed");
    }
}

class LoggingDecorator implements Payment {

    private Payment payment;

    public LoggingDecorator(Payment payment) {
        this.payment = payment;
    }

    public void execute() {
        System.out.println("Log Start");
        payment.execute();
        System.out.println("Log End");
    }
}
```

---

# 🏦 REFACTORING 7 – Adapter (Legacy Bank Integration)

---

## ❌ BAD CODE

```java
LegacyBankAPI bank = new LegacyBankAPI();
bank.makeTxn(5000);
```

---

## 🎯 Refactor Goal

Decouple from vendor API.

---

## ✅ Adapter

```java
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

# 🏦 FINAL ENTERPRISE PAYMENT FLOW (ALL PATTERNS TOGETHER)

1. Builder → Build transaction
2. Factory → Create processor
3. Strategy → Fraud + Fee
4. Chain → Validation pipeline
5. Decorator → Logging + Metrics
6. Observer → Notifications
7. Adapter → Bank integration

Enterprise PaymentEngine:

```java
public class PaymentEngine {

    public void execute(PaymentTransaction txn) {

        validatorChain.validate(txn);

        if (!fraudStrategy.check(txn))
            throw new RuntimeException("Fraud");

        double fee = feeStrategy.calculate(txn.getAmount());

        Payment decorated =
            new LoggingDecorator(
                new CorePayment());

        decorated.execute();

        event.notifyObservers("Payment Success");
    }
}
```

---

# 🚀 If You Want Next

I can now provide:

* 🔥 Complete runnable enterprise payment engine
* 🧠 Architect-level interview breakdown
* 🏗 UML diagrams
* ⚡ Concurrency integrated version
* 🧪 TDD + Test cases
* 📂 Production folder structure

Tell me which level you want next, Abhi 🔥
