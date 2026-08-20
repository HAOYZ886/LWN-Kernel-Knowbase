---
title: "Debugging the kernel using Ftrace - part 1"
url: https://lwn.net/Articles/365835/
date: "December 9, 2009"
category: "Development tools-Kernel tracing; Ftrace; Tracing-Ftrace"
author: "December 9, 2009 This article was contributed by Steven Rostedt"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

December 9, 2009

This article was contributed by Steven Rostedt

Ftrace is a tracing utility built directly into the Linux kernel. Many distributions already have various configurations of Ftrace enabled in their most recent releases. One of the benefits that Ftrace brings to Linux is the ability to see what is happening inside the kernel. As such, this makes finding problem areas or simply tracking down that strange bug more manageable. 

Ftrace's ability to show the events that lead up to a crash gives a better chance of finding exactly what caused it and can help the developer in creating the correct solution. This article is a two part series that will cover various methods of using Ftrace for debugging the Linux kernel. This first part will talk briefly about setting up Ftrace, using the function tracer, writing to the Ftrace buffer from within the kernel, and various ways to stop the tracer when a problem is detected.

Ftrace was derived from two tools. One was the "latency tracer" by Ingo Molnar used in the -rt tree. The other was my own "logdev" utility that had its primary use on debugging the Linux kernel. This article will mostly describe features that came out of logdev, but will also look at the function tracer that originated in the latency tracer.

#### Setting up Ftrace

Currently the API to interface with Ftrace is located in the Debugfs file system. Typically, that is mounted at `/sys/kernel/debug`. For easier accessibility, I usually create a `/debug` directory and mount it there. Feel free to choose your own location for Debugfs.

When Ftrace is configured, it will create its own directory called `tracing` within the Debugfs file system. This article will reference those files in that directory as though the user first changed directory to the Debugfs tracing directory to avoid any confusion as to where the Debugfs file system has been mounted.

```
[~]# cd /sys/kernel/debug/tracing
        [tracing]#
```

This article is focusing on using Ftrace as a debugging tool. Some configurations for Ftrace are used for other purposes, like finding latency or analyzing the system. For the purpose of debugging, the kernel configuration parameters that should be enabled are: 

```
CONFIG_FUNCTION_TRACER
        CONFIG_FUNCTION_GRAPH_TRACER
        CONFIG_STACK_TRACER
        CONFIG_DYNAMIC_FTRACE
```

#### Function tracing - no modification necessary

One of the most powerful tracers of Ftrace is the function tracer. It uses the `-pg` option of `gcc` to have every function in the kernel call a special function "`mcount()`". That function must be implemented in assembly because the call does not follow the normal C ABI. 

When CONFIG_DYNAMIC_FTRACE is configured the call is converted to a NOP at boot time to keep the system running at 100% performance. During compilation the mcount() call-sites are recorded. That list is used at boot time to convert those sites to NOPs. Since NOPs are pretty useless for tracing, the list is saved to convert the call-sites back into trace calls when the function (or function graph) tracer is enabled. 

It is highly recommended to enable CONFIG_DYNAMIC_FTRACE because of this performance enhancement. In addition, CONFIG_DYNAMIC_FTRACE gives the ability to filter which function should be traced. Note, even though the NOPs do not show any impact in benchmarks, the addition of frame pointers that come with the `-pg` option has been known to cause a slight overhead. 

To find out which tracers are available, simply cat the `available_tracers` file in the `tracing` directory:

```
[tracing]# cat available_tracers 
        function_graph function sched_switch nop
```

To enable the function tracer, just echo "function" into the `current_tracer` file.

```
[tracing]# echo function > current_tracer
        [tracing]# cat current_tracer
        function
    
        [tracing]# cat trace | head -10
        # tracer: function
        #
        #           TASK-PID    CPU#    TIMESTAMP  FUNCTION
        #              | |       |          |         |
                    bash-16939 [000]  6075.461561: mutex_unlock <-tracing_set_tracer
                  <idle>-0     [001]  6075.461561: _spin_unlock_irqrestore <-hrtimer_get_next_event
                  <idle>-0     [001]  6075.461562: rcu_needs_cpu <-tick_nohz_stop_sched_tick
                    bash-16939 [000]  6075.461563: inotify_inode_queue_event <-vfs_write
                  <idle>-0     [001]  6075.461563: mwait_idle <-cpu_idle
                    bash-16939 [000]  6075.461563: __fsnotify_parent <-vfs_write
```

The header explains the format of the output pretty well. The first two items are the traced task name and PID. The CPU that the trace was executed on is within the brackets. The timestamp is the time since boot, followed by the function name. The function in this case is the function being traced with its parent following the "`<-`" symbol.

This information is quite powerful and shows the flow of functions nicely. But it can be a bit hard to follow. The function graph tracer, created by Frederic Weisbecker, traces both the entry and exit of a function, which gives the tracer the ability to know the depth of functions that are called. The function graph tracer can make following the flow of execution within the kernel much easier to follow with the human eye:

