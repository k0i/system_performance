perf(1) is the standard Linux profiler, a multi-tool with many uses.\\

# CPU Summary Profiling

The following uses perf(1) to sample stack traces (-g) across all CPUs (-a) at 51 Hertz (-F 49: samples per second) for 30 seconds, and then to list the samples:

```bash
perf record -F 49 -a -g -- sleep 30
perf script
```

# CPU Flame Graphs

```bash
perf record -F 49 -a -g -- sleep 10; perf script --header > out.stacks
git clone https://github.com/brendangregg/FlameGraph; cd FlameGraph
./stackcollapse-perf.pl < ../out.stacks | ./flamegraph.pl --hash > out.svg
```

# Syscall Tracing

The perf(1) `trace` subcommand traces system calls by default, and is perf(1)’s version of strace(1).\
For example, tracing a MySQL server process:

```shell
perf trace -p $(pgrep mysqld)
```

The advantage of perf(1) is that it uses per-CPU buffers to reduce the overhead, making it much safer to use than the current implementation of strace(1).\
It can also trace system-wide, whereas strace(1) is limited to a set of processes (typically a single process), and it can trace events other than syscalls.\
perf(1), however, does not have as many syscall argument translations as strace(1);

# Kernel Time Analysis

As perf(1) trace shows time in syscalls, it helps explain the system CPU time commonly shown by monitoring tools,\
although it is easier to start with a summary than the event-by-event output. perf(1) trace summarizes syscalls with `-s`.

It helps to start with this summary first, and then to use perf(1) trace with a filter to inspect only the syscall types of interest:

```shell
perf trace -s -p $(pgrep mysqld)

mysqld (14169), 225186 events, 99.1%

   syscall            calls    total       min       avg       max      stddev
                               (msec)    (msec)    (msec)    (msec)        (%)
   --------------- -------- --------- --------- --------- ---------     ------
   sendto             27239   267.904     0.002     0.010     0.109      0.28%
   recvfrom           69861   212.213     0.001     0.003     0.069      0.23%
   ppoll              15478   201.183     0.002     0.013     0.412      0.75%

[...]
```

# I/O Profiling

I/O syscalls are particularly interesting, and some were seen in the previous output.\
Tracing the sendto(2) calls using a filter `(-e)`:

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
While perf(1) trace can be used for some I/O profiling, I often wish to dig further into the arguments and summarize them in custom ways.\
For example, this sendto(2) trace shows the file descriptor (37) and socket number (833323), but I’d rather see the socket type, IP addresses, and ports. For such custom tracing, you can switch to bpftrace.
