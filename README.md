# Java Low-Latency & Performance Optimization Guide

Welcome to the comprehensive guide on Java Low-Latency and Performance Optimization. This guide is structured sequentially, moving from low-level hardware constraints up to high-performance Java library design patterns.

## Master Table of Contents

### [Module 1: The Hardware Foundation (Mechanical Sympathy)](./Module_1_Hardware_Foundation/Chapter_1_CPU_Architecture.md)
*Before writing high-performance code, you must understand the machine executing it.*
- [Chapter 1: CPU Architecture & Memory Hierarchy](./Module_1_Hardware_Foundation/Chapter_1_CPU_Architecture.md)

### [Module 2: The Java Memory Model (JMM)](./Module_2_Java_Memory_Model/Chapter_2_Concurrency_Foundations.md)
*Understanding how Java abstracts the underlying hardware and the rules of memory visibility.*
- [Chapter 2: Concurrency Foundations & The JMM](./Module_2_Java_Memory_Model/Chapter_2_Concurrency_Foundations.md)
- [Chapter 3: Volatility & Memory Barriers](./Module_2_Java_Memory_Model/Chapter_3_Volatility_and_Memory_Barriers.md)

### [Module 3: Locking Mechanisms & Concurrency Control](./Module_3_Locking_Mechanisms/Chapter_4_Intrinsic_Locking.md)
*Moving from traditional synchronization to high-performance explicit locking primitives.*
- [Chapter 4: Intrinsic Locking (`synchronized`)](./Module_3_Locking_Mechanisms/Chapter_4_Intrinsic_Locking.md)
- [Chapter 5: Explicit Locking (`java.util.concurrent.locks`)](./Module_3_Locking_Mechanisms/Chapter_5_Explicit_Locking.md)

### [Module 4: Lock-Free & Wait-Free Programming](./Module_4_Lock_Free_Programming/Chapter_6_Hardware_Primitives.md)
*Eliminating thread blocking entirely by leveraging hardware-level instructions.*
- [Chapter 6: Hardware Primitives & Atomics](./Module_4_Lock_Free_Programming/Chapter_6_Hardware_Primitives.md)
- [Chapter 7: Advanced Low-Level Memory Access](./Module_4_Lock_Free_Programming/Chapter_7_Advanced_Low_Level_Memory.md)

### [Module 5: Data Structures & Cache Sympathy](./Module_5_Data_Structures_and_Cache/Chapter_8_Cost_of_Objects.md)
*Optimizing memory layouts to prevent CPU cache misses and minimize garbage collection.*
- [Chapter 8: The Cost of Objects in Java](./Module_5_Data_Structures_and_Cache/Chapter_8_Cost_of_Objects.md)
- [Chapter 9: The False Sharing Problem](./Module_5_Data_Structures_and_Cache/Chapter_9_False_Sharing_Problem.md)
- [Chapter 10: High-Performance Collections & Messaging](./Module_5_Data_Structures_and_Cache/Chapter_10_High_Performance_Collections.md)

---
*Created as a guided learning path for low-latency Java engineers.*
