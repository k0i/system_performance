Compared with other tracers, perf(1) is especially suited for CPU analysis:

- profiling (sampling) CPU stack traces,
- tracing CPU scheduler behavior
- examining PMCs to understand micro-architectural level CPU performance including cycle behavior.

Its tracing capabilities allow it to analyze other targets as well, including disk I/O and software functions.

______________________________________________________________________

perf(1) can be used to answer questions such as:

- Which code paths are consuming CPU resources?
- Are the CPUs stalled on memory loads/stores?
- For what reasons are threads leaving the CPU?
- What is the pattern of disk I/O?

# Subcommands Overview

The following example sampled any program running on any CPU at 99 Hertz for 30 seconds, and then showed the most frequently sampled functions.

```bash
perf record -F 99 -a -- sleep 30
perf report --stdio
```

Selected subcommands from a recent perf(1) version (from Linux 5.6) are:

| Command       | Description                                                         |
| ------------- | ------------------------------------------------------------------- |
| annotate      | Read perf.data (created by perf record) and display annotated code. |
| archive       | Create a portable perf.data file containing debug and symbol info.  |
| bench         | System microbenchmarks.                                             |
| buildid-cache | Manage build-id cache (used by USDT probes).                        |
| c2c           | Cache line analysis tools.                                          |
| diff          | Read two perf.data files and display the differential profile.      |
| evlist        | List the event names in a perf.data file.                           |
| ftrace        | A perf(1) interface to the Ftrace tracer.                           |
| inject        | Filter to augment the events stream with additional information.    |
| kmem          | Trace/measure kernel memory (slab) properties.                      |
| kvm           | Trace/measure kvm guest instances.                                  |
| list          | List event types.                                                   |
| lock          | Analyze lock events.                                                |
| mem           | Profile memory access.                                              |
| probe         | Define new dynamic tracepoints.                                     |
| record        | Run a command and record its profile into perf.data.                |
| report        | Read perf.data (created by perf record) and display the profile.    |
| sched         | Trace/measure scheduler properties (latencies).                     |
| script        | Read perf.data (created by perf record) and display trace output.   |
| stat          | Run a command and gather performance counter statistics.            |
| timechart     | Visualize total system behavior during a workload.                  |
| top           | System profiling tool with real-time screen updates.                |
| trace         | A live tracer (system calls by default).                            |

