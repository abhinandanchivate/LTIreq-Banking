
---

## 1) God Processor: SRP + DIP + OCP

### ❌ Broken code (candidate refactors)

```java
import java.util.UUID;

class PaymentProcessorBad {

    public String process(String type, double amount, String customerId) {
        if (amount <= 0) throw new IllegalArgumentException("amount");

        // Fraud
        if (amount > 100_000) System.out.println("[FRAUD] manual review");

        // Payment routing (OCP)
        if ("CARD".equalsIgnoreCase(type)) {
            System.out.println("Charging card: " + amount);
        } else if ("UPI".equalsIgnoreCase(type)) {
            System.out.println("Collect UPI: " + amount);
        } else {
            throw new RuntimeException("Unsupported type");
        }

        // Persistence (DIP)
        MySqlLedgerRepo repo = new MySqlLedgerRepo();
        String txnId = UUID.randomUUID().toString();
        repo.save(txnId, type, amount);

        // Notification
        System.out.println("SMS sent to " + customerId);
        System.out.println("EMAIL sent to " + customerId);

        // Audit
        System.out.println("[AUDIT] txn=" + txnId + " ok");
        return txnId;
    }

    static class MySqlLedgerRepo {
        void save(String txnId, String type, double amount) {
            System.out.println("Saved to MySQL " + txnId);
        }
    }
}
```

**Identify:** SRP, OCP, DIP
**Refactor tasks:** Extract services; introduce `PaymentMethod` strategy; inject `LedgerRepository` + `Notifier`
**Expected patterns:** Service Extraction + Strategy + DIP (constructor injection)

**Target shape (evaluation):**

* `PaymentProcessor` orchestrates only
* `PaymentMethod` implementations for CARD/UPI
* `LedgerRepository` interface + MySQL impl
* `Notifier` interface

---

## 2) Fake OCP: Strategy used, but factory switch breaks OCP

### ❌ Broken code

```java
interface PaymentMethod { void pay(double amount); }

class CardPayment implements PaymentMethod {
    public void pay(double amount) { System.out.println("Card pay " + amount); }
}
class UpiPayment implements PaymentMethod {
    public void pay(double amount) { System.out.println("UPI pay " + amount); }
}

class PaymentFactoryBad {
    static PaymentMethod create(String type) {
        if ("CARD".equalsIgnoreCase(type)) return new CardPayment();
        if ("UPI".equalsIgnoreCase(type)) return new UpiPayment();
        throw new IllegalArgumentException("Unsupported: " + type);
    }
}

class Processor {
    void process(String type, double amount) {
        PaymentMethod m = PaymentFactoryBad.create(type);
        m.pay(amount);
    }
}
```

**Identify:** OCP violation hidden in factory
**Refactor tasks:** Replace switch-factory with registry/plugin registration
**Expected patterns:** Registry / Map-based lookup (true OCP), optionally SPI/plugin loading

**Target shape:**

* `PaymentRegistry.register(method)` or DI map injection
* Add new payment type without modifying factory/processor

---

## 3) LSP violation: Subclass strengthens constraints / throws

### ❌ Broken code

```java
abstract class Payment {
    abstract void pay(double amount);
}

class CardPayment extends Payment {
    @Override void pay(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("amount");
        System.out.println("Card paid " + amount);
    }
}

// Violates LSP: changes contract by forbidding amounts that base allows (or throws differently)
class UpiPaymentNightOnly extends Payment {
    @Override void pay(double amount) {
        int hour = java.time.LocalTime.now().getHour();
        if (hour < 6) throw new UnsupportedOperationException("UPI disabled before 6AM");
        System.out.println("UPI paid " + amount);
    }
}

class Checkout {
    void checkout(Payment payment, double amt) {
        payment.pay(amt); // may explode unexpectedly depending on subtype
    }
}
```

**Identify:** LSP
**Refactor tasks:** Encode capability/availability in policy, not subtype; separate “availability check” from `pay()`
**Expected patterns:** Composition over inheritance; Policy/Rule object; Strategy + Validator

**Target shape:**

* `PaymentMethod.pay(request)` never throws “unsupported by time” for a valid method; time rules handled by `AvailabilityPolicy` or `RiskPolicy`

---

## 4) ISP violation: Fat interface forces dummy implementations

### ❌ Broken code

