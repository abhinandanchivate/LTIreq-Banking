

---

# 🏦 PART 1 — 10 HANDS-ON ASSIGNMENTS (WITH SOLUTIONS)

---

# 🔵 Assignment 1 — Identify SRP Violations

## Problem

```java
class PaymentService {
    void process(String type, double amount) {
        validate(amount);
        fraudCheck(amount);
        saveToDB(amount);
        sendSMS();
        log();
    }
}
```

### Task

List all SRP violations.

---

## ✅ Solution

Responsibilities:

1. Validation
2. Fraud
3. Persistence
4. Notification
5. Logging

Reason to change:

* Fraud rules change
* DB vendor change
* Notification provider change
* Audit change

➡ Violates SRP.

---

# 🔵 Assignment 2 — Refactor SRP

### Task

Split responsibilities properly.

---

## ✅ Solution

```java
class PaymentProcessor {
    private Validator validator;
    private FraudService fraud;
    private LedgerService ledger;
    private NotificationService notify;

    void process(PaymentRequest request) {
        validator.validate(request);
        fraud.check(request);
        ledger.save(request);
        notify.send(request);
    }
}
```

Each class now single responsibility.

---

# 🔵 Assignment 3 — Remove OCP Violation

## Problem

```java
if(type.equals("CARD")) {}
else if(type.equals("UPI")) {}
```

### Task

Refactor using Strategy.

---

## ✅ Solution

```java
interface PaymentMethod {
    void pay(double amount);
}

class CardPayment implements PaymentMethod {}
class UpiPayment implements PaymentMethod {}
```

Processor depends on abstraction.

---

# 🔵 Assignment 4 — Remove Factory Switch

### Problem

```java
class PaymentFactory {
    static PaymentMethod create(String type) {
        if(type.equals("CARD")) return new CardPayment();
    }
}
```

### Task

Make it OCP compliant.

---

## ✅ Solution (Registry Pattern)

```java
class PaymentRegistry {
    private Map<String, PaymentMethod> methods = new HashMap<>();

    void register(PaymentMethod method) {
        methods.put(method.getType(), method);
    }

    PaymentMethod get(String type) {
        return methods.get(type);
    }
}
```

No modification needed for new types.

---

# 🔵 Assignment 5 — Identify DIP Violation

## Problem

```java
class PaymentProcessor {
    private MySqlRepository repo = new MySqlRepository();
}
```

---

## ✅ Solution

Violation:
High-level class depends on concrete class.

Fix:

```java
interface Repository {
    void save();
}
```

Inject via constructor.

---

# 🔵 Assignment 6 — Refactor DIP

```java
class PaymentProcessor {
    private final Repository repo;

    PaymentProcessor(Repository repo) {
        this.repo = repo;
    }
}
```

Now testable and flexible.

---

# 🔵 Assignment 7 — Identify LSP Violation

```java
class WalletPayment extends Payment {
    void pay(double amount) {
        throw new UnsupportedOperationException();
    }
}
```

---

## ✅ Solution

Violation:
Subclass breaks behavioral contract.

Fix:
Redesign abstraction so Wallet is not forced to implement invalid behavior.

---

# 🔵 Assignment 8 — Apply ISP

## Problem

```java
interface BankingService {
    void pay();
    void refund();
    void generateReport();
}
```

---

## ✅ Solution

```java
interface Payable { void pay(); }
interface Refundable { void refund(); }
interface Reportable { void generateReport(); }
```

---

# 🔵 Assignment 9 — Multi-Tenant DB Support

### Task

Support:

* Tenant A → MySQL
* Tenant B → PostgreSQL

Without modifying processor.

---

## ✅ Solution

Inject repository implementation per tenant.

```java
PaymentProcessor processor =
    new PaymentProcessor(new PostgresRepository());
```

Core untouched.

---

# 🔵 Assignment 10 — Add New Payment Type

Add BNPL without modifying PaymentProcessor.

---

## ✅ Solution

Create:

```java
class BnplPayment implements PaymentMethod {
    public void pay(double amount) {}
}
```

Register in registry.

No modification in processor.

---

---


