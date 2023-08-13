CPU profiling is an essential activity for application performance analysis.\
There are many CPU profilers for Linux, including `perf(1)` and `profile(8)`, both of which used timed sampling.\
These profilers run in kernel mode and can capture both the kernel and user stacks, producing a mixed-mode profile.\
This provides (almost) complete visibility for CPU usage.\\

______________________________________________________________________

# Methodology

## CPU Profiling

Applications and runtimes sometimes provide their own profiler that runs in user mode, which cannot show kernel CPU usage.\
These user-based profilers may have a skewed notion of CPU time as they may be unaware of when the kernel has descheduled the application, and do not account for it.\
Let's always start with kernel-based profilers (perf(1) and profile(8)) and use user-based ones as a last resort.

### CPU Flame Graphs

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

## Off-CPU Analysis

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

Once you’ve established in which states the threads are spending their time, you can investigate them further:

- User or Kernel: Profiling can determine which code paths are consuming CPU, including time spent spinning on locks. See `CPU Profiling`.
- Runnable: Time in this state means the application wants more CPU resources. Examine CPU load for the entire system, and any CPU limits present for the application (e.g., resource controls).
- Swapping (anonymous paging): A lack of available main memory for the application can cause swapping delays. Examine memory usage for the entire system and any memory limits present.
- Disk: This state includes direct disk I/O and page faults. Workload characterization can help solve many disk I/O problems; examine file names, I/O sizes, and I/O types.
- Network: This state is for time blocked during network I/O (send/receive), but not listening for new connections (that’s idle time).Workload characterization can also be useful for network I/O problems; examine hostnames, protocols, and throughput.
- Sleeping: Analyze the reason (code path) and duration of the sleeps.
- Lock: Identify the lock, the thread holding it, and the reason why the holder held it for so long. The reason may be that the holder was blocked on another lock, which requires further unwinding. This is an advanced activity, usually performed by the software developer who has intimate knowledge of the application and its locking hierarchy. There is a BCC tool to aid this type of analysis: `offwaketime(8)` (included in BCC), which shows the blocking stack trace along with the waker.

### Linux

