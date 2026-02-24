

# Topic: Concurrent Transaction Validation

---

## Concept

Concurrent transaction validation is about protecting shared data when more than one request can change it at the same time.

In real systems, many users interact with the same account. If two requests try to withdraw money from the same balance at almost the same moment, both may read the same value before either updates it.

If we do not protect that section properly, both requests may think there is enough money and both may deduct it.

So we must ensure that checking the balance and updating it behave like one safe step.

To solve this, we use:

* Atomic variables
* synchronized blocks
* Locks
* Database transactions

The goal is simple:
No two threads should update the same value based on an old copy.

---

## Scenario

An account has a balance of ₹8000 stored in the system.

The application runs on a server that handles many user requests at the same time using multiple threads.

Now imagine this:

* The same user opens the banking app on two devices.
* On both devices, they try to transfer ₹5000.
* Both transfer requests reach the server almost at the same time.

On the server:

* Request 1 is handled by Thread A.
* Request 2 is handled by Thread B.
* Both threads execute the withdrawal logic independently.

At this moment:

* Thread A reads balance → 8000
* Thread B reads balance → 8000

Neither thread knows that the other thread is working on the same account.

Both believe there is enough money.

---

## Goal

* Only one transfer should succeed.
* The other should fail.
* The balance must never become negative.
* The system must behave correctly even when many requests come together.

---

## What Can Go Wrong

Let’s continue from the scenario.

After reading the balance:

Thread A subtracts 5000 → balance becomes 3000
Thread B subtracts 5000 → balance becomes -2000

Now the balance is incorrect.

Why did this happen?

Because:

* Reading the balance
* Checking the balance
* Updating the balance

were done as separate steps.

There was no protection between these steps.

Both threads worked using the same old value before either finished updating it.

That small timing overlap caused the issue.

---

## Incorrect Example (Runnable)

```java
import java.util.concurrent.*;

public class UnsafeWalletDemo {

    static class Wallet {
        private long balance = 8000;

        public boolean withdraw(long amount) {
            if (balance >= amount) {
                simulateDelay();
                balance -= amount;
                return true;
            }
            return false;
        }

        private void simulateDelay() {
            try {
                Thread.sleep(100);
            } catch (InterruptedException ignored) {}
        }

        public long getBalance() {
            return balance;
        }
    }

    public static void main(String[] args) throws InterruptedException {

        Wallet wallet = new Wallet();
        ExecutorService executor = Executors.newFixedThreadPool(2);

        Runnable task = () -> {
            boolean result = wallet.withdraw(5000);
            System.out.println(Thread.currentThread().getName()
                    + " Success: " + result
                    + " Balance: " + wallet.getBalance());
        };

        executor.submit(task);
        executor.submit(task);

        executor.shutdown();
        Thread.sleep(500);

        System.out.println("Final Balance: " + wallet.getBalance());
    }
}
```

When you run this multiple times, sometimes you may see:

Success: true Balance: 3000
Success: true Balance: -2000
Final Balance: -2000

That is the race condition happening.

---

## Why It Fails

* `balance` is a normal variable.
* It is shared by multiple threads.
* Two threads can read the same value.
* Both pass validation.
* Both subtract the amount.

The delay increases the chance of overlap.

When many users send requests at once, this situation becomes more likely.

---

## Correct Implementation (Runnable)

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicLong;

public class SafeWalletDemo {

    static class Wallet {
        private final AtomicLong balance = new AtomicLong(8000);

        public boolean withdraw(long amount) {
            while (true) {
                long current = balance.get();

                if (current < amount)
                    return false;

                if (balance.compareAndSet(current, current - amount))
                    return true;
            }
        }

        public long getBalance() {
            return balance.get();
        }
    }

    public static void main(String[] args) throws InterruptedException {

        Wallet wallet = new Wallet();
        ExecutorService executor = Executors.newFixedThreadPool(2);

        Runnable task = () -> {
            boolean result = wallet.withdraw(5000);
            System.out.println(Thread.currentThread().getName()
                    + " Success: " + result
                    + " Balance: " + wallet.getBalance());
        };

        executor.submit(task);
        executor.submit(task);

        executor.shutdown();
        Thread.sleep(500);

        System.out.println("Final Balance: " + wallet.getBalance());
    }
}
```

Now you will see:

One thread succeeds.
The other fails.
Balance remains correct.

---

## Explanation of the Correct Version

* We replaced `long` with `AtomicLong`.
* `balance.get()` reads the latest value safely.
* `compareAndSet` checks and updates in one atomic step.
* If another thread updates first, the operation retries.
* Only one thread can update based on a specific balance value.

This removes the timing gap problem.

---

## Common Mistakes

* Using double for money
* Doing slow work inside retry loop
* Not testing with concurrent threads
* Updating multiple related fields without proper locking

---

## Best Practices

* Store money in smallest unit using long
* Keep update logic small and focused
* Protect shared data whenever multiple threads can access it
* Use database transactions if updating multiple tables





---

# Topic: Async Fraud Checks using CompletableFuture

---

## Concept

In many systems, before approving a payment, we call an external fraud service.

That fraud service may take time to respond. Sometimes it responds in milliseconds, sometimes it takes seconds.

If we make the main request thread wait for the fraud system to respond, that thread becomes blocked. When many users send requests at the same time, blocked threads reduce system capacity.

Async fraud checks allow us to call the fraud system in the background without blocking the main flow.

The idea is simple:

Don’t make the request thread sit idle while waiting for another system.

---

## Scenario

A payment service does the following steps:

1. Validate amount and account quickly.
2. Call fraud system.
3. Based on fraud score, approve or review the transaction.

Now assume:

* Fraud system takes 2 seconds to respond.
* The server has a thread pool of size 5.
* 20 users initiate payments at the same time.

If each thread blocks for 2 seconds waiting for fraud, only 5 payments can be processed at a time. The remaining 15 wait in queue.

This slows down the system.

---

## Goal

* Call the fraud system without blocking request threads.
* Keep the system responsive under load.
* Handle fraud delays safely.

---

## What Can Go Wrong

If we write async code but still block using `get()` or `join()`, we lose the benefit.

Example problem:

* Thread submits fraud call using CompletableFuture.
* Immediately calls `future.get()`.
* Thread waits.
* Thread pool becomes blocked.
* Throughput drops.

Even though we used async API, behavior becomes synchronous.

---

## Incorrect Example (Runnable)

```java
import java.util.concurrent.*;

