execsnoop(8) is a BCC and bpftrace tool that traces new process execution system-wide.\
It can find issues of short-lived processes that consume CPU resources and can also be used to debug software execution, including application start scripts.

```shell
execsnoop

PCOMM            PID    PPID   RET ARGS
oltp_read_write  13044  18184    0 /usr/share/sysbench/oltp_read_write.lua --db-
driver=mysql --mysql-password=... --table-size=100000 run
oltp_read_write  13047  18184    0 /usr/share/sysbench/oltp_read_write.lua --db-
driver=mysql --mysql-password=... --table-size=100000 run
sh               13050  13049    0 /bin/sh -c command -v debian-sa1 > /dev/null &&
debian-sa1 1 1 -S XALL
debian-sa1       13051  13050    0 /usr/lib/sysstat/debian-sa1 1 1 -S XALL
sa1              13051  13050    0 /usr/lib/sysstat/sa1 1 1 -S XALL
sadc             13051  13050    0 /usr/lib/sysstat/sadc -F -L -S DISK 1 1 -S XALL
/var/log/sysstat
[...]
```

I ran this on my database system in case it would find anything interesting, and it did: the first two lines show that a read/write microbenchmark was still running,\
aunching oltp_read_write commands in a loop—I had accidentally left this running for days!\
Since the database is handling a different workload, it wasn’t obvious from other system metrics that showed CPU and disk load.\
The lines after oltp_read_write show sar(1) collecting system metrics.

______________________________________________________________________

execsnoop(8) works by tracing the execve(2) system call, and prints a one-line summary for each.\
The tool supports some options, including -t for timestamps.
There is also a threadsnoop(8) tool for bpftrace to trace the creation of threads via libpthread pthread_create().
