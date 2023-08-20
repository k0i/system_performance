The strace(1) command is the Linux system call tracer.13 It can trace syscalls, printing a one-line summary for each, and can also count syscalls and print a report.\
Syscall tracers for other operating systems are: BSD has ktrace(1), Solaris has truss(1), OS X has dtruss(1) (a tool I originally developed), and Windows has a number of options including logger.exe and ProcMon.\
For example, tracing syscalls by PID 1884:

```shell
# -ttt: Prints the first column of time-since-epoch, in units of seconds with microsecond resolution.
# -T: Prints the last field (<time>), which is the duration of the system call, in units of seconds with microsecond resolution.
# -p PID: Trace this process ID. A command can also be specified so that strace(1) launches and traces it.

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

A feature of strace(1) can be seen in the output—translation of syscall arguments into a human-readable form.\
This is especially useful for understanding ioctl(2) calls.
The `-c` option can be used to summarize system call activity.\
The following example also invokes and traces a command (dd(1)) rather than attaching to a PID:

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

The output begins with three lines from dd(1) followed by the strace(1) summary.\
The columns are:

- time: Percentage showing where system CPU time was spent
- seconds: Total system CPU time, in seconds
- usecs/call: Average system CPU time per call, in microseconds
- calls: Number of system calls
- syscall: System call name

## strace Overhead

The current version of strace(1) employs breakpoint-based tracing via the Linux ptrace(2) interface.\
This sets breakpoints for the entry and return of all syscalls (even if the -e option is used to select only some).\
This is invasive, and applications with high syscall rates may find their performance worsened by an order of magnitude.

______________________________________________________________________

Depending on application requirements, this style of tracing may be acceptable to use for short durations to determine the syscall types being called.\
strace(1) would be of greater use if the overhead was not such a problem.\
Other tracers, including perf(1), Ftrace, BCC, and bpftrace, greatly reduce tracing overhead by using buffered tracing, where events are written to a shared kernel ring buffer and the user-level tracer periodically reads the buffer.\
This reduces context switching between user and kernel context, lowering overhead.\
A future version of strace(1) may solve its overhead problem by becoming an alias to the perf(1) trace subcommand.\
Other higher-performing syscall tracers for Linux, based on BPF, include: vltrace by Intel, and a Linux version of the Windows ProcMon tool by Microsoft.
