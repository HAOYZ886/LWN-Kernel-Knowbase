---
title: "Ftrace: The hidden light switch"
url: https://lwn.net/Articles/608497/
date: "August 13, 2014"
category: "Development tools-Kernel tracing; Ftrace; Tracing-Ftrace"
author: "August 13, 2014 This article was contributed by Brendan Gregg"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

August 13, 2014

This article was contributed by [Brendan Gregg](<http://www.brendangregg.com>)

You may think, as I did, that analyzing the Linux kernel is like venturing through a dark dungeon: without the addition of advanced tracers like SystemTap, there's much that can't be seen, and can only be inferred. However, I've recently found hidden switches that turn on some bright lights, strategically placed by Steven Rostedt and others since the 2.6.27 release. These are the ftrace profilers. I haven't even tried all the switches yet, but I'm stunned at what I've seen so far, and I'm having to rethink what I previously believed about Linux kernel performance analysis.

Recently at Netflix (where I work), a Cassandra database was performing poorly after a system upgrade, and disk I/O inflation (a massive increase in the number of I/O operations submitted) was suspected. There can be many causes for this: a worse cache-hit ratio, record-size inflation, readahead inflation, other applications, even other asynchronous kernel tasks (file system background scrubs). The question was: which one, and how do we fix it?

#### 1\. iosnoop

I started with basic server health checks, and then my [`iosnoop`](<http://www.brendangregg.com/blog/2014-07-16/iosnoop-for-linux.html>) tool. `iosnoop` is a shell script that uses the `/sys/kernel/debug` ftrace facilities, and is in my [perf-tools](<https://github.com/brendangregg/perf-tools>) collection on GitHub, along with other ftrace hacks. These work on Netflix's existing servers (which often run Linux 3.2 with security patches) without any other addition to the system, and without requiring kernel debuginfo. In this case, `iosnoop` was run both with and without `-Q` to see the effect of queuing:

```
# **./iosnoop -ts**
      STARTs          ENDs            COMM         PID    TYPE DEV      BLOCK        BYTES     LATms
      13370264.614265 13370264.614844 java         8248   R    202,32   1431244248   45056      0.58
      13370264.614269 13370264.614852 java         8248   R    202,32   1431244336   45056      0.58
      13370264.614271 13370264.614857 java         8248   R    202,32   1431244424   45056      0.59
      13370264.614273 13370264.614868 java         8248   R    202,32   1431244512   45056      0.59
      [...]
      # **./iosnoop -Qts**
      STARTs          ENDs            COMM         PID    TYPE DEV      BLOCK        BYTES     LATms
      13370410.927331 13370410.931182 java         8248   R    202,32   1596381840   45056      3.85
      13370410.927332 13370410.931200 java         8248   R    202,32   1596381928   45056      3.87
      13370410.927332 13370410.931215 java         8248   R    202,32   1596382016   45056      3.88
      13370410.927332 13370410.931226 java         8248   R    202,32   1596382104   45056      3.89
      [...]
```

I didn't see anything out of the ordinary: a higher disk I/O load was causing higher queue times.

The tools that follow are all from the same collection: they demonstrate existing capabilities of ftrace, and how they are useful for solving real problems.

#### 2\. tpoint

To investigate these disk reads in more detail, I used `tpoint` to trace the `block:block_rq_insert` tracepoint:

```
# **./tpoint -H block:block_rq_insert**
      Tracing block:block_rq_insert. Ctrl-C to end.
      # tracer: nop
      #
      #       TASK-PID    CPU#    TIMESTAMP  FUNCTION
      #          | |       |          |         |
              java-16035 [000] 13371565.253582: block_rq_insert: 202,16 WS 0 () 550505336 + 88 [java]
              java-16035 [000] 13371565.253582: block_rq_insert: 202,16 WS 0 () 550505424 + 56 [java]
              java-8248  [007] 13371565.278372: block_rq_insert: 202,32 R 0 () 660621368 + 88 [java]
              java-8248  [007] 13371565.278373: block_rq_insert: 202,32 R 0 () 660621456 + 88 [java]
              java-8248  [007] 13371565.278374: block_rq_insert: 202,32 R 0 () 660621544 + 24 [java]
              java-8249  [007] 13371565.311507: block_rq_insert: 202,32 R 0 () 660666416 + 88 [java]
      [...]
```

I was checking for anything obviously odd, but the I/O details looked normal. The `-H` option prints column headers.

Next, I traced the code path that led to this I/O by printing stack traces (`-s`), to see if they contained an explanation. I also added an in-kernel filter to match reads only (when the "`rwbs`" flag field contains "`R`"):

```
# **./tpoint -s block:block_rq_insert 'rwbs ~ "*R*"' | head -1000**
      Tracing block:block_rq_insert. Ctrl-C to end.
              java-8248  [005] 13370789.973826: block_rq_insert: 202,16 R 0 () 1431480000 + 8 [java]
              java-8248  [005] 13370789.973831: <stack trace>
       => blk_flush_plug_list
       => blk_queue_bio
       => generic_make_request.part.50
       => generic_make_request
       => submit_bio
       => do_mpage_readpage
       => mpage_readpages
       => xfs_vm_readpages
       => read_pages
       => __do_page_cache_readahead
       => ra_submit
       => do_sync_mmap_readahead.isra.24
       => filemap_fault
       => __do_fault
       => handle_pte_fault
       => handle_mm_fault
       => do_page_fault
       => page_fault
              java-8248  [005] 13370789.973831: block_rq_insert: 202,16 R 0 () 1431480024 + 32 [java]
              java-8248  [005] 13370789.973836: <stack trace>
       => blk_flush_plug_list
       => blk_queue_bio
       => generic_make_request.part.50
      [...]
```

Great! The output is similar to the previous example, but with stack traces beneath each tracepoint event. I limited the output using `head` as it is verbose.

`tpoint` is another ftrace-based tool. It's usually better to use perf events (the `perf` command) for this particular use case, as it can handle multi-user access to performance data and a higher event rate, although it is more time-consuming to use. I just wanted to quickly eyeball a few dozen stack traces for a given tracepoint.

The stacks were mostly the same as the example above, which provided a couple of leads: page faults and readahead. This Ubuntu system was using 2MB direct-mapped pages, instead of 4KB like the old system. It also had readahead set to 2048KB, instead of 128KB. Either of these differences could be causing the inflation, although tuning readahead had already been tested, and found to make no difference.

#### 3\. funccount

I wanted to understand that stack trace better, so I started by counting calls using `funccount`, which uses ftrace function profiling. Starting with the per-second rate of `submit_bio()`:

```
# **./funccount -i 1 submit_bio**
      Tracing "submit_bio"... Ctrl-C to end.
    
      FUNC                              COUNT
      submit_bio                        27881
    
      FUNC                              COUNT
      submit_bio                        28478
      [...]
```

This rate, about 28,000 calls per second, is on par with what the disks are doing as seen from `iostat`. `funccount` is counting events in the kernel for efficiency.

Now checking the rate of `filemap_fault()`, which is closer in the stack to the database:

```
# **./funccount -i 1 filemap_fault**
      Tracing "filemap_fault"... Ctrl-C to end.
    
      FUNC                              COUNT
      filemap_fault                      2203
    
      FUNC                              COUNT
      filemap_fault                      3227
      [...]
```

This is consistent with what we believed the application was requesting from the filesystem. There is about a 10x inflation between these calls and the issued disk I/O (as evidenced by the `submit_bio()` calls).

#### 4\. funcslower

Just to confirm that the database is suffering latency caused by the stack trace I was studying, I used `funcslower` (another ftrace-based tool, which uses in-kernel timing and filtering for efficiency) to measure `filemap_fault()` calls taking longer than 1000 microseconds (1ms):

```
# **./funcslower -P filemap_fault 1000**
      Tracing "filemap_fault" slower than 1000 us... Ctrl-C to end.
       0)   java-8210    | ! 5133.499 us |  } /* filemap_fault */
       0)   java-8258    | ! 1120.600 us |  } /* filemap_fault */
       0)   java-8235    | ! 6526.470 us |  } /* filemap_fault */
       2)   java-8245    | ! 1458.30 us  |  } /* filemap_fault */
      [...]
```

These latencies look similar to those seen from disk I/O (with queue time). I'm in the right area.

#### 5\. funccount (again)

I noticed that the stack has "readpage" calls and then "readpage**s** ". Tracing them both at the same time:

```
# **./funccount -i 1 '*mpage_readpage*'**
      Tracing "*mpage_readpage*"... Ctrl-C to end.
    
      FUNC                              COUNT
      mpage_readpages                     364
      do_mpage_readpage                122930
    
      FUNC                              COUNT
      mpage_readpages                     318
      do_mpage_readpage                110344
      [...]
```

Here's our inflation: `mpage_readpages()` is being called about 300 times per second, and then `do_mpage_readpage()` over 100k times per second. This still looks like readahead, although we did try to adjust readahead sizes as an experiment, and it didn't make a difference.

#### 6\. kprobe

Maybe our readahead tuning didn't take effect? I can check this using dynamic tracing of kernel functions. Starting with the above stack trace, I saw that `__do_page_cache_readahead()` has `nr_to_read` (number of pages to read) as an argument, which comes from the readahead setting. Using `kprobe`, an ftrace- and kprobes-based tool, to dynamically trace this function and argument:

```
# **./kprobe -H 'p:do __do_page_cache_readahead nr_to_read=%cx'**
      Tracing kprobe m. Ctrl-C to end.
      # tracer: nop
      #
      #   TASK-PID    CPU#    TIMESTAMP  FUNCTION
      #      | |       |          |         |
          java-8714  [000] 13445354.703793: do: (__do_page_cache_readahead+0x0/0x180) nr_to_read=200
          java-8716  [002] 13445354.819645: do: (__do_page_cache_readahead+0x0/0x180) nr_to_read=200
          java-8734  [001] 13445354.820965: do: (__do_page_cache_readahead+0x0/0x180) nr_to_read=200
          java-8709  [000] 13445354.825280: do: (__do_page_cache_readahead+0x0/0x180) nr_to_read=200
      [...]
```

I used `-H` to print the header, and `p:` to specify that we will create a probe on function entry, which we'll call "do" (that alias is optional). The rest of that line specifies the function and optional arguments. Without kernel debuginfo, I can't refer to the `nr_to_read` symbol, so I need to use registers instead. I guessed %cx: If true, then our tuning hasn't taken hold, as 200 in hex is 512 pages: the original 2048KB.

#### 7\. funcgraph

To be sure, I read the code to see how this value is passed around, and used `funcgraph` to illustrate it:

```
# **./funcgraph -P filemap_fault | head -1000**
       2)   java-8248    |               |  filemap_fault() {
       2)   java-8248    |   0.568 us    |    find_get_page();
       2)   java-8248    |               |    do_sync_mmap_readahead.isra.24() {
       2)   java-8248    |   0.160 us    |      max_sane_readahead();
       2)   java-8248    |               |      ra_submit() {
       2)   java-8248    |               |        __do_page_cache_readahead() {
       2)   java-8248    |               |          __page_cache_alloc() {
       2)   java-8248    |               |            alloc_pages_current() {
       2)   java-8248    |   0.228 us    |              interleave_nodes();
       2)   java-8248    |               |              alloc_page_interleave() {
       2)   java-8248    |               |                __alloc_pages_nodemask() {
       2)   java-8248    |   0.105 us    |                  next_zones_zonelist();
       2)   java-8248    |               |                  get_page_from_freelist() {
       2)   java-8248    |   0.093 us    |                    next_zones_zonelist();
       2)   java-8248    |   0.101 us    |                    zone_watermark_ok();
       2)   java-8248    |               |                    zone_statistics() {
       2)   java-8248    |   0.073 us    |                      __inc_zone_state();
       2)   java-8248    |   0.074 us    |                      __inc_zone_state();
       2)   java-8248    |   1.209 us    |                    }
       2)   java-8248    |   0.142 us    |                    prep_new_page();
       2)   java-8248    |   3.582 us    |                  }
       2)   java-8248    |   4.810 us    |                }
       2)   java-8248    |   0.094 us    |                inc_zone_page_state();
      [...]
```

`funcgraph` uses another ftrace feature: the function graph profiler. It has moderate overhead, since it traces all kernel functions, so I only use it for exploratory purposes like this one-off. The output shows the code-flow in the kernel, and even has time deltas in microseconds. It's the call to `max_sane_readahead()` that is interesting, as that fetches the readahead value it wants to use.

#### 8\. kprobe (again)

This time I'll trace the return of the `max_sane_readahead()` function:

```
# **./kprobe 'r:m max_sane_readahead $retval'**
      Tracing kprobe m. Ctrl-C to end.
          java-8700  [000] 13445377.393895: m: (do_sync_mmap_readahead.isra.24+0x62/0x9c <- \
      			max_sane_readahead) arg1=200
          java-8723  [003] 13445377.396362: m: (do_sync_mmap_readahead.isra.24+0x62/0x9c <- \
      			max_sane_readahead) arg1=200
          java-8701  [001] 13445377.398216: m: (do_sync_mmap_readahead.isra.24+0x62/0x9c <- \
      			max_sane_readahead) arg1=200
          java-8738  [000] 13445377.399793: m: (do_sync_mmap_readahead.isra.24+0x62/0x9c <- \
      			max_sane_readahead) arg1=200
          java-8728  [000] 13445377.408529: m: (do_sync_mmap_readahead.isra.24+0x62/0x9c <- \
      			max_sane_readahead) arg1=200
      [...]
```

This is also 0x200 pages: 2048KB, and this time I used the `$retval` alias instead of guessing registers. So the tuning really did not take effect. Studying the kernel source, I saw that the readahead property was set by a function called `file_ra_state_init()`. Under what circumstances is _that_ called, and how do I trigger it? ftrace/kprobes to the rescue again:

```
# **./kprobe -s p:file_ra_state_init**
      Tracing kprobe m. Ctrl-C to end.
                kprobe-20331 [002] 13454836.914913: file_ra_state_init: (file_ra_state_init+0x0/0x30)
                kprobe-20331 [002] 13454836.914918: <stack trace>
       => vfs_open
       => nameidata_to_filp
       => do_last
       => path_openat
       => do_filp_open
       => do_sys_open
       => sys_open
       => system_call_fastpath
                kprobe-20332 [007] 13454836.915191: file_ra_state_init: (file_ra_state_init+0x0/0x30)
                kprobe-20332 [007] 13454836.915194: <stack trace>
       => vfs_open
       => nameidata_to_filp
      [...]
```

This time I used `-s` to print stack traces, which showed that this often happens from the `open()` syscall. As I'd left Cassandra running while tuning readahead, it may not have reopened its files and run `file_ra_stat_init()`. So I restarted Cassandra to see if the readahead tuning would then take effect, and re-measured:

```
# **./kprobe 'r:m max_sane_readahead $retval'**
      Tracing kprobe m. Ctrl-C to end.
          java-11918 [007] 13445663.126999: m: (ondemand_readahead+0x3b/0x230 <- \
       			max_sane_readahead) arg1=80
          java-11918 [007] 13445663.128329: m: (ondemand_readahead+0x3b/0x230 <- \
      			max_sane_readahead) arg1=80
          java-11918 [007] 13445663.129795: m: (ondemand_readahead+0x3b/0x230 <- \
      			max_sane_readahead) arg1=80
          java-11918 [007] 13445663.131164: m: (ondemand_readahead+0x3b/0x230 <- \
      			max_sane_readahead) arg1=80
      [...]
```

Success!

`iostat` showed a large drop in disk I/O, and the database latency measurements were much better. This was simply a readahead change, where the new Ubuntu instances defaulted to 2048KB. What had misled us earlier was that tuning it had not appeared to make a difference, as the setting wasn't taking effect.

Another tunable we checked was the I/O scheduler, and whether changing it from "deadline" to "noop" was immediate. Using: 

```
./funccount -i 1 'deadline*'
      ./funccount -i 1 'noop*'
```

gave us the answer: they showed the rate of the related kernel functions, and that the tuning was indeed immediate.

#### ftrace and perf-tools

All the tools I used here are from my perf-tools collection, which are front-ends to ftrace and related tracers (kprobes, tracepoints, and the function profiler). I've described some of them as hacks, which they are, as they use creative workarounds for the lack of some in-kernel features. For example, `iosnoop` reads both issue and completion events in user space, and calculates the latencies there, instead of doing that more efficiently in kernel context.

These tools are for older Linux systems without kernel debuginfo, and show what Linux can do using only its built-in ftrace. There's even more to ftrace that I didn't show here: profiling function average latency, tracing wakeup events, function probe triggers, etc. There is also Steven Rostedt's own front-end to ftrace, a multi-tool called [trace-cmd](<http://git.kernel.org/cgit/linux/kernel/git/rostedt/trace-cmd.git>) ([covered on LWN](<https://lwn.net/Articles/410200/>) in 2010), which can do more than my collection of smaller tools, and is also much easier to use than operating on the `/sys` files directly.

To do even more, perf_events (which is also part of the Linux kernel source) with kernel debuginfo lets me examine complex data structures, and even trace kernel code by line number and watch local variables. What I still can't do, however, is perform complex in-kernel aggregations until I add other advanced tracers like [SystemTap](<https://sourceware.org/systemtap/>), [ktap](<https://lwn.net/Articles/551314/>), or coming up: [eBPF](<https://lwn.net/Articles/603983/>).

#### Future Work

Changes are on the way for the Linux kernel, as regular LWN readers likely know. The capabilities I need, to avoid using hacks, may be provided by eBPF, which may be merged [as soon as Linux 3.18](<http://lwn.net/Articles/606089/>). If that happens, I'll be happy to create new versions of these tools making use of eBPF. They will likely look and feel the same, but their implementation will be much more efficient and reliable. I'll also be able to create many more tools without the fear of maintaining too many hacks.

Although, even if Linux does bring these capabilities in an upcoming release, it’ll be some time before I can really use them in production in my work environment, depending on how quickly we can or want to be on the latest kernel. So, while my hacks are temporary workarounds, they may be useful for some time to come.

#### Conclusion

We have a large number of Linux cloud instances to analyze at Netflix, and some interesting and advanced performance issues to solve. While we've been looking at using advanced tracers like SystemTap, I've also been studying what's there already, including the ftrace profilers. The scripts I showed in this post use ftrace, so they work on our our existing instances as-is, without even installing kernel debuginfo.

In this particular example, a Cassandra database experienced a disk I/O inflation issue caused by an increased readahead setting. I traced disk I/O events and latencies, and how these were created in the kernel. This included examining stack traces, counting function-call rates, measuring slow function times, tracing call graphs, and dynamic tracing of function calls and returns, with their arguments and return values.

I did all of this using ftrace, which has been in the Linux kernel for years. I found the hidden light switches.

If you are curious about the inner workings of these ftrace tools, see "[Secrets of the Ftrace function tracer](<http://lwn.net/Articles/370423/>)" by Steven Rostedt, "Debugging the kernel using Ftrace" [part 1](<http://lwn.net/Articles/365835/>) and [part 2](<http://lwn.net/Articles/366796/>), and [Documentation/trace/ftrace.txt](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/Documentation/trace/ftrace.txt>) in the kernel source. The most important lesson here isn't about my tools, but that this level of tracing is even possible on existing Linux kernels. And if eBPF is added, a _lot_ more will be possible.