public class BlockingFraudDemo {

    static int callFraudService() {
        try {
            Thread.sleep(2000); // simulate slow external system
        } catch (InterruptedException ignored) {}
        return 85;
    }

    public static void main(String[] args) throws Exception {

        ExecutorService requestPool = Executors.newFixedThreadPool(2);

        Runnable paymentTask = () -> {
            try {
                CompletableFuture<Integer> fraudFuture =
                        CompletableFuture.supplyAsync(
                                BlockingFraudDemo::callFraudService);

                // Blocking call
                int score = fraudFuture.get();

                System.out.println(Thread.currentThread().getName()
                        + " Fraud Score: " + score);

            } catch (Exception e) {
                e.printStackTrace();
            }
        };

        requestPool.submit(paymentTask);
        requestPool.submit(paymentTask);
        requestPool.submit(paymentTask);

        requestPool.shutdown();
    }
}
```

---

## Why It Fails

* `supplyAsync` starts work in background.
* But `get()` blocks the current thread.
* The request thread waits for fraud response.
* If many requests come, all threads block.
* New requests must wait.

This reduces concurrency.

Even though we used CompletableFuture, we turned it back into blocking behavior.

---

## Correct Implementation (Runnable)

```java
import java.util.concurrent.*;

public class NonBlockingFraudDemo {

    static int callFraudService() {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException ignored) {}
        return 85;
    }

    public static void main(String[] args) throws InterruptedException {

        ExecutorService fraudPool = Executors.newFixedThreadPool(4);

        for (int i = 0; i < 3; i++) {

            CompletableFuture
                    .supplyAsync(NonBlockingFraudDemo::callFraudService, fraudPool)
                    .thenApply(score -> {
                        if (score > 80)
                            return "REVIEW";
                        else
                            return "APPROVE";
                    })
                    .thenAccept(decision ->
                            System.out.println(Thread.currentThread().getName()
                                    + " Decision: " + decision));
        }

        Thread.sleep(5000);
        fraudPool.shutdown();
    }
}
```

---

## Explanation of the Correct Version

* Fraud call runs in a separate thread pool.
* We do not call `get()`.
* `thenApply` transforms the result.
* `thenAccept` consumes the final result.
* The main thread is not blocked.

Multiple fraud checks can run in parallel without blocking request threads.

---

## Common Mistakes

* Calling `get()` or `join()` inside request thread.
* Using the same pool for fast and slow tasks.
* Not handling exceptions in async chain.
* Forgetting to shut down executor.
* Assuming async automatically improves performance.

---

## Best Practices

* Use dedicated executor for slow external calls.
* Avoid blocking calls in async pipelines.
* Add timeout to external calls.
* Handle exceptions properly.
* Keep async chain readable and small.


---

# Topic: ExecutorService and Thread Pool Design

---

## Concept

In a Java application, every request runs on a thread.

Creating a new thread for every request is expensive because:

* Threads consume memory.
* Too many threads increase CPU context switching.
* System becomes unstable under heavy load.

Instead of creating threads again and again, we use a thread pool.

A thread pool:

* Creates a limited number of threads.
* Reuses them for multiple tasks.
* Controls how many tasks can run at the same time.

ExecutorService is used to manage these thread pools.

---

## Scenario

Imagine:

* Your payment service receives 10,000 requests.
* Each request takes 1 second to complete.
* Your server can only safely handle a limited number of concurrent tasks.

If thread management is not done properly:

* Either too many tasks wait in memory.
* Or too many threads are created.
* Both situations can crash the system.

---

## Goal

* Control the number of running threads.
* Prevent unlimited memory usage.
* Keep the system stable during high traffic.

---

# What Can Go Wrong

There are two common mistakes.

---

## Case 1: Fixed Thread Pool with Unlimited Queue

When you create a fixed thread pool like this:

```java
ExecutorService pool = Executors.newFixedThreadPool(2);
```

This means:

* Only 2 threads can run at the same time.
* All extra tasks wait in a queue.
* That queue has no size limit.

Now imagine:

* 10,000 tasks are submitted.
* Each task takes 1 second.

What happens?

* 2 tasks start running.
* 9,998 tasks are stored in memory inside the queue.
* If more requests keep coming, the queue keeps growing.
* Memory usage increases.
* Eventually, the system may crash with OutOfMemoryError.

---

### Runnable Example (Fixed Thread Pool Problem)

```java
import java.util.concurrent.*;

public class FixedPoolProblem {

    public static void main(String[] args) {

        ExecutorService pool = Executors.newFixedThreadPool(2);

        for (int i = 0; i < 10000; i++) {
            pool.submit(() -> {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException ignored) {}
            });
        }

