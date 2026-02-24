
Digital Banking System
Features:

* Fund Transfer
* Card Authorization
* AML (Anti Money Laundering) Screening
* Daily Limit Validation
* Balance Update
* Payment Status Notification

---

# Lab 1 — Concurrent Fund Transfer Validation

## Background

A customer has ₹15,000 in their savings account.

Two fund transfer requests of ₹10,000 are received at the same time:

* Transfer A → To Vendor X
* Transfer B → To Vendor Y

Both are processed by different threads.

---

## Problem

Current implementation:

```java
class BankAccount {

    private long balance = 15000;

    public boolean transfer(long amount) {
        if (balance >= amount) {
            balance -= amount;
            return true;
        }
        return false;
    }
}
```

---

## Task

1. Make transfer operation safe.
2. Ensure balance never becomes negative.
3. Do not use synchronized.

---

## What Can Go Wrong

Thread A reads balance = 15000
Thread B reads balance = 15000

Both deduct 10000

Final balance becomes -5000

System becomes financially inconsistent.

---

## Solution

```java
import java.util.concurrent.atomic.AtomicLong;

class BankAccount {

    private final AtomicLong balance = new AtomicLong(15000);

    public boolean transfer(long amount) {
        while (true) {
            long current = balance.get();

            if (current < amount)
                return false;

            if (balance.compareAndSet(current, current - amount))
                return true;
        }
    }
}
```

---

## Why This Works

* Only one thread can successfully update balance at a time.
* If another thread updates first, second thread retries.
* Negative balance is prevented.

---

# Lab 2 — Async AML Screening for High-Value Transaction

## Background

Any transfer above ₹50,000 must go through AML screening.

AML service takes 2 seconds.

You do not want to block request thread.

---

## Current Blocking Code

```java
int riskScore = amlCheck(transaction);
if (riskScore > 70)
    markForManualReview();
```

---

## Task

1. Convert AML check into asynchronous call.
2. Do not block main thread.
3. Print final decision.

---

## Solution

```java
import java.util.concurrent.CompletableFuture;

CompletableFuture
    .supplyAsync(() -> amlCheck(transaction))
    .thenAccept(score -> {
        if (score > 70)
            System.out.println("Manual Review Required");
        else
            System.out.println("Approved");
    });
```

---

## Why This Is Better

* AML runs in background thread.
* Main request thread remains free.
* System scales better under heavy traffic.

---

# Lab 3 — Card Authorization + Fraud Check in Parallel

## Background

During card payment:

You must check:

1. Card validity
2. Fraud score

Both are independent.

---

## Task

Run both in parallel and approve only if both pass.

---

## Solution

```java
import java.util.concurrent.CompletableFuture;

CompletableFuture<Boolean> cardCheck =
    CompletableFuture.supplyAsync(() -> validateCard());

CompletableFuture<Integer> fraudCheck =
    CompletableFuture.supplyAsync(() -> getFraudScore());

CompletableFuture<Boolean> decision =
    cardCheck.thenCombine(fraudCheck,
        (cardValid, fraudScore) ->
            cardValid && fraudScore < 80
    );

decision.thenAccept(result -> {
    if (result)
        System.out.println("Payment Approved");
    else
        System.out.println("Payment Rejected");
});
```

---

## Why This Is Important

* Both checks run in parallel.
* Reduces overall payment processing time.
* No blocking between checks.

---

# Lab 4 — Handle Timeout for External Payment Gateway

## Background

You call an external payment gateway.

Normally response is within 1 second.

Sometimes it takes 5 seconds.

You must fail after 2 seconds.

---

## Task

Add timeout handling.

---

## Solution

```java
import java.util.concurrent.*;

CompletableFuture
    .supplyAsync(() -> callPaymentGateway())
    .orTimeout(2, TimeUnit.SECONDS)
    .exceptionally(ex -> {
        System.out.println("Gateway Timeout");
        return "FAILED";
    })
    .thenAccept(System.out::println);
```

---

## Why This Is Critical

* Prevents request thread from waiting too long.
* Protects system during gateway slowdown.
* Keeps response predictable.

---

# Lab 5 — Limit Concurrent Risk Engine Calls

## Background

Risk engine can handle only 3 concurrent calls.

If more requests come:

* It slows down.
* Response time increases drastically.

---

## Task

Allow only 3 concurrent risk evaluations.

---

## Solution

```java
import java.util.concurrent.*;

class RiskService {

    private static final Semaphore semaphore = new Semaphore(3);

    public static void evaluate() {
        try {
            semaphore.acquire();

            System.out.println("Evaluating by " +
                Thread.currentThread().getName());

            Thread.sleep(1000);

        } catch (InterruptedException ignored) {
        } finally {
            semaphore.release();
        }
    }
}
```

---

## Why This Is Necessary

* Protects downstream service.
* Smooths traffic spikes.
* Avoids overload.


---

# Lab 6 — Daily Transaction Limit Validation (Concurrent Safe)

## Background

A bank allows maximum ₹25,000 transfer per day per customer.

Multiple transfers may happen simultaneously.

You must:

* Track daily transferred amount
* Reject transaction if limit exceeded
* Ensure thread safety

---

## Problem

Unsafe version:

```java
class DailyLimit {

    private long dailyTotal = 0;

    public boolean validate(long amount) {
        if (dailyTotal + amount <= 25000) {
            dailyTotal += amount;
            return true;
        }
        return false;
    }
}
```