![05fig07](https://github.com/k0i/system_performance/assets/100127291/d74d64e1-6f26-45d6-98bb-9418d89d1b15)

The kernel thread state is based on the kernel `task_struct` state member: Runnable is TASK_RUNNING, Disk is TASK_UNINTERRUPTIBLE, and Sleep is TASK_INTERRUPTIBLE.\
These states are shown by tools including `ps(1)` and `top(1)` using single-letter codes: R, D, and S, respectively.\
While this provides some clues for further analysis, it is far from dividing time into the nine states described earlier.\
More information is required: for example, Runnable can be split into user and kernel time using `/proc` or `getrusage(2)` statistics.

### Clue-Based

You can start by using common OS tools, such as `pidstat`(1) and `vmstat`(8), to suggest where thread state time may be spent. The tools and column of interest are:

- User: pidstat(1) “%usr” (this state is measured directly)
- Kernel: pidstat(1) “%system” (this state is measured directly)
- Runnable: vmstat(8) “r” (system-wide)
- Swapping: vmstat(8) “si” and “so” (system-wide)
- Disk I/O: pidstat(1) -d “iodelay” (includes the swapping state)
- Network I/O: sar(1) -n DEV “rxkB/s” and “txkB/s” (system-wide)
- Sleeping: Not easily available
- Lock: perf(1) top (may identify spin lock time directly)
- Idle: Not easily available

Some of these statistics are system-wide.\
If you find via `vmstat`(8) that there is a system-wide rate of swapping, you could investigate that state using deeper tools to confirm that the application is affected.

### Off-CPU Analysis

As many of the states are off-CPU (everything except User and Kernel), you can apply off-CPU analysis to determine the thread state. See Section Off-CPU Analysis.

### Direct Measurement

Measure thread time accurately by thread state as follows:

- User: User-mode CPU is available from a number of tools and in /proc/PID/stat and `getrusage(2)`. `pidstat(1)` reports this as `%usr`.
- Kernel: Kernel-mode CPU is also in /proc/PID/stat and `getrusage(2)`. `pidstat(1)` reports this as `%system`.
- Runnable: This is tracked by the kernel schedstats feature in nanoseconds and is exposed via /proc/PID/schedstat. It can also be measured, at the cost of some overhead, using tracing tools including the `perf(1)` sched subcommand and BCC `runqlat(8)`.See *Chapter CPUs*.
- Swapping: Time swapping (anonymous paging) in nanoseconds can be measured by delay accounting included an example tool: getdelays.c. Tracing tools can also be used to instrument swapping latency.
- Disk: `pidstat(1)` **-d** shows “iodelay” as the number of clock ticks during which a process was delayed by block I/O and swapping; if there was no system-wide swapping (as reported by `vmstat(8)`), you could conclude that any iodelay was the I/O state. Delay accounting and other accounting features, if enabled, also provide block I/O time, as used by `iotop(8)`. You can also use tracing tools such as `biotop(8)` from BCC.
- Network: Network I/O can be investigated using tracing tools such as BCC and bpftrace, including the `tcptop(8)` tool for TCP network I/O. The application may also have instrumentation to track time in I/O (network and disk).
- Sleeping: Time entering voluntary sleep can be examined using tracers and events including the syscalls:sys_enter_nanosleep tracepoint. `naptime.bt` tool traces these sleeps and prints the PID and duration.
- Lock: Lock time can be investigated using tracing tools, including `klockstat(8)` from BCC and, from the bpf-perf-tools-book repository, `pmlock.bt` and `pmheld.bt` for pthread mutex locks, and `mlock.bt` and `mheld.bt` for kernel mutexes.
- Idle: Tracing tools can be used to instrument the application code paths that handle waiting for work.

Sometimes applications can appear to be completely asleep: they remain blocked off-CPU without a rate of I/O or other events.\
To determine what state the application threads are in, it may be necessary to use a debugger such as `pstack(1)` or `gdb(1)` to inspect thread stack traces, or to read them from the /proc/PID/stack files.\
Note that debuggers like these can pause the target application and cause performance problems of their own: understand how to use them and their risks before trying them in production.

# Observability Tools

This section introduces application performance observability tools for Linux-based operating systems.

| Tool      | Description                                  |
| --------- | -------------------------------------------- |
| strace    | Syscall tracing                              |
| execsnoop | New process tracing                          |
| syscount  | Syscall counting                             |
| bpftrace  | Signal tracing, I/O profiling, lock analysis |

These begin with CPU profiling tools then tracing tools.\
Many of the tracing tools are BPF-based, and use the BCC and bpftrace frontends; they are: `profile(8`), `offcputime(8)`, `execsnoop(8)`, and `syscount(8)`.\
See the documentation for each tool, including its man pages, for full references of its features.

## perf

`perf(1)` is the standard Linux profiler, a multi-tool with many uses.\
It is explained in *Chapter perf*.As CPU profiling is critical for application analysis, a summary of CPU profiling using `perf(1)` is included here.\
*Chapter CPUs*, covers CPU profiling and flame graphs in more detail.

### CPU Profiling

The following uses `perf(1)` to sample stack traces (-g) across all CPUs (-a) at 49 Hertz (-F 49: samples per second) for 30 seconds, and then to list the samples:

