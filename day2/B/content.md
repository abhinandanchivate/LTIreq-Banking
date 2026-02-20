

We will cover:

* Happens-Before (volatile / synchronized)
* Visibility & Reordering
* Heap / Stack / Metaspace
* G1 GC + Heap Pressure
* Memory Leak Creation & Detection

Everything plain Java. No shortcuts.

---

# 🔵 PART 1 — HAPPENS-BEFORE (volatile / synchronized)

---

# 📌 CONCEPT

## What

Happens-before is a rule in Java Memory Model (JMM) that guarantees:

> If A happens-before B, then all memory writes in A are visible to B.

---

## Why

Without happens-before:

* Threads see stale values
* Reordering causes broken logic
* Banking transactions may see inconsistent state

---

## When

* Multi-threaded programs
* Shared mutable state
* Concurrent transaction processing

---

## How

Established via:

* volatile write → read
* unlock → subsequent lock
* thread.start()
* thread.join()

---

## Where (Banking Use Cases)

* Payment shutdown flag
* Fraud score sharing
* Transaction state visibility
* Balance consistency

---

# 🏦 Banking Example Scenario

A fraud engine sets `transactionApproved = true`,
Settlement thread must see it immediately.

Without happens-before → settlement may never see update.

---

# ✅ Best Practices

* Use volatile for flags
* Use synchronized for compound operations
* Avoid unsynchronized shared state
* Prefer immutable objects

---

# 🟢 SIMPLE EXAMPLE 1 — Without volatile (Visibility Bug)

```java
public class HB_Simple1 {

    static boolean approved = false;

    public static void main(String[] args) throws Exception {

        Thread fraud = new Thread(() -> {
            sleep(1000);
            approved = true;
        });

        Thread settlement = new Thread(() -> {
            while (!approved) {}
            System.out.println("Settled");
        });

        settlement.start();
        fraud.start();
    }

    static void sleep(int ms) {
        try { Thread.sleep(ms); } catch (Exception e) {}
    }
}
```

May hang forever.

---

# 🟢 SIMPLE EXAMPLE 2 — Fix Using volatile

```java
public class HB_Simple2 {

    static volatile boolean approved = false;

    public static void main(String[] args) throws Exception {

        Thread fraud = new Thread(() -> {
            sleep(1000);
            approved = true;
        });

        Thread settlement = new Thread(() -> {
            while (!approved) {}
            System.out.println("Settled");
        });

        settlement.start();
        fraud.start();
    }

    static void sleep(int ms) {
        try { Thread.sleep(ms); } catch (Exception e) {}
    }
}
```

Now guaranteed visibility.

---

# 🔴 COMPLEX EXAMPLE 1 — Double Checked Locking

```java
public class HB_Complex1 {

    static class Config {}

    private static volatile Config instance;

    public static Config getInstance() {
        if (instance == null) {
            synchronized (HB_Complex1.class) {
                if (instance == null) {
                    instance = new Config();
                }
            }
        }
        return instance;
    }
}
```

Volatile prevents reordering.

---

# 🔴 COMPLEX EXAMPLE 2 — Lock-based Happens-Before

```java
public class HB_Complex2 {

    private static int balance = 0;

    public static void main(String[] args) throws Exception {

        Thread t1 = new Thread(() -> {
            synchronized (HB_Complex2.class) {
                balance = 100;
            }
        });

        Thread t2 = new Thread(() -> {
            synchronized (HB_Complex2.class) {
                System.out.println(balance);
            }
        });

        t1.start();
        t2.start();
    }
}
```

Unlock → subsequent lock guarantees visibility.

---

---

# 🔵 PART 2 — VISIBILITY & REORDERING

---

# 📌 CONCEPT

## What

Visibility: when one thread’s writes are visible to another.

Reordering: JVM/CPU may reorder instructions for optimization.

---

## Why

Performance optimization causes:

* Out-of-order execution
* Stale reads
* Half-initialized objects

---

## When

* High-performance trading engines
* Payment gateways
* Multi-core fraud systems

---

## Banking Failure Example

Fraud score calculated but settlement sees default value.

---

# 🟢 SIMPLE EXAMPLE 1 — Reordering Issue

```java
public class Reorder_Simple1 {

    static int x = 0;
    static boolean ready = false;

    public static void main(String[] args) {

        new Thread(() -> {
            x = 42;
            ready = true;
        }).start();

        new Thread(() -> {
            if (ready) {
                System.out.println(x);
            }
        }).start();
    }
}
```

May print 0.

---

# 🟢 SIMPLE EXAMPLE 2 — Fix with volatile

