---
layout: default
title: "M:N Green Thread Runtime with Work-Stealing Scheduler"
---

# M:N Green Thread Runtime with Work-Stealing Scheduler

**Roshni Ramesh and Wasmir Chowdhury** · Carnegie Mellon University

---

## Summary

We are implementing an M:N threading runtime in C that multiplexes M lightweight green threads across N kernel threads using a work-stealing scheduler, targeting GHC multicore machines and a Raspberry Pi 4. Our runtime implements per-kernel-thread run queues, Chase-Lev work-stealing deques for load balancing, and SIGALRM-based preemption. We evaluate performance against pthreads, Go goroutines, and Java virtual threads (Project Loom) across a variety of workloads.

## Background

Modern concurrent programs need to manage thousands of lightweight concurrent tasks efficiently. The standard approach (one kernel thread per task via pthreads) does not scale because kernel thread creation is expensive, kernel context switches require a full privilege level change, and the kernel scheduler has no knowledge of application-level task structure.

M:N threading addresses this by maintaining a small fixed pool of N kernel threads and multiplexing M lightweight green threads on top of them entirely in userspace. The kernel sees only N threads; the runtime manages the M green threads invisibly.

**Green thread context switches are roughly 10x cheaper than kernel thread context switches**, since they only save/restore callee-saved registers and the stack pointer with no kernel involvement.

### Scheduling Design

A naive approach uses a single global queue protected by a lock, but this creates a sequential bottleneck. Our approach gives each kernel thread its own private run queue, implemented as a **Chase-Lev work-stealing deque**:

- The **owner** schedules from the front of its own queue with no contention
- When a queue is empty, the kernel thread **steals** from the back of another thread's queue
- This minimizes contention between owner and thief

This is the same fundamental design used by Go's goroutine scheduler, Java's ForkJoinPool, and the Cilk runtime.

## The Challenge

**Synchronization in work stealing** -- the owner of a queue accesses the front while a thief accesses the back simultaneously. Getting this lock-free correctly without data races requires careful reasoning about memory ordering. A bug here causes silent state corruption that is hard to reproduce and diagnose.

**Preemption interaction with scheduler state** -- SIGALRM fires asynchronously on a kernel thread that may be in the middle of a scheduler operation. The signal handler must safely preempt the current green thread without corrupting the run queue or leaving locks held.

**Blocking syscall problem** -- if a green thread calls a blocking syscall, it blocks its entire kernel thread, starving all other green threads on that kernel thread. Addressing this with `io_uring` requires integrating an async IO completion loop into the scheduler.

**Memory locality** -- when work stealing migrates a green thread from one kernel thread to another, the thread's stack data is no longer in the stealing core's cache. Measuring and characterizing this tradeoff is one of our evaluation goals.

**Bare metal port** -- on the Raspberry Pi 4 we lose all Linux primitives. We must implement SMP boot, ARM64 context switching in assembly, hardware timer programming for preemption, and a minimal memory allocator.

## Goals and Deliverables

### Plan to Achieve (Core)

- Fully working M:N threading runtime in C on GHC machines: green thread creation and context switching via `ucontext`, per-kernel-thread run queues with fine-grained locking, Chase-Lev work-stealing, and SIGALRM-based preemption
- Green-thread-aware mutex and condition variable primitives that yield the kernel thread rather than blocking it
- Comprehensive benchmark suite measuring throughput, scheduling latency, work-stealing rate, and preemption overhead
- Direct performance comparison against pthreads across CPU-bound, short-lived thread, and producer-consumer workloads
- Speedup graphs showing performance scaling from 1 to 8 kernel threads on GHC machines

### Hope to Achieve (Reach Goals)

- Benchmark comparison against Go goroutines and Java virtual threads (Project Loom)
- `io_uring` integration for true async IO without blocking kernel threads
- Bare metal port to Raspberry Pi 4 with ARM64 assembly context switch, ARM generic timer for preemption, and a bump allocator

### If Things Go Slower

- Drop `io_uring` and Pi port; focus on thorough Linux evaluation with deeper analysis of stealing behavior and cache miss profiling with `perf`
- If work stealing proves too buggy, fall back to per-kernel-thread queues without stealing and analyze the load imbalance cost

## Schedule

| Week | Roshni | Wasmir |
|------|--------|--------|
| **Mar 25** -- Foundation | Repo setup, Makefile, green thread struct, stack allocation | `ucontext` API, single green thread context switch, `test_basic` passing |
| **Apr 1** -- Single KThread Scheduler | Per-KThread run queue, enqueue/dequeue, instrumentation | Scheduler loop, cooperative multi-thread execution, yielding |
| **Apr 7** -- Parallelism & Stealing | Extend to N kernel threads, verify correctness | Work-stealing implementation, `try_steal`, imbalanced workload test |
| **Apr 14** -- Milestone | *Both: milestone report, project page update, preliminary throughput graphs* | |
| **Apr 14** -- Preemption & Sync | SIGALRM preemption, signal masking | Green thread mutex/condvar, producer-consumer test |
| **Apr 21** -- Eval & Reach Goals | Full benchmark suite, Go/Loom comparisons | `io_uring` integration or bare metal Pi boot |
| **Apr 28** -- Writeup & Poster | *Both: final report, poster, source code cleanup* | |

## Platform

**GHC lab machines** (8 cores, x86-64, Linux) -- primary development and evaluation platform. C gives us direct control over memory layout, assembly integration, and signal handling.

**Raspberry Pi 4** (4 cores, ARM Cortex-A72) -- bare metal port target. Both team members have prior bare metal ARM experience from 18-349.

## References

- Blumofe & Leiserson 1999, *"Scheduling Multithreaded Computations by Work Stealing"*
- Chase & Lev 2005, *"Dynamic Circular Work-Stealing Deque"*
- Go scheduler design document (Dmitry Vyukov)
- Anderson et al. 1992, *"Scheduler Activations"*
- BCM2711 ARM Peripherals datasheet and ARM Cortex-A72 TRM
- [raspberry-pi-os](https://github.com/s-matyukevich/raspberry-pi-os) bare metal tutorial
