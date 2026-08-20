---
title: Advanced usage of last branch records
url: https://lwn.net/Articles/680996/
date: "March 30, 2016"
category: "Tracing-Last branch records"
author: "March 30, 2016 This article was contributed by Andi Kleen"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

March 30, 2016

This article was contributed by Andi Kleen

Last branch records (LBRs) are hardware registers on Intel CPUs that allow sampling branches. These registers hold a ring buffer of the most recent branch decisions that can be used to reconstruct the program's behavior. Last week, we examined the [basics of LBRs](<https://lwn.net/Articles/680985/>) using Linux perf. Now we look at some more advanced uses. 

#### Transactional memory

The Transactional Synchronization Extension (TSX) is a hardware transactional memory implementation, available in Intel [Broadwell](<https://en.wikipedia.org/wiki/Broadwell_%28microarchitecture%29>) or later, that can improve performance for critical regions by speculatively executing them in parallel. For tuning TSX programs, the goal is usually to reduce unnecessary transaction aborts. The perf `--branch-history` option can also be useful to see why [TSX transactions](<https://lwn.net/Articles/534758/>) aborted. Normally we cannot see into TSX transactions because any profiling interrupt aborts the transaction. But LBRs can log branches even inside transactions, so `--branch-history` can show why the abort happened. 

This is particularly useful for internal aborts that are caused by the code inside the transaction itself, such as those caused by transaction-aborting instructions like system calls. That allows seeing what led to the system call being executed. For conflict aborts caused by other threads, they may be visible if the abort happens to be near the instruction that touched the conflicting cache line. There is no guarantee of that, though, as the conflict could hit at any point in the transaction. 

The LBRs also have flags that show that a particular branch was in a transaction or is an abort. This is currently not displayed by `--branch-history`, but can be examined manually through `perf report` or [`perf script`](<http://linux.die.net/man/1/perf-script>). For more details see the [perf TSX tutorial](<https://software.intel.com/en-us/blogs/2013/05/03/intelr-transactional-synchronization-extensions-intelr-tsx-profiling-with-linux-0>). 

#### Branch mispredictions

In addition to the "from" and to "addresses" of branches, LBRs also provide fine-grained information on branch misprediction. Modern CPUs have a long pipeline to execute instructions, and rely on accurately predicting the branch targets to keep the pipeline filled. This is done by a [branch predictor](<https://en.wikipedia.org/wiki/Branch_predictor>). When branches are often mispredicted, performance is typically poor, as the CPU has to throw away a lot of work. Often code that is difficult to predict can be restructured to make it easier for the branch predictor to handle (the [Intel optimization manual [644-page PDF]](<http://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-optimization-manual.pdf>) describes some techniques on how to do this). 

Mispredicted branches can be sampled directly using CPU-performance-counter events, but deriving them from LBRs gives better coverage and does not require an additional performance counter. `perf report` can directly display the branch misprediction state, but currently only at function level.

Here is an example based on a [Stack Overflow question](<http://stackoverflow.com/questions/11227809/why-is-processing-a-sorted-array-faster-than-an-unsorted-array>), asking why processing a sorted array is faster than processing an unsorted array. It fills an array with random numbers and then executes a loop with conditional jumps depending on the random data. These jumps are random and often mispredicted. 

```
1 /* mispredict.c */
          2 #include <stdlib.h>
          3 #define N 200000000
          4 int main(void)
          5 {
          6 	int *data = malloc(N * sizeof(int));
          7 	int i, k;
          8 	for (i = 0; i < N; i++)
          9 		data[i] = rand() % 256;
         10 
         11 	volatile int sum;
         12 	for (k = 0; k < 50; k++)
         13 		for (i = 0; i < N; i++)
         14 			if (data[i] >= 128)
         15 				sum += data[i];
         16 }
```

```
% gcc -g -o mispredict mispredict.c
        % perf record -b perf stat ./mispredict
    
         Performance counter stats for './mispredict':
    
    	23,293,062,786      branches                                                      (100.00%)
    	 5,008,912,919      branch-misses             #   21.50% of all branches          (100.00%)
    
        % perf report --sort symbol_from,symbol_to,mispredict --stdio
        ...
        # Overhead  Source Symbol    Target Symbol       Branch Mispredicted
        # ........  ...............  ..................  ...................
        #
    	53.46%  [k] main         [k] main            N                  
    	27.21%  [.] main         [.] main            N                  
    	17.22%  [.] main         [.] main            Y
```

Now we know that 17% of the total branches are mispredicted in `main()`. We could now attempt to fix them to speed up the program, such as by sorting the array first to make the `if` statement more predictable. Unfortunately, perf doesn't tell us directly on which line they occur, at least yet. 

#### LBR filtering

Previously we looked at all branches using LBRs, but the CPU also supports filtering the branch types, so that not all branch types are logged. This can be useful to get more context. `perf record` supports branch filtering using the `-j` option with the following types: 

> Name| Meaning  
> ---|---  
> `any_call`| any function call or system call  
> `any_ret`| any function return or system call return  
> `ind_call`| any indirect branch  
> `u`| only when the branch target is at the user level  
> `k`| only when the branch target is in the kernel  
> `hv`| only when the target is at the hypervisor level  
> `in_tx`| only when the target is in a hardware transaction  
> `no_tx`| only when the target is not in a hardware transaction  
> `abort_tx`| only when the target is a hardware transaction abort  
> `cond`| conditional branches  
  
For example: 

```
% perf record -j any_call,any_ret ...
```

would record any function calls and returns. Note that not all filter types are supported by all CPUs. If a filter is missing, perf will fall back to filtering in software (which does not increase the effective size of the LBR) and the hypervisor option is only supported on POWER CPUs currently. 

Filtering is mostly useful when LBRs are being used for debugging. There is an exception, which is described next, but we usually want to see all branches. 

#### LBR call graph

In many cases while profiling we want the function call graph for each sample (`perf record -g`); otherwise it is not clear why a function was executed. Traditionally this was implemented by using the frame pointer information set up by the compiler that resides on the stack. Since frame pointers can be somewhat expensive on some x86 CPUs, 64-bit binaries often don't include them; GCC defaults to not enabling them. This results in incomplete call graphs. 

Newer perf versions also support using the [DWARF](<https://en.wikipedia.org/wiki/DWARF>) unwind information to get the call graph, but this is quite expensive, since the stack needs to be copied for each sample, and it also doesn't always work. 

Since [Haswell](<https://en.wikipedia.org/wiki/Haswell_%28microarchitecture%29>), the LBRs have a new mode where the CPU logs every call and return into the LBR and treats them as a stack. This results in the CPU keeping track of the current call graph. Since version 3.19, perf supports this mode to collect this LBR-based call graph. In this case, it also saves/restores the LBRs in the context switch. 

```
% perf record --call-graph lbr program
        % perf report
```

There are some limitations; for example, C++ exceptions or user-space threading can corrupt the call stack. But for typical usage, it works quite well. This feature is especially useful with just-in-time (JIT) compilers that cannot generate frame pointers. 

#### Timed LBR

The [Skylake](<https://en.wikipedia.org/wiki/Skylake_%28microarchitecture%29>) CPUs, beyond just extending the number of LBRs to 32 entries, also added a new "timed LBR" feature. The CPU logs how many cycles occurred between the branches logged in the LBR. 

In an aggressive [out-of-order](<https://en.wikipedia.org/wiki/Out-of-order_execution>) CPU like Skylake, the time when something "occurred" can be a somewhat fuzzy concept: the CPU pipeline executes many instructions in parallel, and parts of the instructions in the basic block may have been executed before or after the branches. The LBR cycles are logged when the branches are issued. Still, the cycles provide a useful, rough indication how long the code area between the branches took. 

This allows much more fine-grained accounting of cycles than what is normally possible with sampling. Also, unlike manual instrumentation of programs with timing calls, it has only a minor and tunable overhead — only the sampling overhead, which can be tuned by lowering the sampling frequency.

[ ![\[Cycle annotation\]](https://static.lwn.net/images/2016/lbr-cycles-sm.png) ](<https://lwn.net/Articles/681346/>)

Starting with version 4.2, perf uses the timed LBR information in `perf report` and in `perf annotate` to report "instructions per cycle" and the average number of cycles for specific basic blocks. The aggregated average cycles per basic block are reported by `perf report` in the branch view at a function level. The interactive version of [`perf annotate`](<http://linux.die.net/man/1/perf-annotate>) (available when browsing samples through `perf report`) can also show the average cycles and instructions per cycle directly with the source and assembler code (see screen shot at right): 

```
% perf record -b ./div
        % perf report
        <navigate to a sample>
        <press right cursor key>
        <select annotate>
```

This example uses the [`div.c` program](<https://lwn.net/Articles/681389/>) from last week's article. The first column shows the average number of cycles for a block. In this case, the generated code jumps into the middle of a block, so we have an overlapping short and long block, but we see that the long block, which includes the two divisions, takes ~25 cycles on average. The short block takes about 6 cycles. The second column is the IPC (instructions per cycle) for the block. 

This allows analysis of how long it takes to execute blocks of instructions in real programs without having to write custom micro-benchmarks. 

#### Virtualization

LBRs are a model-specific feature and are normally not available in virtual machines since most hypervisors do not virtualize them. There is [some work](<http://comments.gmane.org/gmane.comp.emulators.kvm.devel/143279>) in KVM to support LBR virtualization (which has not been merged so far), but other hypervisors, such as Xen and VMware, do not support it. That means to use LBRs the workload currently has to run in a non-virtualized environment. 

LBRs work great in containers, though. 

#### Virtual LBR with Processor Trace

When using LBRs it would be often useful to have more entries than the 8-32 that are currently available to see more context. However, the CPU cannot provide more than it has implemented in hardware. There is a way of using [Processor Trace](<https://lwn.net/Articles/648154/>) (PT) to generate virtual, arbitrary-sized LBRs. PT records all jumps for a particular area in memory. The PT decoder can generate virtual perf samples from such a stream. Unlike normal LBRs, PT has more overhead. It requires a CPU with Processor Trace support (`grep intel_pt /proc/cpuinfo`), such as Broadwell or Skylake. 

```
% start program
        # capture 1 second of execution
        % perf record -e intel_pt//u -p $(pidof program) sleep 1
        % perf report --itrace=10usl60c --branch-history
```

This example samples the PT stream every 10μs with a call graph, and attaches the last 60 branches as LBR entries to each sample. Normally, the result is analyzed using `--branch-history`. This allows seeing much longer paths. Note that it is often not feasible to record long program executions, as it may generate data faster than the disk can keep up with. Virtual LBRs with PT have been added in perf version 4.5. 

#### Debugging

Last branch records can be also used for debugging to find out what happened before a crash. One problem is that the crash handler often "pollutes" the LBRs before their contents can be logged. The perf code is able to save the LBRs on each context switch, but there is currently no interface for a user-space debugger like GDB to access that information. 

Using them for system-wide debugging typically requires using LBR filtering or custom kernel changes to disable and re-enable them on exceptions. They work fairly well from JTAG debuggers (such as the [Intel System Debugger](<https://software.intel.com/en-us/intel-system-debugger>)) because the JTAG debugger does not execute branches and pollute the LBRs while executing. Generally Processor Trace is more versatile for debugging because the traces can provide much larger windows into program execution. 

#### Conclusion

LBRs are a powerful mechanism that can help with performance tuning and debugging. They can be used to look into TSX transactions and to get call stacks without the need for frame pointers. On recent CPUs they allow fine-grained timing of instruction blocks, often doing away with the need to write custom micro-benchmarks to understand performance at the instruction level. 

Perf already has good support for many performance tuning uses of LBRs. Some improvements, especially better support for resolving source lines and better display of hot paths, will hopefully be implemented in the future.
