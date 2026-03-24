# Chapter 2: Concurrency Foundations & The JMM

## Table of Contents
- [1. Atomicity vs. Visibility](#1-atomicity-vs-visibility)
- [2. Instruction Reordering](#2-instruction-reordering)
- [3. The Happens-Before Rule](#3-the-happens-before-rule)

---

## 1. Atomicity vs. Visibility

### Concept Definition
- **Atomicity** ensures that a sequence of operations is executed as a single, indivisible unit. If an operation is atomic, it cannot be interrupted or seen in a partially completed state by other threads.
- **Visibility** ensures that when one thread modifies a shared variable, the updated value is immediately visible to other threads, instead of them seeing stale cached values.
- The Java Memory Model (JMM) provides the rules dictating how and when these two guarantees are enforced.

### Everyday Analogy
- Barron and Olivia are writing a joint daily journal.
- **Atomicity**: If Barron is writing a sentence, Olivia cannot grab the pen from him halfway through the sentence. The entire sentence (the atomic operation) must be written before Olivia can write hers.
- **Visibility**: If Barron writes a sentence while Olivia is in another room, Olivia won't know the sentence exists until Barron explicitly shows her the journal. He needs to "publish" his changes so they become *visible* to her.

### Java-Specific Implementation
- In Java, reading and writing reference variables and most primitive variables (except un-`volatile` `long` and `double`) are atomic.
- However, complex operations like `count++` are **not** atomic. They consist of three separate operations: read, modify, and write.
- To guarantee atomicity for compound operations, you must use synchronization (e.g., `synchronized` blocks) or atomic variables (e.g., `AtomicInteger`).
- To guarantee visibility, you must use `volatile`, `synchronized`, or other JMM-defined memory barriers.

> **Important Considerations:**
> - Passing the atomicity check does not automatically guarantee visibility. 
> - A thread can atomically update a variable, but if the variable is not `volatile`, other threads might continue reading the old value from their L1 caches.

[Back to Top](#table-of-contents)

---

## 2. Instruction Reordering

### Concept Definition
- In modern computing, your code is rarely executed in the exact order you wrote it.
- **The Compiler**, **The JIT (Just-In-Time Compiler)**, and **The CPU** are all permitted to wildly reorder instructions to maximize performance, provided the reordering does not change the result *in a single-threaded context*.
- In a multi-threaded context, this reordering becomes extremely dangerous if you rely on the sequence of operations for correctness without proper synchronization.

### Everyday Analogy
- Steve receives a recipe from Barron that says:
  1. Preheat the oven.
  2. Chop the onions.
  3. Slice the tomatoes.
- Steve realizes chopping onions and slicing tomatoes don't depend on the oven being preheated. He preheats the oven *while* slicing the tomatoes (reordering).
- The final meal is identical (single-threaded correctness).
- However, if Olivia arrives expecting the onions to be chopped exactly right after the oven turns on, she might get confused.

### Java-Specific Implementation
- Consider this common (but dangerous) pattern:
  ```java
  boolean ready = false;
  int data = 0;
  
  // Thread 1
  data = 42;
  ready = true;
  
  // Thread 2
  if (ready) {
      System.out.println(data);
  }
  ```
- The JIT or CPU might decide to execute `ready = true;` *before* `data = 42;`. 
- If Thread 2 executes between these reordered instructions, it will print `0` instead of `42`.

> **Best Practices:**
> - Never assume chronological code execution in a concurrent application unless you have explicitly established formal memory barriers.

[Back to Top](#table-of-contents)

---

## 3. The Happens-Before Rule

### Concept Definition
- The JMM uses a concept called the **Happens-Before** relationship to reason about visibility and ordering guarantees.
- If Action A *happens-before* Action B, then the results of Action A are guaranteed to be visible to Action B, and Action A must be ordered before Action B.
- If two operations do not have a happens-before relationship, the JVM is free to aggressively reorder them.

### Everyday Analogy
- Barron ships a very important package to Olivia via a courier service.
- The courier's receipt acts as the *happens-before* guarantee.
- Because Barron physically handed the package to the courier (Action A) before the courier handed it to Olivia (Action B), Olivia is guaranteed to receive Barron's contents.
- Without the courier (the JMM guarantee), Barron and Olivia are just shouting across a canyon, hoping the message arrives.

### Java-Specific Implementation
- The JMM explicitly defines several undeniable happens-before rules:
  1. **Program Order Rule**: Each action in a thread happens-before every action in that thread that comes later in the program order.
  2. **Monitor Lock Rule**: An unlock on a monitor lock (e.g., exiting a `synchronized` block) happens-before a subsequent lock on the same monitor.
  3. **Volatile Variable Rule**: A write to a `volatile` field happens-before every subsequent read of that same field.
  4. **Thread Start Rule**: A call to `Thread.start()` happens-before any action in the started thread.
  5. **Thread Join Rule**: All actions in a thread happen-before any other thread successfully returns from a `Thread.join()` on that thread.
  6. **Transitivity**: If A happens-before B, and B happens-before C, then A happens-before C.

```java
// Thread Start Rule Example
int x = 10;
Thread t = new Thread(() -> {
    // Guaranteed to see x = 10 because the write occurred before Thread.start()
    System.out.println(x); 
});
t.start(); 
```

> **Important Considerations:**
> - To build reliable, low-latency concurrent systems without heavily relying on slow locks, you must master these rules, particularly the Volatile Variable Rule.

[Back to Top](#table-of-contents)

---

[Back to Master Table of Contents](../README.md)
