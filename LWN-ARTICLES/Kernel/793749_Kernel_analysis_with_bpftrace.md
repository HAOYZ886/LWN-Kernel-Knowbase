---
title: Kernel analysis with bpftrace
url: https://lwn.net/Articles/793749/
date: "July 18, 2019"
category: "BPF-Tracing; Tracing-with BPF"
author: "July 18, 2019 This article was contributed by Brendan Gregg"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

July 18, 2019

This article was contributed by [Brendan Gregg](<http://www.brendangregg.com>)

At the [2019 Linux Storage, Filesystem, and Memory-Management Summit](<https://lwn.net/Articles/lsfmm2019/>) (LSFMM) I gave a keynote on [BPF observability](<https://lwn.net/Articles/787131/>) that included a kernel issue I had debugged on Netflix production servers using [`bpftrace`](<https://github.com/iovisor/bpftrace>). In this article I'll provide a crash course on `bpftrace` for kernel developers—to help them more easily analyze their code.

I was recently discussing `tcp_sendmsg()` with another developer who was concerned about large message sizes (such as 100 megabytes) causing failures. 100 MB?? I doubt Netflix is sending messages anywhere near that large in production. Many of you may know the function prototype well, it is (from [`net/ipv4/tcp.c`](<https://elixir.bootlin.com/linux/latest/source/net/ipv4/tcp.c#L1415>)):

```
int tcp_sendmsg(struct sock *sk, struct msghdr *msg, size_t size);
```

`bpftrace` is already installed on Netflix (and other companies) production servers, so I ssh-ed to a busy server to check the size distribution of messages for ten seconds:

```
# **bpftrace -e 'k:tcp_sendmsg { @size = hist(arg2); } interval:s:10 { exit(); }'**
        Attaching 2 probes...
        
        @size: 
        [4, 8)                25 |                                                    |
        [8, 16)               74 |                                                    |
        [16, 32)               5 |                                                    |
        [32, 64)           13603 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
        [64, 128)           2527 |@@@@@@@@@                                           |
        [128, 256)            21 |                                                    |
        [256, 512)           181 |                                                    |
        [512, 1K)           1587 |@@@@@@                                              |
        [1K, 2K)            2992 |@@@@@@@@@@@                                         |
        [2K, 4K)            9367 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@                 |
        [4K, 8K)           12461 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@     |
        [8K, 16K)            995 |@@@                                                 |
        [16K, 32K)          1977 |@@@@@@@                                             |
        [32K, 64K)           428 |@                                                   |
        [64K, 128K)           14 |                                                    |
        [128K, 256K)           2 |                                                    |
```

The largest is in the 128 to 256KB range. This is using a kprobe ("`k`") on `tcp_sendmsg()`, and saving a histogram of `arg2` (`size`) to a BPF map named "`@size`" (the name is unimportant, I'm just using it to annotate the output). An interval event fires after 10 seconds and exits, at which point all BPF maps are printed.

What about errors?:

```
# **bpftrace -e 'kr:tcp_sendmsg { @retvals[retval > 0 ? 0 : retval] = count(); }
            interval:s:10 { exit(); }'**
        Attaching 2 probes...
        
        @retvals[0]: 49794
```

No errors. This time I am using a kretprobe ("`kr`"), and I'm frequency counting `retval`, which is either a negative error code or the size. Since I don't care about the size, I used a ternary operator to set all positive values to zero.

`bpftrace` lets you answer these questions immediately, in production, and without needing debuginfo (Netflix does not typically have it installed). For this case, it showed that Netflix was unlikely to see the [large `tcp_sendmsg()` problem](<https://github.com/iovisor/bcc/issues/2440>) for this workload. 

#### Other one-liners

The previous `tcp_sendmsg()` one-liners demonstrate some `bpftrace` kernel analysis capabilities. To show you some more capabilities, here are several more one-liners that also use `tcp_sendmsg()` as the example. You can imagine switching it to other kernel functions.

Show `tcp_sendmsg()` sizes bigger than 8192 bytes, per-event:

```
bpftrace -e 'k:tcp_sendmsg /arg2 > 8192/ { printf("PID %d: %d bytes\n", pid, arg2); }'
```

Show a histogram of request size for each process (PID and comm):

```
bpftrace -e 'k:tcp_sendmsg { @size[pid, comm] = hist(arg2); }'
```

Frequency count return values:

```
bpftrace -e 'kr:tcp_sendmsg { @return[retval] = count(); }'
```

Show per-second statistics: event count, average size, and total bytes:

```
bpftrace -e 'k:tcp_sendmsg { @size = stats(arg2); }
            interval:s:1 { print(@size); clear(@size); }'
```

Count calling stack traces:

```
bpftrace -e 'k:tcp_sendmsg { @[kstack] = count(); }'
```

Count calling stack traces, three levels deep:

```
bpftrace -e 'k:tcp_sendmsg { @[kstack(3)] = count(); }'
```

Show function latency as a histogram, in nanoseconds:

```
bpftrace -e 'k:tcp_sendmsg { @ts[tid] = nsecs; }
            kr:tcp_sendmsg /@ts[tid]/ { @ns = hist(nsecs - @ts[tid]); delete(@ts[tid]); }'
```

This last example is saving a timestamp in one probe (keyed on thread ID) and retrieving it in another. This pattern can be used for custom latency measurements.

#### `struct` data

There is an important capability missing from those one-liners: `struct` navigation. Here is the function prototype again:

```
int tcp_sendmsg(struct sock *sk, struct msghdr *msg, size_t size);
```

`bpftrace` provides `arg0`-`argN` for kprobe function arguments, simply mapping them to the registers for the calling convention (`arg2` becomes `%rdx` on x86_64, for example). Since `bpftrace` can read kernel headers, which are often installed on production systems, accessing `struct` data is possible by including the right header and casting the arguments:

```
#include <net/sock.h>
        [...]
                $sk = (struct sock *)arg0;
```

Here's an example of a `bpftrace` tool that prints the address information, size, and return value from `tcp_sendmsg()`. Example output:

```
# **./tcp_sendmsg.bt**
        Attaching 2 probes...
        10.0.0.65       49978 -> 52.37.243.173   443  : 63 bytes, retval 63
        127.0.0.1       58566 -> 127.0.0.1       22   : 36 bytes, retval 36
        127.0.0.1       22    -> 127.0.0.1       58566: 36 bytes, retval 36
        [...]
```

The source of `tcp_sendmsg.bt`:

```
#!/usr/local/bin/bpftrace
        
        #include <net/sock.h>
        
        k:tcp_sendmsg
        {
    	    @sk[tid] = arg0;
    	    @size[tid] = arg2;
        }
        
        kr:tcp_sendmsg
        /@sk[tid]/
        {
    	    $sk = (struct sock *)@sk[tid];
    	    $size = @size[tid];
    	    $af = $sk->__sk_common.skc_family;
    	    if ($af == AF_INET) {
    		    $daddr = ntop($af, $sk->__sk_common.skc_daddr);
    		    $saddr = ntop($af, $sk->__sk_common.skc_rcv_saddr);
    		    $lport = $sk->__sk_common.skc_num;
    		    $dport = $sk->__sk_common.skc_dport;
    		    $dport = ($dport >> 8) | (($dport << 8) & 0xff00);
    		    printf("%-15s %-5d -> %-15s %-5d: %d bytes, retval %d\n",
    		        $saddr, $lport, $daddr, $dport, $size, retval);
    	    } else {
    		    printf("IPv6...\n");
    	    }
    	    delete(@sk[tid]);
    	    delete(@size[tid]);
        }
```

In the kprobe, `sk` and `size` are saved in per-thread-ID maps, so they can be retrieved in the kretprobe when `tcp_sendmsg()` returns. The kretprobe casts `sk` and prints out details, if it is an IPv4 message, using the `bpftrace` function `ntop()` to convert the address to a string. The destination port is flipped from network to host order. To keep this short I skipped IPv6, but you can add code to handle it too (`ntop()` does support IPv6 addresses).

There is work underway for `bpftrace` to use [BPF Type Format](<https://www.kernel.org/doc/html/latest/bpf/btf.html>) (BTF) information as well, which brings various advantages including `struct` definitions that are missing from kernel headers.

#### Advanced example

So far I've shown straightforward tracing capabilities. For an advanced example, I'll demonstrate off-CPU profiling.

CPU profiling of a task is typically easy: I can sample stacks, inspect performance-monitoring counters (PMCs) and model-specific registers (MSRs) to see what was running on-CPU and why it was slow. Off-CPU profiling, on the other hand, has been problematic. I can show blocking stacks on context switches, but they often show that the task is blocked on something else (`select()`, `epoll()`, or a lock.) I need to know what that something else is.

BPF has provided a solution at long last to this problem: the ability to save stack traces and retrieve them later (something I've always wanted DTrace to do, but it cannot). Here is a `bpftrace` version of the solution, which shows the process name, blocking stack, waker stack, and a blocking-time histogram in microseconds:

```
# **./offwake.bt**
       Attaching 4 probes...
       Tracing off-CPU time (us) with waker stacks. Ctrl-C to end.
       ^C
       [...]
       
       @us[ssh,
           finish_task_switch+1
           schedule+44
           schedule_hrtimeout_range_clock+185
           schedule_hrtimeout_range+19
           poll_schedule_timeout+69
           do_select+1378
           core_sys_select+471
           sys_select+183
           do_syscall_64+115
           entry_SYSCALL_64_after_hwframe+61
       ,
           try_to_wake_up+1
           pollwake+115
           __wake_up_common+115
           __wake_up_common_lock+128
           __wake_up_sync_key+30
           sock_def_readable+64
           tcp_rcv_established+1281
           tcp_v4_do_rcv+144
           tcp_v4_rcv+2423
           ip_local_deliver_finish+98
           ip_local_deliver+111
           ip_rcv_finish+297
           ip_rcv+655
           __netif_receive_skb_core+1074
           __netif_receive_skb+24
           netif_receive_skb_internal+69
           napi_gro_receive+197
           ieee80211_deliver_skb+200
           ieee80211_rx_handlers+5376
           ieee80211_prepare_and_rx_handle+1865
           ieee80211_rx_napi+914
           iwl_mvm_rx_rx_mpdu+1205
           iwl_mvm_rx+77
           iwl_pcie_rx_handle+571
           iwl_pcie_irq_handler+1577
           irq_thread_fn+38
           irq_thread+325
           kthread+289
           ret_from_fork+53
       ]:
       [64, 128)              1 |@@                                                  |
       [128, 256)             1 |@@                                                  |
       [256, 512)             1 |@@                                                  |
       [512, 1K)              1 |@@                                                  |
       [1K, 2K)               4 |@@@@@@@@@@                                          |
       [2K, 4K)              10 |@@@@@@@@@@@@@@@@@@@@@@@@@@@                         |
       [4K, 8K)              18 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@   |
       [8K, 16K)              3 |@@@@@@@@                                            |
       [16K, 32K)            16 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@         |
       [32K, 64K)            19 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
       [64K, 128K)            3 |@@@@@@@@                                            |
```

I've included one stack pair in this output. This shows an ssh process with a blocking stack in `select()`. The waker stack reveals what it was waiting for: a network packet. The histogram shows the durations for this offcpu/waker path, in microseconds.

The source to `offwake.bt` is:

```
#!/usr/local/bin/bpftrace
       
       #include <linux/sched.h>
       
       BEGIN
       {
    	   printf("Tracing off-CPU time (us) with waker stacks. Ctrl-C to end.\n");
       }
       
       kprobe:try_to_wake_up
       {
    	   $p = (struct task_struct *)arg0;
    	   @waker[$p->pid] = kstack;
       }
       
       kprobe:finish_task_switch
       {
    	   $prev = (struct task_struct *)arg0;
       
    	   // record timestamp of sleeping task:
    	   @ts[$prev->pid] = nsecs;
       
    	   if (@ts[tid]) {
    		   $offcpu_us = (nsecs - @ts[tid]) / 1000;
    		   @us[comm, kstack, @waker[tid]] = hist($offcpu_us);
    		   delete(@ts[tid]);
    		   delete(@waker[tid]);
    	   }
       }
       
       END
       {
    	   clear(@waker);
    	   clear(@ts);
       }
```

The kernel stack on `try_to_wake_up()` is saved with the task ID, which is later retrieved on `finish_task_switch()`. This is a simple `bpftrace` version of the [`offwaketime`](<https://github.com/iovisor/bcc/blob/master/tools/offwaketime.py>) tool from BCC, and the `samples/bpf/offwaketime*` examples in the kernel source.

I explained this problem and the BPF solution in my Performance@Scale talk ([slides [Slideshare]](<http://www.slideshare.net/brendangregg/linux-bpf-superpowers>), [video](<https://www.facebook.com/atscaleevents/videos/1693888610884236/>)), showing how these stack traces can be [visualized as flame graphs](<http://www.brendangregg.com/FlameGraphs/offcpuflamegraphs.html>). Sometimes you need to show who woke the waker, and who woke them, and so on. By walking the wakeup chain, I can construct a "chain graph" to show the original source of the latency (usually interrupts).

#### Tracepoints

The previous examples used kprobes, and these will need updating as the kernel changes. Tracepoints are much preferable to kprobes for `bpftrace` programs, even though they are "best effort" and might change. That's still better than kprobes, which can change anytime, or show up as missing because they were inlined. However, some kprobes are more stable than others, particularly those for internal kernel interfaces, such as VFS, `struct file_operations`, `struct proto`, etc.

Just as a simple example, here are the arguments provided by the `timer:hrtimer_start` tracepoint:

```
# **bpftrace -lv 't:timer:hrtimer_start'**
       tracepoint:timer:hrtimer_start
           void * hrtimer;
           void * function;
           s64 expires;
           s64 softexpires;
```

Frequency counting the function argument:

```
# **bpftrace -e 't:timer:hrtimer_start { @[ksym(args- >function)] = count(); }'**
       Attaching 1 probe...
       ^C
       
       @[sched_rt_period_timer]: 6
       @[watchdog_timer_fn]: 16
       @[intel_uncore_fw_release_timer]: 290
       @[it_real_fn]: 376
       @[hrtimer_wakeup]: 12301
       @[tick_sched_timer]: 36433
```

The tracepoint arguments are accessible via `args`. In this case, I used the `ksym()` `bpftrace` function to return the kernel symbol name for the address.

#### More examples and information

I stepped through a Netfilx production example in my LSFMM keynote ([slides [Slideshare]](<https://www.slideshare.net/brendangregg/lsfmm-2019-bpf-observability-143092820>)). More examples of programs can be found in the `bpftrace` repository under [tools](<https://github.com/iovisor/bpftrace/tree/master/tools>). At LSFMM, I previewed more tools I had developed for an upcoming BPF analysis book for Addison-Wesley; they will be made available on the [BPF book page](<http://www.brendangregg.com/bpf-performance-tools-book.html>).

To try `bpftrace` yourself, see the [INSTALL](<https://github.com/iovisor/bpftrace/blob/master/INSTALL.md>) guide, and get the latest version: 0.9.1 or newer. There are packages for different distributions, companies including Netflix and Facebook have their own internal package, or you can build from source. `bpftrace` currently uses LLVM and Clang for dependencies (as does [BCC](<https://github.com/iovisor/BCC>)), although a future version may make them optional.

Also see the [`bpftrace` Cheat Sheet](<http://www.brendangregg.com/BPF/bpftrace-cheat-sheet.html>) for a summary of the language as well as the full [`bpftrace` Reference Guide](<https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md>).

#### What about perf, ftrace?

I use the best tool for the job, and many times that's not `bpftrace`: it is perf or ftrace. Examples:

  * Frequency counting many functions (e.g, `tcp_*`): ftrace, because it's fast to initialize. BPF kprobe attaching will become faster with multi-attach `perf_event_open()` once it is developed.
  * Function flow tracing: ftrace function graphing.
  * PMC statistics: perf.
  * CPU sampling with timestamps: perf, as its `perf.data` output has been optimized.

For the last one, sometimes I need timestamps on samples, but often I don't and can just use `bpftrace`; for example, sampling kernel stacks at 99 Hertz:

```
# **bpftrace -e 'profile:hz:99 { @[kstack] = count(); }'**
```

#### Conclusion

Times have changed for Linux. It now has an advanced tracer, `bpftrace`, built from the ground-up for extended BPF and Linux that is solving real production problems at Netflix and other companies. With simple one-liners or short tools you can inspect your own code in custom ways. This article showed lots of examples.

If you'd like to see how your code is running in Netflix production, email me your `bpftrace` program and I can reply with the output (so long as it isn't company/customer revealing). If it helps you discover some inefficiencies with how Linux is executing our workload, and you develop a fix, that's great for Netflix. I am bgregg@netflix.com.

[_Thanks to Alastair Robertson for creating` bpftrace`, and the `bpftrace`, BCC, and BPF communities for all the work over the past five years._]