        pool.shutdown();
    }
}
```

In this program:

* Only 2 tasks execute.
* The rest wait in memory.
* If traffic continues, memory grows continuously.

---

## Case 2: Cached Thread Pool Creating Too Many Threads

Now consider this:

```java
ExecutorService pool = Executors.newCachedThreadPool();
```

This pool works differently.

* It creates new threads when needed.
* It does not limit the number of threads.

If 10,000 tasks arrive quickly:

* It may create thousands of threads.
* Each thread uses memory.
* Too many threads cause heavy CPU context switching.
* System becomes slow or unstable.

---

### Runnable Example (Cached Thread Pool Problem)

```java
import java.util.concurrent.*;

public class CachedPoolProblem {

    public static void main(String[] args) {

        ExecutorService pool = Executors.newCachedThreadPool();

        for (int i = 0; i < 10000; i++) {
            pool.submit(() -> {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException ignored) {}
            });
        }

        pool.shutdown();
    }
}
```

Here:

* Instead of queue growing,
* Thread count grows.
* Memory usage increases quickly.

---

## Why These Problems Happen

In Case 1:
Memory grows because too many tasks are waiting in queue.

In Case 2:
Memory grows because too many threads are created.

In both cases:
System becomes unstable under heavy load.

---

# Correct Implementation

Instead of using default Executors blindly, we create a ThreadPoolExecutor manually.

This allows us to:

* Limit number of threads.
* Limit queue size.
* Decide what happens when queue is full.

---

### Runnable Example (Controlled Thread Pool)

```java
import java.util.concurrent.*;

public class ControlledThreadPool {

    public static void main(String[] args) {

        ThreadPoolExecutor pool =
                new ThreadPoolExecutor(
                        2,                      // core threads
                        2,                      // max threads
                        0L,
                        TimeUnit.MILLISECONDS,
                        new ArrayBlockingQueue<>(5),  // bounded queue
                        new ThreadPoolExecutor.CallerRunsPolicy()
                );

        for (int i = 0; i < 20; i++) {
            pool.submit(() -> {
                try {
                    Thread.sleep(1000);
                    System.out.println(Thread.currentThread().getName() + " processed task");
                } catch (InterruptedException ignored) {}
            });
        }

        pool.shutdown();
    }
}
```

---

## Explanation of the Correct Version

* We set both core and max threads to 2.
* We use a bounded queue of size 5.
* If the queue is full, CallerRunsPolicy runs the task in the calling thread.
* This slows down the producer instead of crashing the system.
* Memory remains controlled.

Now:

* Tasks do not grow infinitely.
* Threads do not grow infinitely.
* System behaves predictably.

---

## Common Mistakes

* Using default Executors without understanding their behavior.
* Ignoring queue size.
* Not defining rejection policy.
* Mixing slow and fast tasks in same pool.
* Forgetting to shut down executor.

---

## Best Practices

* Prefer ThreadPoolExecutor for full control.
* Always use bounded queues.
* Define a proper rejection policy.
* Use separate thread pools for different task types.
* Monitor pool size and queue size in production.

---

# Topic: ScheduledExecutorService (Delayed and Periodic Tasks)

---

## Concept

Sometimes we don’t want to run a task immediately.

We may want to:

* Run it after some delay
* Run it repeatedly at fixed intervals

Instead of using `Thread.sleep()` inside normal threads, Java provides `ScheduledExecutorService`.

It allows us to:

* Schedule a task to run later
* Schedule a task to run again and again

This keeps threads free and makes the system cleaner.

---

## Scenario

Imagine:

* A payment fails due to temporary network issue.
* We want to retry after 2 seconds.
* Or we want to check pending transactions every 10 seconds.

If we use `Thread.sleep()` inside a worker thread:

* That thread becomes blocked.
* It cannot process other tasks.
* Under load, this wastes threads.

We need a better way.

---

## Goal

* Run tasks after a delay.
* Run tasks periodically.
* Avoid blocking threads unnecessarily.
* Keep thread pool efficient.

---

# What Can Go Wrong

A common mistake is doing this:

```java
Thread.sleep(2000);
retryPayment();
```

What happens here?

* The thread calling `sleep()` becomes blocked.
* It does nothing for 2 seconds.
* If many threads do this, thread pool becomes full.
* System slows down.

Another mistake:

* Using a normal ExecutorService and manually managing delays.
* This makes code messy and harder to manage.

---

## Incorrect Example (Blocking Sleep)

```java
import java.util.concurrent.*;

public class BadRetryExample {

    public static void main(String[] args) {

        ExecutorService pool = Executors.newFixedThreadPool(2);

        pool.submit(() -> {
            System.out.println("Payment failed. Waiting to retry...");
            try {
                Thread.sleep(2000); // blocks thread
            } catch (InterruptedException ignored) {}

            System.out.println("Retrying payment...");
        });

        pool.shutdown();
    }
}
```

Problem:

* That thread is doing nothing for 2 seconds.
* If many retries happen, threads stay blocked.
* Thread pool capacity reduces.

---

# Correct Implementation

Use ScheduledExecutorService.

---

### Runnable Example (Delayed Task)

```java
import java.util.concurrent.*;

public class ScheduledRetryExample {

    public static void main(String[] args) {

        ScheduledExecutorService scheduler =
                Executors.newScheduledThreadPool(2);

        System.out.println("Payment failed. Scheduling retry...");

        scheduler.schedule(() -> {
            System.out.println("Retrying payment after delay...");
        }, 2, TimeUnit.SECONDS);

        scheduler.shutdown();
    }
}
```

Here:

* The task is scheduled.
* No thread is blocked waiting.
* Scheduler runs the task after 2 seconds.

---

## Periodic Task Example

```java
import java.util.concurrent.*;

