


----------

# 🟢 LEVEL 1 — Core Concurrency Foundations

----------

## ✅ Assignment 1 — Race Condition Demonstration

**Task:**  
Create an `Account` class with a normal `long balance`.  
Spawn 20 threads debiting ₹100 each from ₹1000.  
Observe incorrect final balance.

**Solution (Runnable):**

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class Assignment01_RaceCondition {

    static class Account {
        long balance = 1000;

        void debit(long amount) {
            if (balance >= amount) {
                // intentionally unsafe: race window
                balance -= amount;
            }
        }
    }

    public static void main(String[] args) throws Exception {
        Account acc = new Account();
        ExecutorService pool = Executors.newFixedThreadPool(5);

        for (int i = 0; i < 20; i++) {
            pool.submit(() -> acc.debit(100));
        }

        pool.shutdown();
        pool.awaitTermination(2, TimeUnit.SECONDS);

        System.out.println("Final balance (unsafe) = " + acc.balance);
        System.out.println("Expected (if perfect) = 0");
    }
}

```

----------

## ✅ Assignment 2 — Fix Using synchronized

**Task:**  
Modify Assignment 1 to use `synchronized` debit method.  
Verify correct balance.

**Solution (Runnable):**

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class Assignment02_Synchronized {

    static class Account {
        long balance = 1000;

        synchronized boolean debit(long amount) {
            if (balance < amount) return false;
            balance -= amount;
            return true;
        }
    }

    public static void main(String[] args) throws Exception {
        Account acc = new Account();
        ExecutorService pool = Executors.newFixedThreadPool(5);

        for (int i = 0; i < 20; i++) {
            pool.submit(() -> acc.debit(100));
        }

        pool.shutdown();
        pool.awaitTermination(2, TimeUnit.SECONDS);

        System.out.println("Final balance (synchronized) = " + acc.balance);
        System.out.println("Expected = 0");
    }
}

```

----------

## ✅ Assignment 3 — Replace synchronized with AtomicLong (CAS)

**Task:**  
Use `AtomicLong.compareAndSet` instead of synchronized.

**Solution (Runnable):**

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;

public class Assignment03_CAS {

    static class Account {
        private final AtomicLong balance = new AtomicLong(1000);

        boolean debit(long amount) {
            while (true) {
                long cur = balance.get();
                if (cur < amount) return false;
                long next = cur - amount;
                if (balance.compareAndSet(cur, next)) return true;
            }
        }

        long getBalance() {
            return balance.get();
        }
    }

    public static void main(String[] args) throws Exception {
        Account acc = new Account();
        ExecutorService pool = Executors.newFixedThreadPool(5);

        for (int i = 0; i < 20; i++) {
            pool.submit(() -> acc.debit(100));
        }

        pool.shutdown();
        pool.awaitTermination(2, TimeUnit.SECONDS);

        System.out.println("Final balance (CAS) = " + acc.getBalance());
        System.out.println("Expected = 0");
    }
}

```

----------

## ✅ Assignment 4 — Add Idempotency Protection

**Task:**  
Maintain a `ConcurrentHashMap.newKeySet()` for processed transaction IDs.  
Reject duplicate tx IDs.

**Solution (Runnable):**

```java
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

public class Assignment04_Idempotency {

    static final Set<String> processed = ConcurrentHashMap.newKeySet();

    static boolean accept(String txId) {
        return processed.add(txId); // true only first time
    }

    public static void main(String[] args) {
        System.out.println("TX-1 accepted? " + accept("TX-1"));
        System.out.println("TX-1 accepted? " + accept("TX-1"));
        System.out.println("TX-2 accepted? " + accept("TX-2"));
        System.out.println("TX-2 accepted? " + accept("TX-2"));
    }
}

```

----------

## ✅ Assignment 5 — Implement Reserve + Commit + Rollback

**Task:**  
Implement:

-   reserve()
    
-   commit()
    
-   rollback()
    

Simulate fraud rejection → rollback funds.

**Solution (Runnable):**

```java
import java.util.concurrent.atomic.AtomicLong;

public class Assignment05_ReserveCommitRollback {

    static class Account {
        private final AtomicLong balance = new AtomicLong(1000);

        // reserve = debit for demo (real systems may "hold" separately)
        boolean reserve(long amount) {
            while (true) {
                long cur = balance.get();
                if (cur < amount) return false;
                if (balance.compareAndSet(cur, cur - amount)) return true;
            }
        }

        void commit(String txId) {
            // for demo: commit is a no-op because reserve already deducted
            System.out.println("Commit success for " + txId);
        }

