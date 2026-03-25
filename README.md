M:N Green Thread Runtime with Work-Stealing Scheduler 
Roshni Ramesh and Wasmir Chowdhury
URL
https://roshnir.github.io/mn-runtime (set up GitHub Pages before submission)
SUMMARY
We are implementing an M:N threading runtime in C that multiplexes M lightweight green threads across N kernel threads using a work-stealing scheduler, targeting GHC multicore machines and a Raspberry Pi 4. Our runtime implements per-kernel-thread run queues, Chase-Lev work-stealing deques for load balancing, and SIGALRM-based preemption, and we will evaluate its performance against pthreads, Go goroutines, and Java virtual threads (Project Loom) across a variety of workloads.
BACKGROUND
Modern concurrent programs need to manage thousands of lightweight concurrent tasks efficiently. The standard approach, one kernel thread per task via pthreads, does not scale because kernel thread creation is expensive, kernel context switches require a full privilege level change, and the kernel scheduler has no knowledge of application-level task structure. M:N threading addresses this by maintaining a small fixed pool of N kernel threads and multiplexing M lightweight green threads on top of them entirely in userspace. The kernel sees only N threads; the runtime manages the M green threads invisibly.
The core data structure is the green thread, a struct containing a saved register context, an allocated stack, and a state field (runnable, running, blocked, or done). Switching between green threads requires saving and restoring only the callee-saved registers and stack pointer, with no kernel involvement. This makes green thread context switches roughly 10x cheaper than kernel thread context switches.
The scheduling problem is how to distribute M green threads across N kernel threads efficiently. A naive approach uses a single global queue protected by a lock, but this creates a sequential bottleneck, all N kernel threads contend on one lock for every scheduling decision. Our approach gives each kernel thread its own private run queue. Each kernel thread schedules from the front of its own queue with no contention. When a kernel thread's queue is empty it steals work from the back of another kernel thread's queue, the Chase-Lev work-stealing deque design that minimizes contention between owner and thief. This is the same fundamental design used by Go's goroutine scheduler, Java's ForkJoinPool, and the Cilk runtime.
The aspects of this problem that benefit from parallelism are exactly the scheduling and load balancing decisions, N kernel threads can make scheduling decisions simultaneously without coordination as long as their queues are independent. The interesting parallel systems questions are how much contention the work-stealing protocol introduces, how cache locality is affected when green threads migrate between kernel threads, and how preemption overhead scales with thread count.
THE CHALLENGE
The core challenge is correctness and performance of the work-stealing implementation under parallel execution. Several specific difficulties:
Synchronization in work stealing, the owner of a queue accesses the front while a thief accesses the back simultaneously. Getting this lock-free or fine-grained-locked correctly without data races requires careful reasoning about memory ordering. A bug here causes silent state corruption that is hard to reproduce and diagnose.
Preemption interaction with scheduler state, SIGALRM fires asynchronously on a kernel thread that may be in the middle of a scheduler operation. The signal handler must safely preempt the current green thread without corrupting the run queue or leaving locks held. Getting the signal masking right around critical sections is subtle.
Blocking syscall problem, if a green thread calls a blocking syscall it blocks its entire kernel thread, starving all other green threads on that kernel thread. Addressing this correctly with io_uring requires integrating an async IO completion loop into the scheduler, which adds significant complexity to the core scheduling loop.
Memory locality, when work stealing migrates a green thread from one kernel thread to another the thread's stack data is no longer in the stealing core's cache. For workloads with significant stack state this migration penalty can offset the load balancing benefit. Measuring and characterizing this tradeoff is one of our evaluation goals.
Bare metal port, on the Raspberry Pi 4 we lose all Linux primitives. We must implement SMP boot, ARM64 context switching in assembly, hardware timer programming for preemption, and a minimal memory allocator. The debugging environment is significantly more constrained, OpenOCD over JTAG instead of gdb and tsan.
The workload characteristics vary significantly, CPU-bound workloads have no blocking and benefit directly from work stealing, IO-bound workloads stress the blocking syscall problem, and mixed workloads test the interaction between preemption and stealing. We expect different configurations of N kernel threads to be optimal for different workload types and measuring this tradeoff is central to our evaluation.
RESOURCES
We are starting from scratch, no existing codebase. Our primary references are:
Blumofe & Leiserson 1999, "Scheduling Multithreaded Computations by Work Stealing", the foundational work stealing algorithm
Chase & Lev 2005, "Dynamic Circular Work-Stealing Deque", the specific data structure for our per-kernel-thread queues
Go scheduler design document by Dmitry Vyukov, canonical M:N design reference
Anderson et al. 1992, "Scheduler Activations", original M:N threading paper
CS 423 MP3 specification (UIUC), closely related assignment, useful for ucontext API guidance
BCM2711 ARM Peripherals datasheet and ARM Cortex-A72 TRM, for bare metal Pi port
s-matyukevich/raspberry-pi-os on GitHub, bare metal Pi tutorial
Machines:
GHC lab machines (8 cores, x86-64, Linux), primary development and evaluation platform
Raspberry Pi 4 (4 cores, ARM Cortex-A72), bare metal port target, we own this hardware
We do not need any special machine access beyond GHC and our own Pi.
GOALS AND DELIVERABLES
Plan to achieve (core deliverables)
Fully working M:N threading runtime in C on GHC machines with green thread creation and context switching via ucontext, per-kernel-thread run queues with fine-grained locking, work-stealing between kernel threads using Chase-Lev deque design, and SIGALRM-based preemption
Green-thread-aware mutex and condition variable primitives that yield the kernel thread rather than blocking it
Comprehensive benchmark suite measuring throughput (green threads completed per second), scheduling latency (time from runnable to running), work-stealing rate and effectiveness, and preemption overhead
Direct performance comparison against pthreads as baseline across CPU-bound, short-lived thread, and producer-consumer workloads
Speedup graphs showing performance scaling as N kernel threads increases from 1 to 8 on GHC machines
We believe we can achieve throughput competitive with pthreads for CPU-bound workloads with many short-lived threads, because our context switch overhead is significantly lower than a kernel thread context switch
Hope to achieve (reach goals)
Benchmark comparison against Go goroutines and Java virtual threads (Project Loom) on identical workloads, with analysis of where and why each runtime wins or loses
io_uring integration so green threads doing IO yield back to the scheduler instead of blocking their kernel thread, enabling true async IO without blocking kernel threads
Bare metal port to Raspberry Pi 4, replace Linux primitives with ARM64 assembly context switch, ARM generic timer for preemption, and a simple bump allocator. Run the same benchmarks on bare metal and compare against the Linux version to isolate OS overhead. We believe this is achievable given both team members have bare metal ARM experience from 18-349
If things go slower than expected
Drop io_uring and Pi port, focus on making the Linux evaluation extremely thorough, more workload types, deeper analysis of stealing behavior, cache miss profiling with perf
If work stealing proves too buggy to stabilize in time, fall back to per-kernel-thread queues without stealing and do a rigorous analysis of the load imbalance cost
Poster session demo
We plan to show a live demo where we spawn a large number of green threads doing visible work, and show real-time graphs of per-kernel-thread queue sizes to visualize work stealing happening. We will also show side-by-side throughput numbers comparing our runtime against pthreads and Go on the same workload.
PLATFORM CHOICE
C on Linux GHC machines is the right choice for several reasons. C gives us direct control over memory layout, assembly integration for context switching, and signal handling, all of which are necessary for a correct and efficient green thread runtime. A higher level language would hide the details we need to control. Linux on GHC provides pthreads for kernel threads, POSIX signals for preemption, and perf for profiling, all of which are exactly what this project requires. The 8-core GHC machines give us enough parallelism to make work stealing interesting and measurable.
The Raspberry Pi 4 is the right bare metal target because we own it, it has 4 ARM Cortex-A72 cores which is enough to demonstrate SMP work stealing, the BCM2711 is well documented, and both team members have prior experience with bare metal ARM programming from 18-349 which significantly reduces the bring-up risk.
SCHEDULE
Week of March 25, foundation
Roshni: project repo setup, Makefile, header files, green thread struct and stack allocation
Wasmir: ucontext API, context switch working for single green thread, test_basic passing
Week of April 1, single kernel thread scheduler
Roshni: per-kernel-thread run queue data structure, enqueue/dequeue, instrumentation counters
Wasmir: kernel thread scheduler loop, multiple green threads running cooperatively on single kernel thread, yielding correctly
Both: verify test_yield and test_cooperative passing, run tsan clean
Week of April 7, parallelism and work stealing
Roshni: extend to N kernel threads, verify parallel execution correct
Wasmir: work stealing implementation, try_steal logic, imbalanced workload test
Both: debug race conditions with tsan, first throughput measurements
milestone prep: have working parallel runtime with stealing by April 14
April 14, milestone report due
Both: write milestone, update project page, preliminary graphs showing throughput vs N kernel threads
Week of April 14, preemption and synchronization primitives
Roshni: SIGALRM preemption, signal masking around critical sections
Wasmir: green thread mutex and condition variable, producer-consumer test
Week of April 21, evaluation and reach goals
Roshni: full benchmark suite, Go and Java Loom comparison benchmarks
Wasmir: begin io_uring integration or bare metal Pi boot depending on progress
Both: collect all evaluation data, produce graphs
Week of April 28, writeup and poster
Both: final report, poster, source code cleanup
April 28: final report due
April 29: poster session