public class PeriodicCheckExample {

    public static void main(String[] args) throws InterruptedException {

        ScheduledExecutorService scheduler =
                Executors.newScheduledThreadPool(1);

        scheduler.scheduleAtFixedRate(() -> {
            System.out.println("Checking pending transactions...");
        }, 0, 3, TimeUnit.SECONDS);

        Thread.sleep(10000);
        scheduler.shutdown();
    }
}
```

This runs every 3 seconds.

---

## Explanation

There are two main methods:

### schedule()

Runs task once after a delay.

### scheduleAtFixedRate()

Runs task repeatedly at fixed interval.

Difference:

* schedule() → one-time execution
* scheduleAtFixedRate() → repeated execution

No thread is blocked unnecessarily.

Scheduler manages execution timing.

---

## Common Mistakes

* Using Thread.sleep() inside request threads.
* Forgetting to shut down scheduler.
* Scheduling too many periodic tasks.
* Using one scheduler for unrelated heavy tasks.
* Ignoring exception handling inside scheduled task.

---

## Best Practices

* Use ScheduledExecutorService for delayed or repeated tasks.
* Keep scheduled tasks small and fast.
* Handle exceptions inside scheduled tasks.
* Use separate scheduler for retry logic if needed.
* Always shut down scheduler properly.

---
Good point 👍 — let’s check the **sequence (flow)** of the Fork/Join topic and make sure it follows your required structure properly.

Your required flow was:

Concept
→ Scenario
→ Goal
→ What Can Go Wrong
→ Incorrect Example
→ Why It Fails
→ Correct Implementation
→ Explanation of Correct Version
→ Common Mistakes
→ Best Practices

---

# Topic: Fork/Join and RecursiveTask for Aggregation

---

## Concept

When processing a very large amount of data, using a single thread may not fully use the CPU.

Modern systems have multiple cores. Fork/Join helps us split large work into smaller parts, process them in parallel, and combine the results.

The idea is simple:

Split → Process → Combine

---

## Scenario

You have 1 million transaction records.

You want to calculate the total transaction amount.

If only one thread processes all records, other CPU cores remain unused.

---

## Goal

* Use multiple CPU cores.
* Improve performance for CPU-heavy tasks.
* Combine results correctly.

---

# What Can Go Wrong

Before jumping into Fork/Join, let’s see common mistakes.

### Mistake 1: Doing everything in one thread

Only one core works. Performance is limited.

### Mistake 2: Splitting too much

If tasks are too small, overhead of managing them becomes high.

### Mistake 3: Blocking inside compute()

Fork/Join is meant for CPU work, not waiting.

### Mistake 4: Using it for small datasets

Overhead may be more than benefit.

---

# Incorrect Example (Single Thread Aggregation)

```java
public class SingleThreadSum {

    public static void main(String[] args) {

        long[] data = new long[1_000_000];

        for (int i = 0; i < data.length; i++) {
            data[i] = 1;
        }

        long sum = 0;

        for (int i = 0; i < data.length; i++) {
            sum += data[i];
        }

        System.out.println("Total Sum: " + sum);
    }
}
```

---

## Why It Fails

* Only one thread runs.
* Multiple CPU cores are unused.
* Large datasets take longer to process.

It works correctly, but not efficiently.

---

# Correct Implementation Using Fork/Join

```java
import java.util.concurrent.*;

public class ForkJoinSum {

    static class SumTask extends RecursiveTask<Long> {

        private long[] array;
        private int start;
        private int end;
        private static final int THRESHOLD = 10_000;

        SumTask(long[] array, int start, int end) {
            this.array = array;
            this.start = start;
            this.end = end;
        }

        @Override
        protected Long compute() {

            if (end - start <= THRESHOLD) {
                long sum = 0;
                for (int i = start; i < end; i++) {
                    sum += array[i];
                }
                return sum;
            }

            int mid = (start + end) / 2;

            SumTask left = new SumTask(array, start, mid);
            SumTask right = new SumTask(array, mid, end);

            left.fork();
            long rightResult = right.compute();
            long leftResult = left.join();

            return leftResult + rightResult;
        }
    }

    public static void main(String[] args) {

        long[] data = new long[1_000_000];

        for (int i = 0; i < data.length; i++) {
            data[i] = 1;
        }

        ForkJoinPool pool = new ForkJoinPool();

        long result = pool.invoke(new SumTask(data, 0, data.length));

        System.out.println("Total Sum: " + result);
    }
}
```

---

## Explanation of the Correct Version

* If the data chunk is small, calculate directly.
* If large, split into two halves.
* One half is forked (sent to another worker thread).
* The other half is computed immediately.
* Results are combined using join().
* ForkJoinPool distributes work across CPU cores.

---

## Common Mistakes

* Choosing very small threshold.
* Choosing very large threshold.
* Doing blocking operations inside compute().
* Using Fork/Join for I/O tasks.

---

## Best Practices

* Use Fork/Join only for CPU-heavy tasks.
* Tune threshold using performance testing.
* Avoid blocking calls.
* Measure performance before and after.

---


Topic: **thenApply vs thenCompose**

---

# Concept

Both `thenApply` and `thenCompose` are used with `CompletableFuture`.

They are used to perform the next step after a previous async task completes.

The difference is simple:

* `thenApply` is used when the next step returns a normal value.
* `thenCompose` is used when the next step itself returns another `CompletableFuture`.

If you use the wrong one, you may end up with nested futures.

---

# Scenario

You have a payment flow:

1. Fetch user details asynchronously.
2. Call fraud service asynchronously.
3. Based on fraud score, decide approval.

Both operations are async.

Now you need to chain them correctly.

---

# Goal

* Chain async tasks properly.
* Avoid nested CompletableFuture objects.
* Keep async flow clean and readable.

---

# What Can Go Wrong

If you use `thenApply` when the next method already returns a `CompletableFuture`, you will get this:

`CompletableFuture<CompletableFuture<Result>>`

Now you have a future inside a future.

This makes handling results complicated.

---

# Incorrect Example (Nested Future Problem)

```java
import java.util.concurrent.*;

