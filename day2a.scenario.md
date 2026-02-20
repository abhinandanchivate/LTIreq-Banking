
---

# 🔥 Refactoring Case 1 — Payment API Hanging Under Load

---

## 🏦 Business Context

You built a high-throughput payment API.

* Fraud and validation both use same thread pool.
* Under load, API latency jumps to 5–10 seconds.
* Some requests never complete.

Production reports:

> “Payments stuck randomly when traffic spikes.”

---

## ❌ Existing Implementation

```java
import java.util.concurrent.*;

public class Case01_Before_Starvation {

    static ExecutorService pool = Executors.newFixedThreadPool(2);

    static boolean validate() {
        sleep(200);
        return true;
    }

    static int fraud() {
        sleep(800); // slow external call
        return 88;
    }

    public static void main(String[] args) {

        CompletableFuture<Boolean> validation =
                CompletableFuture.supplyAsync(() -> validate(), pool);

        CompletableFuture<Integer> fraud =
                CompletableFuture.supplyAsync(() -> fraud(), pool);

        String decision =
                validation.thenCombine(fraud, (v, f) ->
                        f >= 85 ? "REVIEW" : "APPROVE"
                ).join();

        System.out.println("Decision=" + decision);
        pool.shutdown();
    }

    static void sleep(int ms) {
        try { Thread.sleep(ms); } catch (Exception e) {}
    }
}
```

---

## 🚨 Production Problem

* Fraud blocks pool threads
* Validation cannot execute
* Thread starvation
* High latency

---

## 🎯 Refactoring Objective

* Separate IO-bound and CPU-bound tasks
* Prevent starvation
* Improve throughput

---

## ✅ Refactored Implementation

```java
import java.util.concurrent.*;

public class Case01_After_SeparatePools {

    static ExecutorService validationPool =
            Executors.newFixedThreadPool(2);

    static ExecutorService fraudPool =
            Executors.newFixedThreadPool(5);

    static boolean validate() {
        sleep(200);
        return true;
    }

    static int fraud() {
        sleep(800);
        return 88;
    }

    public static void main(String[] args) {

        CompletableFuture<Boolean> validation =
                CompletableFuture.supplyAsync(
                        Case01_After_SeparatePools::validate,
                        validationPool);

        CompletableFuture<Integer> fraud =
                CompletableFuture.supplyAsync(
                        Case01_After_SeparatePools::fraud,
                        fraudPool);

        String decision =
                validation.thenCombine(fraud, (v, f) ->
                        f >= 85 ? "REVIEW" : "APPROVE"
                ).join();

        System.out.println("Decision=" + decision);

        validationPool.shutdown();
        fraudPool.shutdown();
    }

    static void sleep(int ms) {
        try { Thread.sleep(ms); } catch (Exception e) {}
    }
}
```

---

## 🧠 Why This Is Correct

* Validation never waits for fraud threads
* Fraud IO doesn't block CPU work
* Scales independently

---

# 🔥 Refactoring Case 2 — Money Stuck After Timeout

---

## 🏦 Business Context

Customers complain:

> “Money debited but transaction failed.”

Logs show fraud timed out.

---

## ❌ Existing Code

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicLong;

public class Case02_Before_MoneyLeak {

    static class Account {
        AtomicLong bal = new AtomicLong(1000);

        boolean reserve(long amt) {
            while (true) {
                long cur = bal.get();
                if (cur < amt) return false;
                if (bal.compareAndSet(cur, cur - amt))
                    return true;
            }
        }
    }

    static int fraud() {
        sleep(1000);
        return 70;
    }

    public static void main(String[] args) {

        Account acc = new Account();
        acc.reserve(500);

        CompletableFuture<Integer> fraudFuture =
                CompletableFuture.supplyAsync(() -> fraud())
                        .orTimeout(200, TimeUnit.MILLISECONDS);

        try {
            fraudFuture.join();
        } catch (Exception ignored) {}

        System.out.println("Balance=" + acc.bal.get());
    }