```shell
perf record -F 49 -a -g -- sleep 30
[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 0.560 MB perf.data (2940 samples) ]
perf script
mysqld 10441 [000] 64918.205722:   10101010 cpu-clock:pppH:
        5587b59bf2f0 row_mysql_store_col_in_innobase_format+0x270 (/usr/sbin/mysqld)
        5587b59c3951 [unknown] (/usr/sbin/mysqld)
        5587b58803b3 ha_innobase::write_row+0x1d3 (/usr/sbin/mysqld)
        5587b47e10c8 handler::ha_write_row+0x1a8 (/usr/sbin/mysqld)
        5587b49ec13d write_record+0x64d (/usr/sbin/mysqld)
        5587b49ed219 Sql_cmd_insert_values::execute_inner+0x7f9 (/usr/sbin/mysqld)
        5587b45dfd06 Sql_cmd_dml::execute+0x426 (/usr/sbin/mysqld)
        5587b458c3ed mysql_execute_command+0xb0d (/usr/sbin/mysqld)
        5587b4591067 mysql_parse+0x377 (/usr/sbin/mysqld)
        5587b459388d dispatch_command+0x22cd (/usr/sbin/mysqld)
        5587b45943b4 do_command+0x1a4 (/usr/sbin/mysqld)
        5587b46b22c0 [unknown] (/usr/sbin/mysqld)
        5587b5cfff0a [unknown] (/usr/sbin/mysqld)
        7fbdf66a9669 start_thread+0xd9 (/usr/lib/x86_64-linux-gnu/libpthread-2.30.so)
[...]
```

There are 2,940 stack samples in this profile; only one stack has been included here.\
The `perf(1)` script subcommand prints each stack sample in a previously recorded profile (the perf.data file).\
`perf(1)` also has a report subcommand for summarizing the profile as a code-path hierarchy. The profile can also be visualized as a CPU flame graph.

### Syscall Tracing

The `perf(1)` trace subcommand traces system calls by default,and is `perf(1)`’s version of `strace(1)` .\
For example, tracing a MySQL server process:

```shell
perf trace -p $(pgrep mysqld)
         ? (         ): mysqld/10120  ... [continued]: futex())
= -1 ETIMEDOUT (Connection timed out)
     0.014 ( 0.002 ms): mysqld/10120 futex(uaddr: 0x7fbddc37ed48, op: WAKE|
PRIVATE_FLAG, val: 1)           = 0
     0.023 (10.103 ms): mysqld/10120 futex(uaddr: 0x7fbddc37ed98, op: WAIT_BITSET|
PRIVATE_FLAG, utime: 0x7fbdc9cfcbc0, val3: MATCH_ANY) = -1 ETIMEDOUT (Connection
timed out)
[...]
```

### Kernel Time Analysis

As `perf(1)` trace shows time in syscalls, it helps explain the system CPU time commonly shown by monitoring tools,\
although it is easier to start with a summary than the event-by-event output. `perf(1)` trace summarizes syscalls with `-s`:

```shell
# perf trace -s -p $(pgrep mysqld)
 mysqld (14169), 225186 events, 99.1%

   syscall            calls    total       min       avg       max      stddev
                               (msec)    (msec)    (msec)    (msec)        (%)
   --------------- -------- --------- --------- --------- ---------     ------
   sendto             27239   267.904     0.002     0.010     0.109      0.28%
   recvfrom           69861   212.213     0.001     0.003     0.069      0.23%
   ppoll              15478   201.183     0.002     0.013     0.412      0.75%

[...]
```

The output shows syscall counts and timing for each thread.\
The earlier output showing `futex(2)` calls is not very interesting in isolation, and running `perf(1)` trace on any busy application will produce an avalanche of output.\
It helps to start with this summary first, and then to use `perf(1)` trace with a filter to inspect only the syscall types of interest.

### I/O Profiling

I/O syscalls are particularly interesting, and some were seen in the previous output.\
Tracing the `sendto(2)` calls using a filter `(-e)`:

```shell
perf trace -e sendto -p $(pgrep mysqld)
     0.000 ( 0.015 ms): mysqld/14097 sendto(fd: 37<socket:[833323]>, buff:
0x7fbdac072040, len: 12664, flags: DONTWAIT) = 12664
     0.451 ( 0.019 ms): mysqld/14097 sendto(fd: 37<socket:[833323]>, buff:
0x7fbdac072040, len: 12664, flags: DONTWAIT) = 12664
     0.624 ( 0.011 ms): mysqld/14097 sendto(fd: 37<socket:[833323]>, buff:
0x7fbdac072040, len: 11, flags: DONTWAIT) = 11
     0.788 ( 0.010 ms): mysqld/14097 sendto(fd: 37<socket:[833323]>, buff:
0x7fbdac072040, len: 11, flags: DONTWAIT) = 11
[...]
```