public class WrongComposeExample {

    static CompletableFuture<String> callFraudService(String user) {
        return CompletableFuture.supplyAsync(() -> {
            sleep();
            return "FraudCheckDone for " + user;
        });
    }

    static void sleep() {
        try { Thread.sleep(1000); }
        catch (InterruptedException ignored) {}
    }

    public static void main(String[] args) throws Exception {

        CompletableFuture<CompletableFuture<String>> result =
                CompletableFuture
                        .supplyAsync(() -> "UserA")
                        .thenApply(user -> callFraudService(user));

        System.out.println(result.get().get());
    }
}
```

---

# Why It Fails

Look at this line:

```java
.thenApply(user -> callFraudService(user))
```

* `callFraudService()` already returns `CompletableFuture<String>`
* `thenApply` wraps the result inside another future

So you get:

Future<Future<String>>

To get actual result, you must call `.get().get()`

This is messy and unnecessary.

---

# Correct Implementation (Using thenCompose)

```java
import java.util.concurrent.*;

public class CorrectComposeExample {

    static CompletableFuture<String> callFraudService(String user) {
        return CompletableFuture.supplyAsync(() -> {
            sleep();
            return "FraudCheckDone for " + user;
        });
    }

    static void sleep() {
        try { Thread.sleep(1000); }
        catch (InterruptedException ignored) {}
    }

    public static void main(String[] args) throws Exception {

        CompletableFuture<String> result =
                CompletableFuture
                        .supplyAsync(() -> "UserA")
                        .thenCompose(user -> callFraudService(user));

        System.out.println(result.get());
    }
}
```

---

# Explanation

`thenCompose` automatically flattens the nested future.

Instead of:

Future<Future<String>>

It gives:

Future<String>

It works like:

* Run first async task.
* Take its result.
* Start next async task.
* Return its result directly.

You don’t need double get().

---

# When to Use What

Use `thenApply` when:

* The next step returns a normal value.
* Example: transform data.

Example:

```java
future.thenApply(value -> value.toUpperCase());
```

Use `thenCompose` when:

* The next step returns another CompletableFuture.
* Example: calling another async service.

Example:

```java
future.thenCompose(value -> asyncCall(value));
```

---

# Common Mistakes

* Using thenApply for async methods.
* Getting nested futures.
* Calling get() inside async chain.
* Blocking inside async pipeline.

---

# Best Practices

* Remember: Apply → normal value.
* Compose → async value.
* Keep async chain readable.
* Avoid nested futures.
* Avoid blocking calls inside chain.

---

Topic: **thenCombine**

---

# Concept

`thenCombine` is used when you have **two independent asynchronous tasks**, and you want to combine their results.

Important:

* Both tasks run in parallel.
* Neither depends on the other.
* After both complete, you merge their results.

Simple rule:

If Task A and Task B are independent, use `thenCombine`.

---

# Scenario

You are processing a payment.

You need:

1. User account details (async call)
2. Fraud score (async call)

These two calls:

* Do not depend on each other.
* Can run at the same time.

After both finish:

* You combine the results.
* Make final decision.

---

# Goal

* Run independent async tasks in parallel.
* Combine their results safely.
* Avoid blocking calls.

---

# What Can Go Wrong

Common mistake:

Calling `get()` on first future before starting the second.

That makes execution sequential instead of parallel.

Example mistake:

```java id="krzovr"
String user = userFuture.get();
String fraud = fraudFuture.get();
```

This blocks and removes parallel benefit.

Another mistake:

Using `thenCompose` when tasks are independent.

`thenCompose` creates dependency chain.
`thenCombine` is meant for parallel tasks.

---

# Incorrect Example (Sequential Execution)

```java id="e3gkxf"
import java.util.concurrent.*;

public class WrongCombineExample {

    static String fetchUser() {
        sleep();
        return "UserData";
    }

    static String fetchFraud() {
        sleep();
        return "FraudScore";
    }

    static void sleep() {
        try { Thread.sleep(1000); }
        catch (InterruptedException ignored) {}
    }

    public static void main(String[] args) throws Exception {

        CompletableFuture<String> userFuture =
                CompletableFuture.supplyAsync(WrongCombineExample::fetchUser);

        // BAD: blocking call
        String user = userFuture.get();

        CompletableFuture<String> fraudFuture =
                CompletableFuture.supplyAsync(WrongCombineExample::fetchFraud);

        String fraud = fraudFuture.get();

        System.out.println(user + " + " + fraud);
    }
}
```

---

# Why It Fails

* `get()` blocks the thread.
* Fraud call starts only after user call completes.
* Total time becomes 2 seconds instead of 1 second.
* No parallel execution.

---

# Correct Implementation Using thenCombine

```java id="d3om9q"
import java.util.concurrent.*;

public class CorrectCombineExample {

    static String fetchUser() {
        sleep();
        return "UserData";
    }

    static String fetchFraud() {
        sleep();
        return "FraudScore";
    }

    static void sleep() {
        try { Thread.sleep(1000); }
        catch (InterruptedException ignored) {}
    }

