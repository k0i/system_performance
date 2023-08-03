# Tools

## Fixed Counter

Kernels maintain various counters for providing system statistics.

### System-Wide

These tools examine system-wide activity in the context of system software or hardware resources, using kernel counters. Linux tools include:

- vmstat(8): Virtual and physical memory statistics, system-wide
- mpstat(1): Per-CPU usage
- iostat(1): Per-disk I/O usage, reported from the block device interface
- nstat(8): TCP/IP stack statistics
- sar(1): Various statistics; can also archive them for historical reporting

These tools are typically viewable by all users on the system (non-root). Their statistics are also commonly graphed by monitoring software.

### Per-Process

These tools are process-oriented and use counters that the kernel maintains for each process. Linux tools include:

- ps(1): Shows process status, shows various process statistics, including memory and CPU usage.
- top(1): Shows top processes, sorted by CPU usage or another statistic.
- pmap(1): Lists process memory segments with usage statistics.
- These tools typically read statistics from the /proc file system.

## Profiling

Profiling characterizes the target by collecting a set of samples or snapshots of its behavior.

### System-Wide

- perf(1): The standard Linux profiler, which includes profiling subcommands.
- profile(8): A BPF-based CPU profiler from the BCC repository that frequency counts stack traces in kernel context.
- Intel VTune Amplifier XE: Linux and Windows profiling, with a graphical interface including source browsing.

These can also be used to target a single process.

### Per-Process

- gprof(1): The GNU profiling tool, which analyzes profiling information added by compilers (e.g., gcc -pg).
- cachegrind: A tool from the valgrind toolkit, can profile hardware cache usage (and more) and visualize profiles using kcachegrind.
- Java Flight Recorder (JFR): Programming languages often have their own special-purpose profilers that can inspect language context. For example, JFR for Java.

## Tracing

Tracing instruments every occurrence of an event, and can store event-based details for later analysis or produce a summary.
This is similar to profiling, but the intent is to collect or inspect all events, not just a sample.

### System-Wide

These tracing tools examine system-wide activity in the context of system software or hardware resources, using kernel tracing facilities. Linux tools include:

- tcpdump(8): Network packet tracing (uses libpcap)
- biosnoop(8): Block I/O tracing (uses BCC or bpftrace)
- execsnoop(8): New processes tracing (uses BCC or bpftrace)
- perf(1): The standard Linux profiler, can also trace events
- perf trace: A special perf subcommand that traces system calls system-wide
- Ftrace: The Linux built-in tracer
- BCC: A BPF-based tracing library and toolkit
- bpftrace: A BPF-based tracer (bpftrace(8)) and toolkit

### Per-Process

These tracing tools are process-oriented, as are the operating system frameworks on which they are based. Linux tools include:

- strace(1): System call tracing
- gdb(1): A source-level debugger

# Observability Sources

| Type                              | Source                                                                                  |
| --------------------------------- | --------------------------------------------------------------------------------------- |
| Per-process counters              | /proc                                                                                   |
| System-wide counters              | /proc, /sys                                                                             |
| Device configuration and counters | /sys                                                                                    |
| Cgroup statistics                 | /sys/fs/cgroup                                                                          |
| Per-process                       | tracing ptrace                                                                          |
| Hardware counters (PMCs)          | perf_event                                                                              |
| Network statistics                | netlink                                                                                 |
| Network packet capture            | libpcap                                                                                 |
| Per-thread latency metrics        | Delay accounting                                                                        |
| System-wide tracing               | Function profiling (Ftrace), tracepoints, software events, kprobes, uprobes, perf_event |