```
[tracing]# echo function_graph > current_tracer 
        [tracing]# cat trace | head -20
        # tracer: function_graph
        #
        # CPU  DURATION                  FUNCTION CALLS
        # |     |   |                     |   |   |   |
         1)   1.015 us    |        _spin_lock_irqsave();
         1)   0.476 us    |        internal_add_timer();
         1)   0.423 us    |        wake_up_idle_cpu();
         1)   0.461 us    |        _spin_unlock_irqrestore();
         1)   4.770 us    |      }
         1)   5.725 us    |    }
         1)   0.450 us    |    mutex_unlock();
         1) + 24.243 us   |  }
         1)   0.483 us    |  _spin_lock_irq();
         1)   0.517 us    |  _spin_unlock_irq();
         1)               |  prepare_to_wait() {
         1)   0.468 us    |    _spin_lock_irqsave();
         1)   0.502 us    |    _spin_unlock_irqrestore();
         1)   2.411 us    |  }
         1)   0.449 us    |  kthread_should_stop();
         1)               |  schedule() {
```

This gives the start and end of a function denoted with the C like annotation of "`{`" to start a function and "`}`" at the end. Leaf functions, which do not call other functions, simply end with a "`;`". The DURATION column shows the time spent in the corresponding function. The function graph tracer records the time the function was entered and exited and reports the difference as the duration. These numbers only appear with the leaf functions and the "`}`" symbol. Note that this time also includes the overhead of all functions within a nested function as well as the overhead of the function graph tracer itself. The function graph tracer hijacks the return address of the function in order to insert a trace callback for the function exit. This breaks the CPU's branch prediction and causes a bit more overhead than the function tracer. The closest true timings only occur for the leaf functions.

The lonely "`+`" that is there is an annotation marker. When the duration is greater than 10 microseconds, a "`+`" is shown. If the duration is greater than 100 microseconds a "`!`" will be displayed. 

#### Using `trace_printk()`

`printk()` is the king of all debuggers, but it has a problem. If you are debugging a high volume area such as the timer interrupt, the scheduler, or the network, `printk()` can lead to bogging down the system or can even create a live lock. It is also quite common to see a bug "disappear" when adding a few `printk()`s. This is due to the sheer overhead that `printk()` introduces.

Ftrace introduces a new form of `printk()` called `trace_printk()`. It can be used just like `printk()`, and can also be used in any context (interrupt code, NMI code, and scheduler code). What is nice about `trace_printk()` is that it does not output to the console. Instead it writes to the Ftrace ring buffer and can be read via the `trace` file.

Writing into the ring buffer with `trace_printk()` only takes around a tenth of a microsecond or so. But using `printk()`, especially when writing to the serial console, may take several milliseconds per write. The performance advantage of `trace_printk()` lets you record the most sensitive areas of the kernel with very little impact. 

For example you can add something like this to the kernel or module:

```
trace_printk("read foo %d out of bar %p\n", bar->foo, bar);
```

Then by looking at the `trace` file, you can see your output.

```
[tracing]# cat trace
        # tracer: nop
        #
        #           TASK-PID    CPU#    TIMESTAMP  FUNCTION
        #              | |       |          |         |
                   <...>-10690 [003] 17279.332920: : read foo 10 out of bar ffff880013a5bef8
```

The above example was done by adding a module that actually had a `foo` and `bar` construct.

`trace_printk()` output will appear in any tracer, even the function and function graph tracers.

```
[tracing]# echo function_graph > current_tracer
        [tracing]# insmod ~/modules/foo.ko
        [tracing]# cat trace
        # tracer: function_graph
        #
        # CPU  DURATION                  FUNCTION CALLS
        # |     |   |                     |   |   |   |
         3) + 16.283 us   |      }
         3) + 17.364 us   |    }
         3)               |    do_one_initcall() {
         3)               |      /* read foo 10 out of bar ffff88001191bef8 */
         3)   4.221 us    |    }
         3)               |    __wake_up() {
         3)   0.633 us    |      _spin_lock_irqsave();
         3)   0.538 us    |      __wake_up_common();
         3)   0.563 us    |      _spin_unlock_irqrestore();
```

Yes, the `trace_printk()` output looks like a comment in the function graph tracer.

#### Starting and stopping the trace

Obviously there are times where you only want to trace a particular code path. Perhaps you only want to trace what is happening when you run a specific test. The file `tracing_on` is used to disable the ring buffer from recording data:

```
[tracing]# echo 0 > tracing_on
```

This will disable the Ftrace ring buffer from recording. Everything else still happens with the tracers and they will still incur most of their overhead. They do notice that the ring buffer is not recording and will not attempt to write any data, but the calls that the tracers make are still performed.

To re-enable the ring buffer, simply write a '1' into that file:

```
[tracing]# echo 1 > tracing_on
```

Note, it is very important that you have a space between the number and the greater than sign "`>`". Otherwise you may be writing standard input or output into that file.

```
[tracing]# echo 0> tracing_on   /* this will not work! */
```

A common run might be:

```
[tracing]# echo 0 > tracing_on
        [tracing]# echo function_graph > current_tracer
        [tracing]# echo 1 > tracing_on; run_test; echo 0 > tracing_on
```

The first line disables the ring buffer from recording any data. The next enables the function graph tracer. The overhead of the function graph tracer is still present but nothing will be recorded into the trace buffer. The last line enables the ring buffer, runs the test program, then disables the ring buffer. This narrows the data stored by the function graph tracer to include mostly just the data accumulated by the `run_test` program.

#### What's next?

The next article will continue the discussion on debugging the kernel with Ftrace. The method above to disable the tracing may not be fast enough. The latency between the end of the program `run_test` and echoing the 0 into the `tracing_on` file may cause the ring buffer to overflow and lose the relevant data. I will discuss other methods to stop tracing a bit more efficiently, how to debug a crash, and looking at what functions in the kernel are stack hogs. The best way to find out more is to enable Ftrace and just play with it. You can learn a lot about how the kernel works by just following the function graph tracer.