    public static void main(String[] args) throws Exception {

        CompletableFuture<String> userFuture =
                CompletableFuture.supplyAsync(CorrectCombineExample::fetchUser);

        CompletableFuture<String> fraudFuture =
                CompletableFuture.supplyAsync(CorrectCombineExample::fetchFraud);

        CompletableFuture<String> combined =
                userFuture.thenCombine(
                        fraudFuture,
                        (user, fraud) -> user + " + " + fraud
                );

        System.out.println(combined.get());
    }
}
```

---

# Explanation

* Both async tasks start immediately.
* They run in parallel.
* When both complete, `thenCombine` runs.
* It receives both results.
* It merges them.

Total time ≈ 1 second (not 2 seconds).

Parallel execution achieved.

---

# When to Use thenCombine

Use when:

* You have two independent async tasks.
* You need both results.
* Neither task depends on the other.

---

# Common Mistakes

* Blocking with get() before combining.
* Using thenCompose for independent tasks.
* Mixing dependent and independent flows incorrectly.
* Forgetting exception handling.

---

# Best Practices

* Start independent futures first.
* Combine them using thenCombine.
* Avoid blocking calls.
* Handle exceptions in both futures.
* Keep combination logic simple.

---

Topic: **allOf**

---

# Concept

`CompletableFuture.allOf()` is used when you have **multiple async tasks**, and you want to wait until all of them finish.

It does not combine results directly.
It simply tells you: “All tasks are complete.”

After that, you can collect results from each future.

Use it when:

* You launch many independent async tasks.
* You want to continue only after all of them complete.

---

# Scenario

Suppose you are generating a payment report.

You need to fetch:

1. User details
2. Fraud score
3. Transaction history
4. Account limits

All these are independent async calls.

You want to:

* Start all calls in parallel.
* Wait until all are finished.
* Then generate final report.

---

# Goal

* Start multiple async tasks together.
* Wait until all complete.
* Avoid blocking each one individually.
* Keep execution parallel.

---

# What Can Go Wrong

Common mistake:

Calling `get()` one by one:

```java
id="bqsnjm"
String user = userFuture.get();
String fraud = fraudFuture.get();
String history = historyFuture.get();
```

This blocks repeatedly and may destroy parallel benefit.

Another mistake:

Assuming `allOf()` returns combined results automatically.

It does not.

It only returns `CompletableFuture<Void>`.

You must manually extract results.

---

# Incorrect Example (Sequential Blocking)

```java
import java.util.concurrent.*;

public class WrongAllOfExample {

    static String slowTask(String name) {
        try { Thread.sleep(1000); }
        catch (InterruptedException ignored) {}
        return name;
    }

    public static void main(String[] args) throws Exception {

        CompletableFuture<String> f1 =
                CompletableFuture.supplyAsync(() -> slowTask("User"));

        CompletableFuture<String> f2 =
                CompletableFuture.supplyAsync(() -> slowTask("Fraud"));

        // BAD: blocking one by one
        String r1 = f1.get();
        String r2 = f2.get();

        System.out.println(r1 + " " + r2);
    }
}
```

Problem:

* You are manually blocking.
* Logic becomes messy.
* Harder to scale when futures increase.

---

# Correct Implementation Using allOf

```java
import java.util.concurrent.*;

public class CorrectAllOfExample {

    static String slowTask(String name) {
        try { Thread.sleep(1000); }
        catch (InterruptedException ignored) {}
        return name;
    }

    public static void main(String[] args) throws Exception {

        CompletableFuture<String> f1 =
                CompletableFuture.supplyAsync(() -> slowTask("User"));

        CompletableFuture<String> f2 =
                CompletableFuture.supplyAsync(() -> slowTask("Fraud"));

        CompletableFuture<String> f3 =
                CompletableFuture.supplyAsync(() -> slowTask("History"));

        CompletableFuture<Void> all =
                CompletableFuture.allOf(f1, f2, f3);

        CompletableFuture<String> finalResult =
                all.thenApply(v ->
                        f1.join() + " " +
                        f2.join() + " " +
                        f3.join()
                );

        System.out.println(finalResult.get());
    }
}
```

---

# Explanation

1. All three async tasks start immediately.
2. `allOf()` waits for all to finish.
3. `thenApply()` runs only after all complete.
4. We use `join()` to get individual results.
5. No sequential blocking logic.

Parallel execution remains intact.

---

# Important Note

`allOf()` returns `CompletableFuture<Void>`.

It does not merge results automatically.

You must combine results manually inside `thenApply()`.

---

# Common Mistakes

* Calling get() repeatedly.
* Assuming allOf returns combined result.
* Ignoring exception handling.
* Forgetting that if one future fails, allOf fails.

---

# Best Practices

* Start all independent futures first.
* Use allOf to wait for completion.
* Use join() inside thenApply to collect results.
* Handle exceptions properly.
* Keep result-combining logic clean.

---


Topic: **Error Handling in CompletableFuture — exceptionally vs handle**

---

# Concept

When working with asynchronous tasks, failures can happen:

* Network call fails
* Database timeout
* Invalid input
* Unexpected exception

If you don’t handle errors properly, your async pipeline may:

* Stop silently
* Propagate exception unexpectedly
* Crash later when calling `get()`

`CompletableFuture` provides two common methods to deal with errors:

* `exceptionally()`
* `handle()`

Both are used to react to exceptions, but they behave slightly differently.

---

# Scenario

You call a fraud service asynchronously.

Sometimes:

* The fraud system may be down.
* It may throw a runtime exception.

You want to:

* Catch the error
* Return a safe fallback value (like default score)
* Continue the flow without crashing

---

# Goal

* Catch async exceptions properly.
* Provide fallback value if needed.
* Keep async flow stable.
* Avoid unexpected crashes.

---

# What Can Go Wrong

Common mistake:

Not handling exceptions at all.

```java id="hmp9xy"
CompletableFuture<String> future =
        CompletableFuture.supplyAsync(() -> {
            throw new RuntimeException("Fraud service failed");
        });

