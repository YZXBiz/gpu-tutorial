# Chapter 1: Introduction to GPU Programming

## Table of Contents

1. [Why GPUs Matter Now](#1-why-gpus-matter-now)
   - 1.1. [The Speed Wall (2003)](#11-the-speed-wall-2003)
   - 1.2. [Two Paths Forward](#12-two-paths-forward)
2. [How CPUs and GPUs Think Differently](#2-how-cpus-and-gpus-think-differently)
   - 2.1. [CPU Design: Minimize Waiting](#21-cpu-design-minimize-waiting)
   - 2.2. [GPU Design: Maximize Throughput](#22-gpu-design-maximize-throughput)
   - 2.3. [The Performance Gap](#23-the-performance-gap)
3. [What Needs This Much Speed?](#3-what-needs-this-much-speed)
4. [The Parallelization Reality Check](#4-the-parallelization-reality-check)
   - 4.1. [Amdahl's Law in Plain English](#41-amdahls-law-in-plain-english)
   - 4.2. [Real-World Speedups](#42-real-world-speedups)
5. [Four Hard Problems in Parallel Programming](#5-four-hard-problems-in-parallel-programming)
6. [The Programming Landscape](#6-the-programming-landscape)
   - 6.1. [Why CUDA?](#61-why-cuda)
   - 6.2. [How This Relates to Other Tools](#62-how-this-relates-to-other-tools)
7. [What You'll Learn](#7-what-youll-learn)
8. [How This Book Works](#8-how-this-book-works)

---

## 1. Why GPUs Matter Now

### 1.1. The Speed Wall (2003)

**The Real-World Version:** Imagine your car manufacturer promising you a faster car every year for 20 years. Then suddenly in 2003, they stop. The engine can't run any hotter without melting, so they give you a car with two engines instead.

**In Computing Terms:** Single-core CPUs increased in speed dramatically from the 1980s to 2003, reaching gigahertz speeds. Then physics intervened: heat dissipation and power consumption made it impossible to keep cranking up clock speeds. The industry pivoted to multicore processors—multiple CPUs on one chip.

**Why This Pattern:** You can't make one worker infinitely fast, but you can hire more workers.

> **💡 Key Pattern**
>
> When you hit a physical limit on individual performance, the only path forward is parallel execution.

This shift had a profound effect: **sequential programs stopped getting faster automatically**. The same code that would speed up with each new processor generation suddenly plateaued. Only programs that could divide work across multiple cores would benefit from new hardware.

### 1.2. Two Paths Forward

The industry split into two design philosophies:

**Multicore CPUs (Latency-Oriented):**
- Started with 2 cores, now up to 128 cores (ARM Ampere)
- Each core is sophisticated: out-of-order execution, branch prediction, large caches
- Goal: Make each individual task finish quickly
- Best for: Sequential code, complex logic, varied workloads

**Many-Thread GPUs (Throughput-Oriented):**
- Started with hundreds of threads, now tens of thousands
- Each execution unit is simpler: in-order pipelines, smaller caches
- Goal: Complete massive amounts of work in parallel
- Best for: Data-parallel operations, numerical computation

---

## 2. How CPUs and GPUs Think Differently

### 2.1. CPU Design: Minimize Waiting

A CPU is built like a high-end sports car with every feature to minimize latency:

**Architecture Features:**
- **Large caches:** Frequently accessed data stored on-chip for instant retrieval
- **Branch prediction:** Hardware guesses which way your `if` statement will go before it executes
- **Out-of-order execution:** Rearranges instructions on the fly to avoid waiting
- **Complex control logic:** Sophisticated circuitry to manage all these optimizations

**The Tradeoff:** All this complexity consumes chip area and power that could otherwise be used for more execution units.

**Example (2021):**
- Intel 24-core CPU: 0.33 TFLOPS (double-precision)
- Each core is large and sophisticated

### 2.2. GPU Design: Maximize Throughput

A GPU is built like a fleet of delivery trucks optimized for total cargo moved per day:

**Architecture Features:**
- **Many simple execution units:** More workers, each less sophisticated
- **Small caches:** Just enough to reduce redundant memory trips
- **Long-latency operations:** Acceptable because other threads keep the units busy
- **High memory bandwidth:** Approximately 10× more than contemporary CPUs

**The Tradeoff:** Individual operations take longer, but total work completed per second is much higher.

**Example (2021):**
- NVIDIA A100 GPU: 9.7 TFLOPS (double-precision), 156 TFLOPS (single-precision)
- 30× more throughput than the Intel CPU above

> **💡 Key Pattern**
>
> Reducing latency is expensive (quadratic power cost). Increasing throughput is cheap (linear cost). GPUs choose throughput.

### 2.3. The Performance Gap

```
CPU Approach (Latency-Oriented):
────────────────────────────────
┌────────────────────┐
│  Complex Core 1    │ ← Large cache, branch prediction
├────────────────────┤
│  Complex Core 2    │
├────────────────────┤
│       ...          │
└────────────────────┘
Result: Fast individual threads

GPU Approach (Throughput-Oriented):
────────────────────────────────────
┌──┬──┬──┬──┬──┬──┬──┬──┐
│S │S │S │S │S │S │S │S │ ← Simple execution units
├──┼──┼──┼──┼──┼──┼──┼──┤
│S │S │S │S │S │S │S │S │   (S = Simple Core)
├──┼──┼──┼──┼──┼──┼──┼──┤
│  ... thousands ...   │
└──────────────────────┘
Result: Massive parallel throughput
```

**Why this works:** Graphics applications (the original GPU use case) perform the same operations on millions of pixels. Individual pixel latency doesn't matter—total frames per second does.

---

## 3. What Needs This Much Speed?

Beyond gaming, several application areas demand continually increasing speed:

**Molecular Biology:**
- Traditional microscopes have physical limits
- Computational simulation reveals molecular-level interactions
- Larger systems + longer simulations = better predictions

**Media Processing:**
- Video: Standard definition → HD → 4K → 8K
- Each step requires 4× more computation
- Real-time features: View synthesis, upscaling, focus correction

**Deep Learning (Since 2012):**
- Neural networks require massive labeled datasets and computation
- Training used to take weeks; now takes hours on GPUs
- Enabled: Computer vision, natural language processing, self-driving cars

**Physical Simulation:**
- Gaming: Pre-scripted scenes → dynamic physics
- Engineering: Digital twins for stress testing
- Weather: Higher resolution models = more accurate forecasts

> **💡 Key Pattern**
>
> Applications that simulate or represent the concurrent physical world are naturally parallel and demand increasing throughput.

---

## 4. The Parallelization Reality Check

### 4.1. Amdahl's Law in Plain English

**The Real-World Version:** You're renovating a house. Painting can be parallelized (hire 10 painters), but foundation work cannot (only one crew can work on it). If the foundation takes 50% of the time, hiring 1000 painters won't reduce total project time by more than 50%.

**In Computing Terms:** Speedup is limited by the sequential portion of your program.

**Formula:**
```
Speedup = 1 / (Sequential Fraction + Parallel Fraction / Number of Processors)
```

**Concrete Example:**

Let's say your program takes 100 seconds total:
- 30 seconds: Sequential (file I/O, initialization)
- 70 seconds: Parallelizable computation

**Scenario 1: 100× speedup on parallel portion**
```
New Time = 30 + (70/100) = 30.7 seconds
Total Speedup = 100/30.7 = 3.26×
```

Despite a 100× speedup on the parallel part, total speedup is only 3.26× because 30% remains sequential.

**Scenario 2: 99% is parallelizable**
```
Sequential: 1 second
Parallel: 99 seconds → 0.99 seconds (with 100× speedup)
Total Time: 1.99 seconds
Total Speedup = 100/1.99 = 50.25×
```

**The Lesson:** To achieve 100× speedups, you need 99.9%+ of execution time in parallel sections.

### 4.2. Real-World Speedups

**Typical Results:**
- **Straightforward parallelization:** 10× speedup (often memory-bandwidth limited)
- **With memory optimization:** 50× speedup (using GPU on-chip memory cleverly)
- **After extensive tuning:** 100×+ speedup (requires 99.9%+ parallel coverage)

**Memory Bandwidth Wall:**

Sequential View:
```
CPU → DRAM (Bandwidth: X GB/s)
```

Parallel View (Naive):
```
GPU (100× more cores) → DRAM (Still X GB/s)
↑ All cores competing for same bandwidth
```

**The Fix:** Use on-chip memory (shared memory, caches) to reduce DRAM trips. This is a major focus of GPU optimization.

---

## 5. Four Hard Problems in Parallel Programming

### Problem 1: Algorithm Complexity

**Issue:** Some parallel algorithms do more total work than their sequential counterparts.

**Example:** Parallel algorithms for naturally sequential problems (recurrences) may perform redundant work, potentially making them slower on large datasets.

**Solution:** Primitive patterns like prefix sum (scan) convert sequential formulations into efficient parallel forms. (Covered in Chapter 11)

### Problem 2: Memory Bandwidth

**Issue:** Many applications are memory-bound—limited by data access speed, not computation speed.

**Example:** Simple element-wise operations (adding two arrays) are entirely memory-bound on modern processors.

**Solution:** Tiling, data reuse, exploiting on-chip memories. (Covered in Chapters 5-6)

### Problem 3: Data Characteristics

**Issue:** Real-world data often has uneven distributions, causing load imbalance across threads.

**Example:** Graph algorithms where some nodes have 2 neighbors and others have 10,000.

**Solution:** Dynamic work distribution, data regularization. (Covered in pattern chapters)

### Problem 4: Synchronization Overhead

**Issue:** Threads need to coordinate, which requires synchronization operations where threads wait instead of working.

**Types:**
- **Embarrassingly parallel:** No synchronization needed (e.g., image filters)
- **Coordinated parallel:** Requires barriers or atomic operations (e.g., histogram)

**Solution:** Minimize synchronization through careful algorithm design, privatization. (Covered throughout)

> **💡 Key Pattern**
>
> High-performance parallel programming is about managing resources (memory bandwidth, synchronization) as much as dividing work.

---

## 6. The Programming Landscape

### 6.1. Why CUDA?

**Historical Context:**

**Before 2007 (GPGPU):**
- Had to express computation as "painting pixels"
- Required graphics API knowledge (OpenGL, Direct3D)
- Limited to specific computation types

**2007: CUDA Changes Everything**
- General-purpose programming interface
- Hardware support (not just software layer)
- C/C++ with minimal extensions
- Direct memory management

**Current Status:**
- 1+ billion CUDA-enabled GPUs deployed
- Standard tool for HPC, deep learning, scientific computing

### 6.2. How This Relates to Other Tools

**OpenMP** (Multicore CPUs):
- Compiler directives for parallelization
- Good performance portability across vendors
- Abstracts many details
- Now supports GPU offloading
- **Learning Value:** Understanding CUDA concepts helps you use OpenMP effectively

**MPI** (Cluster Computing):
- Message passing for distributed memory systems
- No shared memory across nodes
- Used in 100,000+ node supercomputers
- **In Practice:** Often combined with CUDA for multi-GPU clusters (Chapter 20)

**OpenCL** (Cross-Platform):
- Standardized alternative to CUDA
- Runs on CPUs, GPUs, FPGAs from multiple vendors
- More API-heavy, less language-extension focused
- **Key Insight:** CUDA and OpenCL concepts are nearly identical; learning one makes learning the other trivial

---

## 7. What You'll Learn

**Goal 1: High-Performance Parallel Programming**

You'll learn computational thinking techniques—ways of formulating problems that are amenable to parallel execution. The focus is practical: working code, measurable performance, concrete examples.

**Goal 2: Correct and Debuggable Code**

Parallel code can have subtle bugs. You'll learn:
- Simple synchronization patterns
- Debugging techniques
- Tools for finding performance bottlenecks
- Data-parallel programming for reliability

**Goal 3: Future Scalability**

Code that runs faster on each new GPU generation by:
- Localizing memory accesses
- Regularizing data access patterns
- Minimizing resource contention

**Not Required:** Deep hardware expertise. **But Required:** Conceptual understanding of how hardware executes your code to reason about performance.

---

## 8. How This Book Works

### Part I: Fundamentals (Chapters 2-6)

**Chapter 2:** CUDA C basics, CPU/GPU cooperation, data transfer
**Chapter 3:** Multidimensional thread organization, handling multidimensional data
**Chapter 4:** GPU architecture—how cores are organized, threads scheduled
**Chapter 5:** Memory architecture—using specialized memories for speed
**Chapter 6:** Performance considerations—patterns of execution and memory access

### Part II: Primitive Patterns (Chapters 7-12)

Each chapter introduces a fundamental parallel pattern:

- **Convolution** (Ch 7): Data locality, constant memory
- **Stencil** (Ch 8): 3D organization, thread granularity
- **Histogram** (Ch 9): Atomic operations, privatization
- **Reduction** (Ch 10): Tree patterns, control divergence
- **Prefix Sum** (Ch 11): Sequential → parallel conversion, work efficiency
- **Merge** (Ch 12): Dynamic data organization

### Part III: Advanced Patterns & Applications (Chapters 13-19)

Putting it together in real applications:

- Sorting, Sparse matrices, Graph traversal
- Deep learning, Medical imaging, Molecular visualization
- Computational thinking principles

### Part IV: Advanced Practices (Chapters 20-22)

- Multi-GPU clusters (MPI + CUDA)
- Dynamic parallelism
- Advanced features: Unified memory, debugging, profiling

### Learning Approach

**Concrete → Abstract:** Learn concepts in CUDA context first, then generalize
**Pattern-Based:** Techniques taught through reusable patterns
**Application-Driven:** No isolated theory—everything in application context

> **💡 Key Pattern**
>
> Master one parallel programming model deeply, then transfer knowledge to others easily.

---

**← Previous:** [Acknowledgments](acknowledgments.md) | **Next: →** [Chapter 2: Heterogeneous Data Parallel Computing](chapter2.md)

---

## Key Takeaways

1. **The 2003 Pivot:** Single-core speed hit a wall; parallelism became mandatory
2. **Two Philosophies:** CPUs minimize latency; GPUs maximize throughput (30× performance gap)
3. **Amdahl's Law:** 100× speedups require 99.9%+ parallel coverage
4. **Four Challenges:** Algorithm complexity, memory bandwidth, data characteristics, synchronization
5. **CUDA's Role:** First general-purpose GPU programming interface with hardware support
6. **This Book:** Teaches practical parallel programming through patterns and real applications
