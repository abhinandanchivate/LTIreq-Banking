
---

# 1️⃣ Happens-Before (volatile / synchronized)

---

## Concept

Happens-before is a rule in the Java Memory Model that guarantees:

If one action happens-before another, then:

* The second action sees all changes made by the first.
* Execution order is preserved for visibility.

It solves visibility and ordering issues between threads.

---

## Scenario

Thread A validates a payment and sets `approved = true`.
Thread B waits for approval before settlement.

---

## Goal

Ensure Thread B always sees the updated value of `approved`.

---

## What Can Go Wrong

* Thread B never sees the update.
* Infinite loop.
* Payment stuck forever.

---

## ❌ Incorrect Example

```java
class Payment {
    boolean approved = false;

    void approve() {
        approved = true;
    }

    void waitForApproval() {
        while (!approved) { }
        System.out.println("Settled");
    }
}
```

---

## Why It Fails

* Each thread may cache the variable.
* No memory barrier.
* JVM may reorder instructions.
* No happens-before relationship exists.

---

## ✅ Correct Implementation (volatile)

```java
class Payment {
    private volatile boolean approved = false;

    void approve() {
        approved = true;
    }

    void waitForApproval() {
        while (!approved) { }
        System.out.println("Settled");
    }
}
```

---

## Explanation

`volatile`:

* Forces write to main memory.
* Forces read from main memory.
* Prevents reordering around that variable.
* Creates happens-before between write and read.

---

## Common Mistakes

* Using volatile for increment operations.
* Assuming volatile makes operations atomic.
* Accessing same variable sometimes synchronized, sometimes not.

---

## Best Practices

* Use volatile only for simple state flags.
* Use synchronized for compound operations.
* Use Atomic classes for counters.

---

# 2️⃣ Visibility & Reordering (Double Checked Locking)

---

## Concept

JVM and CPU may reorder instructions for optimization.
Without proper memory barriers, another thread may observe partially constructed objects.

---

## Scenario

Lazy initialization of a singleton fraud engine.

---

## Goal

Initialize object once and safely publish it to all threads.

---

## What Can Go Wrong

Thread sees non-null object reference but object fields are not initialized.

---

## ❌ Incorrect Example

```java
class FraudEngine {
    private static FraudEngine instance;

    public static FraudEngine getInstance() {
        if (instance == null) {
            synchronized (FraudEngine.class) {
                if (instance == null) {
                    instance = new FraudEngine();
                }
            }
        }
        return instance;
    }
}
```

---

## Why It Fails

Object creation may be reordered:

1. Allocate memory
2. Assign reference
3. Initialize object

Another thread may see step 2 before step 3.

---

## ✅ Correct Implementation

```java
class FraudEngine {
    private static volatile FraudEngine instance;

    public static FraudEngine getInstance() {
        if (instance == null) {
            synchronized (FraudEngine.class) {
                if (instance == null) {
                    instance = new FraudEngine();
                }
            }
        }
        return instance;
    }
}
```

---

## Explanation

`volatile`:

* Prevents instruction reordering.
* Guarantees safe publication.
* Ensures visibility across threads.

---

## Common Mistakes

* Forgetting volatile.
* Using unsafe lazy initialization.
* Locking entire method unnecessarily.

---

## Best Practices

* Prefer enum singleton.
* Prefer static initialization.
* Use volatile only when needed.

---

# 3️⃣ Heap, Stack, Metaspace

---

## Concept

JVM memory areas:

* Stack → per thread, stores method calls & local variables
* Heap → shared, stores objects
* Metaspace → stores class metadata

---

## Scenario

Application handles thousands of requests creating many objects.

---

## Goal

Understand memory areas to avoid memory errors.

---

## What Can Go Wrong

* StackOverflowError
* OutOfMemoryError (Heap)
* OutOfMemoryError (Metaspace)

---

## ❌ Incorrect Example (Stack Overflow)

```java
void recursive() {
    recursive();
}
```

---

## Why It Fails

Each method call creates a new stack frame.
Stack memory is limited.

---

