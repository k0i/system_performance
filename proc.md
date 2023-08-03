# /proc

The contents of /proc are documented in the `proc(5)` man page and in the Linux kernel documentation: `Documentation/filesystems/proc.txt`.\
Some parts have extended documentation, such as diskstats in `Documentation/iostats`.txt and scheduler stats in `Documentation/scheduler/sched-stats.txt`.\
/proc contains a number of directories, where each directory is named after the process ID for the process it represents.\
You can also examine /proc/self for your current process (shell).\\

## Per-Process Statistics

```shell
ls -F /proc/18733
arch_status      environ     mountinfo      personality   statm
attr/            exe@        mounts         projid_map    status
autogroup        fd/         mountstats     root@         syscall
auxv             fdinfo/     net/           sched         task/
cgroup           gid_map     ns/            schedstat     timers
clear_refs       io          numa_maps      sessionid     timerslack_ns
cmdline          limits      oom_adj        setgroups     uid_map
comm             loginuid    oom_score      smaps         wchan
coredump_filter  map_files/  oom_score_adj  smaps_rollup
cpuset           maps        pagemap        stack
cwd@             mem         patch_state    stat
```

Those related to per-process performance observability include:

- limits: In-effect resource limits
- maps: Mapped memory regions
- sched: Various CPU scheduler statistics
- schedstat: CPU runtime, latency, and time slices
- smaps: Mapped memory regions with usage statistics
- stat: Process status and statistics, including total CPU and memory usage
- statm: Memory usage summary in units of pages
- status: stat and statm information, labeled
- fd: Directory of file descriptor symlinks (also see fdinfo)
- cgroup: Cgroup membership information
- task: Directory of per-task (thread) statistics

The following shows how per-process statistics are read by top(1), traced using strace(1):

```shell
stat("/proc/14704", {st_mode=S_IFDIR|0555, st_size=0, ...}) = 0
open("/proc/14704/stat", O_RDONLY)      = 4
read(4, "14704 (sshd) S 1 14704 14704 0 -"..., 1023) = 232
close(4)
```

## System-Wide Statistics

```shell
cd /proc; ls -Fd [a-z]*
acpi/      dma          kallsyms     mdstat        schedstat      thread-self@
buddyinfo  driver/      kcore        meminfo       scsi/          timer_list
bus/       execdomains  keys         misc          self@          tty/
cgroups    fb           key-users    modules       slabinfo       uptime
cmdline    filesystems  kmsg         mounts@       softirqs       version
consoles   fs/          kpagecgroup  mtrr          stat           vmallocinfo
cpuinfo    interrupts   kpagecount   net@          swaps          vmstat
crypto     iomem        kpageflags   pagetypeinfo  sys/           zoneinfo
devices    ioports      loadavg      partitions    sysrq-trigger
diskstats  irq/         locks        sched_debug   sysvipc/
```

System-wide files related to performance observability include:

- cpuinfo: Physical processor information, including every virtual CPU, model name, clock speed, and cache sizes.
- diskstats: Disk I/O statistics for all disk devices
- interrupts: Interrupt counters per CPU
- loadavg: Load averages
- meminfo: System memory usage breakdowns
- net/dev: Network interface statistics
- net/netstat: System-wide networking statistics
- net/tcp: Active TCP socket information
- pressure/: Pressure stall information (PSI) files
- schedstat: System-wide CPU scheduler statistics
- self: A symlink to the current process ID directory, for convenience
- slabinfo: Kernel slab allocator cache statistics
- stat: A summary of kernel and system resource statistics: CPUs, disks, paging, swap, processes
- zoneinfo: Memory zone information

These are read by system-wide tools. For example, here’s vmstat(8) reading /proc, as traced by strace(1):

```shell
open("/proc/meminfo", O_RDONLY)         = 3
lseek(3, 0, SEEK_SET)                   = 0
read(3, "MemTotal:         889484 kB\nMemF"..., 2047) = 1170
open("/proc/stat", O_RDONLY)            = 4
read(4, "cpu  14901 0 18094 102149804 131"..., 65535) = 804
open("/proc/vmstat", O_RDONLY)          = 5
lseek(5, 0, SEEK_SET)                   = 0
read(5, "nr_free_pages 160568\nnr_inactive"..., 2047) = 1998
```

# CPU Statistic Accuracy

The `/proc/stat` file provides system-wide CPU utilization statistics and is used by many tools (vmstat(8), mpstat(1), sar(1), monitoring agents).\
The accuracy of these statistics depends on the kernel configuration. The default configuration (`CONFIG_TICK_CPU_ACCOUNTING`) measures CPU utilization with a granularity of clock ticks, which may be four milliseconds (depending on CONFIG_HZ).\
This is generally sufficient. There are options to improve accuracy by using higher-resolution counters, though with a small performance cost (`VIRT_CPU_ACCOUNTING_NATIVE` and `VIRT_CPU_ACCOUTING_GEN`),\
as well an option to for more accurate IRQ time (`IRQ_TIME_ACCOUNTING`). A different approach to obtaining accurate CPU utilization measurements is to use MSRs or PMCs.
