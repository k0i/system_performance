# Tools

| Tool        | Description                          |
| ----------- | ------------------------------------ |
| uptime      | Load averages                        |
| vmstat      | Includes system-wide CPU averages    |
| mpstat      | Per-CPU statistics                   |
| sar         | Historical statistics                |
| ps          | Process statusA                      |
| top         | Monitor per-process/thread CPU usage |
| pidstat     | Per-process/thread CPU breakdowns    |
| time, ptime | Time a command, with CPU breakdowns  |
| turboboost  | Show CPU clock rate and other states |
| showboost   | Show CPU clock rate and turbo boost  |
| pmcarch     | Show high-level CPU cycle usage      |
| tlbstat     | Summarize TLB cycles                 |
| perf        | CPU profiling and PMC analysis       |
| profile     | Sample CPU stack traces              |
| cpudist     | Summarize on-CPU time                |
| runqlat     | Summarize CPU run queue latency      |
| runqlen     | Summarize CPU run queue length       |
| softirqs    | Summarize soft interrupt time        |
| hardirqs    | Summarize hard interrupt time        |
| bpftrace    | Tracing programs for CPU analysis    |

## uptime

uptime(1) is one of several commands that print the system load averages:

```sh
 10:06:39 up  1:17,  1 user,  load average: 1,95, 1,84, 1,82
```

The last three numbers are the 1-, 5-, and 15-minute load averages.  
By comparing the three numbers, you can determine if the load is increasing, decreasing, or steady during the last 15 minutes (or so).

## vmstat

It prints system-wide CPU averages in the last few columns, and a count of runnable threads in the first column:

```sh
vmstat 1

procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 25030140 220468 4378312    0    0   346   237 2996   12  7  2 92  0  0
 1  0      0 25031972 220484 4374720    0    0     0   560 2980 7091  3  1 95  0  0
 2  0      0 25033668 220492 4374996    0    0     0    48 4436 8117  8  2 90  0  0
```

- r: Run-queue length - the total number of runnable threads
- us: User-time percent
- sy: System-time (kernel) percent
- id: Idle percent
- wa: Wait I/O percent, which measures CPU idle when threads are blocked on disk I/O
- st: Stolen percent, which for virtualized environments shows CPU time spent servicing other tenants

---

All of these values are system-wide averages across all CPUs, with the exception of `r`, which is the total.

On Linux, the `r` column is the total number of tasks waiting plus those running.  
For other operating systems (e.g., Solaris) the `r` column only shows tasks waiting, not those running.  
The original `vmstat(1)` by Bill Joy and Ozalp Babaoglu for 3BSD in 1979 begins with an `RQ` column for the number of runnable and running processes, as the Linux `vmstat(8)` currently does.

## mpstat

The multiprocessor statistics tool, `mpstat(1)`, can report statistics per CPU:

```sh
mpstat -P ALL 1

Linux 6.2.10-surface (kali)     30.07.2023      _x86_64_        (8 CPU)

10:24:39     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
10:24:40     all    3,84    0,00    1,02    0,13    0,00    0,00    0,00    0,00    0,00   95,01
10:24:40       0   10,31    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00   89,69
10:24:40       1   10,10    0,00    2,02    0,00    0,00    0,00    0,00    0,00    0,00   87,88
10:24:40       2    1,03    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00   98,97
10:24:40       3    1,02    0,00    1,02    0,00    0,00    0,00    0,00    0,00    0,00   97,96
10:24:40       4    1,04    0,00    1,04    0,00    0,00    0,00    0,00    0,00    0,00   97,92
10:24:40       5    2,97    0,00    3,96    0,00    0,00    0,00    0,00    0,00    0,00   93,07
10:24:40       6    3,12    0,00    0,00    1,04    0,00    0,00    0,00    0,00    0,00   95,83
10:24:40       7    1,03    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00   98,97
```

The `-P ALL` option was used to print the per-CPU report. By default, `mpstat(1)` prints only the system-wide summary line (all).  
The columns are:

- CPU: Logical CPU ID, or all for summary
- %usr: User-time, excluding %nice
- %nice: User-time for processes with a nice’d priority
- %sys: System-time (kernel)
- %iowait: I/O wait
- %irq: Hardware interrupt CPU usage
- %soft: Software interrupt CPU usage
- %steal: Time spent servicing other tenants
- %guest: CPU time spent in guest virtual machines
- %gnice: CPU time to run a niced guest
- %idle: Idle

Key columns are `%usr`, `%sys`, and `%idle`.  
These identify CPU usage per CPU and show the user-time/kernel-time ratio.  
This can also identify “hot” CPUs — those running at 100% utilization (`%usr` + `%sys`) while others are not—which can be caused by single-threaded application workloads or device interrupt mapping.

## sar

The system activity reporter, `sar(1)`, can be used to observe current activity and can be configured to archive and report historical statistics.

![Снимок экрана_2023-07-30_10-42-16](https://github.com/k0i/system_performance/assets/100127291/b0849c0e-e819-4046-947e-fc54d64797e9)
