---
title: Perfcounters added to the mainline
url: https://lwn.net/Articles/339361/
date: "July 1, 2009"
category: Performance monitoring
author: "By Jake Edge July 1, 2009"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
July 1, 2009

We last [looked](<http://lwn.net/Articles/311850/>) at the perfcounters patches back in December, shortly after they appeared. Since that time, a great deal of work has been done, culminating in perfcounters being included into the mainline during the recently completed 2.6.31 merge window. Along the way, a tool to use perfcounters, called `perf`, was added to the mainline as well. 

Adding `perf` to the kernel `tools/` directory is one of the more surprising aspects of the perfcounters merge. Kernel hackers have long been leery of adding user-space tools into the kernel source tree, but Linus Torvalds was [unconvinced](<https://lwn.net/Articles/339406/>) by multiple complaints about that approach. He pointed to `oprofile` to explain: 

It took literally months for the user mode tools to catch up and get the patches to support new functionality into CVS (or is it SVN?), and after that it took even longer for them to become part of a release and be picked up by distributions. In fact, I'm not sure it is part of a release even now - I had to make a bug report to Fedora to get atom and Nehalem support in my tools: I think they took the unofficial patch. 

Others were not so sure that the `oprofile` being developed separately from the kernel was the root cause of those failures. Christoph Hellwig [had other ideas](<https://lwn.net/Articles/339409/>): ""I don't think oprofile has been a [disaster] because of any kind of split, but because the design has been a failure from day 1."" But, Torvalds wants to [try including the tool](<https://lwn.net/Articles/339410/>) to see where it leads: ""Let's give a _new_ approach a chance, and see if we can avoid the mistakes of yesteryear this time."" 

The `perf` tool itself is a fairly simple command-line tool, which can be built from the `tools/perf` directory. It also includes some documentation, in the form of man pages that are also available via `perf help` (as well as in HTML and other formats). At its simplest, it gathers and reports some statistics for a particular command: 

```
$ perf stat ./hackbench 10
        Time: 4.174
    
         Performance counter stats for './hackbench 10':
    
    	8134.135358  task-clock-msecs     #      1.859 CPUs
    	      23524  context-switches     #      0.003 M/sec
    	       1095  CPU-migrations       #      0.000 M/sec
    	      16964  page-faults          #      0.002 M/sec
    	10734363561  cycles               #   1319.669 M/sec
    	12281522014  instructions         #      1.144 IPC
    	  121964514  cache-references     #     14.994 M/sec
    	   10280836  cache-misses         #      1.264 M/sec
    
    	4.376588249  seconds time elapsed.
```

This summarizes the performance events that occurred while running the `hackbench` micro-benchmark program. There are a combination of hardware events (cycles, instructions, cache-references, and cache-misses) as well as software events (task-clock-msecs, context-switches, CPU-migrations, and page-faults) that are derived from the kernel code and not the processor-specific performance monitoring unit (PMU). Currently, support for hardware events is available for Intel, AMD, and PowerPC PMUs, but other architectures still have support for the software events. 

There is also a `top`-like mode for observing which kernel functions are being executed most frequently in a continuously updating display: 

```
$ perf top -c 1000 -p 3216
    
        ------------------------------------------------------------------------------
           PerfTop:     360 irqs/sec  kernel:65.0% [1000 cycles],  (target_pid: 3216)
        ------------------------------------------------------------------------------
    
    		 samples    pcnt         RIP          kernel function
          ______     _______   _____   ________________   _______________
    
    		 1214.00 -  5.3% - 00000000c045cb4d : lock_acquire
    		 1148.00 -  5.0% - 00000000c045d1d3 : lock_release
    		  911.00 -  4.0% - 00000000c045d377 : lock_acquired
    		  509.00 -  2.2% - 00000000c05a0cbc : debug_locks_off
    		  490.00 -  2.2% - 00000000c05a2f08 : _raw_spin_trylock
    		  489.00 -  2.1% - 00000000c041d1d8 : read_hpet
    		  488.00 -  2.1% - 00000000c04419b8 : run_timer_softirq
    		  483.00 -  2.1% - 00000000c04d5f72 : do_sys_poll
    		  477.00 -  2.1% - 00000000c05a34a0 : debug_smp_processor_id
    		  462.00 -  2.0% - 00000000c043df85 : __do_softirq
    		  404.00 -  1.8% - 00000000c074d93f : sub_preempt_count
    		  353.00 -  1.5% - 00000000c074d9d2 : add_preempt_count
    		  338.00 -  1.5% - 00000000c0408a76 : native_sched_clock
    		  318.00 -  1.4% - 00000000c074b4c3 : _spin_lock_irqsave
    		  309.00 -  1.4% - 00000000c044ea10 : enqueue_hrtimer
```

This is a static version of the output from looking at a largely quiescent firefox process (pid 3216), sampling every 1000 cycles. 

There is quite a bit more that `perf` can do. There is a `record` sub-function that gathers the performance counter data into a `perf.data` file which can be used by other commands: 

```
$ perf record ./hackbench 10
        Time: 4.348
        [ perf record: Captured and wrote 2.528 MB perf.data (~110448 samples) ]
    
        $ perf report --sort comm,dso,symbol | head -15
    
        #
        # (110146 samples)
        #
        # Overhead           Command  Shared Object              Symbol
        # ........  ................  .........................  ......
        #
    	10.70%         hackbench  [kernel]                   [k] check_bytes_and_report
    	 9.07%         hackbench  [kernel]                   [k] slab_pad_check
    	 5.67%         hackbench  [kernel]                   [k] on_freelist
    	 5.28%         hackbench  [kernel]                   [k] lock_acquire
    	 5.03%         hackbench  [kernel]                   [k] lock_release
    	 3.19%         hackbench  [kernel]                   [k] init_object
    	 3.02%         hackbench  [kernel]                   [k] lock_acquired
    	 2.47%         hackbench  [kernel]                   [k] _raw_spin_trylock
```

This output shows the top eight kernel functions executed while running `hackbench`. The same data file can also be used by `perf annotate` (when given a symbol name and the appropriate `vmlinux` file) to show the disassembled code for a function, along with the number of samples recorded on each instruction. There is clearly a wealth of information that can be derived from the tool. 

The original posting of the perfcounters patches [came as somewhat of a surprise](<http://lwn.net/Articles/310260/>) to Stephane Eranian, who had long been working on another performance monitoring solution, "perfmon". While he is still a bit skeptical of perfcounters, which were originally proposed by Ingo Molnar and Thomas Gleixner, he has been reviewing the patches, and providing [lengthy comments](<https://lwn.net/Articles/339432/>). Molnar, also [responded at length](<https://lwn.net/Articles/339433/>), breaking his reply into multiple chunks which can be found in the thread. 

Perfmon was targeted at exposing as much of the underlying PMU data as possible to user space, but Molnar explicitly [rejects that goal](<https://lwn.net/Articles/339436/>): 

So for every "will you support advanced PMU feature X, Y and Z" question you ask, the first-level answer is: 'please show the developer usecase and integrate it into our tools so we can see how it all works and how useful it is'. 

"A tool might want to do this" is not a good enough answer. We now have a working OSS tool-space with 'perf' where such arguments for more PMU features can be made in very specific terms: patches, numbers and comparisons. Actual hands-on utility, happy developers and faster apps is what matters in the end - not just the list of PMU features we expose. 

His focus, presumably shared with his co-maintainers Peter Zijlstra and Paul Mackerras, is to generalize performance measurement features so that they are not dependent on any particular CPU and that they fit well with developer work flow: ""I do claim we had few if any sane performance analysis tools before under Linux, and i think we are still in the stone ages and still have a lot of work to do in this area."" From Molnar's perspective, that ease of use for users and developers is one of the main areas where perfmon fell short. 

Molnar is not shy about pointing out that perfcounters still needs a lot of work, but the framework is there, so features can be added to that. As yet, there is no documentation in the kernel `Documentation/` directory, but one presumes that will be handled sometime soon. Overall, perfcounters and the `perf` tool look to be a highly useful addition to the kernel, one that should start providing benefits--in the form of better performance--in the near term.
