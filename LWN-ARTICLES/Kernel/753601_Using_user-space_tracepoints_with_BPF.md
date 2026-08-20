---
title: "Using user-space tracepoints with BPF"
url: https://lwn.net/Articles/753601/
date: "May 11, 2018"
category: "BPF-Tracing; Tracing-with BPF"
author: "May 11, 2018 This article was contributed by Matt Fleming BPF in the kernel"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

May 11, 2018

This article was contributed by [Matt Fleming](<http://www.codeblueprint.co.uk/>)

* * *

[BPF in the kernel](<https://lwn.net/Articles/740157/>)

Much has been written on LWN about dynamically instrumenting kernel code. These features are also available to user-space code with a special kind of probe known as a User Statically-Defined Tracing (USDT) probe. These probes provide a low-overhead way of instrumenting user-space code and provide a convenient way to debug applications running in production. In this final article of the BPF and BCC series we'll look at where USDT probes come from and how you can use them to understand the behavior of your own applications. 

The origins of USDT probes can be found in Sun's DTrace utility. While DTrace can't claim to have invented static tracepoints (various implementations are described in the ["related work" section](<https://www.usenix.org/legacy/event/usenix04/tech/general/full_papers/cantrill/cantrill_html/#sec-related>) of the [original DTrace paper](<https://www.usenix.org/legacy/event/usenix04/tech/general/full_papers/cantrill/cantrill_html/>)), it certainly made them much more popular. With the emergence of DTrace, many applications began adding USDT probes to important functions to aid with tracing and diagnosing run-time behavior. Given that, it's perhaps not surprising that these probes are usually enabled (as part of configuring the build) with the `--enable-dtrace` switch. 

For example, MySQL provides a number of [probes](<https://dev.mysql.com/doc/refman/5.6/en/dba-dtrace-mysqld-ref.html>) to aid database administrators and to help them understand who is connecting to the database, which SQL commands they're running, and low-level details on data transferred between clients and MySQL servers. Other popular tools such as Java, PostgreSQL, Node.js, and even the GNU C Library also come with the option of enabling probes. These probes cover a wide range of activities, from memory allocation to garbage-collection events. 

There is a variety of tools on Linux to work with USDT probes. [SystemTap](<https://sourceware.org/systemtap/>) is a popular choice and an alternative to DTrace since it's only recently that [DTrace has been supported on Linux](<https://oss.oracle.com/git/gitweb.cgi?p=dtrace-linux-kernel.git;a=summary>). Support for USDT probes (termed "statically defined traces" inside the kernel) for perf was [merged](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7e4dc77b2869a683fc43c0394fca5441816390ba>) in v4.8-rc1, and even [LTTng](<https://lttng.org>) has been able to emit [USDT-compatible probes](<https://github.com/lttng/lttng-ust/commit/b2f60c7986bb69f81b79b68f1bfe83aafa3278a7>) since 2012. The most recent additions to the developer's USDT tool chest — and arguably the most user-friendly — are the tools and scripts in the [BPF Compiler Collection (BCC)](<https://github.com/iovisor/bcc>). 

#### BCC tools for working with USDT probes

BCC has had support for USDT probes since March 2016, when Sasha Goldshtein sent a [pull request](<https://github.com/iovisor/bcc/pull/451>) to the GitHub project adding support to the existing `tplist` and `trace` tools. 

The [`tplist`](<https://github.com/iovisor/bcc/blob/master/tools/tplist.py>) tool allows you to see which probes (if any) are available for the kernel, an application, or a library, and it can be used to discover names of probes to enable with [`trace`](<https://github.com/iovisor/bcc/blob/master/tools/trace.py>). Running it on a version of the C library compiled with SDT support shows: 

```
# tplist.py -l /lib64/libc-2.27.so
        /lib64/libc-2.27.so libc:setjmp
        /lib64/libc-2.27.so libc:longjmp
        /lib64/libc-2.27.so libc:longjmp_target
        /lib64/libc-2.27.so libc:memory_mallopt_arena_max
        /lib64/libc-2.27.so libc:memory_mallopt_arena_test
        /lib64/libc-2.27.so libc:memory_tunable_tcache_max_bytes
        /lib64/libc-2.27.so libc:memory_tunable_tcache_count
        /lib64/libc-2.27.so libc:memory_tunable_tcache_unsorted_limit
        /lib64/libc-2.27.so libc:memory_mallopt_trim_threshold
        /lib64/libc-2.27.so libc:memory_mallopt_top_pad
        [ ... ]
```

The `-l` parameter tells `tplist` that the file argument is a library or executable. Omitting `-l` instructs `tplist` to print the list of kernel tracepoints. 

A filter can be applied to the list of tracepoints and probes, which helps to shorten the (potentially very long) default output of `tplist`. For example, using the filter `sbrk` prints only those probes with the string "`sbrk`" in their name. And using the `-vv` parameter prints the arguments available at the probe site. For example: 

```
./tplist.py -vv -l /lib64/libc-2.27.so sbrk
        /lib64/libc-2.27.so libc:memory_sbrk_less [sema 0x0]
        location #1 0x816dd
        argument #1 8 unsigned bytes @ ax
        argument #2 8 signed   bytes @ bp
        /lib64/libc-2.27.so libc:memory_sbrk_more [sema 0x0]
        location #1 0x826af
        argument #1 8 unsigned bytes @ ax
        argument #2 8 signed   bytes @ r12
```

The argument details are necessary to understand which registers contain function parameters. Knowing the location of arguments allows us to print their contents with the BCC `trace` tool with a command like: 

```
# trace.py 'u:/lib64/libc-2.27.so:memory_sbrk_more "%u", arg1' -T
        TIME     PID     TID     COMM            FUNC             -
        21:46:51 12781   12781   ls              memory_sbrk_more 114974720
```

The `trace` utility takes a number of arguments and accepts a probe specifier, an example of which was used above. Probe specifiers allow users to describe exactly what they want to be printed when the probe fires. A list of examples (and a more thorough explanation of the format) is provided in the [trace_example.txt](<https://github.com/iovisor/bcc/blob/master/tools/trace_example.txt>) file in the BCC repository. The output above shows one hit when a process running `ls` hit the `memory_sbrk_more` probe. 

Additional tools in BCC enable USDT probes for popular high-level languages like Java, Python, Ruby, and PHP — `[lib/calls.py](<https://github.com/iovisor/bcc/blob/master/tools/lib/ucalls.py>)` summarizes method calls, `[lib/uflow.py](<https://github.com/iovisor/bcc/blob/master/tools/lib/uflow.py>)` traces function entry and exit and prints a visual flow graph, `[lib/ugc.py](<https://github.com/iovisor/bcc/blob/master/tools/lib/ugc.py>)` traces garbage-collection events, `[lib/uobjnew.py](<https://github.com/iovisor/bcc/blob/master/tools/lib/uobjnew.py>)` prints summary statistics for new object allocations, and `[lib/uthreads.py](<https://github.com/iovisor/bcc/blob/master/tools/lib/uthreads.py>)` prints details on thread creation. `[lib/ustat.py](<https://github.com/iovisor/bcc/blob/master/tools/lib/ustat.py>)` is a monitoring tool that pulls all of these together and displays their events with a `top`-like interface: 

```
# ustat.py
        Tracing... Output every 10 secs. Hit Ctrl-C to end
        12:17:17 loadavg: 0.33 0.08 0.02 5/211 26284
    
        PID    CMDLINE      METHOD/s   GC/s   OBJNEW/s   CLOAD/s  EXC/s  THR/s
        3018   node/node    0          3      0          0        0      0
```

The output above shows that pid 3018, a `node` process, generated three garbage collection events within a ten-second period. Like most of these scripts, `ustat.py` runs until interrupted by the user. 

In addition to the language-specific tools, BCC also includes specialized scripts for specific applications. For example, [`bashreadline.py`](<https://github.com/iovisor/bcc/blob/master/tools/bashreadline.py>) prints commands from all running bash shells: 

```
# bashreadline.py
        TIME      PID    COMMAND
        05:28:25  21176  ls -l
        05:28:28  21176  date
        05:28:35  21176  echo hello world
        05:28:43  21176  foo this command failed
        05:28:45  21176  df -h
        05:29:04  3059   echo another shell
        05:29:13  21176  echo first shell again
```

[`dbslower.py`](<https://github.com/iovisor/bcc/blob/master/tools/dbslower.py>) prints database (MySQL or PostgreSQL) operations with a latency that exceeds a specified threshold: 

```
# dbslower.py mysql -m 1000
        Tracing database queries for pids 25776 slower than 1000 ms...
        TIME(s)        PID          MS QUERY
        1.421264       25776  2002.183 call getproduct(97)
        3.572617       25776  2001.381 call getproduct(97)
        5.661411       25776  2001.867 call getproduct(97)
        7.748296       25776  2001.329 call getproduct(97)
```

#### Adding USDT probes to your application

SystemTap provides an API for adding static probes to an application. To create them, you'll need the `systemtap-sdt-devel` package, which provides the `sys/sdt.h` header file. The documentation for the SystemTap project provides an [example of adding a probe](<https://sourceware.org/systemtap/wiki/AddingUserSpaceProbingToApps>), but we'll add one to a simple C program and use the BCC tools to list and enable the probe: 

```
#include <sys/sdt.h>
        #include <sys/time.h>
        #include <unistd.h>
    
        int main(int argc, char **argv)
        {
            struct timeval tv;
    
            while(1) {
                gettimeofday(&tv, NULL);
                DTRACE_PROBE1(test-app, test-probe, tv.tv_sec);
                sleep(1);
            }
            return 0;
        }
```

This simple program runs until interrupted. It fires a probe and then calls `sleep()` to wait for one second until the loop starts again. The `DTRACE_PROBE()` macro is used to create probe points at desired locations, in this case, immediately before sleeping. This macro takes a provider name, probe name, and arguments as parameters. There's a separate `DTRACE_PROBEn()` macro for each argument count. For example, if your probe has three arguments, you need to use `DTRACE_PROBE3()`. 

The `DTRACE_PROBEn()` macros are implemented by placing a no-op assembly instruction at the probe site and writing an ELF note in the application image that includes things like the probe address and name. Since the runtime overhead of an inactive probe is the cost of executing a no-op instruction and, given that the ELF note isn't loaded into memory, the impact on performance is minimal. 

The provider name allows you to create a namespace for your probe. The most common value (and the one suggested in the SystemTap example) is to use the name of your application or library. In the example above I've used `test-app`, and the probe name was imaginatively titled, `test-probe`. The one and only argument to the probe is the time in seconds. 

Using tplist, we can see the probe and its argument: 

```
# tplist.py -vv -l ./simple-c
        simple-c test-app:test-probe [sema 0x0]
        location #1 0x40057b
        argument #1 8 signed   bytes @ ax
```

We can then construct a probe specifier to print the first argument with `trace`, assuming the simple-c program above is running: 

```
# trace.py 'u:./simple-c:test-probe "%u", arg1' -T -p $(pidof simple-c)
        TIME     PID     TID     COMM            FUNC             -
        21:55:44 13450   13450   simple-c        test-probe       1524430544
        21:55:45 13450   13450   simple-c        test-probe       1524430545
        21:55:46 13450   13450   simple-c        test-probe       1524430546
```

The final column in the output shows the current time, in seconds, when the probe fires. This is information passed as the first argument in the probe declaration. You do need to be aware of the data type of the probe arguments, since it is reflected in the `printf()`-style format string used in the probe specifier. 

#### Conclusion

USDT probes bring the flexibility of kernel tracepoints to user-space applications. Thanks to the rise of DTrace, many popular applications and high-level programming languages grew support for USDT probes. BCC provides simple tools for working with probes that allow developers to list available probes in libraries and applications, and trace them to print diagnostic data. Adding probes to your own code is possible with SystemTap's API and the collection of `DTRACE_PROBE()` macros. USDT probes can help you troubleshoot your applications in production with minimal run-time overhead.