System.out.println(future.get());
```

What happens?

* `get()` throws ExecutionException.
* If not handled, application may crash.

Another mistake:

Using only `thenApply()` and assuming exceptions are handled automatically.

They are not.

If an exception occurs, subsequent stages are skipped.

---

# Incorrect Example (No Error Handling)

```java id="ohb9v3"
import java.util.concurrent.*;

public class NoErrorHandling {

    public static void main(String[] args) throws Exception {

        CompletableFuture<String> future =
                CompletableFuture.supplyAsync(() -> {
                    throw new RuntimeException("Service failed");
                });

        System.out.println(future.get()); // Exception here
    }
}
```

---

# Why It Fails

* Exception occurs inside async task.
* Future completes exceptionally.
* Calling `get()` throws ExecutionException.
* If not caught, program terminates.

Async does not mean automatic error handling.

---

# Correct Implementation Using exceptionally()

```java id="5r5x1k"
import java.util.concurrent.*;

public class ExceptionallyExample {

    public static void main(String[] args) throws Exception {

        CompletableFuture<String> future =
                CompletableFuture.supplyAsync(() -> {
                    throw new RuntimeException("Service failed");
                })
                .exceptionally(ex -> {
                    System.out.println("Error occurred: " + ex.getMessage());
                    return "DefaultValue";
                });

        System.out.println(future.get());
    }
}
```

---

# Explanation (exceptionally)

* If no error occurs → original result continues.
* If error occurs → exceptionally runs.
* It returns fallback value.
* Flow continues normally.

Use this when:

You only want to handle failures.

---

# Correct Implementation Using handle()

```java id="s6h4d7"
import java.util.concurrent.*;

public class HandleExample {

    public static void main(String[] args) throws Exception {

        CompletableFuture<String> future =
                CompletableFuture.supplyAsync(() -> {
                    throw new RuntimeException("Service failed");
                })
                .handle((result, ex) -> {
                    if (ex != null) {
                        System.out.println("Error occurred: " + ex.getMessage());
                        return "FallbackValue";
                    }
                    return result;
                });

        System.out.println(future.get());
    }
}
```

---

# Explanation (handle)

* `handle()` always runs.
* It receives:

  * result (if successful)
  * exception (if failed)
* You decide what to return.
* It can process both success and failure.

Use this when:

You want to handle both success and error in one place.

---

# Difference Between exceptionally and handle

exceptionally:

* Runs only when error happens.
* Simpler for fallback.

handle:

* Runs always.
* More flexible.
* Can modify success result too.

---

# Common Mistakes

* Ignoring async exceptions.
* Calling get() without try-catch.
* Using exceptionally when success logic also needs modification.
* Swallowing exceptions silently without logging.

---

# Best Practices

* Always handle exceptions in async pipelines.
* Log the error before returning fallback.
* Use exceptionally for simple fallback.
* Use handle when both success and failure need logic.
* Avoid blocking get() in production flows.

---


Topic: **Timeout Handling — orTimeout() and completeOnTimeout()**

---

# Concept

When calling an external service asynchronously, sometimes the service may:

* Take too long
* Hang indefinitely
* Never respond

If you don’t control waiting time, your application may:

* Keep threads occupied
* Delay other requests
* Slow down the entire system

Timeout handling ensures:

If a task takes too long, we stop waiting and take a decision.

Java 9+ provides:

* `orTimeout()`
* `completeOnTimeout()`

---

# Scenario

You call a fraud service asynchronously.

Normally, it responds within 500 milliseconds.

But sometimes:

* It takes 5 seconds.
* Or it never responds.

You don’t want your payment system to wait forever.

You want:

* Either fail after 1 second
* Or return a default value

---

# Goal

* Prevent infinite waiting.
* Keep system responsive.
* Handle slow services safely.
* Maintain predictable behavior.

---

# What Can Go Wrong

Common mistake:

Not setting any timeout.

```java
CompletableFuture<String> future =
        CompletableFuture.supplyAsync(() -> slowService());

future.get();  // waits forever if service hangs
```

If the service hangs:

* The thread blocks.
* The system slows down.
* Under load, many threads may block.

Another mistake:

Using Thread.sleep to simulate timeout manually.

That does not stop the async task.

---

# Incorrect Example (No Timeout)

```java
import java.util.concurrent.*;

public class NoTimeoutExample {

    static String slowService() {
        try { Thread.sleep(5000); }
        catch (InterruptedException ignored) {}
        return "FraudScore";
    }

    public static void main(String[] args) throws Exception {

        CompletableFuture<String> future =
                CompletableFuture.supplyAsync(NoTimeoutExample::slowService);

        System.out.println(future.get()); // waits 5 seconds
    }
}
```

Problem:

* If service takes 30 seconds, your code waits 30 seconds.
* No control over delay.

---

# Correct Implementation Using orTimeout()

```java
import java.util.concurrent.*;

public class OrTimeoutExample {

    static String slowService() {
        try { Thread.sleep(5000); }
        catch (InterruptedException ignored) {}
        return "FraudScore";
    }

