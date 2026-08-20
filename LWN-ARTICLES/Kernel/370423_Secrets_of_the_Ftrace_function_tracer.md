---
title: Secrets of the Ftrace function tracer
url: https://lwn.net/Articles/370423/
date: "January 20, 2010"
category: "Ftrace; Tracing-Ftrace"
author: "January 20, 2010 This article was contributed by Steven Rostedt"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

January 20, 2010

This article was contributed by Steven Rostedt

Probably the most powerful tracer derived from Ftrace is the function tracer. It has the ability to trace practically every function in the kernel. It can be run not just for debugging or analyzing, but also to learn and observe the flow of the Linux kernel.

Two previous articles, Debugging the Linux Kernel Using Ftrace [parts I](<http://lwn.net/Articles/365835/>) and [II](<http://lwn.net/Articles/366796/>), explain some of the basic features of Ftrace and the function tracer; this article is written with the assumption that the reader has already read them. As with the previous articles, the examples in this article expect that the user has already changed to the debug file system `tracing` directory. The kernel configuration options that are need to be enabled to follow the examples in this article are:

  * CONFIG_FUNCTION_TRACER 
  * CONFIG_DYNAMIC_FTRACE 
  * CONFIG_FUNCTION_GRAPH_TRACER 

Note, the `CONFIG_**HAVE** _FUNCTION_TRACER`, `CONFIG_**HAVE** _DYNAMIC_FTRACE`, and `CONFIG_**HAVE** _FUNCTION_GRAPH_TRACER` options are enabled when the architecture supports the corresponding feature. Do not confuse them with the listed options. The features are only enabled when the listed configuration options are enabled and not when only the `_HAVE_` options are.

As shown in the previous articles, here is a quick example of how to enable the function tracer.

```
[tracing]# echo function > current_tracer
       [tracing]# cat trace
              <idle>-0     [000] 1726568.996435: hrtimer_get_next_event <-get_next_timer_interrupt
              <idle>-0     [000] 1726568.996436: _spin_lock_irqsave <-hrtimer_get_next_event
              <idle>-0     [000] 1726568.996436: _spin_unlock_irqrestore <-hrtimer_get_next_event
              <idle>-0     [000] 1726568.996437: rcu_needs_cpu <-tick_nohz_stop_sched_tick
              <idle>-0     [000] 1726568.996438: enter_idle <-cpu_idle
              ...
```

The above shows you the process name (`<idle>`), PID (`0`) the CPU that the trace executed on (`[000]`), a time-stamp in seconds with the decimal places down to microseconds (`1726568.996435`) the function being traced (`hrtimer_get_next_event`) and the parent that called that function (`get_next_timer_interrupt`).

#### Function filtering

Running the function tracer can be overwhelming. The amount of data may be vast, and very hard to get a hold of by the human brain. Ftrace provides a way to limit what functions you see. Two files exist that let you limit what functions are traced:

```
set_ftrace_filter
       set_ftrace_notrace
```

These filtering features depend on the `CONFIG_DYNAMIC_FTRACE` option. As explained in the previous articles, when this configuration is enabled all of the `mcount` caller locations are stored and at boot time are converted into NOPs. These locations are saved and used to enable tracing when the function tracer is activated. But this also has a nice side effect: not all functions must be enabled. The above files will determine which functions gets enabled and which do not.

When any function is listed in the `set_ftrace_filter`, only those functions will be traced. This will help the performance of the system when the trace is active. Tracing every function incurs a large overhead, but when using the `set_ftrace_filter`, only those functions listed in that file will have the NOPs changed to call the tracer. Depending on which functions are being traced, just having a couple of hundred functions enabled is hardly noticeable.

The `set_ftrace_notrace` file is the opposite of `set_ftrace_filter`. Instead of limiting the trace to a set of functions, functions listed in `set_ftrace_notrace` will not be traced. Some functions show up quite often and not only does tracing these functions slow down the system, they can fill up the trace buffer and make it harder to analyze the functions you care about. Functions such as `rcu_read_lock()` and `spin_lock()` fall into this category.

The process to add functions to these files typically uses bash redirection. Using the symbol '>' will remove all existing functions in the file and add what is being echoed into the file. Appending to the file using '>>' will keep the existing functions and add new ones.

```
[tracing]# echo sys_read > set_ftrace_filter
       [tracing]# cat set_ftrace_filter
       sys_read
       [tracing]# echo sys_write >> set_ftrace_filter
       [tracing]# cat set_ftrace_filter
       sys_write
       sys_read
       [tracing]# echo sys_open > set_ftrace_filter
       [tracing]# cat set_ftrace_filter
       sys_open
```

To remove all functions just echo a blank line into the filter file.

```
[tracing]# echo sys_read sys_open sys_write > set_ftrace_notrace 
       [tracing]# cat set_ftrace_notrace
       sys_open
       sys_write
       sys_read
       [tracing]# echo > set_ftrace_notrace
       [tracing]# cat set_ftrace_notrace
       [tracing]#
```

The functions listed in these files can also be set on the kernel command line. The options `ftrace_notrace` and `ftrace_filter` will preset these files by listing a comma delimited set of functions.

```
ftrace_notrace=rcu_read_lock,rcu_read_unlock,spin_lock,spin_unlock
       ftrace_filter=kfree,kmalloc,schedule,vmalloc_fault,spurious_fault
```

Functions added by the kernel command line set what will be in the corresponding filter files. These options only pre-load the files, functions can still be removed or added using the bash redirection as explained above.

The functions listed in `set_ftrace_notrace` take precedence. That is, if a function is listed in both `set_ftrace_notrace` and `set_ftrace_filter`, that function will not be traced. 

#### Wildcard filters

A list of functions that can be added to the filter files is shown in the `available_filter_functions` file. This list of functions was derived from the list of stored `mcount` callers previously mentioned.

```
[tracing]# cat available_filter_functions | head -8
       _stext
       do_one_initcall
       run_init_process
       init_post
       name_to_dev_t
       create_dev
       T.627
       set_personality_64bit
```

You can grep this file and redirect the result into one of the filter files:

```
[tracing]# grep sched available_filter_functions > set_ftrace_filter
       [tracing]# cat set_ftrace_filter | head -8
       save_stack_address_nosched
       mce_schedule_work
       smp_reschedule_interrupt
       native_smp_send_reschedule
       sys32_sched_rr_get_interval
       sched_avg_update
       proc_sched_set_task
       sys_sched_get_priority_max
```

Unfortunately, adding lots of functions to the filtering files is slow and you will notice that the above grep took several seconds to execute. This is because each function name written into the filter file will be processed individually. The above grep produces over 300 function names. Each of those 300 names will be compared (using `strcmp()`) against every function name in the kernel, which is quite a lot.

```
[tracing]# wc -l  available_filter_functions 
       24331 available_filter_functions
```

So the grep above caused `set_ftrace_filter` to generate over 300 * 24331 (7,299,300) comparisons!

Fortunately, these files also take wildcards; the following glob expressions are valid:

  * `value*` \- Select all functions that begin with `value`. 
  * `*value*` \- Select all functions that contain the text `value`. 
  * `*value` \- Select all functions that end with `value`. 

The kernel contains a rather simple parser, and will not process `value*value` in the expected way. It will ignore the second `value` and select all functions that start with `value` regardless of what it ends with. Wildcards passed to the filter files are processed directly for each available function, which is much faster than passing in individual functions in a list.

Because the star (*) is also used by bash, it is best to wrap the input with quotes: 

```
[tracing]# echo set* > set_ftrace_filter
       [tracing]# cat set_ftrace_filter
       #### all functions enabled ####
       [tracing]# echo 'set*' > set_ftrace_filter
       [tracing]# cat set_ftrace_filter | head -5
       set_personality_64bit
       set_intr_gate_ist
       set_intr_gate
       set_intr_gate
       set_tsc_mode
```

The filters can also select only those functions that belong to a specific module by using the 'mod' command in the input to the filter file:

```
[tracing]# echo ':mod:tg3' > set_ftrace_filter
       [tracing]# cat set_ftrace_filter |head -8
       tg3_write32
       tg3_read32
       tg3_write_flush_reg32
       tw32_mailbox_flush
       tg3_write32_tx_mbox
       tg3_read32_mbox_5906
       tg3_write32_mbox_5906
       tg3_disable_ints
```

This is very useful if you are debugging a single module, and only want to see the functions that belong to that module in the trace.

In the earlier articles, enabling and disabling recording to the ring buffer was done using the `tracing_on` file and the `tracing_on()` and `tracing_off()` kernel functions. But if you do not want to recompile the kernel, and you want to stop the tracing at a particular function, `set_ftrace_filter` has a method to do so.

The format of the command to have the function trace enable or disable the ring buffer is as follows:

```
function:command[:count]
```

This will execute the `command` at the start of the `function`. The `command` is either `traceon` or `traceoff`, and an optional `count` can be added to have the command only execute a given number of times. If the count is left off (including the leading colon) then the command will be executed every time the function is called.

A while back, I was debugging a change to the kernel I made that was causing a segmentation fault to some programs. I was having a hard time catching the trace, because by the time I was able to stop the trace after seeing the segmentation fault, the data had already been overwritten. But the backtrace on the console showed that the function `__bad_area_nosemaphore` was being called. I was then able to stop the tracer with the following command:

```
[tracing]# echo '__bad_area_nosemaphore:traceoff' > set_ftrace_filter
      [tracing]# cat set_ftrace_filter
      #### all functions enabled ####
      __bad_area_nosemaphore:traceoff:unlimited
      [tracing]# echo function > current_tracer
```

Notice that functions with commands do not affect the general filters. Even though a command has been added to `__bad_area_nosemaphore`, the filter still allowed all functions to be traced. Commands and filter functions are separate and do not affect each other. With the above command attached to the function `__bad_area_nosemaphore`, the next time the segmentation fault occurred, the trace stopped and contained the data I needed to debug the situation.

#### Removing functions from the filters

As stated earlier, echoing in nothing with '>' will clear the filter file. But what if you only want to remove a few functions from the filter?

```
[tracing]# cat set_ftrace_filter > /tmp/filter
       [tracing]# grep -v lock /tmp/filter > set_ftrace_filter
```

The above works, but as mentioned, it may take a while to complete if there were several functions already in `set_ftrace_filter`. The following does the same thing but is much faster:

```
[tracing]# echo '!*lock*' >> set_ftrace_filter
```

The '!' symbol will remove functions listed in the filter file. As shown above, the '!' works with wildcards, but could also be used with a single function. Since '!' has special meaning in bash it must be wrapped with single quotes or bash will try to execute what follows it. Also note the '>>' is used. If you make the mistake of using '>' you will end up with no functions in the filter file.

Because the commands and filters do not interfere with each other, clearing the `set_ftrace_filter` will not clear the commands. The commands must be cleared with the '!' symbol.

```
[tracing]# echo 'sched*' > set_ftrace_filter
       [tracing]# echo 'schedule:traceoff' >> set_ftrace_filter
       [tracing]# cat trace | tail -5
       schedule_console_callback
       schedule_bh
       schedule_iso_resource
       schedule_reallocations
       schedule:traceoff:unlimited
       [tracing]# echo > set_ftrace_filter
       [tracing]# cat set_ftrace_filter
       #### all functions enabled ####
       schedule:traceoff:unlimited
       [tracing]# echo '!schedule:traceoff' >> set_ftrace_filter
       [tracing]# cat set_ftrace_filter
       #### all functions enabled ####
       [tracing]#
```

This may seem awkward, but having the '>' and '>>' only affect the functions to be traced and not the function commands, actually simplifies the control between filtering functions and adding and removing commands.

#### Tracing a specific process

Perhaps you only need to trace a specific process, or set of processes. The file `set_ftrace_pid` lets you specify specific processes that you want to trace. To just trace the current thread you can do the following:

```
[tracing]# echo $$ > set_ftrace_pid
```

The above will set the function tracer to only trace the bash shell that executed the `echo` command. If you want to trace a specific process, you can create a shell script wrapper program.

```
[tracing]# cat ~/bin/ftrace-me
       #!/bin/sh
       DEBUGFS=`grep debugfs /proc/mounts | awk '{ print $2; }'`
       echo $$ > $DEBUGFS/tracing/set_ftrace_pid
       echo function > $DEBUGFS/tracing/current_tracer
       exec $*
       [tracing]# ~/bin/ftrace-me ls -ltr
```

Note, you must clear the `set_ftrace_pid` file if you want to go back to generic function tracing after performing the above.

```
[tracing]# echo -1 > set_ftrace_pid
```

#### What calls a specific function?

Sometimes it is useful to know what is calling a particular function. The immediate predecessor is helpful, but an entire backtrace is even better. The function tracer contains an option that will create a backtrace in the ring buffer for every function that is called by the tracer. Since creating a backtrace for every function has a large overhead, which could live lock the system, care must be taken when using this feature. Imagine the timer interrupt on a slower system where it is run at 1000 HZ. It is quite possible that having every function that the timer interrupt calls produce a backtrace could take 1 millisecond to complete. By the time the timer interrupt returns, a new one will be triggered before any other work can be done, which leads to a live lock.

To use the function tracer backtrace feature, it is imperative that the functions being called are limited by the function filters. The option to enable the function backtracing is unique to the function tracer and activating it can only be done when the function tracer is enabled. This means you must first enable the function tracer before you have access to the option:

```
[tracing]# echo kfree > set_ftrace_filter
       [tracing]# cat set_ftrace_filter
       kfree
       [tracing]# echo function > current_tracer
       [tracing]# echo 1 > options/func_stack_trace
       [tracing]# cat trace | tail -8
        => sys32_execve
        => ia32_ptregs_common
                    cat-6829  [000] 1867248.965100: kfree <-free_bprm
                    cat-6829  [000] 1867248.965100: <stack trace>
    
        => free_bprm
        => compat_do_execve
        => sys32_execve
        => ia32_ptregs_common
       [tracing]# echo 0 > options/func_stack_trace
       [tracing]# echo > set_ftrace_filter
```

Notice that I was careful to cat the `set_ftrace_filter` before enabling the `func_stack_trace` option to ensure that the filter was enabled. At the end, I disabled the `options/func_stack_trace` before disabling the filter. Also note that the option is non-volatile, that is, even if you enable another tracer plugin in `current_tracer`, the option will still be enabled if you re-enable the function tracer. 

#### The function_graph tracer

The function tracer is very powerful, but it may be difficult to understand the linear format that it produces. Frederic Weisbecker has extended the function tracer into the function_graph tracer. The function_graph tracer piggy-backs off of most of the code created by the function tracer, but adds its own hook in the `mcount` call. Because it still uses the `mcount` calling methods most of the function filtering explained above also applies to the function_graph tracer, with the exception of the `traceon`/`traceoff` commands and `set_ftrace_pid` (although the latter may change in the future).

The function_graph tracer was also explained in the previous articles, but the `set_graph_function` file was not described. The `func_stack_trace` used in the previous section can see what might call a function, but `set_graph_function` can be used to see what a function calls:

```
[tracing]# echo kfree > set_graph_function
       [tracing]# echo function_graph > current_tracer
       [tracing]# cat trace
       # tracer: function_graph
       #
       # CPU  DURATION                  FUNCTION CALLS
       # |     |   |                     |   |   |   |
        0)               |  kfree() {
        0)               |    virt_to_cache() {
        0)               |      virt_to_head_page() {
        0)   0.955 us    |        __phys_addr();
        0)   2.643 us    |      }
        0)   4.299 us    |    }
        0)   0.855 us    |    __cache_free();
        0)   ==========> |
        0)               |    smp_apic_timer_interrupt() {
        0)               |      apic_write() {
        0)   0.849 us    |        native_apic_mem_write();
        0)   2.853 us    |      }
       [tracing]# echo > set_graph_function
```

This displays the call graph performed only by `kfree()`. The "`==========>`" shows that an interrupt happened during the call. The trace records all functions within the `kfree()` block, even those functions called by an interrupt that triggered while in the scope of `kfree()`. 

The function_graph tracer shows the time a function took in the duration field. In the previous articles, it was mentioned that only the leaf functions, the ones that do not call other functions, have an accurate duration, since the duration of parent functions also includes the overhead of the function_graph tracer calling the child functions. By using the `set_ftrace_filter` file, you can force any function into becoming a leaf function in the function_graph tracer, and this will allow you to see an accurate duration of that function. 

```
[tracing]# echo smp_apic_timer_interrupt > set_ftrace_filter
       [tracing]# echo function_graph > current_tracer
       [tracing]# cat trace | head
       # tracer: function_graph
       #
       # CPU  DURATION                  FUNCTION CALLS
       # |     |   |                     |   |   |   |
        1)   ==========> |
        1) + 16.433 us   |  smp_apic_timer_interrupt();
        1)   ==========> |
        1) + 25.897 us   |  smp_apic_timer_interrupt();
        1)   ==========> |
        1) + 24.764 us   |  smp_apic_timer_interrupt();
```

The above shows that the timer interrupt takes between 16 and 26 microseconds to complete. 

#### Function profiling

`oprofile` and `perf` are very powerful profiling tools that take periodic samples of the system and can show where most of the time is spent. With the function profiler, it is possible to take a good look at the actual function execution and not just samples. If `CONFIG_FUNCTION_GRAPH_TRACER` is configured in the kernel, the function profiler will use the function graph infrastructure to record how long a function has been executing. If just `CONFIG_FUNCTION_TRACER` is configured, the function profiler will just count the functions being called. 

```
[tracing]# echo nop > current_tracer
       [tracing]# echo 1 > function_profile_enabled
       [tracing]# cat trace_stat/function 0 |head
         Function                               Hit    Time            Avg
         --------                               ---    ----            ---
         schedule                             22943    1994458706 us     86931.03 us 
         poll_schedule_timeout                 8683    1429165515 us     164593.5 us 
         schedule_hrtimeout_range              8638    1429155793 us     165449.8 us 
         sys_poll                             12366    875206110 us     70775.19 us 
         do_sys_poll                          12367    875136511 us     70763.84 us 
         compat_sys_select                     3395    527531945 us     155384.9 us 
         compat_core_sys_select                3395    527503300 us     155376.5 us 
         do_select                             3395    527477553 us     155368.9 us
```

The above also includes the times that a function has been preempted or `schedule()` was called and the task was swapped out. This may seem useless, but it does give an idea of what functions get preempted often. Ftrace also includes options that allow you to have the function graph tracer ignore the time the task was scheduled out. 

```
[tracing]# echo 0 > options/sleep-time
       [tracing]# echo 0 > function_profile_enabled
       [tracing]# echo 1 > function_profile_enabled
       [tracing]# cat trace_stat/function0  | head
         Function                               Hit    Time            Avg
         --------                               ---    ----            ---
         default_idle                          2493    6763414 us     2712.962 us 
         native_safe_halt                      2492    6760641 us     2712.938 us 
         sys_poll                              4723    714243.6 us     151.226 us  
         do_sys_poll                           4723    692887.4 us     146.704 us  
         sys_read                              9211    460896.3 us     50.037 us   
         vfs_read                              9243    434521.2 us     47.010 us   
         smp_apic_timer_interrupt              3940    275747.4 us     69.986 us   
         sock_poll                            80613    268743.2 us     3.333 us
```

Note that the `sleep-time` option contains a "-" and is not `sleep_time`.

Disabling the function profiler and then re-enabling it causes the numbers to reset. The list is sorted by the `Avg` times, but using scripts you can easily sort by any of the numbers. The `trace_stat/function0` only represents CPU 0. There exists a `trace_stat/function#` for each CPU on the system. All functions that are traced and have been hit are in this file.

```
[tracing]# cat trace_stat/function0  | wc -l
       2978
```

Functions that were not hit are not listed. The above shows that 2978 functions were hit since I started the profiling. 

Another option that affects profiling is `graph-time` (again with a "-"). By default it is enabled. When enabled, the times for a function include the times of all the functions that were called within the function. As you can see from the output in the above examples, several system calls are listed with the highest average. When disabled, the times only include the execution of the function itself, and do not contain the times of functions that are called from the function:

```
[tracing]# echo 0 > options/graph-time
       [tracing]# echo 0 > function_profile_enabled
       [tracing]# echo 1 > function_profile_enabled
       [tracing]# cat trace_stat/function0  | head
         Function                               Hit    Time            Avg
         --------                               ---    ----            ---
         mwait_idle                           10132    246835458 us     24361.96 us 
         tg_shares_up                        154467    389883.5 us     2.524 us    
         _raw_spin_lock_irqsave              343012    263504.3 us     0.768 us    
         _raw_spin_unlock_irqrestore         351269    175205.6 us     0.498 us    
         walk_tg_tree                         14087    126078.4 us     8.949 us    
         __set_se_shares                     274937    88436.65 us     0.321 us    
         _raw_spin_lock                      100715    82692.61 us     0.821 us    
         kstat_irqs_cpu                      257500    80124.96 us     0.311 us
```

Note that both `sleep-time` and `graph-time` also affect the duration times displayed by the function_graph tracer. 

#### Conclusion

The function tracer is very powerful with lots of different options. It is already available in mainline Linux, and hopefully will be enabled by default in most distributions. It allows you to see into the depths of the kernel and with its arsenal of features, gives you a good idea of why things are happening the way they do. Start using the function tracer to open up the black box that we call the kernel. Have fun!