    static void sleep(int ms) {
        try { Thread.sleep(ms); } catch (Exception e) {}
    }
}
```

---

## 🚨 Problem

* Fraud timed out
* No rollback executed
* ₹500 permanently deducted

---

## 🎯 Refactoring Objective

* Guarantee compensation on timeout
* No financial leak

---

## ✅ Refactored Implementation

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicLong;

public class Case02_After_Compensation {

    static class Account {
        AtomicLong bal = new AtomicLong(1000);

        boolean reserve(long amt) {
            while (true) {
                long cur = bal.get();
                if (cur < amt) return false;
                if (bal.compareAndSet(cur, cur - amt))
                    return true;
            }
        }

        void rollback(long amt) {
            bal.addAndGet(amt);
        }
    }

    static int fraud() {
        sleep(1000);
        return 70;
    }

    public static void main(String[] args) {

        Account acc = new Account();
        long amount = 500;

        boolean reserved = acc.reserve(amount);

        CompletableFuture<Integer> fraudFuture =
                CompletableFuture.supplyAsync(() -> fraud())
                        .orTimeout(200, TimeUnit.MILLISECONDS)
                        .handle((score, ex) -> {
                            if (ex != null) {
                                acc.rollback(amount);
                                return 96;
                            }
                            return score;
                        });

        fraudFuture.join();

        System.out.println("Balance=" + acc.bal.get());
    }

    static void sleep(int ms) {
        try { Thread.sleep(ms); } catch (Exception e) {}
    }
}
```

---

## 🧠 Why Correct

* Compensation always executed
* No silent fund leak
* Timeout-safe design

---

# 🔥 Refactoring Case 3 — Nested Futures Causing Complexity

---

## 🏦 Business Context

Senior engineer implemented fraud after validation.
Code review comment:

> “Why do we have Future<Future<Integer>>?”

---

## ❌ Problem Code

```java
import java.util.concurrent.*;

public class Case03_Before_Nested {

    public static void main(String[] args) {

        CompletableFuture<Boolean> validation =
                CompletableFuture.supplyAsync(() -> true);

        CompletableFuture<CompletableFuture<Integer>> nested =
                validation.thenApply(ok ->
                        CompletableFuture.supplyAsync(() -> 80)
                );

        System.out.println(nested.join().join());
    }
}
```

---

## 🚨 Problem

* Hard to read
* Wrong abstraction
* Extra join required

---

## 🎯 Refactoring Objective

Flatten the async chain.

---

## ✅ Refactored Code

```java
import java.util.concurrent.*;

public class Case03_After_Compose {

    public static void main(String[] args) {

        CompletableFuture<Boolean> validation =
                CompletableFuture.supplyAsync(() -> true);

        CompletableFuture<Integer> fraud =
                validation.thenCompose(ok ->
                        CompletableFuture.supplyAsync(() -> 80)
                );

        System.out.println(fraud.join());
    }
}
```

---

# 🔥 Refactoring Case 4 — Fraud Overloading External Vendor

---

## 🏦 Business Context

Fraud vendor complains:

> “You are sending 1000 parallel calls.”

Vendor rate limit = 3 concurrent requests.

---

## ❌ Problem Code

```java
CompletableFuture.supplyAsync(() -> fraud());
```

No rate control.

---

## 🎯 Refactoring Objective

Add concurrency limit.

---

## ✅ Refactored Code

```java
import java.util.concurrent.*;

public class Case04_After_Semaphore {

    static Semaphore sem = new Semaphore(3);

    static int fraud() {
        sleep(300);
        return 80;
    }

    public static void main(String[] args) {

        CompletableFuture<Integer> fraudFuture =
                CompletableFuture.supplyAsync(() -> {
                    try {
                        sem.acquire();
                        return fraud();
                    } catch (Exception e) {
                        throw new RuntimeException(e);
                    } finally {
                        sem.release();
                    }
                });

        System.out.println(fraudFuture.join());
    }

    static void sleep(int ms) {
        try { Thread.sleep(ms); } catch (Exception e) {}
    }
}
```

---

# 🔥 Refactoring Case 5 — Blocking inside Controller Thread

---

## 🏦 Business Context

API latency spikes.
Thread dump shows many threads stuck at `Future.get()`.

---

## ❌ Problem Code

```java
CompletableFuture<Integer> fraud =
        CompletableFuture.supplyAsync(() -> 80);

Integer score = fraud.get(); // blocking
```

---

## 🎯 Refactoring Objective

Make it non-blocking.

---

## ✅ Refactored Code

```java
import java.util.concurrent.*;

public class Case05_After_NonBlocking {

    public static void main(String[] args) {

        CompletableFuture<Integer> fraud =
                CompletableFuture.supplyAsync(() -> 80);

        CompletableFuture<String> decision =
                fraud.thenApply(score ->
                        score >= 85 ? "REVIEW" : "APPROVE"
                );

        System.out.println(decision.join());
    }
}
```

---



