
---

# 🔵 ENTERPRISE REFACTORING WORKSHOP

# SOLID PRINCIPLES ON PAYMENT ENGINE

Everything plain Java. No frameworks. No shortcuts.

We will cover:

* SRP – Single Responsibility
* OCP – Open/Closed
* LSP – Liskov Substitution
* ISP – Interface Segregation
* DIP – Dependency Inversion
* Progressive refactoring waves
* Enterprise-grade pattern mapping

---

# 🔵 PART 1 — SRP (Single Responsibility Principle)

---

# 📌 CONCEPT

## What

A class should have **only one reason to change**.

---

## Why

In BFSI systems:

* Fraud rules change
* DB vendors change
* Notification providers change
* Audit compliance changes

If all are inside one class → high regression risk.

---

## When

* You see 200+ line “God classes”
* Business + infrastructure logic mixed
* Validation + DB + logging in same method

---

## Where (Banking Use Cases)

* Payment engines
* Loan approval modules
* Billing systems
* Reconciliation engines

---

# 🏦 Banking Failure Scenario

RBI updates fraud threshold.

Developer edits PaymentEngine class.

Accidentally breaks ledger persistence.

Production outage.

---

# 🔴 SRP VIOLATION EXAMPLE

```java
class PaymentEngine {

    void process(String type, double amount) {

        validate(amount);
        fraudCheck(amount);
        debitAccount(amount);
        saveToDB(amount);
        sendNotification();
        audit();
    }

    void validate(double amount) {}
    void fraudCheck(double amount) {}
    void debitAccount(double amount) {}
    void saveToDB(double amount) {}
    void sendNotification() {}
    void audit() {}
}
```

---

# 🚨 Violations

This class handles:

* Validation
* Fraud logic
* Business logic
* Persistence
* Notification
* Audit logging

6 responsibilities → 6 reasons to change.

---

# ✅ SRP REFACTOR (Service Extraction Pattern)

```java
class PaymentProcessor {

    private final Validator validator;
    private final FraudService fraudService;
    private final LedgerService ledger;
    private final NotificationService notification;
    private final AuditService audit;

    void process(PaymentRequest request) {
        validator.validate(request);
        fraudService.check(request);
        ledger.debit(request);
        notification.notifyUser(request);
        audit.log(request);
    }
}
```

Each service now has one reason to change.

---

---

# 🔵 PART 2 — OCP (Open Closed Principle)

---

# 📌 CONCEPT

## What

Software entities should be:

* Open for extension
* Closed for modification

---

## Why

In payment systems:

Adding new payment types must NOT require editing stable core code.

---

## When

* if-else chain grows
* switch-case grows
* Factory keeps changing

---

## Where

* Payment methods
* Tax rules
* Discount policies
* Fraud rule engines

---

# 🔴 OCP VIOLATION

```java
if(type.equals("CARD")) {}
else if(type.equals("UPI")) {}
else if(type.equals("WALLET")) {}
```

Add BNPL → modify class → regression risk.

---

# 🟢 SIMPLE FIX — Strategy

```java
interface PaymentMethod {
    void pay(double amount);
}
```

---

# 🔴 INCOMPLETE OCP (Factory Switch Still Broken)

```java
class PaymentFactory {
    static PaymentMethod create(String type) {
        if(type.equals("CARD")) return new CardPayment();
        if(type.equals("UPI")) return new UpiPayment();
    }
}
```

Still modifies factory.

---

# 🟢 ENTERPRISE OCP — Registry Pattern

```java
class PaymentRegistry {
    private static Map<String, PaymentMethod> methods = new HashMap<>();

    static void register(PaymentMethod method) {
        methods.put(method.getType(), method);
    }

    static PaymentMethod get(String type) {
        return methods.get(type);
    }
}
```

New payment = new class only.

Core untouched.

---

---

# 🔵 PART 3 — LSP (Liskov Substitution Principle)

---

# 📌 CONCEPT

## What

Subtypes must be substitutable without breaking behavior.

---

## Why

If subclass throws exception or changes meaning → runtime crash.

---

## When

* UnsupportedOperationException in subclass
* Subclass tightens preconditions
* Subclass changes output meaning

---

# 🔴 LSP VIOLATION

```java
abstract class Payment {
    abstract void pay(double amount);
}

class FixedDepositPayment extends Payment {
    void pay(double amount) {
        throw new UnsupportedOperationException();
    }
}
```

FD is not a Payment in behavioral sense.

---

# 🟢 FIX — Better Abstraction

```java
interface DebitOperation {
    void debit(double amount);
}
```

Do not force incorrect hierarchy.

---

---

# 🔵 PART 4 — ISP (Interface Segregation Principle)

---

# 📌 CONCEPT

## What

Clients should not depend on methods they don’t use.

---

## Why

Fat interfaces cause dummy implementations.

---

# 🔴 ISP VIOLATION

```java
interface BankingService {
    void pay();
    void refund();
    void generateTaxStatement();
    void applyLoan();
}
```

UPI does not need loan application.

---

# 🟢 FIX — Role Interfaces

```java
interface Payable {
    void pay();
}

interface Refundable {
    void refund();
}
```

---

---

# 🔵 PART 5 — DIP (Dependency Inversion Principle)

---

# 📌 CONCEPT

## What

High-level modules depend on abstraction, not concrete classes.

---

## Why

Allows DB change, gateway change, test mocking.

---

# 🔴 DIP VIOLATION

```java
class PaymentProcessor {
    private MySqlLedgerRepository repo = new MySqlLedgerRepository();
}
```

---

# 🟢 FIX — Constructor Injection

```java
interface LedgerRepository {
    void save();
}

class PaymentProcessor {
    private final LedgerRepository repo;

    PaymentProcessor(LedgerRepository repo) {
        this.repo = repo;
    }
}
```

---

---

# 🔵 FULL HANDS-ON REFACTORING WORKSHOP

---

# 🏦 Broken Enterprise Payment Module (Complete Example)

```java
class EnterprisePaymentSystem {

    void process(String type, double amount) {

        if(amount <= 0)
            throw new RuntimeException();

        if(amount > 50000)
            System.out.println("Fraud alert");

        if(type.equals("CARD"))
            System.out.println("Card logic");
        else if(type.equals("UPI"))
            System.out.println("UPI logic");

        System.out.println("Saving to MySQL");
        System.out.println("Sending SMS");
        System.out.println("Logging audit");
    }
}
```

---

# 🔎 STEP 1 — Identify Violations

| Principle | Violation                          |
| --------- | ---------------------------------- |
| SRP       | Business + DB + notification mixed |
| OCP       | if-else                            |
| DIP       | MySQL hardcoded                    |
| ISP       | Risk if fat interface used         |
| LSP       | Risk if subclass overrides         |

---

# 🛠 STEP 2 — Refactoring Waves

### Wave 1 — Extract Services (SRP)

### Wave 2 — Strategy + Registry (OCP)

### Wave 3 — Constructor Injection (DIP)

### Wave 4 — Role Interfaces (ISP)

### Wave 5 — Behavioral Contracts (LSP)

---

# 🏁 Final Clean Architecture

```text
Application Layer
   PaymentProcessor
        |
----------------------------------
| Fraud | Strategy | Repo | Notify |
```

---