---

## What Can Go Wrong

Two transfers of ₹20,000 happen simultaneously.

Thread A checks → dailyTotal = 0
Thread B checks → dailyTotal = 0

Both approve.

Final total becomes ₹40,000.

Limit broken.

---

## Solution

```java
import java.util.concurrent.atomic.AtomicLong;

class DailyLimit {

    private final AtomicLong dailyTotal = new AtomicLong(0);

    public boolean validate(long amount) {

        while (true) {

            long current = dailyTotal.get();

            if (current + amount > 25000)
                return false;

            if (dailyTotal.compareAndSet(current, current + amount))
                return true;
        }
    }
}
```

---

## Why This Works

* Atomic update ensures safe addition.
* No race condition.
* Daily cap is enforced correctly.

---

# Lab 7 — Batch Settlement Using ForkJoin

## Background

At end of day, bank must calculate total settlement amount from 1 million transactions.

Single-threaded calculation is slow.

---

## Task

Use ForkJoin to split work across CPU cores.

---

## Solution

```java
import java.util.concurrent.*;

class SettlementTask extends RecursiveTask<Long> {

    private long[] transactions;
    private int start, end;
    private static final int THRESHOLD = 10000;

    SettlementTask(long[] transactions, int start, int end) {
        this.transactions = transactions;
        this.start = start;
        this.end = end;
    }

    protected Long compute() {

        if (end - start <= THRESHOLD) {
            long sum = 0;
            for (int i = start; i < end; i++)
                sum += transactions[i];
            return sum;
        }

        int mid = (start + end) / 2;

        SettlementTask left = new SettlementTask(transactions, start, mid);
        SettlementTask right = new SettlementTask(transactions, mid, end);

        left.fork();
        long rightResult = right.compute();
        long leftResult = left.join();

        return leftResult + rightResult;
    }
}
```

---

## Why This Helps

* Splits large dataset.
* Uses multiple CPU cores.
* Faster aggregation for large data.

---

# Lab 8 — Scheduled Account Reconciliation

## Background

Bank must reconcile accounts every 10 seconds.

You must schedule recurring task.

---

## Solution

```java
import java.util.concurrent.*;

public class ReconciliationJob {

    public static void main(String[] args) {

        ScheduledExecutorService scheduler =
                Executors.newScheduledThreadPool(1);

        scheduler.scheduleAtFixedRate(
                () -> reconcileAccounts(),
                0,
                10,
                TimeUnit.SECONDS
        );
    }

    static void reconcileAccounts() {
        System.out.println("Reconciling accounts...");
    }
}
```

---

## Why This Is Needed

* Automates periodic banking tasks.
* No manual trigger required.
* Runs at fixed interval.

---

# Lab 9 — Retry Failed Payment After Delay

## Background

If payment gateway fails, retry after 3 seconds.

---

## Solution

```java
import java.util.concurrent.*;

public class PaymentRetry {

    static ScheduledExecutorService scheduler =
            Executors.newScheduledThreadPool(1);

    public static void retryPayment() {

        scheduler.schedule(
                () -> processPayment(),
                3,
                TimeUnit.SECONDS
        );
    }

    static void processPayment() {
        System.out.println("Retrying payment...");
    }
}
```

---

## Why This Matters

* Temporary gateway failures can recover.
* Improves payment success rate.
* Avoids immediate rejection.

---

# Lab 10 — Mini Banking Transaction Pipeline

Now we combine everything.

Flow:

1. Validate daily limit
2. Withdraw balance safely
3. Run fraud check asynchronously
4. Add timeout
5. Limit fraud concurrency
6. Final decision

---

## Complete Example

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicLong;

public class MiniBankingPipeline {

    static AtomicLong balance = new AtomicLong(50000);
    static AtomicLong dailyLimit = new AtomicLong(0);
    static Semaphore fraudLimit = new Semaphore(2);

    static int fraudCheck() {
        try { Thread.sleep(1000); }
        catch (InterruptedException ignored) {}
        return 60;
    }

    static boolean withdraw(long amount) {

        while (true) {

            long current = balance.get();
            if (current < amount)
                return false;

            if (balance.compareAndSet(current, current - amount))
                return true;
        }
    }

    static boolean checkDailyLimit(long amount) {

        while (true) {

            long current = dailyLimit.get();
            if (current + amount > 25000)
                return false;

            if (dailyLimit.compareAndSet(current, current + amount))
                return true;
        }
    }

    public static void main(String[] args) {

        long amount = 10000;

        if (!checkDailyLimit(amount)) {
            System.out.println("Daily limit exceeded");
            return;
        }

        if (!withdraw(amount)) {
            System.out.println("Insufficient balance");
            return;
        }

        CompletableFuture
            .supplyAsync(() -> {

                try {
                    fraudLimit.acquire();
                    return fraudCheck();
                } catch (InterruptedException e) {
                    return 100;
                } finally {
                    fraudLimit.release();
                }

            })
            .orTimeout(2, TimeUnit.SECONDS)
            .handle((score, ex) -> {

                if (ex != null)
                    return "Transaction Failed (Timeout)";

                return score < 80 ?
                        "Transaction Approved" :
                        "Manual Review";
            })
            .thenAccept(System.out::println);
    }
}
```

---