![Снимок экрана_2023-08-20_06-52-06](https://github.com/k0i/system_performance/assets/100127291/d3f3171c-7b17-4379-ab30-688bc2eddd67)

# One Liners

See Also: https://www.brendangregg.com/perf.html#OneLiners

## Listing Events

- List all currently known events:

```bash
perf list
```

- List sched tracepoints:

```bash
perf list 'sched:*'
```

- List events with names containing the string “block”:

```bash
perf list block
```

- List currently available dynamic probes:

```bash
perf probe -l
```

## Counting Events

- Show PMC statistics for the specified command:

```bash
perf stat command
```

- Show PMC statistics for the specified PID, until Ctrl-C:

```bash
perf stat -p PID
```

- Show PMC statistics for the entire system, for 5 seconds:

```bash
perf stat -a sleep 5
```

- Show CPU last level cache (LLC) statistics for the command:

```bash
perf stat -e LLC-loads,LLC-load-misses,LLC-stores,LLC-prefetches command
```

- Count unhalted core cycles using a raw PMC specification (Intel):

```bash
perf stat -e r003c -a sleep 5
```

- Count front-end stalls using a verbose PMC raw specification (Intel):

```bash
perf stat -e cpu/event=0x0e,umask=0x01,inv,cmask=0x01/ -a sleep 5
```

- Count syscalls per second system-wide:

```bash
perf stat -e raw_syscalls:sys_enter -I 1000 -a
```

- Count system calls by type for the specified PID:

```bash
  perf stat -e 'syscalls:sys_enter\_\*' -p PID
```

- Count block device I/O events for the entire system, for 10 seconds:

```bash
perf stat -e 'block:\*' -a sleep 10
```

## Profiling

- Sample on-CPU functions for the specified command, at 99 Hertz:

```bash
  perf record -F 99 command
```

- Sample CPU stack traces (via frame pointers) system-wide for 10 seconds:

```bash
  perf record -F 99 -a -g sleep 10
```

- Sample CPU stack traces for the PID, using dwarf (debuginfo) to unwind stacks:

```bash
  perf record -F 99 -p PID --call-graph dwarf sleep 10
```

- Sample CPU stack traces for a container by its /sys/fs/cgroup/perf_event cgroup:

```bash
  perf record -F 99 -e cpu-clock --cgroup=docker/1d567f439319...etc... -a sleep 10
```

- Sample CPU stack traces for the entire system, using last branch record (LBR; Intel):

```bash
  perf record -F 99 -a --call-graph lbr sleep 10
```

- Sample CPU stack traces, once every 100 last-level cache misses, for 5 seconds:

```bash
  perf record -e LLC-load-misses -c 100 -ag sleep 5
```

- Sample on-CPU user instructions precisely (e.g., using Intel PEBS), for 5 seconds:

```bash
  perf record -e cycles:up -a sleep 5
```

- Sample CPUs at 49 Hertz, and show top process names and segments, live:

```bash
  perf top -F 49 -ns comm,dso
```

## Static tracing

- Trace new processes, until Ctrl-C:

```bash
  perf record -e sched:sched_process_exec -a
```

- Sample a subset of context switches with stack traces for 1 second:

```bash
  perf record -e context-switches -a -g sleep 1
```

- Trace all context switches with stack traces for 1 second:
  ```bash
  perf record -e sched:sched_switch -a -g sleep 1
  ```

````
- Trace all context switches with 5-level-deep stack traces for 1 second:
```bash
  perf record -e sched:sched_switch/max-stack=5/ -a sleep 1
````

- Trace connect(2) calls (outbound connections) with stack traces, until Ctrl-C:
  ```bash
  perf record -e syscalls:sys_enter_connect -a -g
  ```

````
- Sample at most 100 block device requests per second, until Ctrl-C:
```bash
  perf record -F 100 -e block:block_rq_issue -a
````

- Trace all block device issues and completions (has timestamps), until Ctrl-C:

```bash
  perf record -e block:block_rq_issue,block:block_rq_complete -a
```

- Trace all block requests, of size at least 64 Kbytes, until Ctrl-C:

```bash
  perf record -e block:block_rq_issue --filter 'bytes >= 65536'
```

- Trace all ext4 calls, and write to a non-ext4 location, until Ctrl-C:

```bash
  perf record -e 'ext4:\*' -o /tmp/perf.data -a
```

- Trace the http\_\_server\_\_request USDT event (from Node.js; Linux 4.10+):

```bash
  perf record -e sdt_node:http\_\_server\_\_request -a
```

- Trace block device requests with live output (no perf.data) until Ctrl-C:

```bash
  perf trace -e block:block_rq_issue
```

- Trace block device requests and completions with live output:

```bash
  perf trace -e block:block_rq_issue,block:block_rq_complete
```

- Trace system calls system-wide with live output (verbose):

```bash
  perf trace
```

## Dynamic Tracing

- Add a probe for the kernel tcp_sendmsg() function entry (--add optional):

```bash
perf probe --add tcp_sendmsg
```

- Remove the tcp_sendmsg() tracepoint (or -d):

```bash
perf probe --del tcp_sendmsg
```

- List available variables for tcp_sendmsg(), plus externals (needs kernel debuginfo):

```bash
perf probe -V tcp_sendmsg --externs
```

- List available line probes for tcp_sendmsg() (needs debuginfo):

```bash
perf probe -L tcp_sendmsg
```

- List available variables for tcp_sendmsg() at line number 81 (needs debuginfo):

```bash
perf probe -V tcp_sendmsg:81
```

- Add a probe for tcp_sendmsg() with entry argument registers (processor-specific):

```bash
perf probe 'tcp_sendmsg %ax %dx %cx'
```

- Add a probe for tcp_sendmsg(), with an alias (“bytes”) for the %cx register:

```bash
perf probe 'tcp_sendmsg bytes=%cx'
```

- Trace previously created probe when bytes (alias) is greater than 100:

```bash
perf record -e probe:tcp_sendmsg --filter 'bytes > 100'
```

- Add a tracepoint for tcp_sendmsg() return, and capture the return value:

```bash
perf probe 'tcp_sendmsg%return $retval'
```

- Add a tracepoint for tcp_sendmsg(), with size and socket state (needs debuginfo):

```bash
perf probe 'tcp_sendmsg size sk->__sk_common.skc_state'
```

- Add a tracepoint for do_sys_open() with the filename as a string (needs debuginfo):

```bash
perf probe 'do_sys_open filename:string'
```

- Add a tracepoint for the user-level fopen(3) function from libc:

```bash
perf probe -x /lib/x86_64-linux-gnu/libc.so.6 --add fopen
```

## Reporting

- Show perf.data in an ncurses browser (TUI) if possible:

```bash
perf report
```

- Show perf.data as a text report, with data coalesced and counts and percentages:

```bash
perf report -n --stdio
```

- List all perf.data events, with data header (recommended):

```bash
perf script --header
```

- List all perf.data events, with my recommended fields (needs record -a; Linux \< 4.1 used -f instead of -F):

\`\`
\`bash
perf script --header -F comm,pid,tid,cpu,time,event,ip,sym,dso

````

- Generate a flame graph visualization (Linux 5.8+):

```bash
perf script report flamegraph
````

- Disassemble and annotate instructions with percentages (needs some debuginfo):

```bash
perf annotate --stdio
```

# perf Events

Events can be listed using `perf list`.

| Event                         | Description                                           |
| ----------------------------- | ----------------------------------------------------- |
| Hardware event                | Mostly processor events (implemented using PMCs)      |
| Software event                | A kernel counter event                                |
| Hardware cache event          | Processor cache events (PMCs)                         |
| Kernel PMU event              | Performance Monitoring Unit (PMU) events (PMCs)       |
| cache, floating point...      | Processor vendor events (PMCs) and brief descriptions |
| Raw hardware event descriptor | PMCs specified using raw codes                        |
| Hardware breakpoint           | Processor breakpoint event                            |
| Tracepoint event              | Kernel static instrumentation events                  |
| SDT event                     | User-level static instrumentation events (USDT)       |
| pfm-events                    | libpfm events (added in Linux 5.8)                    |

The tracepoint and SDT events mostly list static instrumentation points, but if you have created some dynamic instrumentation probes, those will be listed as well.\
The perf list command accepts a search substring as an argument.\
For example, listing events containing “mem_load_l3” with events highlighted in bold:

```bash
# perf list mem_load_l3

List of pre-defined events (to be used in -e):

cache:
  mem_load_l3_hit_retired.xsnp_hit
       [Retired load instructions which data sources were L3 and cross-core snoop
hits in on-pkg core cache Supports address when precise (Precise event)]
  mem_load_l3_hit_retired.xsnp_hitm
       [Retired load instructions which data sources were HitM responses from shared
L3 Supports address when precise (Precise event)]
  mem_load_l3_hit_retired.xsnp_miss
       [Retired load instructions which data sources were L3 hit and cross-core snoop
missed in on-pkg core cache Supports address when precise (Precise event)]
  mem_load_l3_hit_retired.xsnp_none
       [Retired load instructions which data sources were hits in L3 without snoops
required Supports address when precise (Precise event)]
[...]
```

These are hardware events (PMC-based), and the output includes brief descriptions.\
The (Precise event) refers to precise event-based sampling (PEBS) capable events.

#

Hardware Events

Hardware Events are typically implemented using PMCs, which are configured using codes specific for the processor;\
for example, branch instructions on Intel processors can typically be instrumented with perf(1) by using the raw hardware event descriptor “r00c4,”\
short for the register codes: umask 0x0 and event select 0xc4.\
These codes are published in the processor manuals;Intel also makes them available as JSON files.

## Frequency Sampling

When using perf record with PMCs, a default sample frequency is used so that not every event is recorded.\
For example, recording the cycles event:

```bash
perf record -vve cycles -a sleep 1

Using CPUID GenuineIntel-6-8E
intel_pt default config: tsc,mtc,mtc_period=3,psb_period=3,pt,branch
------------------------------------------------------------
perf_event_attr:
  size                             112
  { sample_period, sample_freq }   4000
  sample_type                      IP|TID|TIME|CPU|PERIOD
  disabled                         1
  inherit                          1

 mmap                             1
  comm                             1
  freq                             1
[...]
[ perf record: Captured and wrote 3.360 MB perf.data (3538 samples) ]
```

The output shows that frequency sampling is enabled (freq 1) with a sample frequency of 4000.\
This tells the kernel to adjust the rate of sampling so that roughly 4,000 events per second per CPU are captured.\
This is desirable, since some PMCs instrument events that can occur billions of times per second (e.g., CPU cycles) and the overhead of recording every event would be prohibitive.\
But this is also a gotcha: the default output of perf(1) (without the very verbose option: `-vv`) does not say that frequency sampling is in use, and you may be expecting to record all events.\
This event frequency only affects the record subcommand; stat counts all events.

______________________________________________________________________

The event frequency can be modified using the `-F` option, or changed to a period using `-c`, which captures one-in-every-period events (also known as overflow sampling).\\

```bash
perf record -F 99 -e cycles -a sleep 1
```

This samples at a target rate of 99 Hertz (events per second).\
It is similar to the profiling one-liners in the section One-Liners:\
they do not specify the event (no `-e` cycles), which causes perf(1) to default to cycles if PMCs are available, or to the cpu-clock software event.\
See the following section **CPU Profiling**, for more details.\
Note that there is a limit to the frequency rate, as well as a CPU utilization percent limit for perf(1), which can be viewed and set using sysctl(8):

```bash
sysctl kernel.perf_event_max_sample_rate

kernel.perf_event_max_sample_rate = 15500
```

```bash
sysctl kernel.perf_cpu_time_max_percent

kernel.perf_cpu_time_max_percent = 25
```

This shows the maximum sample rate on this system to be 15,500 Hertz, and the maximum CPU utilization allowed by perf(1) (specifically the PMU interrupt) to be 25%.

# Software Events

These are events that typically map to hardware events, but are instrumented in software.\
Like hardware events, they may have a default sample frequency, typically 4000, so that only a subset is captured when using the record subcommand.\
Note the following difference between the context-switches software event, and the equivalent tracepoint. Starting with the software event:

```bash
perf record -vve context-switches -a -- sleep 1

[...]
------------------------------------------------------------
perf_event_attr:
  type                             1
  size                             112
  config                           0x3
  { sample_period, sample_freq }   4000
  sample_type                      IP|TID|TIME|CPU|PERIOD
[...]
  freq                             1
[...]
[ perf record: Captured and wrote 3.227 MB perf.data (660 samples) ]
```

This output shows that the software event has defaulted to frequency sampling at a rate of 4000 Hertz. Now the equivalent tracepoint:

```bash
perf record -vve sched:sched_switch -a sleep 1
[...]
------------------------------------------------------------
perf_event_attr:
  type                             2
  size                             112
  config                           0x131
  { sample_period, sample_freq }   1
  sample_type                      IP|TID|TIME|CPU|PERIOD|RAW
[...]
[ perf record: Captured and wrote 3.360 MB perf.data (3538 samples) ]
```

This time, period sampling is used (no freq 1), with a sample period of 1 (equivalent to `-c 1`).\
This captures every event. You can do the same with software events by specifying -c 1, for example:

```bash
perf record -vve context-switches -a -c 1 -- sleep 1
```

# Tracepoint Events