        void rollback(long amount) {
            balance.addAndGet(amount);
        }

        long getBalance() {
            return balance.get();
        }
    }

    public static void main(String[] args) {
        Account acc = new Account();

        String txId = "TX-500";
        long amount = 800;

        boolean reserved = acc.reserve(amount);
        System.out.println("Reserved? " + reserved + " balance=" + acc.getBalance());

        boolean fraudApproved = false; // simulate fraud rejection
        if (fraudApproved) {
            acc.commit(txId);
        } else {
            acc.rollback(amount);
            System.out.println("Fraud rejected -> rollback done");
        }

        System.out.println("Final balance=" + acc.getBalance());
    }
}

```

----------

# 🟡 LEVEL 2 — Async Basics (CompletableFuture)

----------

## ✅ Assignment 6 — Convert Fraud Check to Async

**Task:**  
Create fraud method with 300ms delay.  
Run using `CompletableFuture.supplyAsync`.

**Solution (Runnable):**

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class Assignment06_AsyncFraud {

    static int fraudScore() {
        sleep(300);
        return 78;
    }

    public static void main(String[] args) {
        ExecutorService fraudPool = Executors.newFixedThreadPool(2);

        CompletableFuture<Integer> f =
                CompletableFuture.supplyAsync(Assignment06_AsyncFraud::fraudScore, fraudPool);

        System.out.println("Fraud score = " + f.join());

        fraudPool.shutdown();
    }

    static void sleep(int ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { throw new RuntimeException(e); }
    }
}

```

----------

## ✅ Assignment 7 — Use thenApply for Decision

**Task:**  
Transform fraud score to APPROVE / REVIEW.

**Solution (Runnable):**

```java
import java.util.concurrent.CompletableFuture;

public class Assignment07_ThenApplyDecision {

    public static void main(String[] args) {
        CompletableFuture<Integer> score =
                CompletableFuture.supplyAsync(() -> 92);

        CompletableFuture<String> decision =
                score.thenApply(s -> s >= 85 ? "REVIEW" : "APPROVE");

        System.out.println("Decision = " + decision.join());
    }
}

```

----------

## ✅ Assignment 8 — Implement Dependent Async using thenCompose

**Task:**  
Fraud check should execute only if validation passes.

**Solution (Runnable):**

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class Assignment08_ThenCompose {

    static boolean validate() {
        sleep(150);
        return true;
    }

    static int fraudScore() {
        sleep(300);
        return 80;
    }

    public static void main(String[] args) {
        ExecutorService pool = Executors.newFixedThreadPool(2);

        CompletableFuture<Boolean> validation =
                CompletableFuture.supplyAsync(Assignment08_ThenCompose::validate, pool);

        CompletableFuture<Integer> fraud =
                validation.thenCompose(ok -> {
                    if (!ok) return CompletableFuture.completedFuture(0);
                    return CompletableFuture.supplyAsync(Assignment08_ThenCompose::fraudScore, pool);
                });

        System.out.println("Fraud score = " + fraud.join());
        pool.shutdown();
    }

    static void sleep(int ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { throw new RuntimeException(e); }
    }
}

```

----------

## ✅ Assignment 9 — Parallel Balance + Fraud using thenCombine

**Task:**  
Run balance check and fraud check in parallel.  
Combine result.

**Solution (Runnable):**

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class Assignment09_ThenCombine {

    static boolean balanceCheck() {
        sleep(200);
        return true;
    }

    static int fraudScore() {
        sleep(300);
        return 88;
    }

    public static void main(String[] args) {
        ExecutorService pool = Executors.newFixedThreadPool(2);

        CompletableFuture<Boolean> balF =
                CompletableFuture.supplyAsync(Assignment09_ThenCombine::balanceCheck, pool);

        CompletableFuture<Integer> fraudF =
                CompletableFuture.supplyAsync(Assignment09_ThenCombine::fraudScore, pool);

        CompletableFuture<String> decision =
                balF.thenCombine(fraudF, (balOk, score) -> {
                    if (!balOk) return "REJECT";
                    return score >= 85 ? "REVIEW" : "APPROVE";
                });

        System.out.println("Decision = " + decision.join());
        pool.shutdown();
    }

    static void sleep(int ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { throw new RuntimeException(e); }
    }
}

```

----------

## ✅ Assignment 10 — Batch Processing using allOf

**Task:**  
Process 10 fraud calls concurrently.  
Collect results using `allOf`.

**Solution (Runnable):**

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CompletableFuture;

public class Assignment10_AllOfBatch {

