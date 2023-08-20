uptime(1) is one of several commands that print the system load averages:

```bash
 uptime
  9:04pm  up 268 day(s), 10:16,  2 users,  load average: 7.76, 8.32, 8.60
```

The last three numbers are the 1-, 5-, and 15-minute load averages.\
By comparing the three numbers, you can determine if the load is increasing, decreasing, or steady during the last 15 minutes (or so).\
This can be useful to know: if you are responding to a production performance issue and find that the load is decreasing, you may have missed the issue; if the load is increasing, the issue may be getting worse!

# Pressure Stall Information (PSI)

PSI gives averages for CPU, memory, and I/O.\
The average shows the percent of time something was stalled on a resource (saturation only). This is compared with load averages in the following:

| Attribute | Load Averages                   | Pressure Stall Information          |
| --------- | ------------------------------- | ----------------------------------- |
| Resources | System-wide                     | cpu, memory, io (each individually) |
| Metric    | Number of busy and queued tasks | Percent of time stalled (waiting)   |
| Times     | 1 min, 5 min, 15 min            | 10 s, 60 s, 300 s                   |
| Average   | Exponentially damped moving sum | Exponentially damped moving sum     |

The following Table(Linux load average examples versus pressure stall information) shows what the metric shows for different scenarios:

| Example Scenario       | Load Averages | Pressure Stall Information |
| ---------------------- | ------------- | -------------------------- |
| 2 CPUs, 1 busy thread  | 1.0           | 0.0                        |
| 2 CPUs, 2 busy threads | 2.0           | 0.0                        |
| 2 CPUs, 3 busy threads | 3.0           | 50.0                       |
| 2 CPUs, 4 busy threads | 4.0           | 100.0                      |
| 2 CPUs, 5 busy threads | 5.0           | 100.0                      |

For example, showing the 2 CPU with 3 busy threads scenario:

```bash
uptime
07:51:13 up 4 days,  9:56,  2 users,  load average: 3.00, 3.00, 2.55

cat /proc/pressure/cpu
some avg10=50.00 avg60=50.00 avg300=49.70 total=1031438206

#
cat /proc/pressure/memory
```

This 50.0 value means a thread (“some”) has stalled 50% of the time.\
The io and memory metrics include a second line for when all non-idle threads have stalled (“full”).\
PSI best answers the question: how likely is it that a task will have to wait on the resources?
Whether you use load averages or PSI, you should quickly move to more detailed metrics to understand load, such as those provided by vmstat(1) and mpstat(1).
