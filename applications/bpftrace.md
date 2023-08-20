bpftrace is a BPF-based tracer that provides a high-level programming language, allowing the creation of powerful one-liners and short scripts.\
It is well suited for custom application analysis based on clues from other tools.

## Signal Tracing

This bpftrace one-liner traces process signals (via the kill(2) syscall) showing the source PID and process name, and destination PID and signal number:

```shell
bpftrace -e 't:syscalls:sys_enter_kill { time("%H:%M:%S ");
    printf("%s (PID %d) send a SIG %d to PID %d\n",
    comm, pid, args->sig, args->pid); }'

Attaching 1 probe...
09:07:59 bash (PID 9723) send a SIG 2 to PID 9723
09:08:00 systemd-journal (PID 214) send a SIG 0 to PID 501
09:08:00 systemd-journal (PID 214) send a SIG 0 to PID 550
09:08:00 systemd-journal (PID 214) send a SIG 0 to PID 392
```

The output shows a bash shell sending a signal 2 (Ctrl-C) to itself, followed by systemd-journal sending signal 0 to other PIDs.\
Signal 0 does nothing: it is typically used to check if another process still exists based on the syscall return value.\
This one-liner can be useful for debugging strange application issues, such as early terminations.\
Timestamps are included for cross-checking with performance issues in monitoring software.\
Tracing signals is also available as the standalone killsnoop(8) tool in BCC and bpftrace.

## I/O Profiling

bpftrace can be used to analyze I/O in various ways: examining sizes, latency, return values, and stack traces.\
For example, the recvfrom(2) syscall was frequently called in previous examples, and can be examined further using bpftrace.

```shell
bpftrace -e 't:syscalls:sys_enter_recvfrom { @bytes = hist(args->size); }'

Attaching 1 probe...
^C

@bytes:
[4, 8)             40142 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[8, 16)             1218 |@                                                   |
[16, 32)           17042 |@@@@@@@@@@@@@@@@@@@@@@                              |
[32, 64)               0 |                                                    |
[64, 128)              0 |                                                    |
[128, 256)             0 |                                                    |
[256, 512)             0 |                                                    |
[512, 1K)              0 |                                                    |
[1K, 2K)               0 |                                                    |
[2K, 4K)               0 |                                                    |
[4K, 8K)               0 |                                                    |
[8K, 16K)              0 |                                                    |
[16K, 32K)         19477 |@@@@@@@@@@@@@@@@@@@@@@@@@                           |
```

The output shows that about half of the sizes were very small, between 4 and 7 bytes, and the largest sizes were in the 16 to 32 Kbyte range.\
It may also be useful to compare this buffer size histogram to the actual bytes received, by tracing the syscall exit tracepoint:

```shell
# bpftrace -e 't:syscalls:sys_exit_recvfrom { @bytes = hist(args->ret); }'
```

A large mismatch may show an application is allocating larger buffers that it needs to.\
(Note that this exit one-liner will include syscall errors in the histogram as a size of -1.)

______________________________________________________________________

If the received sizes also show some small and some large I/O, this may also affect the latency of the syscall, with larger I/O taking longer.\
To measure recvfrom(2) latency, both the start and end of the syscall can be traced at the same time, as shown by the following bpftrace program.

```shell
# bpftrace -e 't:syscalls:sys_enter_recvfrom { @ts[tid] = nsecs; }
    t:syscalls:sys_exit_recvfrom /@ts[tid]/ {
    @usecs = hist((nsecs - @ts[tid]) / 1000); delete(@ts[tid]); }'

Attaching 2 probes...
^C
@usecs:
[0]                23280 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@                       |
[1]                40468 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[2, 4)               144 |                                                    |
[4, 8)             31612 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@            |
[8, 16)               98 |                                                    |
[16, 32)              98 |                                                    |
[32, 64)           20297 |@@@@@@@@@@@@@@@@@@@@@@@@@@                          |
[64, 128)           5365 |@@@@@@                                              |
[128, 256)          5871 |@@@@@@@                                             |
[256, 512)           384 |                                                    |
[512, 1K)             16 |                                                    |
[1K, 2K)              14 |                                                    |
[2K, 4K)               8 |                                                    |
[4K, 8K)               0 |                                                    |
[8K, 16K)              1 |                                                    |
```

