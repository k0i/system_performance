# CPU Profiling

CPU profiling is an essential activity for application performance analysis.\
There are many CPU profilers for Linux, including `perf(1)` and `profile(8)`, both of which used timed sampling.\
These profilers run in kernel mode and can capture both the kernel and user stacks, producing a mixed-mode profile.\
This provides (almost) complete visibility for CPU usage.\\

______________________________________________________________________

Applications and runtimes sometimes provide their own profiler that runs in user mode, which cannot show kernel CPU usage.\
These user-based profilers may have a skewed notion of CPU time as they may be unaware of when the kernel has descheduled the application, and do not account for it.\
Let's always start with kernel-based profilers (perf(1) and profile(8)) and use user-based ones as a last resort.

## CPU Flame Graphs

A CPU flame graph is shown as follows:\
![05fig03](https://github.com/k0i/system_performance/assets/100127291/04adc7dc-b7e1-4795-b517-5faec3d45dc8)

These are mixed-mode flame graphs, showing both user and kernel stacks.\
crc32_z() is the function that is on-CPU the most, spanning about 40% of this excerpt (the center plateau).\
A tower on the left shows a syscall `write(2)` path into the kernel, spanning about 30% of CPU time in total.\
With a quick glance, we’ve identified these two as possible low-level targets for optimization.\
Browsing the code-path ancestry (downwards) can reveal high-level targets: in this case, all the CPU usage is from the `MYSQL_BIN_LOG::commit()` function.

> As an example, I performed an internet search for MYSQL_BIN_LOG::commit() and quickly found articles describing MySQL binary logging, used for database restoration and replication, and how it can be tuned or disabled entirely. A quick search for crc32_z() shows it is a checksumming function from zlib. Perhaps there is a newer and faster version of zlib? Does the processor have the optimized CRC instruction, and is zlib using it? Does MySQL even need to calculate the CRC, or can it be turned off?

______________________________________________________________________

### Off-CPU Footprints

By browsing the CPU flame graph you may find evidence of file system I/O, disk I/O, network I/O, lock contention, and more.\
The example highlights the `ext4` file system I/O as an example.\
If you browse enough flame graphs, you’ll become familiar with the function names to look for: “tcp\_*” for kernel TCP functions, “blk\_*” for kernel block I/O functions, etc.\
Here are some suggested search terms for Linux systems:

- “ext4” (or “btrfs”, “xfs”, “zfs”): to find file system operations.
- “blk”: to find block I/O.
- “tcp”: to find network I/O.
- “utex”: to show lock contention (“mutex” or “futex”).
- “alloc” or “object”: to show code paths doing memory allocation.

### Off-CPU Analysis

Off-CPU analysis is the study of threads that are not currently running on a CPU: This state is called off-CPU.\
It includes all the reasons that threads block: disk I/O, network I/O, lock contention, explicit sleeps, scheduler preemption, etc.\
The analysis of these reasons and the performance issues they cause typically involves a wide variety of tools.\
Off-CPU analysis is one method to analyze them all, and can be supported by a single off-CPU profiling tool.

- Sampling: Collecting timer-based samples of threads that are off-CPU, or simply all threads (called wallclock sampling).
- Scheduler tracing: Instrumenting the kernel CPU scheduler to time the duration that threads are off-CPU, and recording these times with the off-CPU stack trace. Stack traces do not change when a thread is off-CPU (because it is not running to change it), so the stack trace only needs to be read once for each blocking event.
- Application instrumentation: Some applications have built-in instrumentation for commonly blocking code paths, such as disk I/O. Such instrumentation may include application-specific context. While convenient and useful, this approach is typically blind to off-CPU events (scheduler preemption, page faults, etc.).

The first two approaches are preferable as they work for all applications and can see all off-CPU events; however, they come with major overhead.\
Sampling at 49 Hertz should cost negligible overhead on, say, an 8-CPU system, but off-CPU sampling must sample the pool of threads rather than the pool of CPUs.\
The same system may have 10,000 threads, most of which are idle, so sampling them increases the overhead by 1,000x (imagine CPU-profiling a 10,000-CPU system).\
Scheduler tracing can also cost significant overhead, as the same system may have 100,000 scheduler events or more per second.

Scheduler tracing is the technique now commonly used, based on  tools such as `offcputime(8)`.

> An optimization I use is to record only off-CPU events that exceed a tiny duration, which reduces the number of samples.I also use BPF to aggregate stacks in kernel context, rather than emitting all samples to user space, reducing overhead even further. Much as these techniques help, you should be careful with off-CPU rofiling in production, and evaluate the overhead in a test environment before use.

## Thread State Analysis

### Nine States

This is a list of nine thread states to give better starting points for analysis than the two earlier states (on-CPU and off-CPU):

- User: On-CPU in user mode
- Kernel: On-CPU in kernel mode
- Runnable: And off-CPU waiting for a turn on-CPU
- Swapping (anonymous paging): Runnable, but blocked for anonymous page-ins
- Disk I/O: Waiting for block device I/O: reads/writes, data/text page-ins
- Net I/O: Waiting for network device I/O: socket reads/writes
- Sleeping: A voluntary sleep
- Lock: Waiting to acquire a synchronization lock (waiting on someone else)
- Idle: Waiting for work

![05fig06](https://github.com/k0i/system_performance/assets/100127291/41cc7db0-b5d5-49b0-b878-378737038add)