    static CompletableFuture<Integer> fraudAsync(int i) {
        return CompletableFuture.supplyAsync(() -> {
            sleep(100 + (i % 3) * 50);
            return 60 + i; // fake score
        });
    }

    public static void main(String[] args) {
        List<CompletableFuture<Integer>> list = new ArrayList<>();

        for (int i = 0; i < 10; i++) {
            list.add(fraudAsync(i));
        }

        CompletableFuture.allOf(list.toArray(new CompletableFuture[0])).join();

        List<Integer> scores = list.stream().map(CompletableFuture::join).toList();
        System.out.println("Scores = " + scores);
    }

    static void sleep(int ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { throw new RuntimeException(e); }
    }
}

```

----------

# 🟠 LEVEL 3 — Error Handling & Control

----------

## ✅ Assignment 11 — Add exceptionally Fallback

**Task:**  
Throw exception in fraud call.  
Handle using `exceptionally`.

**Solution (Runnable):**

```java
import java.util.concurrent.CompletableFuture;

public class Assignment11_Exceptionally {

    public static void main(String[] args) {
        CompletableFuture<Integer> fraud =
                CompletableFuture.supplyAsync(() -> {
                    throw new RuntimeException("Fraud service down");
                }).exceptionally(ex -> {
                    System.out.println("Fallback because: " + ex.getMessage());
                    return 95;
                });

        System.out.println("Score = " + fraud.join());
    }
}

```

----------

## ✅ Assignment 12 — Replace exceptionally with handle

**Task:**  
Use `handle()` to manage success and failure.

**Solution (Runnable):**

```java
import java.util.concurrent.CompletableFuture;

public class Assignment12_Handle {

    public static void main(String[] args) {
        CompletableFuture<Integer> fraud =
                CompletableFuture.supplyAsync(() -> 80)
                        .handle((res, ex) -> {
                            if (ex != null) return 95;
                            return res;
                        });

        System.out.println("Score = " + fraud.join());
    }
}

```

----------

## ✅ Assignment 13 — Add Timeout with orTimeout

**Task:**  
Fraud call takes 1s. Add timeout of 200ms.  
Fallback to REVIEW.

**Solution (Runnable):**

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

public class Assignment13_OrTimeout {

    public static void main(String[] args) {
        CompletableFuture<Integer> fraud =
                CompletableFuture.supplyAsync(() -> {
                    sleep(1000);
                    return 70;
                }).orTimeout(200, TimeUnit.MILLISECONDS)
                  .exceptionally(ex -> 96);

        int score = fraud.join();
        String decision = score >= 85 ? "REVIEW" : "APPROVE";
        // here score=96 due to timeout -> decision becomes REVIEW if you want
        decision = (score == 96) ? "REVIEW" : decision;

        System.out.println("Score=" + score + " Decision=" + decision);
    }

    static void sleep(int ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { throw new RuntimeException(e); }
    }
}

```

----------

## ✅ Assignment 14 — Manual Timeout using ScheduledExecutorService

**Task:**  
Use scheduler to complete future exceptionally after delay.

**Solution (Runnable):**

```java
import java.util.concurrent.*;

public class Assignment14_ManualTimeout {

    public static void main(String[] args) {
        ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

        CompletableFuture<Integer> fraud =
                CompletableFuture.supplyAsync(() -> {
                    sleep(1000);
                    return 75;
                });

        scheduler.schedule(() -> fraud.completeExceptionally(
                new TimeoutException("Manual timeout")
        ), 200, TimeUnit.MILLISECONDS);

        int score = fraud.handle((res, ex) -> {
            if (ex != null) {
                System.out.println("Timeout hit -> fallback");
                return 96;
            }
            return res;
        }).join();

        System.out.println("Score=" + score);
        scheduler.shutdown();
    }

    static void sleep(int ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { throw new RuntimeException(e); }
    }
}

```

----------

# 🔵 LEVEL 4 — Thread Pool Engineering

----------

## ✅ Assignment 15 — Separate Executors

**Task:**  
Create validationPool and fraudPool.  
Run both tasks on separate pools.

**Solution (Runnable):**

