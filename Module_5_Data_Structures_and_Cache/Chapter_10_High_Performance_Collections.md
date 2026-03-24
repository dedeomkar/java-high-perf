# Chapter 10: High-Performance Collections & Messaging

## Table of Contents
- [1. Deconstructing `HashMap`](#1-deconstructing-hashmap)
- [2. Open Addressing vs. Separate Chaining](#2-open-addressing-vs-separate-chaining)
- [3. Primitive-Backed Collections](#3-primitive-backed-collections)
- [4. Zero-Allocation IPC (Inter-Process Communication)](#4-zero-allocation-ipc-inter-process-communication)

---

## 1. Deconstructing `HashMap`

### Concept Definition
- `java.util.HashMap` is an incredibly elegant and versatile data structure, but it is fundamentally hostile to CPU caches.
- It uses an array of "buckets." Due to Generics, keys and values must be Objects (forcing primitive boxing).
- For every key-value pair inserted, a `Map.Entry` (Node) object must be allocated on the heap.
- When collisions occur, it chains these nodes together in a Linked List (or a Red-Black tree in JDK 8+).

### Everyday Analogy
- Barron organizes a massive filing system.
- Instead of keeping the files tightly packed in folders (`int[]`), he puts every single piece of paper into its own heavy plastic frame (`Map.Entry`).
- Then, he attaches strings between the frames to link related topics (Separate Chaining).
- Finding related topics requires following strings leading all over the massive warehouse.

### Java-Specific Implementation
- The massive allocation rate of `Map.Entry` structures triggers garbage collection.
- The linked-list structure guarantees pointer chasing and cache misses on get operations.
- `HashMap<Integer, Integer>` is perhaps the most heavily used, yet least optimal structure for low-latency Java.

```java
// Standard HashMap forces boxing and object allocation
Map<Integer, Integer> map = new HashMap<>();
for (int i = 0; i < 1_000_000; i++) {
    // Allocates: 2 Integer objects + 1 Map.Entry Node per iteration!
    map.put(i, i * 2); 
}
```

[Back to Top](#table-of-contents)

---

## 2. Open Addressing vs. Separate Chaining

### Concept Definition
- Low-latency hash maps abandon Java's standard "Separate Chaining" collision strategy in favor of "Open Addressing" (specifically, Linear Probing).
- **Separate Chaining (Java HashMap)**: If two items hash to bucket index 5, attach them together in a linked list starting at index 5.
- **Open Addressing / Linear Probing**: If two items hash to bucket index 5, put the first one in index 5. For the second item, simply walk down the contiguous array (index 6, 7, 8) until you find an empty slot, and place it there.

### Everyday Analogy
- Olivia assigns parking spots based on the last digit of a driver's license.
- **Separate Chaining**: Two drivers show up needing Spot 5. Olivia puts the first car in Spot 5, and forces the second car to park on the roof via an elevator mechanism.
- **Open Addressing**: Spot 5 is taken, so the second driver simply parks in Spot 6 right next to it. 

### Java-Specific Implementation
- Linear probing means all data is stored directly in monolithic arrays.
- During a lookup, the CPU cache line pulls in the target bucket *and* the adjacent buckets simultaneously. If a collision occurred, scanning linearly down the array triggers zero cache misses.

```java
// Conceptual Open Addressing Lookup
int[] keys = ...;
int hash = ThreadLocalRandom.current().nextInt();
int pos = hash & (keys.length - 1);

// Scan contiguous array linearly (extremely cache-friendly)
while (keys[pos] != 0 && keys[pos] != targetKey) {
    pos = (pos + 1) & (keys.length - 1); // Wrap around
}
```

[Back to Top](#table-of-contents)

---

## 3. Primitive-Backed Collections

### Concept Definition
- To achieve real speed and minimal GC overhead, low-latency engineers use domain-specific collections that eliminate object boxing and map entries entirely.
- Instead of allocating instances, these libraries usually allocate two large, contiguous primitive arrays under the hood (e.g., a `long[]` for keys and a `long[]` for values) and rely on open addressing.

### Everyday Analogy
- Barron needs a toolkit for a highly specialized job requiring only metric wrenches.
- **`java.util.Map`**: He is forced to carry a massive, heavy backpack containing every tool imaginable, most of which he doesn't need (Object overhead).
- **Primitive-Backed Map**: He brings a single, lightweight toolbelt with exactly the metric wrenches he needs. No extra weight, maximum efficiency.

### Java-Specific Implementation
- **Eclipse Collections**: Provides `IntObjectHashMap`, `LongLongHashMap`, etc. Heavily optimized for memory footprint.
- **FastUtil**: Provides type-specific maps, sets, and queues (`Int2ObjectOpenHashMap`). Very fast, utilized extensively by data-heavy frameworks like Apache Spark.

> **Best Practices:**
> - When keys and values naturally map to primitives, substituting `java.util.Map` for an `Int2IntOpenHashMap` from FastUtil often yields a 5x memory reduction and a complete cessation of minor GC pauses.

```java
// Using FastUtil's primitive map
// Zero boxing, zero Map.Entry allocations. Backed by contiguous int[] and long[]
Int2IntOpenHashMap fastMap = new Int2IntOpenHashMap();
for (int i = 0; i < 1_000_000; i++) {
    fastMap.put(i, i * 2); // Put raw primitives directly
}

// Fast iteration without Iterator object allocation
for (Int2IntMap.Entry entry : fastMap.int2IntEntrySet()) {
    int key = entry.getIntKey();
    int val = entry.getIntValue();
}
```

[Back to Top](#table-of-contents)

---

## 4. Zero-Allocation IPC (Inter-Process Communication)

### Concept Definition
- For threads to communicate (or for entirely separate JVM processes on the same machine to communicate) moving data through traditional network sockets or locks is far too slow for low-latency systems.
- The gold standard is moving data through **Shared Memory Ring Buffers**.
- A Ring Buffer is a fixed-size, circular array. It requires zero memory allocation once initialized.

### Everyday Analogy
- Olivia (Producer) and Steve (Consumer) are working in a busy restaurant.
- **Sockets/Queues**: Olivia creates a burger, walks over to Steve, waits for him to be free, and hands it to him (blocking and high overhead).
- **Ring Buffer (Disruptor)**: They place a spinning "Lazy Susan" (circular tray) between their stations. Olivia constantly drops burgers onto empty spots on the tray and spins it. Steve constantly grabs burgers as they spin past. They never talk, wait, or bump into each other.

### Java-Specific Implementation
- **Agrona**: A massive lock-free concurrent library used heavily in high-frequency trading. It provides Zero-Copy broadcast buffers directly over memory-mapped files via `VarHandle`/`Unsafe`. It never allocates objects during normal operation.
- **The LMAX Disruptor**: A seminal messaging architecture that eliminates standard queue bottlenecks.
  - It abandons Producer/Consumer Queues in favor of a monolithic Ring Buffer.
  - Uses Sequence Barriers instead of locks to coordinate processing.
  - Aligns the buffer to avoid False Sharing, allowing 10s of millions of messages to be dispatched per second with single-digit microsecond latencies.

> **Important Considerations:**
> - Mechanical Sympathy reaches its ultimate form in the Disruptor pattern: pre-allocating contiguous arrays, padding sequence counters to avoid cache invalidations, and using memory fences instead of locks.

```java
// Conceptual LMAX Disruptor Ring Buffer pre-allocation
// Zero objects are created during the actual message processing phase.
Disruptor<TradeEvent> disruptor = new Disruptor<>(
    TradeEvent::new,              // Factory creates empty objects up front
    1024 * 1024,                  // Ring Buffer Size (Power of 2)
    Executors.defaultThreadFactory(),
    ProducerType.SINGLE,          // Single writer optimization
    new BusySpinWaitStrategy()    // Ultra-low latency, burns CPU
);
```

[Back to Top](#table-of-contents)

---

[Back to Master Table of Contents](../README.md)
