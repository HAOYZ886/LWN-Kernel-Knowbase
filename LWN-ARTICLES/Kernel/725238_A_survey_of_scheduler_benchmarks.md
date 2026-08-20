---
title: A survey of scheduler benchmarks
url: https://lwn.net/Articles/725238/
date: "June 14, 2017"
category: "Benchmarking; Scheduler-Testing and benchmarking"
author: "June 14, 2017 This article was contributed by Matt Fleming"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

June 14, 2017

This article was contributed by [Matt Fleming](<http://www.codeblueprint.co.uk/>)

Many benchmarks have been used by kernel developers over the years to test the performance of the scheduler. But recent kernel commit messages have shown a particular pattern of tools being used (some relatively new), all of which were created specifically for developing scheduler patches. While each benchmark is different, having its own unique genesis story and intended testing scenario, there is a unifying attribute; they were all written to scratch a developer's itch.

#### Hackbench

[Hackbench](<http://manpages.ubuntu.com/manpages/xenial/man8/hackbench.8.html>) is a message-passing scheduler benchmark that allows developers to configure both the communication mechanism (pipes or sockets) and the task configuration (POSIX threads or processes). This benchmark is a stalwart of kernel scheduler testing, and has had more versions than the Batman franchise. It was [originally created](<http://lkml.iu.edu/hypermail/linux/kernel/0112.1/0702.html>) in 2001 by Rusty Russell to demonstrate the improved performance of the [multi-queue scheduler](<http://lse.sourceforge.net/scheduling/mq1.html>) patch series. Over the years, many people have added their contributions to Russell's version, including Ingo Molnar, Yanmin Zhang, and David Sommerseth. Hitoshi Mitake added the most recent incarnation to the kernel source tree as part of the `perf-bench` tool in 2009.

Here's an example of the output of `perf-bench`: 

```
$ perf bench sched pipe
        # Running 'sched/pipe' benchmark:
        # Executed 1000000 pipe operations between two processes
    
             Total time: 3.643 [sec]
    
               3.643867 usecs/op
                 274433 ops/sec
```

The output of the benchmark is the average scheduler wakeup latency — the duration between telling a task it needs to wake up to perform work and that task running on a CPU. When analyzing latency, it's important to look at as many latency samples as possible because outliers (high-latency values) can be hidden by a summary statistic, such as the arithmetic mean. It's quite easy to miss those high latency events if the only data you have is the average latency, but scheduler wakeup delays can quickly lead to major performance issues. 

Because hackbench calculates an average latency for communicating a fixed amount of data between two tasks, it is most often used by developers who are making changes to the scheduler's load-balancing code. On the flip side, the lack of data for analyzing the entire latency distribution makes it difficult to dig into scheduler latency wakeup issues without using tracing tools.

#### Schbench

One benchmark that does provide detailed latency distribution statistics for scheduler wakeups is [schbench](<https://git.kernel.org/pub/scm/linux/kernel/git/mason/schbench.git/>). It allows users to configure the usual parameters — such as number of tasks and test duration — but also the time between wakeups (`--sleeptime`), time spinning once woken (`--cputime`); it also has the ability to automatically increase the task count until the 99th percentile wakeup latencies become extreme.

Schbench was created by Chris Mason in 2016 while forward porting some kernel patches that Facebook was carrying to improve the [performance of its workloads](<https://marc.info/?l=linux-kernel&m=145987986404475>). "Schbench allowed me to quickly test a variety of theories as we were forward porting our old patches", Mason said in a private email. It has since become useful for more than that, and Facebook now uses it for performance regression detection, investigating performance issues, and benchmarking patches before they're posted upstream.

Here's an example showing the detailed statistics produced by schbench: 

```
$ ./schbench -t 16 -m 2
        Latency percentiles (usec)
    	50.0000th: 15
    	75.0000th: 24
    	90.0000th: 26
    	95.0000th: 30
    	*99.0000th: 85
    	99.5000th: 1190
    	99.9000th: 7272
    	min=0, max=7270
```

The scheduler wakeup latency distribution that schbench prints at the end of the benchmark run is one of its distinguishing features, and was one of the main rationales for creating it. Mason continued: "The focus on p99 latencies instead of average latencies is the most important part. For us, lots of problems only show up when you start looking at the long tail in the latency graphs." It's also a true micro-benchmark, including only the bare minimum code required to simulate Facebook's workloads while ensuring the scheduler is the slowest part of the code path.

Publishing this benchmark has provided a common tool for discussing Facebook's workloads with upstream developers, and non-Facebook engineers are now using it to test their scheduler changes, which Mason is very happy with: "I'm really grateful when I see people using schbench to help validate new patches going in."

#### Adrestia

[Adrestia](<https://github.com/mfleming/adrestia>) is a dirt-simple scheduler wakeup latency micro-benchmark that contains even less code than schbench. I wrote it in 2016 to measure scheduler latency without using the `futex()` system call as is done in schbench in order to provide more coverage by testing a different kernel subsystem in the scheduler wakeup path. I also needed something that had fewer bells and whistles and was trivial to configure. While schbench models Facebook's workloads, Adrestia is designed only to provide the 95th-percentile wakeup latency value, which provides a simple answer to the question: "What is the typical maximum wakeup latency value?"

I use adrestia to detect performance regressions of patches merged, and to validate potential patches as they're posted to the linux-kernel mailing list. It has been particularly useful for triggering regressions caused by changes to the cpufreq code, mainly because I test with wakeup times that are a multiple of 32ms — the Linux scheduling period. Using multiples of the scheduling period allows the CPU frequency to be reduced before the next wakeup, and thus provides an understanding of the effects of frequency selection on scheduler wakeup latencies. This turns out to be important when validating performance because many enterprise distributions ship with the intel_pstate driver enabled and the default governor set to "powersave".

#### Rt-app

[Rt-app](<https://github.com/scheduler-tools/rt-app>) is a highly configurable real-time workload simulator that accepts a JSON grammar for describing task execution and periodicity. It was originally created by Giacomo Bagnoli as part of his [master's thesis](<https://retis.sssup.it/?q=node/77>) so that he could create background tasks to induce scheduler latency and test his Linux kernel changes for low-latency audio. Juri Lelli started working on it around 2010 when he began his efforts on the deadline scheduler project, again, for his [master's thesis [PDF]](<https://core.ac.uk/download/pdf/14699805.pdf>). Lelli said (in a private email) that he used rt-app while writing his thesis because it was the in-house testing solution at RetisLab (Scuola Superiore Sant'Anna University) at the time, "I didn't also know about any other tool that was able to create synthetic sets base on a JSON description".

Today, ARM and Linaro are using rt-app to trigger specific scheduler code paths. It is a flexible tool that can be used to test small scheduling and load-balancing changes; it is also useful for generating end-to-end workload performance and power figures. Because of its flexibility (and expressive JSON grammar) it is heavily used to model workloads when they are impractical to run directly, such as Android benchmarks on mainline Linux. "You want to use it to abstract complexity and test for regressions across different platforms/os stacks/back/forward ports", said Lelli.

Lelli himself uses it primarily for handling bug reports because he can model problematic workloads without having to run the actual application stack. He also uses it for regression testing; the rt-app source repository has amassed a large collection of configurations for workloads that have caused regressions in the past. Many developers run rt-app indirectly via [ARM's LISA framework](<https://github.com/ARM-software/lisa>), since LISA further abstracts the creation of rt-app configuration files and also includes libraries to post-process the rt-app trace data.

If modeling of complex workloads is needed when testing scheduler changes, rt-app appears to be the obvious choice. "It's useful to model (almost) any sort of real-world application without coding it from scratch - you just need to be fluent with its own JSON grammar… I'm actually relatively confident that for example it shouldn't be too difficult to create {hackbench,cyclictest,etc.}-like type of workloads with rt-app".

#### In closing

[Benchmarks offer benefits](<https://www.infoq.com/articles/traditional-benchmark-lifespan>) that no other tool can; they can help developers communicate the important bits of a workload by paring it back to its core, making it simple to reproduce reported performance issues, and ensuring that performance doesn't regress. Yet a large number of performance-improving kernel patches contain no benchmark numbers at all. That's slowly starting to change for the scheduler subsystem with the help of the benchmarks mentioned above. But if you can't find a benchmark that represents your workload, maybe it's time to write your own, and finally scratch that itch.
