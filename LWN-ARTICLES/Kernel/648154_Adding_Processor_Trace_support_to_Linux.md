---
title: Adding Processor Trace support to Linux
url: https://lwn.net/Articles/648154/
date: "July 1, 2015"
category: Processor Trace; Tracing
author: "July 1, 2015 This article was contributed by Andi Kleen"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

July 1, 2015

This article was contributed by Andi Kleen

Tracing is a technique that is used for both performance analysis and debugging. A tracer generates its data into a log; these are tracing events that can be later analyzed to understand the program's execution. Linux has [ftrace](<https://lwn.net/Articles/365835/>) that can log function calls and tracepoint data for the kernel. CPUs also support tracing mechanisms for different events. [Processor Trace](<https://software.intel.com/en-us/blogs/2013/09/18/processor-tracing>) (PT) is a new tracing mechanism for Intel CPUs that traces branches executing on the CPU, which allows the reconstruction of the control flow of all executed code. 

Intel CPUs have an older mechanism called BTS (Branch Trace Store) that also allowed branch tracing, but it was slow and not widely used. PT allows tracing with better performance and gathers additional information, such as timing. Intel Processor Trace is supported on Broadwell CPUs. More information can be found in the Processor Trace specification in the [Intel Architecture Software Developer Manuals](<http://www.intel.com/sdm>), volume 3, chapter 36. 

Hardware tracing mechanisms were traditionally used from hardware debuggers. But it is also possible to integrate them into the operating system, which allows tight integration with its profiling and debugging mechanisms. This makes it possible to use different trace buffers for different processes, and to make the facility available for non-root users. More importantly, it also allows tracing on every system, without needing to attach special debugging hardware. 

The 4.1 kernel has an implementation of Processor Trace in the kernel's perf_events subsystem that allows the use of PT through the perf interface. The user-tools support is not quite merged yet, but is expected for 4.2. The code was developed by Alex Shishkin and Adrian Hunter and was tested by Emmanuel Lucet. 

Other architectures, such as ARM and MIPS, also have branch tracing mechanisms, but these were not supported by operating system drivers before. ARM is currently merging a separate driver for its [CoreSight](<http://www.arm.com/products/system-ip/debug-trace/>) tracing, but that is using a different, non-perf interface. 

#### Implementation

Tracing branches on a modern CPU that runs multiple cores at multi-GHz speeds can generate huge amounts of data. While the CPU can do the actual tracing with little overhead — hardware can do it in parallel — maintaining the bandwidth to a log buffer can be a challenge. PT uses aggressive compression to make this problem manageable: unconditional direct branches are not logged at all, conditional branches are compressed to a single bit (taken or not-taken), and `CALL`/`RET` instructions can be elided by maintaining a call stack in the decoder. 

A special PT decoder later reads the compressed trace stream. It decodes the instructions in the executable. Every time it reaches a conditional or indirect branch it consults the trace information to decide on the branch target. Asynchronous events, such as interrupts or transaction aborts, are also reported, in addition to time and processor frequency. The information is sufficient to reconstruct the full control flow of the original program. The performance overhead is shifted from tracing to decoding time. Decoding is several orders of magnitude slower than tracing. 

Standard perf PMU (Performance Monitoring Unit) drivers generate "perf events" in the kernel in a standardized format and provide them through a ring buffer to the "`perf record`" command in user space. Using this approach directly is not possible with high-volume hardware tracing, as a PT decoder running in the kernel would never be able to keep up with the realtime instruction stream. 

The perf PT driver instead dumps the tracing data directly to a separate ring buffer and uses a decoder running later in user space to generate perf_events, which are then processed by the standard perf tool chain. The decoder also needs side-band information to identify the executables and associate different processes and threads with the right binaries. Perf already had events for the metadata information needed, as standard perf sampling also relies on user-space post-processing to associate samples with executables. When multiple threads are traced on the same CPU, it also needs context switch information, which is also available. PT decoding uses both the tracing buffers per logical CPU, and the side-band stream generated by perf. 

#### User interface

The merging of the the PT code was a long, drawn-out exercise that took about two years. The main sticking point was how to represent the extra trace buffer in the perf_events API. After multiple iterations, the final version uses an auxiliary data (AUX data) buffer associated with the main perf ring buffer, which is shared between kernel and user space via `mmap()`. The AUX buffer is allocated by `mmap()`ing an area above the original perf ring buffer. The perf driver then configures the tracing hardware to dump data into this buffer. When the buffer is filled up the perf_events kernel code sends an AUX data event to the main perf event ring buffer, which signals the amount of data available to "`perf record`". 

In addition, normal perf events are used to collect the side-band data needed for decoding. The AUX data infrastructure is generic and can be used for other kinds of high-volume trace data. 

After recording, the compressed data is decoded into standard perf events. This is done at "`perf record`" (or perf script/inject) time. To synchronize the side-band data with the hardware decoding stream, time stamps are used. The side-band data is used to determine the executables and threads, which are needed by the decoder to reconstruct the branch stream. The trace stream reports events in Time Stamp Counter (TSC) format, while perf events by default use the Linux trace clock. The kernel reports the necessary conversion factors and offsets. The decoder interpolates timing for branch events between time packets, and uses the resulting estimated time stamps to synchronize with tracepoint events for context switch and executable load. 

To walk the executables and find the static and conditional branches, the decoder uses Masami Hiramatsu's x86 instruction decoder from the kernel tree. 

The PT decoder then generates perf "sample" events from the trace stream. It supports different sampling options, controlled by the new `--itrace` option: 

```
--itrace={letter}period
        i       synthesize "instructions" events
        b       synthesize "branches" events
        x       synthesize "transactions" events
        c       synthesize branches events (calls only)
        r       synthesize branches events (returns only)
        e       synthesize tracing error events
        d       create a debug log
        g[size] synthesize a call chain (use with i or x)
        period can be NUMBER unit (e.g. 10us)
```

By default instructions are sampled approximately every 100µs (with the default `--itrace=i100us` option). 

Here is an example that records the execution of a simple program 

```
% perf record -e intel_pt//u ls
        [ perf record: Woken up 1 times to write data ]
        [ perf record: Captured and wrote 0.190 MB perf.data ]
```

After the recording, the trace needs to be decoded. `perf report` will run the decoder and generate a histogram of the samples. 

```
% perf report --stdio
        ...
        # Samples: 36  of event 'instructions:u'
        # Event count (approx.): 8613072
        #
        # Overhead  Command  Shared Object     Symbol                          
        # ........  .......  ................  ................................
        #
    	27.78%  ls       libc-2.17.so      [.] get_next_seq                
    		     |
    		     ---get_next_seq
    			__strcoll_l
        ...
```

The tracer recorded 8,613,072 instructions, and the decoder sampled them every 100µs. This is a somewhat faster sampling frequency than normal cycle sampling with `perf record`, which can give better results. 

The tracing supports the usual perf filters, like `/u` for user code only, and `/k` for kernel code. There are also more tunables for the PMU that can be optionally specified between the "`/`" delimiters: 

```
tsc=0  	 Disable time stamps
        noretcomp	 Disable return compression
```

In the example above, we used `u` to only record user space. Recording kernel space is also possible with the right permissions, but requires running the decoder as root. 

Alternatively, we can also look at the exact trace, for example for debugging or to find specific patterns for performance analysis. This can be done with `perf script`. Since the default `perf script` output is quite verbose, we limit the fields, but also add the source line if available. This traces the startup of the `ls` command, which begins by executing the dynamic linker (`ld-2.17.so`). 

```
% perf script -F time,comm,cpu,sym,dso,ip,srcline
    		  ls [001] 454279.326516:                 0 [unknown] ([unknown])
    		  ls [001] 454279.326516:      7fdeb58b1630 _start (/lib/x86_64-linux-gnu/ld-2.17.so)
          .:0
    		  ls [001] 454279.326527:                 0 [unknown] ([unknown])
    		  ls [001] 454279.326527:      7fdeb58b1633 _start (/lib/x86_64-linux-gnu/ld-2.17.so)
          .:0
    		  ls [001] 454279.326527:      7fdeb58b4fbf _dl_start (/lib/x86_64-linux-gnu/ld-2.17.so)
          get-dynamic-info.h:44
    		  ls [001] 454279.326532:                 0 [unknown] ([unknown])
    		  ls [001] 454279.326532:      7fdeb58b4fc6 _dl_start (/lib/x86_64-linux-gnu/ld-2.17.so)
          rtld.c:385 
    		  ls [001] 454279.326539:                 0 [unknown] ([unknown])
    		  ls [001] 454279.326539:      7fdeb58b4fe1 _dl_start (/lib/x86_64-linux-gnu/ld-2.17.so)
          rtld.c:414
    		  ls [001] 454279.326546:                 0 [unknown] ([unknown])
    		  ls [001] 454279.326546:      7fdeb58b500e _dl_start (/lib/x86_64-linux-gnu/ld-2.17.so)
          get-dynamic-info.h:52
    		  ls [001] 454279.326546:      7fdeb58b502f _dl_start (/lib/x86_64-linux-gnu/ld-2.17.so)
          get-dynamic-info.h:46
    		  ls [001] 454279.326546:      7fdeb58b502f _dl_start (/lib/x86_64-linux-gnu/ld-2.17.so)
          get-dynamic-info.h:46
    		  ls [001] 454279.326546:      7fdeb58b502f _dl_start (/lib/x86_64-linux-gnu/ld-2.17.so)
          get-dynamic-info.h:46
```

`perf script` outputs branches instead of instructions by default. [ ![\[Flame Graph\]](https://static.lwn.net/images/2015/pt-flamegraph-sm.png) ](<https://lwn.net/Articles/649627/>)

The format is standard perf format, so existing scripts processing samples will work. For example we could generate a [Flame Graph](<http://www.brendangregg.com/flamegraphs.html>) using PT data (as seen at right). 

`perf script` also supports running Perl or Python scripts. We can use that to directly analyze a trace. This is important because instruction traces are often too long to be read by a human. It is also possible to use other tools that can process perf output to view the information. 

Perf also has the ability to store trace data into a database. The `export-to-postgresql.py` script that is included with perf stores branch information into a PostgreSQL database for later analysis with standard database tools. The database pairs up calls and returns, which allows determining the lifetime and the call-stack of every function call. 

#### Limitations for non-root users

Perf supports recording as non-root. There are some limitations. When tracing multiple threads on a single CPU, perf uses the same tracing buffer. To reconstruct the trace later, a context switch side-band event is needed to find the correct executables. The PT decoder uses the `sched::sched_switch` tracepoint to do this. Tracepoints traditionally need root rights (unless perf security is disabled with `kernel.perf_event_paranoid=-1`). If the tracepoint is not available, newly created threads cannot be decoded. This implies that only the single-threaded program that is started by perf or already existing threads selected with `--per-thread --uid/--pid/--tid` can be traced. In the future, a non-root context switch event, such as the one proposed in [this patch](<https://lkml.org/lkml/2015/6/9/513>), would fix this problem. 

Another issue is that the kernel code is self-modifying. To handle kernel code, the decoder has to see the patched "live" image through `/proc/kcore`, which is only available to root. The new `perf-with-kcore.sh` wrapper script allows saving `kcore` as part of a perf session, which allows the decoder to see the same state of the kernel code as during tracing. 

#### Snapshot mode

The record example above attempts to store a complete trace of the program to the disk. Storing complete traces typically only works for programs with limited computation. For longer workloads, which do significant computation with many branches, the disk I/O bandwidth is eventually unable to keep up with the CPU, so there can be data loss. 

An alternative is to store the trace as a ring buffer to memory only, and only stop and save the trace when an "event of interest" happens. Memory has enough bandwidth to keep up with the traces in most cases. 

This is especially useful for debugging: we're looking for the trace before the bug to see how it happened. It is also useful for performance analysis. For example, a common problem both in server and user interface tuning is the need to analyze occasional latency spikes that cause visible delay to the user. 

To handle this situation with PT, perf supports the new snapshot mode. The trace is started in ring-buffer mode, recording the trace constantly in a memory ring-buffer, and saving the perf metadata to disk. 

When the event of interest happens, we send a `SIGUSR2` signal to the `perf record` process, which then dumps the in-memory trace buffer to the `perf.data` file. 

So we need a program that sends the signal when something interesting happens. For example, we can use a simple REST key-value store that retrieves values from a database. Assume the database has an occasional latency glitch that makes a transaction take too long; we want to understand why. So we instrument the program to send the signal to perf when a database operation takes too long. We can then look at the trace later with `perf script` to find out what caused the latency spikes. 

To avoid having to run the test program as root, we disable perf security and run the `perf record` as the same user ID as the test. Otherwise the test program could not send signals to perf. 

```
% sudo sysctl -w kernel.perf_event_paranoid=-1
```

Start the tracing and run the [example server](<https://lwn.net/Articles/648186/>) (`simple-kv.py`) in the background: 

```
% perf record --snapshot -a -e intel_pt//u sleep 100 &
        % export PERFPID=$!
        % python simple-kv.py &
        % curl http://localhost:4000/foo
        # perf script -F time,comm,cpu,sym,dso,ip,srcline
        ... tracing output ...
```

#### Debugging

When we use tracing for debugging, the obvious point of interest is a crash or abort. With tracing information we gain the ability to do limited backward debugging. The original PT patch kit had built-in support for core dumps. A new rlimit allowed enabling tracing in ring buffer mode for a program, and the trace would then be automatically written to the core dump file. The PT code that is merged in 4.1 does not yet have this capability, but it will hopefully eventually be re-added. 

This raises the question of what to do with the core dump. The original PT patch kit API was supported by GDB, which could both read PT core dumps and also do live backward debugging for control flow. The debugger allows you to virtually move inside the trace and allows examining the program state (as available) at different points in the past. GDB had support to do this earlier using [record and replay](<https://sourceware.org/gdb/download/onlinedocs/gdb/Process-Record-and-Replay.html#Process-Record-and-Replay>) and using the earlier Branch Trace Store, but it was extremely slow. PT will be a faster and more functional alternative. 

A version of GDB supporting the current auxtrace kernel API is [currently in development](<https://sourceware.org/ml/gdb-patches/2015-06/msg00453.html>) by Markus Metzger. For decoding PT, GDB uses the [libipt](<https://github.com/01org/processor-trace>) PT decoder, which provides a C API for decoding that can be also used for other PT tools. 

#### Conclusion

Hardware-based branch tracing is a powerful technique for debugging and performance analysis. It can generate vast amounts of data that allow the analysis of programs in new ways. In fact, the challenge will not be to collect data on performance or debugging problems any more, but instead more to find the right information in all of the information that can now be collected. As of the 4.2 kernel, perf and other tools will provide a range of powerful capabilities to generate branch traces. It will also provide basic capabilities to display the data. It will be interesting to see what new visualization and analysis tools can be developed on top of that to make better use of this rich source of data.
