# Chapter 6: Hardware Primitives & Atomics

## Table of Contents
- [1. Compare-And-Swap (CAS)](#1-compare-and-swap-cas)
- [2. The Contention Problem](#2-the-contention-problem)
- [3. The `java.util.concurrent.atomic` Package](#3-the-javautilconcurrentatomic-package)
- [4. The ABA Problem](#4-the-aba-problem)
- [5. High-Contention Counters](#5-high-contention-counters)

---

## 1. Compare-And-Swap (CAS)

### Concept Definition
- Compare-And-Swap (CAS) is the fundamental hardware instruction powering lock-free programming.
- In x86 architecture, it is executed via the `CMPXCHG` instruction.
- It atomically checks if a memory location currently holds an *expected* value; if and only if it does, it updates the location to a *new* value. If the value does not match, the operation fails (and usually retries in a "CAS loop").

### Everyday Analogy
- Barron wants to update the high score on an arcade machine, but other players might be updating it simultaneously.
- **CAS Strategy**: Barron remembers the current high score is 100. He walks to the machine with his new score of 150.
- He tells the machine: "If the high score is *still* exactly 100, update it to 150. If someone else changed it to 120 while I was walking over here, don't do anything, just tell me the new score."
- If he fails, he goes back, calculates a new strategy based on the 120, and tries again.

### Java-Specific Implementation
- Lock-free algorithms in Java completely rely on CAS. It eliminates the need for thread context switching (sleeping) entirely, keeping threads hot on the CPU.
- Under the hood, Java calls down to the CPU hardware via intrinsic methods to execute the CAS instruction.

[Back to Top](#table-of-contents)

---

## 2. The Contention Problem

### Concept Definition
- While CAS is generally faster than a heavyweight lock, it degrades severely under high contention.
- If 10 threads continuously execute a CAS loop trying to update the same single memory address, only 1 thread succeeds per cycle. The other 9 threads fail, instantly retry, and burn CPU cycles endlessly.
- **Cache Line Bouncing**: When multiple CPU cores attempt native CAS operations on the same variable, the MESI cache coherence protocol frantically invalidates and transfers the cache line back and forth between the cores (an "invalidation storm").

### Everyday Analogy
- Olivia organizes a massive scavenger hunt where 50 people must all sign a single, small notepad at the finish line to prove they finished.
- The notepad (cache line) is constantly being snatched out of people's hands. Only one person can sign at a time, while the other 49 people just aggressively grab at the notepad, wasting pure energy and creating chaos.

### Java-Specific Implementation
- A single `AtomicInteger` under massive thread contention will bring a multi-core machine to its knees because the L1/L2 caches spend all their time syncing via the L3 cache instead of actually computing.

> **Important Considerations:**
> - CAS is an optimistic concurrency model. It only performs well when the *probability of collision* is relatively low.

[Back to Top](#table-of-contents)

---

## 3. The `java.util.concurrent.atomic` Package

### Concept Definition
- This package provides lock-free, thread-safe wrappers for single variables, primarily `AtomicInteger`, `AtomicLong`, and `AtomicReference`.
- Internally, they hold a `volatile` value and expose methods like `compareAndSet()` or `incrementAndGet()`, which use `Unsafe` or `VarHandle` to execute the hardware CAS.
- **Footprint Optimization**: An `AtomicInteger` is an object, meaning it carries the 12-16 byte Object Header overhead. If you have an array of 1,000,000 objects and each needs an atomic flag, creating 1,000,000 `AtomicInteger` instances wastes massive amounts of memory.

### Everyday Analogy
- Steve needs to track the inventory for millions of tiny screws.
- Putting each screw in its own heavy, individual lockbox (`AtomicInteger`) takes up too much warehouse space.
- Instead, he keeps the screws in standard bins (`volatile int`), and uses a single, special robotic arm (`AtomicIntegerFieldUpdater`) to safely manipulate the counts inside any bin.

### Java-Specific Implementation
- To save memory, you can declare a primitive `volatile int` field in your class and use an `AtomicIntegerFieldUpdater` to perform CAS operations on it.
- This gives you the atomic safety of `AtomicInteger` without the object allocation overhead.

```java
public class Node {
    volatile int state = 0;
    private static final AtomicIntegerFieldUpdater<Node> STATE_UPDATER =
        AtomicIntegerFieldUpdater.newUpdater(Node.class, "state");

    public boolean tryUpdate() {
        // CAS operation directly on the primitive field
        return STATE_UPDATER.compareAndSet(this, 0, 1);
    }
}
```

[Back to Top](#table-of-contents)

---

## 4. The ABA Problem

### Concept Definition
- A critical flaw in naive CAS algorithms is the ABA problem.
- CAS only checks if the *value* matches the expected value. However, the value could have changed from A to B, and then *back* to A, while a thread was paused.
- The paused thread wakes up, sees the value is A, mistakenly assumes nothing has changed, and proceeds to corrupt the system state (this is especially dangerous in lock-free linked lists or queues where object pointers are being recycled).

### Everyday Analogy
- Barron is guarding a parking spot (Spot A). He leaves for lunch.
- While he is gone, Steve parks his car in Spot A (changes to B).
- Later, Steve completes his errand and leaves. Another driver in a car identical to Barron's pulls into Spot A (changes back to A).
- Barron returns, sees a car identical to his in the spot, and assumes nobody ever used the spot while he was gone. He is completely unaware of the intermediate state changes.

### Java-Specific Implementation
- Java provides mitigation strategies via specialized classes.
- **`AtomicStampedReference`**: Pairs an object reference with an integer "stamp" (version number). Even if the reference goes from A -> B -> A, the stamp goes from 1 -> 2 -> 3. The CAS checks *both* the reference and the stamp, preventing the ABA trap.
- **`AtomicMarkableReference`**: Similar concept, but pairs the reference with a `boolean` mark instead of an integer sequence.

[Back to Top](#table-of-contents)

---

## 5. High-Contention Counters

### Concept Definition
- To solve the cache line bouncing and MESI invalidation storms caused by high CAS contention on a single variable, Java 8 introduced `LongAdder` (built on `Striped64`).
- Instead of forcing all threads to fight over a single `volatile long`, `LongAdder` automatically distributes the contention across an array of independent cells.
- When you ask for the total, it simply sums up all the cells.

### Everyday Analogy
- Olivia is taking donations at a massive charity event.
- **`AtomicLong` approach**: She has one single donation bucket. 1,000 people are trying to put money in at the exact same time, creating a massive bottleneck.
- **`LongAdder` approach**: She sets up 16 different donation buckets spaced far apart. People put money in whichever bucket has the shortest line. At the end of the night, Olivia just counts all 16 buckets and adds them together.

### Java-Specific Implementation
- The internal array of cells in `Striped64` is carefully padded using `@Contended` to completely eliminate false sharing between the individual cells.
- Threads hash their thread ID to pick a specific cell. If they detect contention on that cell, they re-hash and pick a different cell.

> **Best Practices:**
> - ALWAYS use `LongAdder` or `DoubleAdder` instead of `AtomicLong` for simple metrics gathering (like hit counters or throughput tracking) in highly concurrent, low-latency applications. Only use `AtomicLong` if you strictly require atomic `compareAndSet` behavior, which `LongAdder` does not support.

[Back to Top](#table-of-contents)

---

[Back to Master Table of Contents](../README.md)