```java
interface BankingOps {
    void pay(double amount);
    void refund(double amount);
    void chargeback(String txnId);
    void generateMonthlyReport();
}

class UpiModule implements BankingOps {
    public void pay(double amount) { System.out.println("UPI pay"); }
    public void refund(double amount) { System.out.println("UPI refund"); }

    // ISP smell: not applicable but forced
    public void chargeback(String txnId) { throw new UnsupportedOperationException("No chargeback"); }
    public void generateMonthlyReport() { throw new UnsupportedOperationException("No report"); }
}
```

**Identify:** ISP + LSP risk (throws)
**Refactor tasks:** Split into `Payable`, `Refundable`, `ChargebackCapable`, `Reportable`
**Expected patterns:** Role interfaces / capability segregation

**Target shape:**

* UPI implements only needed interfaces; callers depend only on required capability

---

## 5) DIP violation: High-level depends on concrete gateways

### ❌ Broken code

```java
class SmsGateway {
    void send(String to, String msg) { System.out.println("SMS: " + msg); }
}
class EmailGateway {
    void send(String to, String msg) { System.out.println("EMAIL: " + msg); }
}

class ReceiptServiceBad {
    private final SmsGateway sms = new SmsGateway();
    private final EmailGateway email = new EmailGateway();

    void sendReceipt(String customerId, String txnId, double amount) {
        String msg = "Receipt txn=" + txnId + " amt=" + amount;
        sms.send(customerId, msg);
        email.send(customerId, msg);
    }
}
```

**Identify:** DIP (hardcoded concretes), testability issue
**Refactor tasks:** Introduce `Notifier` port; inject implementations; optionally support multi-channel via list
**Expected patterns:** DIP + Adapter pattern

**Target shape:**

* `interface Notifier { void notify(customerId, message); }`
* `CompositeNotifier(List<Notifier>)`

---

## 6) SRP + Hidden coupling: “Validation + persistence + domain decisions” mixed

### ❌ Broken code

```java
class SettlementBad {

    void settle(String txnId, String status, double amount) {
        if (txnId == null || txnId.isBlank()) throw new IllegalArgumentException("txnId");
        if (!status.equals("APPROVED") && !status.equals("DECLINED"))
            throw new IllegalArgumentException("status");

        // domain decision mixed with storage
        if (status.equals("APPROVED") && amount > 50_000) {
            System.out.println("Apply MDR override");
        }

        // persistence hardcoded
        System.out.println("UPDATE settlement SET status='" + status + "' WHERE txn='" + txnId + "'");
        System.out.println("[AUDIT] settlement updated");
    }
}
```

**Identify:** SRP, DIP (SQL inline), security smell (string SQL), maintainability
**Refactor tasks:** Extract validation, policy, repository; parameterize; separate audit
**Expected patterns:** SRP extraction + Repository + Audit port

**Target shape:**

* `SettlementPolicy` decides MDR override
* `SettlementRepository.updateStatus(txnId,status)`
* `AuditLogger.log(event)`

---

## 7) OCP violation in fraud rules: growing if-else thresholds

### ❌ Broken code

```java
class FraudEngineBad {

    boolean approve(String merchantTier, double amount, int txnsLastHour) {
        if ("TIER1".equals(merchantTier)) {
            if (amount > 200_000) return false;
            if (txnsLastHour > 200) return false;
        } else if ("TIER2".equals(merchantTier)) {
            if (amount > 100_000) return false;
            if (txnsLastHour > 100) return false;
        } else {
            if (amount > 50_000) return false;
        }
        return true;
    }
}
```

**Identify:** OCP + SRP (policy sprawl)
**Refactor tasks:** Turn rules into composable strategies; add new tier without editing core
**Expected patterns:** Chain of Responsibility / Specification / Rule Engine pattern

**Target shape:**

* `List<FraudRule>` applied in order
* `TierPolicyProvider` supplies rule set per tier

---

## 8) LSP + ISP in refunds: inconsistent behavior across methods

### ❌ Broken code

```java
interface PaymentMethodBad {
    String pay(double amount);
    String refund(String txnId, double amount);
}

class CardMethod implements PaymentMethodBad {
    public String pay(double amount) { return "CARD_TXN"; }
    public String refund(String txnId, double amount) { return "CARD_REFUND"; }
}

class UpiMethod implements PaymentMethodBad {
    public String pay(double amount) { return "UPI_TXN"; }

    // LSP/ISP issue: refund not supported (or different semantics)
    public String refund(String txnId, double amount) {
        throw new UnsupportedOperationException("UPI refund not supported");
    }
}
```