The output shows two 12664-byte sends followed by two 11-byte sends, all with the DONTWAIT flag.\
If I saw a flood of small sends, I might wonder if performance could be improved by coalescing them, or avoiding the DONTWAIT flag.\
While `perf(1)` trace can be used for some I/O profiling, I often wish to dig further into the arguments and summarize them in custom ways.\
For example, this `sendto(2)` trace shows the file descriptor (37) and socket number (833323), but I’d rather see the socket type, IP addresses, and ports.\
For such custom tracing, you can switch to bpftrace in Section bpftrace.

## profile

`profile(8)` is timer-based CPU profiler from BCC (See: Chapter BCC).\
It uses BPF to reduce overhead by aggregating stack traces in kernel context, and only passes unique stacks and their counts to user space.\
The following `profile(8)` example samples at 49 Hertz across all CPUs, for 10 seconds:

```shell
profile -F 49 10
Sampling at 49 Hertz of all threads by user + kernel stack for 10 secs.
[...]

    SELECT_LEX::prepare(THD*)
    Sql_cmd_select::prepare_inner(THD*)
    Sql_cmd_dml::prepare(THD*)
    Sql_cmd_dml::execute(THD*)
    mysql_execute_command(THD*, bool)
    Prepared_statement::execute(String*, bool)
    Prepared_statement::execute_loop(String*, bool)
    mysqld_stmt_execute(THD*, Prepared_statement*, bool, unsigned long, PS_PARAM*)
    dispatch_command(THD*, COM_DATA const*, enum_server_command)
    do_command(THD*)
    [unknown]
    [unknown]
    start_thread
    -                mysqld (10106)
        13
```

Only one stack trace is included in this output, showing that SELECT_LEX::prepare() was sampled on-CPU with that ancestry 13 times.\
`profile(8)` is further discussed in Chapter CPUs, which lists its various options and includes instructions for generating CPU flame graphs from its output.

## offcputime

`offcputime(8)` is a BCC and bpftrace tool (Chapter 15) to summarize time spent by threads blocked and off-CPU, showing stack traces to explain why.\
It supports Off-CPU analysis. `offcputime(8)` is the counterpart to `profile(8)`: between them, they show the entire time spent by threads on the system.\
The following shows `offcputime(8)` from BCC, tracing for 5 seconds:

```shell
offcputime 5
Tracing off-CPU time (us) of all threads by user + kernel stack for 5 secs.
[...]

    finish_task_switch
    schedule
    jbd2_log_wait_commit
    jbd2_complete_transaction
    ext4_sync_file
    vfs_fsync_range
    do_fsync
    __x64_sys_fdatasync
    do_syscall_64
    entry_SYSCALL_64_after_hwframe
    fdatasync
    IO_CACHE_ostream::sync()
    MYSQL_BIN_LOG::sync_binlog_file(bool)
    MYSQL_BIN_LOG::ordered_commit(THD*, bool, bool)
    MYSQL_BIN_LOG::commit(THD*, bool)
    ha_commit_trans(THD*, bool, bool)
    trans_commit(THD*, bool)
    mysql_execute_command(THD*, bool)
    Prepared_statement::execute(String*, bool)
    Prepared_statement::execute_loop(String*, bool)
    mysqld_stmt_execute(THD*, Prepared_statement*, bool, unsigned long, PS_PARAM*)
    dispatch_command(THD*, COM_DATA const*, enum_server_command)
    do_command(THD*)
    [unknown]
    [unknown]
    start_thread
    -                mysqld (10441)
        352107

[...]
```

The output shows unique stack traces and their time spent off-CPU in microseconds.\
This particular stack shows `ext4` file system sync operations via a code path through `MYSQL_BIN_LOG::sync_binlog_file()`, totaling 352 milliseconds during this trace.\
For efficiency, `offcputime(8)` aggregates these stacks in kernel context, and emits only unique stacks to user space.\
It also only records stack traces for off-CPU durations that exceed a threshold, one microsecond by default, which can be tuned using the `-m` option.\
There is also a `-M` option to set the maximum time for recording stacks.Why would we want to exclude long-duration stacks?\\
This can be an effective way to filter out uninteresting stacks: threads waiting for work and blocking for one or more seconds in a loop. Try using `-M 900000`, to exclude durations longer than 900 ms.