```java
static volatile boolean ready = false;
```

Prevents reordering.

---

# 🔴 COMPLEX EXAMPLE 1 — Bank Transaction State Machine

```java
public class Reorder_Complex1 {

    static class Tx {
        int amount;
        boolean processed;
    }

    static Tx tx = new Tx();

    public static void main(String[] args) {

        new Thread(() -> {
            tx.amount = 500;
            tx.processed = true;
        }).start();

        new Thread(() -> {
            if (tx.processed) {
                System.out.println(tx.amount);
            }
        }).start();
    }
}
```

Fix: make processed volatile.

---

# 🔴 COMPLEX EXAMPLE 2 — Lazy Fraud Engine Init

Without volatile, reference may be seen before construction completes.

---

---

# 🔵 PART 3 — HEAP / STACK / METASPACE

---

# 📌 CONCEPT

## Heap

Shared memory for objects.

## Stack

Per-thread call frames.

## Metaspace

Stores class metadata.

---

# 🏦 Banking Impact

* Heap overflow → payment outage
* Stack overflow → recursive fraud loop crash
* Metaspace leak → dynamic proxy explosion

---

# 🟢 SIMPLE EXAMPLE 1 — Stack Overflow

```java
public class StackOverflowDemo {
    static void recurse() { recurse(); }

    public static void main(String[] args) {
        recurse();
    }
}
```

---

# 🟢 SIMPLE EXAMPLE 2 — Heap Allocation

```java
public class HeapDemo {
    public static void main(String[] args) {
        byte[] data = new byte[10_000_000];
        System.out.println("Allocated");
    }
}
```

---

# 🔴 COMPLEX EXAMPLE 1 — Heap Pressure Simulation (G1)

Run with:

```
-Xms256m -Xmx256m -XX:+UseG1GC -Xlog:gc
```

```java
import java.util.*;

public class HeapPressure {

    public static void main(String[] args) throws Exception {
        List<byte[]> list = new ArrayList<>();

        while (true) {
            list.add(new byte[1024 * 1024]);
            Thread.sleep(100);
        }
    }
}
```

Observe GC behavior.

---

# 🔴 COMPLEX EXAMPLE 2 — Metaspace Leak via ClassLoader

```java
import java.lang.reflect.*;
import java.util.*;

public class MetaLeak {

    static List<Class<?>> classes = new ArrayList<>();

    public static void main(String[] args) throws Exception {

        while (true) {
            Proxy.getProxyClass(
                MetaLeak.class.getClassLoader(),
                Runnable.class
            );
        }
    }
}
```

Run with small metaspace:

```
-XX:MaxMetaspaceSize=64m
```

---

---

# 🔵 PART 4 — G1 GC TUNING

---

## What

Region-based GC optimized for predictable pause time.

---

## Banking Why

Low-latency payment engines need predictable pauses.

---

## Key Flags

```
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:InitiatingHeapOccupancyPercent=45
```

---

# 🟢 SIMPLE EXAMPLE — Observe GC Logs

```
-Xlog:gc
```

Run heap pressure example.

---

# 🔴 COMPLEX EXAMPLE — Promotion Failure Simulation

Allocate large arrays repeatedly until old gen fills.

---

---

# 🔵 PART 5 — MEMORY LEAK CREATION & DETECTION

---

# 📌 What

Memory leak = unreachable objects still referenced.

---

# 🟢 SIMPLE EXAMPLE 1 — Static Leak

```java
import java.util.*;

public class StaticLeak {
    static List<byte[]> cache = new ArrayList<>();

    public static void main(String[] args) {
        while (true) {
            cache.add(new byte[1024 * 1024]);
        }
    }
}
```

---

# 🟢 SIMPLE EXAMPLE 2 — ThreadLocal Leak

```java
public class ThreadLocalLeak {

    static ThreadLocal<byte[]> local = new ThreadLocal<>();

    public static void main(String[] args) {
        local.set(new byte[10_000_000]);
    }
}
```

---

# 🔴 COMPLEX EXAMPLE 1 — Listener Leak

Store listeners in static map without removal.

---

# 🔴 COMPLEX EXAMPLE 2 — ExecutorService Leak

```java
public class ExecutorLeak {
    public static void main(String[] args) {
        Executors.newFixedThreadPool(5);
    }
}
```

Never shutdown → threads never die.

---

# 🔍 Detection

## Using JVisualVM

1. Run app
2. Monitor heap growth
3. Take heap dump
4. Inspect retained objects

## Using MAT

1. Open heap dump
2. Dominator tree
3. Find largest retained object

---

---