**Identify:** ISP + LSP
**Refactor tasks:** Split `Refundable`; model refund support as capability; checkout uses only `Payable`
**Expected patterns:** Role interfaces + capability checking

**Target shape:**

* `interface Payable { String pay(...); }`
* `interface Refundable { String refund(...); }`
* UPI implements only Payable

---

## 9) DIP + Testability: static globals & hidden state

### ❌ Broken code

```java
class Config {
    static String DB_URL = "jdbc:mysql://prod:3306/ledger";
    static int FRAUD_LIMIT = 100_000;
}

class PaymentLimitsBad {
    boolean allowed(double amount) {
        return amount <= Config.FRAUD_LIMIT; // hidden dependency
    }
}

class LedgerClientBad {
    void connect() {
        System.out.println("Connecting " + Config.DB_URL); // hidden dependency
    }
}
```

**Identify:** DIP violation via global config; hard to test; environment leakage
**Refactor tasks:** Inject config via constructor; use immutable settings object
**Expected patterns:** Dependency Injection + Immutable Config

**Target shape:**

* `final class PaymentSettings { final int fraudLimit; ... }`
* Inject into services

---

## 10) Transaction workflow sprawl: Template Method candidate

### ❌ Broken code

```java
class PaymentWorkflowBad {

    void execute(String type, double amount) {
        System.out.println("validate");
        if (amount <= 0) throw new IllegalArgumentException();

        System.out.println("pre-auth checks");
        if (amount > 100_000) System.out.println("manual review");

        if ("CARD".equals(type)) {
            System.out.println("card capture");
            System.out.println("card receipt");
        } else if ("UPI".equals(type)) {
            System.out.println("upi collect");
            System.out.println("upi receipt");
        }

        System.out.println("persist ledger");
        System.out.println("audit");
    }
}
```

**Identify:** SRP + OCP (workflow variations + core mixed)
**Refactor tasks:** Extract invariant workflow + variable steps
**Expected patterns:** Template Method (for flow) + Strategy (for method-specific steps)

**Target shape:**

* `abstract class PaymentFlowTemplate { final execute(); abstract authorize(); abstract capture(); }`
* Payment method variations injected

---


---

# 1) Solution — God Processor (SRP + OCP + DIP)

### ✅ Refactored (Orchestrator + Ports + Strategies)

```java
import java.util.*;

record PaymentRequest(String type, double amount, String customerId) {}

interface Validator { void validate(PaymentRequest req); }
interface FraudService { void check(PaymentRequest req); }
interface LedgerRepository { void save(String txnId, PaymentRequest req); }
interface Notifier { void notify(String customerId, String message); }

interface PaymentMethod {
    String type();
    void pay(PaymentRequest req);
}

final class PaymentProcessor {
    private final Validator validator;
    private final FraudService fraud;
    private final LedgerRepository ledger;
    private final Notifier notifier;
    private final PaymentRegistry registry;

    PaymentProcessor(Validator validator, FraudService fraud, LedgerRepository ledger,
                     Notifier notifier, PaymentRegistry registry) {
        this.validator = validator;
        this.fraud = fraud;
        this.ledger = ledger;
        this.notifier = notifier;
        this.registry = registry;
    }

    public String process(PaymentRequest req) {
        validator.validate(req);
        fraud.check(req);

        PaymentMethod method = registry.get(req.type());
        method.pay(req);

        String txnId = UUID.randomUUID().toString();
        ledger.save(txnId, req);

        notifier.notify(req.customerId(), "Payment success txn=" + txnId);
        return txnId;
    }
}

final class PaymentRegistry {
    private final Map<String, PaymentMethod> methods = new HashMap<>();
    void register(PaymentMethod m) { methods.put(m.type().toUpperCase(), m); }
    PaymentMethod get(String type) {
        PaymentMethod m = methods.get(type.toUpperCase());
        if (m == null) throw new IllegalArgumentException("Unsupported type: " + type);
        return m;
    }
}
```

---

# 2) Solution — “Fake OCP” Factory Switch → Registry

### ✅ Replace switch-factory with registry (true OCP)

```java
import java.util.*;

interface PaymentMethod { String type(); void pay(double amount); }

final class PaymentRegistry {
    private final Map<String, PaymentMethod> map = new HashMap<>();
    public void register(PaymentMethod m) { map.put(m.type().toUpperCase(), m); }
    public PaymentMethod resolve(String type) {
        PaymentMethod m = map.get(type.toUpperCase());
        if (m == null) throw new IllegalArgumentException("Unsupported: " + type);
        return m;
    }
}

final class Processor {
    private final PaymentRegistry registry;
    Processor(PaymentRegistry registry) { this.registry = registry; }

    void process(String type, double amount) {
        registry.resolve(type).pay(amount);
    }
}

// Add new payment type = add class + register it; no modifications to Processor/Registry.
```