The output shows that recvfrom(2) was often less than 8 microseconds, with a slower mode between 32 and 256 microseconds.\
Some latency outliers are present, the slowest reaching the 8 to 16 millisecond range.\
You can continue to drill down further. For example, the output map declaration (@usecs = ...) can be changed to:

- @usecs\[args->ret\]: To break down by syscall return value, showing a histogram for each. Since the return value is the number of bytes received, or -1 for error, this breakdown will confirm if larger I/O sizes caused higher latency.
- @usecs\[ustack\]: To break down by user stack trace, showing a latency histogram for each code path.

I would also consider adding a filter after the first tracepoint so that this showed the MySQL server only, and not other processes:

```shell
 bpftrace -e 't:syscalls:sys_enter_recvfrom /comm == "mysqld"/ {...
```

## Lock Tracing

bpftrace can be used to investigate application lock contention in a number of ways.\
For a typical pthread mutex lock, uprobes can be used to trace the pthread library functions: pthread_mutex_lock(), etc.;\
and tracepoints can be used to trace the futex(2) syscall that manages lock blocking.\
He previously developed the pmlock(8) and pmheld(8) bpftrace tools for instrumenting the pthread library functions, and have published these as open source.\\

```shell
pmlock.bt $(pgrep mysqld)

Attaching 4 probes...
Tracing libpthread mutex lock latency, Ctrl-C to end.
^C
[...]

@lock_latency_ns[0x7f37280019f0,
    pthread_mutex_lock+36
    THD::set_query(st_mysql_const_lex_string const&)+94
    Prepared_statement::execute(String*, bool)+336
    Prepared_statement::execute_loop(String*, bool, unsigned char*, unsigned char*...
    mysqld_stmt_execute(THD*, unsigned long, unsigned long, unsigned char*, unsign...
, mysqld]:
[1K, 2K)              47 |                                                    |
[2K, 4K)             945 |@@@@@@@@                                            |
[4K, 8K)            3290 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@                      |
[8K, 16K)           5702 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
```

This output has been truncated to show only one of the many stacks printed.\
This stack shows that lock address 0x7f37280019f0 was acquired via the THD::setquery() codepath, and acquisition was often in the 8 to 16 microsecond range.\
Why did this lock take this long? pmheld.bt shows the stack trace of the holder, by tracing the lock to unlock duration:

```shell
pmheld.bt $(pgrep mysqld)

Attaching 5 probes...
Tracing libpthread mutex held times, Ctrl-C to end.
^C
[...]
@held_time_ns[0x7f37280019f0,
    __pthread_mutex_unlock+0
    THD::set_query(st_mysql_const_lex_string const&)+147
    dispatch_command(THD*, COM_DATA const*, enum_server_command)+1045
    do_command(THD*)+544
    handle_connection+680
, mysqld]:
[2K, 4K)            3848 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@             |
[4K, 8K)            5038 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[8K, 16K)              0 |                                                    |
[16K, 32K)             0 |                                                    |
[32K, 64K)             1 |                                                    |
```

This shows a different code path for the holder.\
If the lock has a symbol name, it is printed instead of the address.\
Without the symbol name, you can identify the lock from the stack trace: this is a lock in THD::set_query() at instruction offset 147.\
The source code to that function shows it only acquires one lock: LOCK_thd_query.\
Tracing of locks does add overhead, and lock events can be frequent.\
It may be possible to develop similar tools based on kprobes of kernel futex functions, reducing the overhead somewhat.\
An alternate approach with negligible overhead is to use CPU profiling instead.\
CPU profiling typically costs little overhead as it is bounded by the sample rate, and heavy lock contention can use enough CPU cycles to show up in a CPU profile.
