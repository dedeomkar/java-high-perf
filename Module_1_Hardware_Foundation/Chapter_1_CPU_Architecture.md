# Chapter 1: CPU Architecture & Memory Hierarchy

## Table of Contents
- [1. Latency Numbers](#1-latency-numbers)
- [2. The Cache Hierarchy](#2-the-cache-hierarchy)
- [3. The Cache Line](#3-the-cache-line)
- [4. Prefetching](#4-prefetching)

---

## 1. Latency Numbers

### Concept Definition
- A CPU can perform operations much faster than it can retrieve data from main memory.
- There are distinct layers of memory between the CPU core and the system's main memory (RAM).
- The time it takes to fetch data increases dramatically the further away the data is from the CPU core.
- Measuring performance in nanoseconds (ns) is essential for low-latency engineering.

### Everyday Analogy
- Barron is cooking a complex dish and needs ingredients.
- **CPU Registers (1ns)**: Ingredients already on his cutting board. Instant access.
- **L1 Cache (1-2ns)**: Ingredients in a small bowl right next to the cutting board.
- **L2 Cache (3-5ns)**: Ingredients in the cupboard above his workstation.
- **L3 Cache (10-20ns)**: Ingredients in the refrigerator across the kitchen.
- **Main Memory (100ns)**: Ingredients at the grocery store down the street. It is a massive delay compared to grabbing from the cutting board.

### Java-Specific Implementation
- Java hides the hardware layer, but memory latency profoundly impacts performance.
- An object reference stored in a local variable might be cached in registers or L1.
- Following a pointer to an object on the Java heap usually means a trip to main memory (a cache miss).

> **Best Practices:**
> - Avoid "pointer chasing" (e.g., traversing `LinkedLists` or deep object graphs) in hot loops.
> - Keep data flat and contiguously allocated to avoid paying the main memory latency tax.

[Back to Top](#table-of-contents)

---

## 2. The Cache Hierarchy

### Concept Definition
- Modern processors use a multi-tiered caching system to masquerade the slowness of main memory.
- **L1 Cache**: Smallest and fastest. Split into L1d (data) and L1i (instructions). Located strictly on the core.
- **L2 Cache**: Larger and slightly slower than L1. Also located per core.
- **L3 Cache**: Largest and slowest cache. Generally shared across all cores on the socket.

### Everyday Analogy
- Olivia organizes her kitchen workspace efficiently to avoid running to the store.
- She keeps small, frequently used spices (L1) at arm's length.
- She keeps slightly larger, less frequently used baking supplies (L2) in her pantry.
- The household shares a large chest freezer (L3) in the garage.
- If Olivia needs something and it's not in the kitchen (L1/L2), she checks the shared freezer (L3) before driving to the store (Main Memory).

### Java-Specific Implementation
- When multiple threads (running on different cores) access the same Java object, the object is placed in their respective L1/L2 caches.
- If one thread modifies the object, the cache line must be flushed to L3/Main Memory to ensure visibility to other cores (cache coherence).

> **Important Considerations:**
> - Sharing data between threads causes cache invalidation.
> - Maximize thread-local computation so that data remains safely in the fast, per-core L1 and L2 caches.

[Back to Top](#table-of-contents)

---

## 3. The Cache Line

### Concept Definition
- CPUs rarely fetch single bytes from memory. They fetch chunks called "Cache Lines".
- Standard x86 processors use a cache line size of **64 bytes**.
- When you request a single `int` (4 bytes), the CPU grabs that `int` plus the adjoining 60 bytes.
- This is designed to optimize sequential access and spatial locality.

### Everyday Analogy
- Steve goes to the grocery store to buy a single can of soda.
- Instead of selling him one can, the store only sells soda in 12-packs (a cache line).
- He is forced to bring the entire 12-pack home.
- If he plans to drink more soda later, this is highly efficient. He already has it.
- If his roommate only drinks water and throws the rest of the soda away, the extra weight was wasted effort.

### Java-Specific Implementation
- A Java array of primitives (like `int[]` or `long[]`) is stored contiguously in memory.
- Iterating through an `int[]` is incredibly fast because one 64-byte cache line fetch brings in 16 `int`s at once.
- An array of objects (like `String[]`) stores *references*, not the objects themselves. Scanning it results in cache misses as the CPU jumps around the heap.

> **Best Practices:**
> - Use primitive-backed data structures (like `int[]` or specialized high-performance libraries) for hot paths.
> - Understand that updating independent variables located on the same 64-byte cache line can cause "False Sharing" (covered in a later module).

[Back to Top](#table-of-contents)

---

## 4. Prefetching

### Concept Definition
- Modern CPUs possess dedicated hardware prefetchers.
- The prefetcher monitors memory access patterns.
- If it detects a predictable pattern (e.g., accessing elements sequentially: 0, 1, 2, 3), it will automatically fetch the next elements (4, 5, 6...) into the L1 cache before the program even asks for them.

### Everyday Analogy
- Barron is building a brick wall.
- He lays bricks one by one, moving left to right.
- Olivia acts as his helper (the prefetcher). 
- Seeing his predictable pattern, Olivia starts placing the next brick right next to Barron's hand before he even reaches for it.
- Barron never has to wait, maintaining maximum velocity. If Barron suddenly jumps to the other side of the yard, Olivia gets confused and drops the bricks (a prefetcher miss).

### Java-Specific Implementation
- To trigger the hardware prefetcher, your data access patterns must be linear and predictable.
- Looping consecutively over flat arrays (e.g., `long[]`) is the best way to leverage hardware prefetching in Java.
- Unpredictable branching (excessive `if/else` statements) and random pointer chasing disrupt the hardware prefetcher, forcing the CPU to stall.

> **Important Considerations:**
> - Write mechanical sympathetic code: arrange data linearly in memory to help the CPU predict your next move.
> - Predictability is often more important than theoretical algorithmic complexity in low-latency systems.

[Back to Top](#table-of-contents)

---

[Back to Master Table of Contents](../README.md)
