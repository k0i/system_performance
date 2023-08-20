syscount(8) is a BCC and bpftrace tool to count system calls system-wide.

```shell
syscount

Tracing syscalls, printing top 10... Ctrl+C to quit.
[05:01:28]
SYSCALL                   COUNT
recvfrom                 114746
sendto                    57395
ppoll                     28654
futex                       953
io_getevents                 55
bpf                          33
rt_sigprocmask               12
epoll_wait                   11
select                        7
nanosleep                     6

Detaching...
```

This shows the most frequent syscall was recvfrom(2), which was called 114,746 times while tracing.\
You can explore further using other tracing tools to examine the syscall arguments, latency, and calling stack trace.\
For example, you can use perf(1) trace with a `-e` recvfrom filter, or use bpftrace to instrument the syscalls:sys_enter_recvfrom tracepoint.

```shell
syscount -P
Tracing syscalls, printing top 10... Ctrl+C to quit.
^C[05:09:49]
PID    COMM               COUNT
2351   slack             115656
946    Xorg               89786
2534   slack              55483
2911   chrome             32014
1876   compiz             25304
26743  nvim                5638
13200  qterminal           5278
3166   chrome              3263
26734  nvim                3184
730    NetworkManager      2075
```