---

# 3) Solution — LSP (Don’t encode “time unsupported” in subtype)

### ✅ Move “availability” into policy, keep method substitutable

```java
import java.time.*;

record PaymentRequest(double amount, LocalTime atTime) {}

interface PaymentMethod { void pay(PaymentRequest req); }

interface AvailabilityPolicy {
    void checkAllowed(PaymentRequest req); // throws only when policy blocks, not subtype “unsupported”
}

final class TimeWindowPolicy implements AvailabilityPolicy {
    private final int startHourInclusive;
    TimeWindowPolicy(int startHourInclusive) { this.startHourInclusive = startHourInclusive; }

    public void checkAllowed(PaymentRequest req) {
        if (req.atTime().getHour() < startHourInclusive) {
            throw new IllegalStateException("Payment blocked by policy before " + startHourInclusive + ":00");
        }
    }
}

final class UpiPayment implements PaymentMethod {
    private final AvailabilityPolicy policy;
    UpiPayment(AvailabilityPolicy policy) { this.policy = policy; }

    public void pay(PaymentRequest req) {
        policy.checkAllowed(req);
        System.out.println("UPI paid " + req.amount());
    }
}

final class CardPayment implements PaymentMethod {
    public void pay(PaymentRequest req) {
        System.out.println("CARD paid " + req.amount());
    }
}
```

**Why this fixes LSP:** `UpiPayment` doesn’t violate base contract with “unsupported” surprises; restrictions are modeled as explicit policy.

---

# 4) Solution — ISP Fat Interface → Role Interfaces

```java
interface Payable { void pay(double amount); }
interface Refundable { void refund(String txnId, double amount); }
interface ChargebackCapable { void chargeback(String txnId); }
interface Reportable { void generateMonthlyReport(); }

final class UpiModule implements Payable, Refundable {
    public void pay(double amount) { System.out.println("UPI pay " + amount); }
    public void refund(String txnId, double amount) { System.out.println("UPI refund " + txnId); }
}

// Caller depends only on what it needs:
final class Checkout {
    void checkout(Payable payable, double amount) { payable.pay(amount); }
}
```

---

# 5) Solution — DIP for Notifications (Composite Notifier)

```java
import java.util.*;

interface Notifier {
    void notify(String to, String msg);
}

final class SmsNotifier implements Notifier {
    public void notify(String to, String msg) { System.out.println("SMS-> " + to + ": " + msg); }
}
final class EmailNotifier implements Notifier {
    public void notify(String to, String msg) { System.out.println("EMAIL-> " + to + ": " + msg); }
}

final class CompositeNotifier implements Notifier {
    private final List<Notifier> notifiers;
    CompositeNotifier(List<Notifier> notifiers) { this.notifiers = List.copyOf(notifiers); }

    public void notify(String to, String msg) {
        for (Notifier n : notifiers) n.notify(to, msg);
    }
}

final class ReceiptService {
    private final Notifier notifier;
    ReceiptService(Notifier notifier) { this.notifier = notifier; }

    void sendReceipt(String customerId, String txnId, double amount) {
        notifier.notify(customerId, "Receipt txn=" + txnId + " amt=" + amount);
    }
}
```

---

# 6) Solution — Settlement SRP + DIP + safer persistence

```java
record SettlementRequest(String txnId, String status, double amount) {}

interface SettlementValidator { void validate(SettlementRequest req); }
interface SettlementPolicy { boolean needsMdrOverride(SettlementRequest req); }
interface SettlementRepository { void updateStatus(String txnId, String status); }
interface AuditLogger { void log(String event); }

final class SettlementService {
    private final SettlementValidator validator;
    private final SettlementPolicy policy;
    private final SettlementRepository repo;
    private final AuditLogger audit;

    SettlementService(SettlementValidator validator, SettlementPolicy policy,
                      SettlementRepository repo, AuditLogger audit) {
        this.validator = validator; this.policy = policy; this.repo = repo; this.audit = audit;
    }

    void settle(SettlementRequest req) {
        validator.validate(req);

        if ("APPROVED".equals(req.status()) && policy.needsMdrOverride(req)) {
            audit.log("MDR override applied txn=" + req.txnId());
        }

        repo.updateStatus(req.txnId(), req.status());
        audit.log("Settlement updated txn=" + req.txnId() + " status=" + req.status());
    }
}
```