```java
import java.util.concurrent.*;

public class Assignment15_SeparateExecutors {

    static boolean validate() {
        sleep(150);
        return true;
    }

    static int fraudScore() {
        sleep(300);
        return 82;
    }

    public static void main(String[] args) {
        ExecutorService validationPool = Executors.newFixedThreadPool(2);
        ExecutorService fraudPool = Executors.newFixedThreadPool(2);

        CompletableFuture<Boolean> v =
                CompletableFuture.supplyAsync(Assignment15_SeparateExecutors::validate, validationPool);

        CompletableFuture<Integer> f =
                v.thenCompose(ok -> ok
                        ? CompletableFuture.supplyAsync(Assignment15_SeparateExecutors::fraudScore, fraudPool)
                        : CompletableFuture.completedFuture(0));

        System.out.println("Fraud=" + f.join());

        validationPool.shutdown();
        fraudPool.shutdown();
    }

    static void sleep(int ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { throw new RuntimeException(e); }
    }
}

```

----------

## ✅ Assignment 16 — Create Bounded ThreadPoolExecutor

**Task:**  
ThreadPoolExecutor with core=5, queue size=10, CallerRunsPolicy.  
Simulate 100 tasks.

**Solution (Runnable):**

```java
import java.util.concurrent.*;

public class Assignment16_BoundedExecutor {

    public static void main(String[] args) throws Exception {
        ThreadPoolExecutor pool = new ThreadPoolExecutor(
                5, 5,
                0L, TimeUnit.MILLISECONDS,
                new ArrayBlockingQueue<>(10),
                new ThreadPoolExecutor.CallerRunsPolicy()
        );

        long start = System.currentTimeMillis();
        for (int i = 1; i <= 100; i++) {
            int id = i;
            pool.execute(() -> {
                sleep(20);
                if (id % 25 == 0) {
                    System.out.println("Done task " + id + " on " + Thread.currentThread().getName());
                }
            });
        }

        pool.shutdown();
        pool.awaitTermination(5, TimeUnit.SECONDS);

        System.out.println("Finished in ms = " + (System.currentTimeMillis() - start));
    }

    static void sleep(int ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { throw new RuntimeException(e); }
    }
}

```

----------

## ✅ Assignment 17 — Detect Thread Starvation

**Task:**  
Block fraud tasks intentionally.  
Observe queue growth and latency.

**Solution (Runnable):**

```java
import java.util.concurrent.*;

public class Assignment17_ThreadStarvation {

    public static void main(String[] args) throws Exception {
        // single pool -> validation can suffer if fraud blocks
        ThreadPoolExecutor pool = new ThreadPoolExecutor(
                4, 4,
                0L, TimeUnit.MILLISECONDS,
                new ArrayBlockingQueue<>(20),
                new ThreadPoolExecutor.CallerRunsPolicy()
        );

        // Submit many "fraud" tasks that block
        for (int i = 1; i <= 30; i++) {
            int id = i;
            pool.execute(() -> {
                sleep(800); // blocking fraud
                if (id % 10 == 0) System.out.println("Fraud done " + id);
            });
        }

        // Submit some "validation" tasks and measure delay
        long t0 = System.currentTimeMillis();
        Future<String> validation = pool.submit(() -> {
            long delay = System.currentTimeMillis() - t0;
            return "Validation executed after " + delay + " ms on " + Thread.currentThread().getName();
        });

        System.out.println(validation.get());
        System.out.println("Queue size seen = " + pool.getQueue().size());

        pool.shutdown();
        pool.awaitTermination(5, TimeUnit.SECONDS);
    }

    static void sleep(int ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { throw new RuntimeException(e); }
    }
}

```

----------

# 🟣 LEVEL 5 — Advanced Banking Patterns

----------

## ✅ Assignment 18 — Implement Semaphore Rate Limiting

**Task:**  
Allow only 3 concurrent fraud calls. Others must return REVIEW.

**Solution (Runnable):**

```java
import java.util.concurrent.*;
import java.util.concurrent.Semaphore;

public class Assignment18_SemaphoreRateLimit {

    static final Semaphore permits = new Semaphore(3, true);

    static String fraudCall(int txId) {
        boolean acquired = false;
        try {
            acquired = permits.tryAcquire(150, TimeUnit.MILLISECONDS);
            if (!acquired) return "TX-" + txId + " => REVIEW (rate limited)";

            sleep(400);
            return "TX-" + txId + " => OK (fraud completed)";
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } finally {
            if (acquired) permits.release();
        }
    }

    public static void main(String[] args) throws Exception {
        ExecutorService pool = Executors.newFixedThreadPool(8);

        for (int i = 1; i <= 10; i++) {
            int id = i;
            pool.submit(() -> System.out.println(fraudCall(id)));
        }

        pool.shutdown();
        pool.awaitTermination(5, TimeUnit.SECONDS);
    }

    static void sleep(int ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { throw new RuntimeException(e); }
    }
}

```

----------

## ✅ Assignment 19 — ForkJoin Settlement Aggregation