## ✅ Correct Implementation

```java
void loop() {
    while (true) {
        // logic
    }
}
```

---

## Explanation

Stack:

* Fixed size
* Per thread

Heap:

* Managed by GC
* Stores all objects

Metaspace:

* Stores class metadata
* Grows dynamically

---

## Common Mistakes

* Deep recursion.
* Large temporary objects.
* Dynamic class loading without control.

---

## Best Practices

* Avoid deep recursion.
* Keep objects lightweight.
* Monitor metaspace growth.
* Tune stack size if needed (-Xss).

---

# 4️⃣ G1 GC Tuning

---

## Concept

G1 (Garbage First) is a region-based garbage collector.
It prioritizes collecting regions with most garbage first.

---

## Scenario

Application allocates many objects per second.

---

## Goal

Maintain low pause time and stable performance.

---

## What Can Go Wrong

* Frequent Full GC
* Long pause times
* Latency spikes

---

## ❌ Incorrect Configuration

```bash
-Xms256m -Xmx4g
```

---

## Why It Fails

Heap resizing causes additional pauses.
Fragmentation increases.

---

## ✅ Correct Configuration

```bash
-XX:+UseG1GC
-Xms4g
-Xmx4g
-XX:MaxGCPauseMillis=200
```

---

## Explanation

* Fixed heap reduces resizing.
* G1 manages regions.
* Pause target helps control latency.

---

## Common Mistakes

* Blindly increasing heap.
* Not checking GC logs.
* Ignoring object allocation rate.

---

## Best Practices

* Keep Xms = Xmx.
* Enable GC logging.
* Monitor pause time.
* Reduce unnecessary object creation.

---

# 5️⃣ Heap Pressure Simulation

---

## Concept

Heap pressure occurs when object allocation rate is high and objects are retained.

---

## Scenario

Application continuously creates objects without releasing them.

---

## Goal

Observe GC behavior under memory stress.

---

## What Can Go Wrong

* Frequent GC
* High CPU usage
* OutOfMemoryError

---

## ❌ Incorrect Example

```java
import java.util.*;

public class HeapPressure {
    static List<byte[]> list = new ArrayList<>();

    public static void main(String[] args) {
        while (true) {
            list.add(new byte[1_000_000]);
        }
    }
}
```

---

## Why It Fails

Objects remain referenced.
GC cannot reclaim memory.

---

## ✅ Correct Implementation

```java
if (list.size() > 100) {
    list.clear();
}
```

---

## Explanation

Objects must become unreachable for GC to reclaim them.

---

## Common Mistakes

* Unbounded collections.
* Long-lived references.
* Static caches.

---

## Best Practices

* Use bounded collections.
* Understand object lifecycle.
* Monitor memory during load testing.

---

# 6️⃣ Memory Leak Creation & Detection

---

## Concept

Memory leak = objects still reachable but no longer needed.

---

## Scenario

Static cache storing data without eviction.

---

## Goal

Detect and remove retained references.

---

## What Can Go Wrong

* Heap keeps growing.
* Frequent Full GC.
* Eventually OutOfMemoryError.

---

## ❌ Incorrect Example

```java
class Leak {
    static List<byte[]> cache = new ArrayList<>();

    public static void main(String[] args) {
        while (true) {
            cache.add(new byte[1024 * 1024]);
        }
    }
}
```

---

## Why It Fails

Static reference prevents GC.

---

## ✅ Correct Implementation

Use bounded cache:

```java
class BoundedCache<K,V> extends LinkedHashMap<K,V> {
    private final int maxSize;

    BoundedCache(int maxSize) {
        super(16, 0.75f, true);
        this.maxSize = maxSize;
    }

    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return size() > maxSize;
    }
}
```

---

## Explanation

Old entries automatically removed.
Memory remains controlled.

---

## Common Mistakes

* Not clearing ThreadLocal.
* Static references.
* Not closing resources.

---

## Best Practices

* Use bounded caches.
* Remove ThreadLocal after use.
* Take heap dump before increasing heap.
* Use profiling tools regularly.

---
