---
title: "Debugging the kernel using Ftrace - part 2"
url: https://lwn.net/Articles/366796/
date: "December 22, 2009"
category: "Development tools-Kernel tracing; Ftrace; Tracing-Ftrace"
author: "December 22, 2009 This article was contributed by Steven Rostedt"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

December 22, 2009

This article was contributed by Steven Rostedt

The Ftrace tracing utility has many different features that will assist in tracking down Linux kernel problems. The [previous article](<http://lwn.net/Articles/365835/>) discussed setting up Ftrace, using the function and function graph tracers, using `trace_printk()`, and a simple way to stop the recording of a trace from user space. This installment will touch on how user space can interact with Ftrace, faster ways of stopping the trace, debugging a crash, and finding what kernel functions are the biggest stack hogs.

#### Trace Markers

Seeing what happens inside the kernel gives the user a better understanding of how their system works. But sometimes there needs to be coordination between what is happening in user space and what is happening inside the kernel. The timestamps that are shown in the traces are all relative to what is happening within the trace, but they do not correspond well with wall time.

To help synchronize between the actions in user space and kernel space, the `trace_marker` file was created. It provides a way to write into the Ftrace ring buffer from user space. This marker will then appear in the trace to give a location in the trace of where a specific event occurred.

```
[tracing]# echo hello world > trace_marker
        [tracing]# cat trace
        # tracer: nop
        #
        #           TASK-PID    CPU#    TIMESTAMP  FUNCTION
        #              | |       |          |         |
                   <...>-3718  [001]  5546.183420: 0: hello world
```

The `<...>` indicates that the name of the task that wrote the marker was not recorded. Future releases may fix this. 

#### Starting, Stopping and Recording in a Program

The `tracing_on` and `trace_marker` files work very well to trace the activities of an application if the source of the application is available. If there is a problem within the application and you need to find out what is happening inside the kernel at a particular location of the application, these two files come in handy.

At the start of the application, you can open these files to have the file descriptors ready:

```
int trace_fd = -1;
        int marker_fd = -1;
    
        int main(int argc, char **argv)
        {
    	    char *debugfs;
    	    char path[256];
    	    [...]
    
    	    debugfs = find_debugfs();
    	    if (debugfs) {
    		    strcpy(path, debugfs);  /* BEWARE buffer overflow */
    		    strcat(path,"/tracing/tracing_on");
    		    trace_fd = open(path, O_WRONLY);
    		    if (trace_fd >= 0)
    			    write(trace_fd, "1", 1);
    
    		    strcpy(path, debugfs);
    		    strcat(path,"/tracing/trace_marker");
    		    marker_fd = open(path, O_WRONLY);
```

Then, at some critical location in the code, markers can be placed to show where the application currently is: 

```
if (marker_fd >= 0)
    	    write(marker_fd, "In critical area\n", 17);
    
        if (critical_function() < 0) {
    	    /* we failed! */
    	    if (trace_fd >= 0)
    		    write(trace_fd, "0", 1);
        }
```

In looking at the example, first you see a function called "find_debugfs()". The proper location to mount the debug file system is at `/sys/kernel/debug` but a robust tool should not depend on the debug file system being mounted there. An example of `find_debugfs()` is located [here](<http://lwn.net/Articles/366800/>). The file descriptors are initialized to -1 to allow this code to work both with and without a tracing enabled kernel.

When the problem is detected, writing the ASCII character "0" into the `trace_fd` file descriptor stops tracing. As discussed in part 1, this only disables the recording into the Ftrace ring buffer, but the tracers are still incurring overhead.

When using the initialization code above, tracing will be enabled at the beginning of the application because the tracer runs in overwrite mode. That is, when the trace buffer fills up, it will remove the old data and replace it with the new. Since only the most recent trace information is relevant when the problem occurs there is no need to stop and start the tracing during the normal running of the application. The tracer only needs to be disabled when the problem is detected so the trace will have the history of what led up to the error. If interval tracing is needed within the application, it can write an ASCII "1" into the trace_fd to enable the tracing. 

Here is an example of a simple program called `simple_trace.c` that uses the initialization process described above:

```
req.tv_sec = 0;
        req.tv_nsec = 1000;
        write(marker_fd, "before nano\n", 12);
        nanosleep(&req, NULL);
        write(marker_fd, "after nano\n", 11);
        write(trace_fd, "0", 1);
```

(No error checking was added due to this being a simple program for example purposes only.)

Here is the process to trace this simple program:

```
[tracing]# echo 0 > tracing_on
        [tracing]# echo function_graph > current_tracer
        [tracing]# ~/simple_trace
        [tracing]# cat trace
```

The first line disables tracing because the program will enable it at start up. Next the function graph tracer is selected. The program is executed, which results in the following trace. Note that the output can be a little verbose so much of it has been cut and replaced with `[...]`:

```
[...]
         0)               |      __kmalloc() {
         0)   0.528 us    |        get_slab();
         0)   2.271 us    |      }
         0)               |      /* before nano */
         0)               |      kfree() {
         0)   0.475 us    |        __phys_addr();
         0)   2.062 us    |      }
         0)   0.608 us    |      inotify_inode_queue_event();
         0)   0.485 us    |      __fsnotify_parent();
        [...]
         1)   0.523 us    |          _spin_unlock();
         0)   0.495 us    |    current_kernel_time();
         1)               |          it_real_fn() {
         0)   1.602 us    |  }
         1)   0.728 us    |            __rcu_read_lock();
         0)               |  sys_nanosleep() {
         0)               |    hrtimer_nanosleep() {
         0)   0.526 us    |      hrtimer_init();
         1)   0.418 us    |            __rcu_read_lock();
         0)               |      do_nanosleep() {
         1)   1.114 us    |            _spin_lock_irqsave();
        [...]
         0)               |      __kmalloc() {
         1)   2.760 us    |  }
         0)   0.556 us    |        get_slab();
         1)               |  mwait_idle() {
         0)   1.851 us    |      }
         0)               |      /* after nano */
         0)               |      kfree() {
         0)   0.486 us    |        __phys_addr();
```

Notice that the writes to `trace_marker` show up as comments in the function graph tracer.

The first column here represents the CPU. When we have the CPU traces interleaved like this, it may become hard to read the trace. The tool `grep` can easily filter this, or the `per_cpu` trace files may be used. The `per_cpu` trace files are located in the debugfs tracing directory under `per_cpu`.

```
[tracing]# ls per_cpu
        cpu0  cpu1  cpu2  cpu3  cpu4  cpu5  cpu6  cpu7
```

There exists a trace file in each one of these CPU directories that only show the trace for that CPU.

To get a nice view of the function graph tracer without the interference of other CPUs just look at `per_cpu/cpu0/trace`.

```
[tracing]# cat per_cpu/cpu0/trace
         0)               |      __kmalloc() {
         0)   0.528 us    |        get_slab();
         0)   2.271 us    |      }
         0)               |      /* before nano */
         0)               |      kfree() {
         0)   0.475 us    |        __phys_addr();
         0)   2.062 us    |      }
         0)   0.608 us    |      inotify_inode_queue_event();
         0)   0.485 us    |      __fsnotify_parent();
         0)   0.488 us    |      inotify_dentry_parent_queue_event();
         0)   1.106 us    |      fsnotify();
        [...]
         0)   0.721 us    |    _spin_unlock_irqrestore();
         0)   3.380 us    |  }
         0)               |  audit_syscall_entry() {
         0)   0.495 us    |    current_kernel_time();
         0)   1.602 us    |  }
         0)               |  sys_nanosleep() {
         0)               |    hrtimer_nanosleep() {
         0)   0.526 us    |      hrtimer_init();
         0)               |      do_nanosleep() {
         0)               |        hrtimer_start_range_ns() {
         0)               |          __hrtimer_start_range_ns() {
         0)               |            lock_hrtimer_base() {
         0)   0.866 us    |              _spin_lock_irqsave();
        [...]
         0)               |      __kmalloc() {
         0)               |        get_slab() {
         0)   1.851 us    |      }
         0)               |      /* after nano */
         0)               |      kfree() {
         0)   0.486 us    |        __phys_addr();
```

#### Disabling the Tracer Within the Kernel

During the development of a kernel driver there may exist strange errors that occur during testing. Perhaps the driver gets stuck in a sleep state and never wakes up. Trying to disable the tracer from user space when a kernel event occurs is difficult and usually results in a buffer overflow and loss of the relevant information before the user can stop the trace.

There are two functions that work well inside the kernel: `tracing_on()` and `tracing_off()`. These two act just like echoing "1" or "0" respectively into the `tracing_on` file. If there is some condition that can be checked for inside the kernel, then the tracer may be stopped by adding something like the following: 

```
if (test_for_error())
    	    tracing_off();
```

Next, add several `trace_printk()`s (see part 1), recompile, and boot the kernel. You can then enable the function or function graph tracer and just wait for the error condition to happen. Examining the `tracing_on` file will let you know when the error condition occurred. It will switch from "1" to "0" when the kernel calls `tracing_off()`.

After examining the trace, or saving it off in another file with: 

```
cat trace > ~/trace.sav
```

you can continue the trace to examine another hit. To do so, just echo "1" into `tracing_on`, and the trace will continue. This is also useful if the condition that triggers the `tracing_off()` call can be triggered legitimately. If the condition was triggered by normal operation, just restart the trace by echoing a "1" back into `tracing_on` and hopefully the next time the condition is hit will be because of the abnormality.

#### ftrace_dump_on_oops

There are times that the kernel will crash and examining the memory and state of the crash is more of a CSI science than a program debugging science. Using `kdump`/`kexec` with the `crash` utility is a valuable way to examine the state of the system at the point of the crash, but it does not let you see what has happened prior to the event that caused the crash.

Having Ftrace configured and enabling `ftrace_dump_on_oops` in the kernel boot parameters, or by echoing a "1" into `/proc/sys/kernel/ftrace_dump_on_oops`, will enable Ftrace to dump to the console the entire trace buffer in ASCII format on oops or panic. Having the console output to a serial log makes debugging crashes much easier. You can now trace back the events that led up to the crash.

Dumping to the console may take a long time since the default Ftrace ring buffer is over a megabyte per CPU. To shrink the size of the ring buffer, write the number of kilobytes you want the ring buffer to be to `buffer_size_kb`. Note that the value is per CPU, not the total size of the ring buffer. 

```
[tracing]# echo 50 > buffer_size_kb
```

The above will shrink the Ftrace ring buffer down to 50 kilobytes per CPU. 

You can also trigger a dump of the Ftrace buffer to the console with `sysrq-z`.

To choose a particular location for the kernel dump, the kernel may call `ftrace_dump()` directly. Note, this may permanently disable Ftrace and a reboot may be necessary to enable it again. This is because `ftrace_dump()` reads the buffer. The buffer is made to be written to in all contexts (interrupt, NMI, scheduling) but the reading of the buffer requires locking. To be able to perform `ftrace_dump()` the locking is disabled and the buffer may end up being corrupted after the output.

```
/*
         * The following code will lock up the box, so we dump out the
         * trace before we hit that location.
         */
        ftrace_dump();
    
        /* code that locks up */
```

#### Stack Tracing

The final topic to discuss is the ability to examine the size of the kernel stack and how much stack space each function is using. Enabling the stack tracer (`CONFIG_STACK_TRACER`) will show where the biggest use of the stack takes place.

The stack tracer is built from the function tracer infrastructure. It does not use the Ftrace ring buffer, but it does use the function tracer to hook into every function call. Because it uses the function tracer infrastructure, it does not add overhead when not enabled. To enable the stack tracer, echo 1 into `/proc/sys/kernel/stack_tracer_enabled`. To see the max stack size during boot up, add "`stacktrace`" to the kernel boot parameters.

The stack tracer checks the size of the stack at every function call. If it is greater than the last recorded maximum, it records the stack trace and updates the maximum with the new size. To see the current maximum, look at the `stack_max_size` file. 

```
[tracing]# echo 1 > /proc/sys/kernel/stack_tracer_enabled
        [tracing]# cat stack_max_size
        2928
        [tracing]# cat stack_trace
                Depth    Size   Location    (34 entries)
                -----    ----   --------
          0)     2952      16   mempool_alloc_slab+0x15/0x17
          1)     2936     144   mempool_alloc+0x52/0x104
          2)     2792      16   scsi_sg_alloc+0x4a/0x4c [scsi_mod]
          3)     2776     112   __sg_alloc_table+0x62/0x103
        [...]
         13)     2072      48   __elv_add_request+0x98/0x9f
         14)     2024     112   __make_request+0x43e/0x4bb
         15)     1912     224   generic_make_request+0x424/0x471
         16)     1688      80   submit_bio+0x108/0x115
         17)     1608      48   submit_bh+0xfc/0x11e
         18)     1560     112   __block_write_full_page+0x1ee/0x2e8
         19)     1448      80   block_write_full_page_endio+0xff/0x10e
         20)     1368      16   block_write_full_page+0x15/0x17
         21)     1352      16   blkdev_writepage+0x18/0x1a
         22)     1336      32   __writepage+0x1a/0x40
         23)     1304     304   write_cache_pages+0x241/0x3c1
         24)     1000      16   generic_writepages+0x27/0x29
        [...]
         30)      424      64   bdi_writeback_task+0x3f/0xb0
         31)      360      48   bdi_start_fn+0x76/0xd7
         32)      312     128   kthread+0x7f/0x87
         33)      184     184   child_rip+0xa/0x20
```

Not only does this give you the size of the maximum stack found, it also shows the breakdown of the stack sizes used by each function. Notice that `write_cache_pages` had the biggest stack with 304 bytes being used, followed by `generic_make_request` with 224 bytes of stack.

To reset the maximum, echo "0" into the `stack_max_size` file.

```
[tracing]# echo 0 > stack_max_size
```

Keeping this running for a while will show where the kernel is using a bit too much stack. But remember that the stack tracer only has no overhead when it is not enabled. When it is running you may notice a bit of a performance degradation.

Note that the stack tracer will not trace the max stack size when the kernel is using a separate stack. Because interrupts have their own stack, it will not trace the stack usage there. The reason is that currently there is no easy way to quickly see what the top of the stack is when the stack is something other than the current task's stack. When using split stacks, a process stack may be two pages but the interrupt stack may only be one. This may be fixed in the future, but keep this in mind when using the stack tracer.

#### Conclusion

Ftrace is a very powerful tool and easy to configure. No extra tools are necessary. Everything that was shown it this tutorial can be used on embedded devices that only have Busybox installed. Taking advantage of the Ftrace infrastructure should cut the time needed to debug that hard-to-find race condition. I seldom use `printk()` any more because using the function and function graph tracers along with `trace_printk()` and `tracing_off()` have become my main tools for debugging the Linux kernel.