**Task:**  
Given array of 10,000 transaction amounts, use `RecursiveTask` to compute total in parallel.

**Solution (Runnable):**

```java
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;

public class Assignment19_ForkJoinSettlement {

    static class SumTask extends RecursiveTask<Long> {
        private static final int THRESHOLD = 500;
        private final long[] data;
        private final int lo, hi;

        SumTask(long[] data, int lo, int hi) {
            this.data = data; this.lo = lo; this.hi = hi;
        }

        @Override
        protected Long compute() {
            int size = hi - lo;
            if (size <= THRESHOLD) {
                long sum = 0;
                for (int i = lo; i < hi; i++) sum += data[i];
                return sum;
            }
            int mid = lo + size / 2;
            SumTask left = new SumTask(data, lo, mid);
            SumTask right = new SumTask(data, mid, hi);
            left.fork();
            long r = right.compute();
            long l = left.join();
            return l + r;
        }
    }

    public static void main(String[] args) {
        long[] tx = new long[10_000];
        for (int i = 0; i < tx.length; i++) tx[i] = 10; // each tx = 10 paise

        ForkJoinPool pool = new ForkJoinPool();
        long total = pool.invoke(new SumTask(tx, 0, tx.length));

        System.out.println("Settlement total = " + total);
        pool.shutdown();
    }
}

```

----------

## ✅ Assignment 20 — Full Async Banking Pipeline

**Task:**  
Build complete flow:

1.  Validate (CAS debit)
    
2.  Idempotency check
    
3.  Async fraud
    
4.  Timeout
    
5.  Rate limit
    
6.  Error handling
    
7.  Commit or rollback
    

**Solution (Runnable):**

```java
import java.util.Set;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicLong;

public class Assignment20_FullPipeline {

    enum Decision { APPROVE, REVIEW, REJECT }

    static class Account {
        private final AtomicLong balance = new AtomicLong(1000);

        boolean reserve(long amt) {
            while (true) {
                long cur = balance.get();
                if (cur < amt) return false;
                if (balance.compareAndSet(cur, cur - amt)) return true;
            }
        }

        void rollback(long amt) { balance.addAndGet(amt); }
        long balance() { return balance.get(); }
    }

    static final Set<String> processed = ConcurrentHashMap.newKeySet();
    static final Semaphore fraudPermits = new Semaphore(3, true);

    static int fraudScore(String txId) {
        boolean acquired = false;
        try {
            acquired = fraudPermits.tryAcquire(120, TimeUnit.MILLISECONDS);
            if (!acquired) return 96; // treat as REVIEW bias due to load

            sleep(300);
            return 88;
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } finally {
            if (acquired) fraudPermits.release();
        }
    }

    public static void main(String[] args) {
        Account acc = new Account();
        ExecutorService validationPool = Executors.newFixedThreadPool(2);
        ExecutorService fraudPool = Executors.newFixedThreadPool(4);

        String txId = "TX-999";
        long amount = 500;

        // 1) Idempotency
        if (!processed.add(txId)) {
            System.out.println("Decision=REJECT reason=DUPLICATE");
            shutdown(validationPool, fraudPool);
            return;
        }

        // 2) Validate + reserve (async)
        CompletableFuture<Boolean> reservedF =
                CompletableFuture.supplyAsync(() -> acc.reserve(amount), validationPool);

        // 3) Fraud async (dependent) + 4) timeout + 6) error handling
        CompletableFuture<Integer> fraudF =
                reservedF.thenCompose(reserved -> {
                    if (!reserved) return CompletableFuture.completedFuture(0);
                    return CompletableFuture.supplyAsync(() -> fraudScore(txId), fraudPool);
                }).orTimeout(200, TimeUnit.MILLISECONDS)
                  .handle((score, ex) -> {
                      if (ex != null) return 96; // timeout/error => REVIEW bias
                      return score;
                  });

        // 7) Commit or rollback decision
        Decision decision = reservedF.thenCombine(fraudF, (reserved, score) -> {
            if (!reserved) return Decision.REJECT;
            if (score >= 85) {
                acc.rollback(amount); // rollback on review in this demo
                return Decision.REVIEW;
            }
            return Decision.APPROVE;
        }).join();

        System.out.println("Decision=" + decision + " finalBalance=" + acc.balance());

        shutdown(validationPool, fraudPool);
    }

    static void shutdown(ExecutorService a, ExecutorService b) {
        a.shutdown();
        b.shutdown();
    }

    static void sleep(int ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { throw new RuntimeException(e); }
    }
}

```

----------