### Off-CPU Time Flame Graphs

Despite only showing unique stacks, the full output from the previous example was still over 200,000 lines.\
To make sense of it, it can be visualized as an off-CPU time flame graph. The commands to generate these are similar to those with `profile(8)`:

```shell
git clone https://github.com/brendangregg/FlameGraph; cd FlameGraph
offcputime -f 5 | ./flamegraph.pl --bgcolors=blue --title="Off-CPU Time Flame Graph"> out.svg
```

## strace

The `strace(1)` command is the Linux system call tracer.\
It can trace syscalls, printing a one-line summary for each, and can also count syscalls and print a report.\\

______________________________________________________________________

Syscall tracers for other operating systems are: BSD has `ktrace(1)`, Solaris has `truss(1)`, OS X has `dtruss(1)`, and Windows has a number of options including `logger.exe` and `ProcMon`.\
For example, tracing syscalls by PID 1884:

```shell
strace -ttt -T -p 1884

1356982510.395542 close(3)              = 0 <0.000267>
1356982510.396064 close(4)              = 0 <0.000293>
1356982510.396617 ioctl(255, TIOCGPGRP, [1975]) = 0 <0.000019>
1356982510.396980 rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0 <0.000024>
1356982510.397288 rt_sigprocmask(SIG_BLOCK, [CHLD], [], 8) = 0 <0.000014>
1356982510.397365 wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WSTOPPED|
WCONTINUED, NULL) = 1975 <0.018187>
[...]
```

- `-ttt`: Prints the first column of time-since-epoch, in units of seconds with microsecond resolution.
- `-T`: Prints the last field (<time>), which is the duration of the system call, in units of seconds with microsecond resolution.
- `-p PID`: Trace this process ID. A command can also be specified so that `strace(1)` launches and traces it.

Other options not used here include `-f` to follow child threads, and `-o` filename to write the strace(1) output to the given file name.\
A feature of `strace(1)` can be seen in the output—translation of syscall arguments into a human-readable form.\
This is especially useful for understanding `ioctl(2)` calls.\
The `-c` option can be used to summarize system call activity.\
The following example also invokes and traces a command (`dd(1)`) rather than attaching to a PID:

```shell
strace -c dd if=/dev/zero of=/dev/null bs=1k count=5000k

5120000+0 records in
5120000+0 records out
5242880000 bytes (5.2 GB) copied, 140.722 s, 37.3 MB/s
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 51.46    0.008030           0   5120005           read
 48.54    0.007574           0   5120003           write
  0.00    0.000000           0        20        13 open
[...]
------ ----------- ----------- --------- --------- ----------------
100.00    0.015604              10240092        19 total
```

The output begins with three lines from `dd(1)` followed by the `strace(1)` summary. The columns are:

- time: Percentage showing where system CPU time was spent
- seconds: Total system CPU time, in seconds
- usecs/call: Average system CPU time per call, in microseconds
- calls: Number of system calls
- syscall: System call name

### strace Overhead

The current version of `strace(1)` employs breakpoint-based tracing via the Linux `ptrace(2)` interface.\
This sets breakpoints for the entry and return of all syscalls (even if the -e option is used to select only some).\
This is invasive, and applications with high syscall rates may find their performance worsened by an order of magnitude.

______________________________________________________________________

Depending on application requirements, this style of tracing may be acceptable to use for short durations to determine the syscall types being called.\
`strace(1)` would be of greater use if the overhead was not such a problem.\
Other tracers, including `perf(1)`, `Ftrace`, `BCC`, and `bpftrace`, greatly reduce tracing overhead by using buffered tracing,\
where events are written to a shared kernel ring buffer and the user-level tracer periodically reads the buffer.\
This reduces context switching between user and kernel context, lowering overhead.

## execsnoop