    public static void main(String[] args) throws Exception {

        CompletableFuture<String> future =
                CompletableFuture.supplyAsync(OrTimeoutExample::slowService)
                        .orTimeout(1, TimeUnit.SECONDS);

        try {
            System.out.println(future.get());
        } catch (Exception e) {
            System.out.println("Timed out!");
        }
    }
}
```

---

## Explanation (orTimeout)

* If task completes within 1 second → normal result.
* If it exceeds 1 second → TimeoutException.
* The future completes exceptionally.
* You handle the exception.

Use this when:

You want to fail fast if service is slow.

---

# Correct Implementation Using completeOnTimeout()

```java
import java.util.concurrent.*;

public class CompleteOnTimeoutExample {

    static String slowService() {
        try { Thread.sleep(5000); }
        catch (InterruptedException ignored) {}
        return "FraudScore";
    }

    public static void main(String[] args) throws Exception {

        CompletableFuture<String> future =
                CompletableFuture.supplyAsync(CompleteOnTimeoutExample::slowService)
                        .completeOnTimeout("DefaultScore", 1, TimeUnit.SECONDS);

        System.out.println(future.get());
    }
}
```

---

## Explanation (completeOnTimeout)

* If task finishes within 1 second → normal result.
* If not → it returns "DefaultScore".
* No exception thrown.
* Flow continues smoothly.

Use this when:

You want a fallback value instead of failing.

---

# Difference Between orTimeout and completeOnTimeout

orTimeout:

* Throws exception on timeout.
* Good when timeout should be treated as failure.

completeOnTimeout:

* Returns default value.
* Good when fallback is acceptable.

---

# Common Mistakes

* Not setting timeout for external calls.
* Assuming async automatically prevents delay.
* Not handling TimeoutException.
* Using very small timeout without testing.

---

# Best Practices

* Always define timeout for external services.
* Choose timeout based on realistic SLAs.
* Log timeout events.
* Combine timeout with error handling.
* Test behavior under slow service simulation.

---

Topic: **Rate Limiting using Semaphore**

---

# Concept

Sometimes your system must limit how many tasks run at the same time.

Example:

* Fraud system can handle only 5 requests at once.
* Database can handle limited concurrent connections.
* External API has rate limits.

If too many requests hit at once:

* System slows down.
* External service may reject calls.
* Your application may crash.

`Semaphore` helps control how many threads can enter a section of code at the same time.

Simple idea:

Only N tasks allowed at once.

---

# Scenario

You are calling a fraud service.

* Fraud service can safely handle 3 concurrent requests.
* If more than 3 hit at the same time, it slows down or fails.

Your system receives 20 payment requests.

You want:

* Only 3 fraud checks running simultaneously.
* Remaining requests should wait.

---

# Goal

* Protect downstream systems.
* Prevent overload.
* Control concurrency.
* Keep system stable under traffic spikes.

---

# What Can Go Wrong

If you do not limit concurrency:

```java id="j2s0ka"
CompletableFuture.runAsync(() -> callFraudService());
```

If 1000 requests come:

* 1000 fraud calls may run at once.
* Fraud service may slow down.
* Response time increases.
* Failures increase.

Another mistake:

Acquiring semaphore but forgetting to release it.

That causes deadlock.

---

# Incorrect Example (No Rate Limiting)

```java id="ipk9jp"
import java.util.concurrent.*;

public class NoRateLimitExample {

    static void callFraudService() {
        try {
            Thread.sleep(1000);
            System.out.println("Fraud check done by " +
                    Thread.currentThread().getName());
        } catch (InterruptedException ignored) {}
    }

    public static void main(String[] args) {

        ExecutorService pool = Executors.newFixedThreadPool(10);

        for (int i = 0; i < 10; i++) {
            pool.submit(NoRateLimitExample::callFraudService);
        }

        pool.shutdown();
    }
}
```

Here:

* All tasks run at once (up to pool size).
* No control over downstream capacity.

---

# Correct Implementation Using Semaphore

```java id="3pt1wk"
import java.util.concurrent.*;

public class RateLimitExample {

    static Semaphore semaphore = new Semaphore(3);

    static void callFraudService() {

        try {
            semaphore.acquire();   // take permit

            System.out.println("Processing by "
                    + Thread.currentThread().getName());

            Thread.sleep(1000);

        } catch (InterruptedException ignored) {
        } finally {
            semaphore.release();   // always release
        }
    }

    public static void main(String[] args) {

        ExecutorService pool = Executors.newFixedThreadPool(10);

        for (int i = 0; i < 10; i++) {
            pool.submit(RateLimitExample::callFraudService);
        }

        pool.shutdown();
    }
}
```

---

# Explanation

* Semaphore initialized with 3 permits.
* Only 3 threads can acquire permit at a time.
* Other threads wait.
* When a thread finishes, it releases permit.
* Next waiting thread proceeds.

This ensures:

Maximum 3 concurrent fraud calls.

---

# Why This Works

* Controls concurrency without blocking entire system.
* Protects external services.
* Smooths traffic spikes.
* Keeps memory and CPU stable.

---

# Common Mistakes

* Forgetting to release permit.
* Using acquire() without try-finally.
* Setting permit count too high or too low.
* Confusing rate limiting with thread pool size.

---

# Best Practices

* Always release permit in finally block.
* Set permit count based on downstream capacity.
* Monitor concurrency metrics.
* Combine semaphore with proper timeout handling.
* Do not block indefinitely; consider tryAcquire with timeout.

Example with timeout:

```java id="pm3h0g"
if (semaphore.tryAcquire(1, TimeUnit.SECONDS)) {
    try {
        callFraudService();
    } finally {
        semaphore.release();
    }
} else {
    System.out.println("Too many requests, rejecting.");
}
```

---













