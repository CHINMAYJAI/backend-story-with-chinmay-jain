_1. The Cost of Idle Time: I/O Bound vs. CPU Bound_
Understanding backend performance requires looking at exactly how much time is spent waiting versus processing.

- _The 70% Rule:_ In a typical backend application, _more than 70%_ of processing time is spent waiting for Input/Output (I/O) operations, not on actual CPU computation.
- _Database Query Latencies:_
  - Local network database query: Takes **1 to 2 milliseconds**.
  - Different availability zone: Takes **20 to 30 milliseconds**.
  - Different region: Takes **90 to 100 milliseconds**.
- _The 95% Waste:_ A typical mid-level API call might involve 5 network operations (like 3-5 database queries and 1-2 external cache/email services). If each takes an average of 50ms, the server spends _250 milliseconds_ just waiting. If the CPU processing only takes _10 milliseconds**, the CPU hardware sits idle **95% of the time_ for that request.
- _Lost Instructions:_ A modern CPU can execute roughly _3 billion instructions per second_ (**3 million instructions per millisecond**). If a single-threaded server waits 100 milliseconds for a database response, it wastes the opportunity to process **300 million instructions**.
- _Typical CPU Times:_ Routine CPU-bound tasks like JSON validation usually finish in just **1 to 2 milliseconds**.

_2. Concurrency vs. Parallelism Requirements_

- _Concurrency:_ This handles multiple tasks at once and can be achieved with just _1 CPU core_ by pausing and resuming tasks.
- _Parallelism:_ This executes multiple instructions simultaneously and requires hardware support—at least **2 CPU cores**.

_3. The Heavy Numbers Behind The Threading Model_
The traditional way to achieve concurrency is by creating native Operating System (OS) threads. However, this comes with massive mathematical overhead:

- _Time Slices:_ An OS scheduler typically assigns a processing time slice of roughly _2 milliseconds_ per thread before pausing it to switch to another.
- _Memory Overhead:_ An OS thread stack on Linux can allocate up to _8 megabytes_ of virtual memory. Even assuming a conservative _500 KB to 1 MB_ of physical memory per thread, receiving a traffic spike of _10,000 requests_ (using 10,000 threads) will instantly consume _8 to 9 Gigabytes_ of memory, easily crashing a server.
- _Creation Latency:_ Just asking the OS to create a new thread takes a **few microseconds to a few milliseconds**.
- _Context Switch Latency:_ When the OS switches the CPU from one thread to another, saving registers and updating state takes _1 to 10 microseconds_ per switch. If a system tries to juggle **1,000 threads**, this unproductive maintenance time stretches beyond milliseconds, slowing down actual work.

_4. The Event Loop Model (Single-Threaded Efficiency)_
Event loops (used in NodeJS, Python, etc.) avoid the mathematical overhead of heavy threads by using an entirely different approach:

- _The 1 Thread Rule:_ The event loop typically operates on exactly **1 thread**, meaning there is zero memory overhead for extra stacks and zero OS context-switching latency.
- _The 100 Millisecond Danger:_ The golden rule of event loops is never to block them. If a heavy CPU-bound task (like rendering an image) takes **100 milliseconds or more**, the entire single thread stops, freezing the processing of all other concurrent requests.

_5. Virtual Threads & Go Routines Scale_
Languages like Go create lightweight "virtual threads" (Go routines) managed by the runtime, rather than native OS threads.

- _CPU to Thread Ratio:_ A Go application might be capped at creating only _4 OS threads_ to perfectly match **4 CPU cores**.
- _Massive Scale:_ Because the Go scheduler handles context switching locally via lightweight pointer switches, those few OS threads can juggle _thousands or millions_ of Go routines at once. You can run _100 times more_ Go routines than standard OS threads before running out of memory.

_6. Race Conditions (The "Lost Update" Problem)_
Race conditions occur when shared memory is modified without proper safeguards, even in single-threaded async code.

- _The Mathematical Error:_ If a global user balance is _100**, and two concurrent requests attempt to withdraw **100\*\*, they might both check if the balance is sufficient at the exact same time (`100 >= 100`). Because of the way operations interleave while waiting for I/O, both requests proceed, subtracting 100 twice, resulting in a mathematical error where the final balance becomes _-100\*\* (`0 - 100`).
