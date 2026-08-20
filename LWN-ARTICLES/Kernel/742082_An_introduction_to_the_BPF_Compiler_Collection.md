---
title: An introduction to the BPF Compiler Collection
url: https://lwn.net/Articles/742082/
date: "December 22, 2017"
category: Development tools
author: "December 22, 2017 This article was contributed by Matt Fleming BPF in the kernel"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

December 22, 2017

This article was contributed by [Matt Fleming](<http://www.codeblueprint.co.uk/>)

* * *

[BPF in the kernel](<https://lwn.net/Articles/740157/>)

In the previous article of this series, I discussed how to use [eBPF to safely run code supplied by user space](<https://lwn.net/Articles/740157/>) inside of the kernel. Yet one of eBPF's biggest challenges for newcomers is that writing programs requires compiling and linking to the eBPF library from the kernel source. Kernel developers might always have a copy of the kernel source within reach, but that's not so for engineers working on production or customer machines. Addressing this limitation is one of the reasons that the [BPF Compiler Collection](<https://github.com/iovisor/bcc>) was created. The project consists of a toolchain for writing, compiling, and loading eBPF programs, along with example programs and battle-hardened tools for debugging and diagnosing performance issues.

Since its release in April 2015, many developers have worked on BCC, and the [113 contributors](<https://github.com/iovisor/bcc/graphs/contributors>) have produced an impressive collection of over 100 [examples](<https://github.com/iovisor/bcc/tree/master/examples>) and ready-to-use [tracing tools](<https://github.com/iovisor/bcc/tree/master/tools>). For example, scripts that use [User Statically-Defined Tracing (USDT) probes](<http://dtrace.org/guide/chp-usdt.html>) (a mechanism from DTrace to place tracepoints in user-space code) are provided for tracing [garbage collection events](<https://github.com/iovisor/bcc/blob/master/tools/lib/ugc.py>), [method calls and system calls](<https://github.com/iovisor/bcc/blob/master/tools/lib/ucalls.py>), and [thread creation and destruction](<https://github.com/iovisor/bcc/blob/master/tools/lib/uthreads.py>) in high-level languages. Many popular applications, particularly databases, also have USDT probes that can be enabled with configuration switches like `--enable-dtrace`. These probes are inserted into user applications, as the name implies, statically at compile-time. I'll be dedicating an entire LWN article to covering USDT probes in the near future.

The project documentation shows how to use the existing scripts and tools to conduct a thorough performance investigation without writing a line of code, and [a handy tutorial](<https://github.com/iovisor/bcc/blob/master/docs/tutorial.md>) is provided in the BCC repository. [Another useful guide to some of the BCC tools](<http://www.brendangregg.com/ebpf.html>) was written by Brendan Gregg, who has the second highest number of patches to `bcc/tools` ([Sasha Goldshtein](<http://blogs.microsoft.co.il/sasha/>) holds the number one spot as of this writing).

Front-ends for the Python and Lua programming languages are available in BCC. Using these high-level languages, it's possible to write short but expressive programs with all the data-manipulation advantages that are missing with C. For example, developers can treat eBPF maps as Python dictionaries and access map contents directly, which is implemented internally by using the BPF helper functions. This helps to lower the bar for would-be developers using eBPF because they can use the standard patterns that they're used to for processing data.

BCC invokes the LLVM Clang compiler, which has a BPF back end, to translate C into eBPF bytecode. BCC then takes care of loading the eBPF bytecode into the kernel with the [`bpf()` system call](<http://man7.org/linux/man-pages/man2/bpf.2.html>). If loading fails, for example if the in-kernel verifier checks fail, then BCC provides hints as to why loading failed, e.g. "HINT: The 'map_value_or_null' error can happen if you dereference a pointer value from a map lookup without first checking if that pointer is NULL." This is another motivation for creating BCC — it's difficult to write obviously correct BPF programs; BCC tells you when you've made a mistake.

#### A really quick "Hello, World!" example

To demonstrate how quickly you can start working with BCC, here's the ["Hello, World!" program example](<https://github.com/iovisor/bcc/blob/master/examples/hello_world.py>) from the BCC repository. It prints into the trace buffer every time the `clone()` system call runs. I've reformatted it slightly to make it easier to read.

```
#!/usr/bin/env python
        from bcc import BPF
    
        program='''
        int kprobe__sys_clone(void *ctx)
        {
        	bpf_trace_printk("Hello, World!\n");
    	return 0;
        }
        '''
```

The entire eBPF program is contained in the `program` variable; this is the code that runs inside the kernel on the eBPF virtual machine. The format of the function name, "`kprobe__sys_clone()`", is important: the [`kprobe__` prefix](<https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#1-kprobes>) directs the BCC toolchain to attach a kprobe to the kernel symbol that follows it. In this case, that's `sys_clone()`. When `sys_clone()` is called and this kprobe fires, the eBPF program runs and `bpf_trace_printk()` prints "Hello, World!" into the kernel's trace buffer.

The remainder of the Python program causes the eBPF code to be loaded into the kernel and run: 

```
b = BPF(text=program)
        b.trace_print()
```

The previously cumbersome task of compiling the program to eBPF bytecode and loading it into the kernel is handled entirely by instantiating a new BPF object; all the low-level work is done behind the scenes, inside the Python bindings and BCC's `libbpf`.

`BPF.trace_print()` performs a blocking read on the kernel's trace buffer file (`/sys/kernel/debug/tracing/trace_pipe`) and prints the contents to the standard output. Here's the output:

```
gnome-terminal--3210  [003] d..2 19252.369014: 0x00000001: Hello, World!
        gnome-terminal--3210  [003] d..2 19252.369080: 0x00000001: Hello, World!
        pool-21543 [001] d..2 19252.382317: 0x00000001: Hello, World!
        bash-21545 [002] d..2 19252.385535: 0x00000001: Hello, World!
        bash-21546 [003] d..2 19252.385752: 0x00000001: Hello, World!
        bash-21545 [002] d..2 19252.386883: 0x00000001: Hello, World!
```

The output shows:

  * The name of the application running when the kprobe fired
  * Its PID
  * The CPU it was running on (in [brackets])
  * Various process context flags
  * A timestamp

The final field is our "Hello, World!" string that we passed to `bpf_trace_printk()`. The penultimate field contains the address `0x00000001`. Normally, when kernel code writes to the trace buffer, the instruction pointer address following the call to `trace_printk()` is printed in that field. Unfortunately, this isn't implemented for `bpf_trace_printk()`, so the hard-coded address `0x00000001` is always used.

#### More examples

[`argdist.py`](<https://github.com/iovisor/bcc/blob/master/tools/argdist.py>) inserts a probe (uprobe, kprobe, tracepoint, or USDT) into to a given function, which can be in the kernel or in user-space code. When the probe fires, `argdist.py` prints the function's parameter values, either as a count or histogram. It runs until interrupted by the user. For example, the following command prints the number of times `irq_handler_entry()` is called, along with which interrupt was raised: 

```
$ tools/argdist.py -C 't:irq:irq_handler_entry():int:args->irq'
        [14:14:24]
        t:irq:irq_handler_entry():int:args->irq
        COUNT      EVENT
        12         args->irq = 45
        16         args->irq = 53
        52         args->irq = 48
        [14:14:25]
        t:irq:irq_handler_entry():int:args->irq
        COUNT      EVENT
        1          args->irq = 49
        5          args->irq = 53
        24         args->irq = 45
```

Because the histogram option (`-H`) uses buckets to group multiple interrupts together, it's less useful than the count option (`-C`) in this case. One scenario where histogram output is helpful, however, is for the [`btrfsdist.py`](<https://github.com/iovisor/bcc/blob/master/tools/btrfsdist.py>) tool, which summarizes the latency of Btrfs reads, writes, opens, and fsync operations into power-of-two buckets:

```
$ tools/btrfsdist.py
        Tracing btrfs operation latency... Hit Ctrl-C to end.
        ^C
    
        operation = 'read'
         usecs               : count     distribution
             0 -> 1          : 775      |****************************************|
             2 -> 3          : 60       |***                                     |
             4 -> 7          : 20       |*                                       |
             8 -> 15         : 3        |                                        |
            16 -> 31         : 3        |                                        |
            32 -> 63         : 0        |                                        |
            64 -> 127        : 0        |                                        |
           128 -> 255        : 1        |                                        |
           256 -> 511        : 19       |                                        |
           512 -> 1023       : 12       |                                        |
    
        operation = 'write'
         usecs               : count     distribution
             0 -> 1          : 0        |                                        |
             2 -> 3          : 2        |**********                              |
             4 -> 7          : 8        |****************************************|
             8 -> 15         : 1        |*****                                   |
            16 -> 31         : 4        |********************                    |
            32 -> 63         : 4        |********************                    |
    
        operation = 'open'
         usecs               : count     distribution
             0 -> 1          : 636      |****************************************|
             2 -> 3          : 22       |*                                       |
             4 -> 7          : 16       |*                                       |
             8 -> 15         : 2        |                                        |
            16 -> 31         : 1        |                                        |
    
        operation = 'fsync'
         usecs               : count     distribution
             0 -> 1          : 0        |                                        |
             2 -> 3          : 0        |                                        |
             4 -> 7          : 0        |                                        |
             8 -> 15         : 0        |                                        |
            16 -> 31         : 0        |                                        |
            32 -> 63         : 0        |                                        |
            64 -> 127        : 0        |                                        |
           128 -> 255        : 0        |                                        |
           256 -> 511        : 0        |                                        |
           512 -> 1023       : 0        |                                        |
          1024 -> 2047       : 0        |                                        |
          2048 -> 4095       : 0        |                                        |
          4096 -> 8191       : 1        |****************************************|
```

#### There's more to come

That was just a quick introduction to BCC. In the next one, we'll explore some of the more complicated topics, like how to access eBPF data structures, how to configure the way your eBPF program is compiled, and how to debug your programs, all using the Python front end.
