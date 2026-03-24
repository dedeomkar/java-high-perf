# Chapter 5: Explicit Locking (`java.util.concurrent.locks`)

## Table of Contents
- [1. AbstractQueuedSynchronizer (AQS)](#1-abstractqueuedsynchronizer-aqs)
- [2. `ReentrantLock`](#2-reentrantlock)
- [3. `ReentrantReadWriteLock`](#3-reentrantreadwritelock)
- [4. `StampedLock`](#4-stampedlock)

---

## 1. AbstractQueuedSynchronizer (AQS)

### Concept Definition
- The `AbstractQueuedSynchronizer` (AQS) is the hidden engine powering almost all explicit locks in Java's `java.util.concurrent` package.
- It is a framework that relies on an atomic `int` to represent state, and a robust FIFO (First-In, First-Out) wait queue (implemented as a doubly-linked list of nodes) to manage blocked threads.
- Instead of using OS-level monitors immediately, AQS extensively uses CPU-level CAS (Compare-And-Swap) operations and aggressive spin-waiting before eventually parking threads.

### Everyday Analogy
- Barron operates a deli. Instead of people violently fighting over the counter (unregulated locks), he installs a "Take a Number" machine (AQS).
- The machine has an internal counter showing the current serving number (the atomic `int` state).
- Customers form a neat line holding their tickets (the FIFO queue of thread nodes).
- Customers periodically glance up to see if their number is called (spinning via CAS) before finally sitting down and falling asleep (thread parking).

### Java-Specific Implementation
- You rarely use AQS directly; instead, you use classes built on top of it, such as `ReentrantLock`, `Semaphore`, and `CountDownLatch`.
- Under heavy contention, AQS locks are significantly faster and more predictable than `synchronized` intrinsic locks because they manage thread parking entirely in Java-space before calling the OS.

[Back to Top](#table-of-contents)

---

## 2. `ReentrantLock`

### Concept Definition
- `ReentrantLock` is the primary alternative to the `synchronized` keyword. It provides the same basic mutual exclusion but with advanced options.
- **Fair Mode**: Threads acquire the lock in the exact order they requested it (strict FIFO). Eliminates starvation but has massive overhead because it forces threads to context-switch constantly.
- **Unfair Mode (Default)**: If a new thread requests the lock at the exact moment the current owner releases it, the new thread can "barge" in and steal it, bypassing the entire waiting queue. This dramatically increases throughput by keeping active threads on the CPU.

### Everyday Analogy
- Olivia manages a single-stall restroom.
- **Fair Mode**: A strict line. Even if the person at the front of the line is looking at their phone and slow to react when the door opens, everyone else must wait for them.
- **Unfair Mode**: When the door opens, if someone happens to be walking exactly past the door, they can just jump in, bypassing the line. It's unfair to the people waiting, but the restroom is utilized continuously with zero downtime.

### Java-Specific Implementation
```java
// Default is Unfair, which provides maximum throughput
Lock lock = new ReentrantLock(); 

lock.lock();
try {
    // Critical Section
} finally {
    // ALWAYS release explicit locks in a finally block
    lock.unlock(); 
}
```

> **Best Practices:**
> - Unless strict chronological ordering is an unbreakable business requirement, always use Unfair locks for low-latency systems.

[Back to Top](#table-of-contents)

---

## 3. `ReentrantReadWriteLock`

### Concept Definition
- A `ReentrantReadWriteLock` allows multiple threads to read a shared resource simultaneously, but requires exclusive access for writing.
- It cleverly packs both the "read count" and the "write state" into the single 32-bit `int` state of its underlying AQS (e.g., higher 16 bits for readers, lower 16 bits for writers).
- **Lock Downgrading**: A thread holding the write lock can acquire the read lock before releasing the write lock, effectively downgrading itself to a reader without allowing other writers to sneak in.

### Everyday Analogy
- Steve manages a bulletin board.
- Multiple people can stand in front of the board and read it at the same time (Shared Read Lock).
- If Steve wants to replace the poster, he must ask everyone to leave. While he is replacing the poster, no one else is allowed to look or touch (Exclusive Write Lock).

### Java-Specific Implementation
- `ReadWriteLock` seems perfect for read-heavy systems, but it suffers from a fatal flaw: the cache coherence traffic needed to increment and decrement the AQS read counter across multiple CPU cores creates a severe bottleneck.

```java
ReadWriteLock rwLock = new ReentrantReadWriteLock();
rwLock.readLock().lock();
try {
    // Read state
} finally {
    rwLock.readLock().unlock();
}
```

> **Important Considerations:**
> - While conceptually elegant, `ReentrantReadWriteLock` scales poorly on modern multi-core machines due to cache-line contention on its internal counters.

[Back to Top](#table-of-contents)

---

## 4. `StampedLock`

### Concept Definition
- Introduced in Java 8, `StampedLock` is designed specifically to replace `ReentrantReadWriteLock` for extremely high-performance scenarios.
- It introduces the concept of an **Optimistic Read**.
- In an optimistic read, the thread reads the data without acquiring a lock or updating any shared counters (zero cache coherence traffic!). It receives a "stamp".
- After reading the data, the thread validates the stamp. If a writer interfered while the thread was reading, validation fails, and the thread falls back to a standard pessimistic read lock.

### Everyday Analogy
- Barron wants to read a clock on the wall.
- **Pessimistic Read (`ReadWriteLock`)**: He walks over, grabs the clock physically so no one can change the time, reads it, and puts it back.
- **Optimistic Read (`StampedLock`)**: He just glances at the clock from across the room. He notes the time, blinks, and looks again just to ensure nobody tampered with the clock hands while he was looking. If it hasn't changed, he proceeds instantly.

### Java-Specific Implementation
- `StampedLock` is complex but provides near-perfect scalability for read-heavy workloads because reader threads don't cause write traffic on the lock's internal state.

```java
StampedLock sl = new StampedLock();
double x, y;

// 1. Optimistic Read - No locking!
long stamp = sl.tryOptimisticRead();

// Copy to local variables
double currentX = x;
double currentY = y;

// 2. Validate the stamp
if (!sl.validate(stamp)) {
    // 3. Fallback to pessimistic lock if modified
    stamp = sl.readLock();
    try {
        currentX = x;
        currentY = y;
    } finally {
        sl.unlockRead(stamp);
    }
}
```

> **Best Practices:**
> - Use `StampedLock` optimism for data structures that are updated rarely but read furiously across dozens of cores.

[Back to Top](#table-of-contents)

---

[Back to Master Table of Contents](../README.md)