---

# 7) Solution — Fraud rules OCP (Rule list / Chain of Responsibility)

```java
import java.util.*;

record FraudContext(String merchantTier, double amount, int txnsLastHour) {}

interface FraudRule {
    Optional<String> rejectReason(FraudContext ctx);
}

final class AmountLimitRule implements FraudRule {
    private final String tier;
    private final double max;
    AmountLimitRule(String tier, double max) { this.tier = tier; this.max = max; }

    public Optional<String> rejectReason(FraudContext ctx) {
        if (!tier.equals(ctx.merchantTier())) return Optional.empty();
        return ctx.amount() > max ? Optional.of("Amount exceeds " + max + " for " + tier) : Optional.empty();
    }
}

final class RateLimitRule implements FraudRule {
    private final String tier;
    private final int maxTxPerHour;
    RateLimitRule(String tier, int maxTxPerHour) { this.tier = tier; this.maxTxPerHour = maxTxPerHour; }

    public Optional<String> rejectReason(FraudContext ctx) {
        if (!tier.equals(ctx.merchantTier())) return Optional.empty();
        return ctx.txnsLastHour() > maxTxPerHour ? Optional.of("Txn rate exceeds " + maxTxPerHour) : Optional.empty();
    }
}

final class FraudEngine {
    private final List<FraudRule> rules;
    FraudEngine(List<FraudRule> rules) { this.rules = List.copyOf(rules); }

    boolean approve(FraudContext ctx) {
        for (FraudRule r : rules) {
            Optional<String> reason = r.rejectReason(ctx);
            if (reason.isPresent()) return false;
        }
        return true;
    }
}
// Add new tier/rule = add a new rule instance; no engine modification.
```

---

# 8) Solution — Refund LSP/ISP (Capability-based)

```java
interface Payable { String pay(double amount); }
interface Refundable { String refund(String txnId, double amount); }

final class CardMethod implements Payable, Refundable {
    public String pay(double amount) { return "CARD_TXN"; }
    public String refund(String txnId, double amount) { return "CARD_REFUND"; }
}

final class UpiMethod implements Payable {
    public String pay(double amount) { return "UPI_TXN"; }
}

final class RefundService {
    String refund(Refundable refundable, String txnId, double amount) {
        return refundable.refund(txnId, amount);
    }
}
```

---

# 9) Solution — DIP against static globals (Immutable Settings injection)

```java
final class PaymentSettings {
    final String dbUrl;
    final int fraudLimit;
    PaymentSettings(String dbUrl, int fraudLimit) {
        this.dbUrl = dbUrl;
        this.fraudLimit = fraudLimit;
    }
}

final class PaymentLimits {
    private final PaymentSettings settings;
    PaymentLimits(PaymentSettings settings) { this.settings = settings; }

    boolean allowed(double amount) { return amount <= settings.fraudLimit; }
}

final class LedgerClient {
    private final PaymentSettings settings;
    LedgerClient(PaymentSettings settings) { this.settings = settings; }

    void connect() { System.out.println("Connecting " + settings.dbUrl); }
}
```

---

# 10) Solution — Workflow sprawl (Template Method + Strategy)

```java
interface PaymentStep {
    void run(double amount);
}

abstract class PaymentFlowTemplate {

    // invariant algorithm
    public final void execute(double amount) {
        validate(amount);
        preAuthChecks(amount);
        authorize(amount);
        capture(amount);
        persist(amount);
        audit(amount);
    }

    protected void validate(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("amount");
    }

    protected void preAuthChecks(double amount) {
        if (amount > 100_000) System.out.println("Manual review");
    }

    protected abstract void authorize(double amount);
    protected abstract void capture(double amount);

    protected void persist(double amount) { System.out.println("persist ledger"); }
    protected void audit(double amount) { System.out.println("audit"); }
}

final class CardFlow extends PaymentFlowTemplate {
    protected void authorize(double amount) { System.out.println("card auth"); }
    protected void capture(double amount) { System.out.println("card capture"); }
}

final class UpiFlow extends PaymentFlowTemplate {
    protected void authorize(double amount) { System.out.println("upi collect"); }
    protected void capture(double amount) { System.out.println("upi confirm"); }
}
```

---


