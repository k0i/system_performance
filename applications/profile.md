profile(8) is timer-based CPU profiler from BCC.\
It uses BPF to reduce overhead by aggregating stack traces in kernel context, and only passes unique stacks and their counts to user space.\
The following profile(8) example samples at 49 Hertz across all CPUs, for 10 seconds:

```shell
profile -F 49 10

Sampling at 49 Hertz of all threads by user + kernel stack for 10 secs.
[...]

SELECT_LEX::prepare(THD*)
Sql_cmd_select::prepare_inner(THD*)
Sql_cmd_dml::prepare(THD*)
Sql_cmd_dml::execute(THD*)
mysql_execute_command(THD*, bool)
Prepared_statement::execute(String*, bool)
Prepared_statement::execute_loop(String*, bool)
mysqld_stmt_execute(THD*, Prepared_statement*, bool, unsigned long, PS_PARAM*)
dispatch_command(THD*, COM_DATA const*, enum_server_command)
do_command(THD*)
[unknown]
[unknown]
start_thread
-                mysqld (10106)
    13
[...]
```

Only one stack trace is included in this output, showing that SELECT_LEX::prepare() was sampled on-CPU with that ancestry 13 times.\
profile(8) is further discussed in Chapter `CPUs`, which lists its various options and includes instructions for generating CPU flame graphs from its output.
