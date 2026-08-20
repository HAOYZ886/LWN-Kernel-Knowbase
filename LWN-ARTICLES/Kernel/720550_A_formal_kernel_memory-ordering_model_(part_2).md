---
title: "A formal kernel memory-ordering model (part 2)"
url: https://lwn.net/Articles/720550/
date: "April 20, 2017"
category: Memory model
author: "April 20, 2017 By Jade Alglave, Luc Maranget, Paul E. McKenney, Andrea Parri, and Alan Stern"
---

April 20, 2017

By Jade Alglave, Luc Maranget, Paul E. McKenney, Andrea Parri, and Alan Stern

This article continues the description of the Linux-kernel memory model that started [here](<https://lwn.net/Articles/718628/>) as follows, with the intended audience for each section in parentheses: 

  1. Memory models and the role of cycles (people interested in using memory-ordering tools). 
  2. Specifying a memory model in terms of prohibited cycles (people interested in understanding the memory model). 
  3. Conclusions (all). 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

This is followed by the inescapable answers to the quick quizzes. 

###  Memory models and the role of cycles

One way of formalizing a memory model is to create an abstract description of how a running system operates internally, and then enumerate all the possible outcomes this abstract operation can give rise to. There are [tools that take this operational approach](<https://lwn.net/Articles/470681/>). Another way is to define the constraints imposed by the memory model, in the form of logical axioms, and then enumerate all the possible outcomes that are consistent with these constraints. A tool using this axiomatic approach (called "herd") is described [here](<https://lwn.net/Articles/608550/>). This tool can be [downloaded](<http://diy.inria.fr/sources/index.html>) and built as described in the `INSTALL.md` file. 

Both approaches take as input a small fragment of code and an assertion (together called a _litmus test_) and produce an output value indicating whether the memory model permits the code fragment to execute in a way that would make the assertion true. Here is a simple example of a litmus test (with line numbers added) that illustrates the so-called "message-passing" pattern: 

> [Litmus Test #1](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/C-MP+o-mb-o+o-mb-o.litmus>)

```
>       1 C C-MP+o-mb-o+o-mb-o
>       2
>       3 {
>       4 }
>       5
>       6 P0(int *x, int *y)
>       7 {
>       8   WRITE_ONCE(*y, 1);
>       9   smp_mb();
>      10   WRITE_ONCE(*x, 1);
>      11 }
>      12
>      13 P1(int *x, int *y)
>      14 {
>      15   int r1;
>      16   int r2;
>      17
>      18   r1 = READ_ONCE(*x);
>      19   smp_mb();
>      20   r2 = READ_ONCE(*y);
>      21 }
>      22
>      23 exists
>      24 (1:r1=1 /\ 1:r2=0)
>
```

Line 1 identifies the source language of the code fragment ("C") and gives the litmus test's name ("C-MP+o-mb-o+o-mb-o").  Lines 3 and 4 are where initial values could be provided. In this program no explicit initialization is needed, because all variables' initial values default to zero. Lines 6-21 provide the code, in this case, one function for each of two processors. You can choose any name you like for these functions as long as it consists of a "P" immediately followed by the processor's number, numbered consecutively starting from zero. By convention, local variable names begin with "`r`" (these variables are treated as though they are stored in CPU registers), and global variables must be passed in by reference as function parameters. The names of these function parameters are significant: they must match the names of the corresponding global variables. 

Finally, lines 23 and 24 provide an "`exists`" assertion expression to evaluate the final state. This final state is evaluated after the dust has settled: both processes have completed and all of their memory references and memory barriers have propagated to all parts of the system. The references to the local variables "`r1`" and "`r2`" in line 24 must be prefixed with "`1:`" to specify which processor they are local to. Note that a single "`=`" in this expression is an equality operator rather than an assignment (the assertion expression is written in the litmus-test language rather than in C). The "`/\`" character combination means "and"; it is an ASCII representation of the mathematical "∧" symbol. Similarly, "`\/`" stands for "or" (the mathematical "∨" symbol); this assertion could have been expressed just as well in negated form by writing: 

> 

```
>     23 forall
>     24 (1:r1=0 \/ 1:r2=1)
>
```

The "`~`" character indicates negation, so this assertion could also have been written in non-negated form as follows: 

> 

```
>     23 exists
>     24 ~(1:r1=0 \/ 1:r2=1)
>
```

The `herd` tool normally only prints variables that appear in the `exists` (or `forall`) clause. Additional variables can be printed using `locations`, which is demonstrated [here](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/Examples.html#litmus22>). 

The software tools mentioned above simply tell you whether the logic expression evaluates to true in all, some, or none of the possible executions of the code. Value judgments are left to the user. `herd` may be run using the `[linux.def](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/linux.def>)` macro file included in the source package, the [Litmus Test #1](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/C-MP+o-mb-o+o-mb-o.litmus>) source file, and the "[bell](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/strong-kernel.bell>)" and "[cat](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/strong-kernel.cat>)" files for the strong kernel memory model described [here](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/StrongModel.html>). The command is as follows: 

> 

```
>     herd7 -macros linux.def -bell strong-kernel.bell -cat strong-kernel.cat \
>           C-MP+o-mb-o+o-mb-o.litmus
>
```

For people who prefer shorter command lines, the [`strong.cfg`](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/strong.cfg>) configuration file specifies these settings already, along with several others related to the style of the plot files `herd` is capable of producing. The command using this configuration file is: 

> 

```
>     herd7 -conf strong.cfg C-MP+o-mb-o+o-mb-o.litmus
>
```

The output from either command is: 

> Outcome for [Litmus Test #1 (strong model)](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/C-MP+o-mb-o+o-mb-o.litmus>)

```
>      1 Test C-MP+o-mb-o+o-mb-o Allowed
>      2 States 3
>      3 1:r1=0; 1:r2=0;
>      4 1:r1=0; 1:r2=1;
>      5 1:r1=1; 1:r2=1;
>      6 No
>      7 Witnesses
>      8 Positive: 0 Negative: 3
>      9 Condition exists (1:r1=1 /\ 1:r2=0)
>     10 Observation C-MP+o-mb-o+o-mb-o Never 0 3
>     11 Hash=3240a31645e46554cb09739d726087ad
>
```

This output indicates the three possible outcomes from running this code in the Linux kernel: 

  1. `r1 == 0 && r2 == 0`. This outcome occurs when P1 completes before P0 begins. 
  2. `r1 == 0 && r2 == 1`. This outcome occurs when P1 executes concurrently with P0, so that P1's read from `x` executes before P0's write to `x`, but P1's read from `y` executes after P0's write to `y`. 
  3. `r1 == 1 && r2 == 1`. This outcome occurs when P1 starts after P0 completes. 

The outcome `r1 == 1 && r2 == 0` is not possible, as indicated by the "`Never 0 3`" near the end of the output. Here, "Never" indicates that the test condition is true; "0 3" reinforces that by saying that the condition was found to be true in zero possible outcomes and false in three outcomes. This forbidden outcome would require a cycle of events, each happening before the next and the last happening before the first: 

  1. P0 writes to `x`, 
  2. P1 reads from `x`, 
  3. P1 reads from `y`, and 
  4. P0 writes to `y`. 
This cycle is illustrated in the following figure: 

> ![cycle.png](https://static.lwn.net/images/2017/mm-model/cycle-new.png)

Please note that the concept of cycles can be thought of as a mathematically precise generalization of the `memory-barriers.txt` concept of memory-barrier pairing. The labels in the diagram are defined as follows: 

  1. `fr` = "from-read", linking each read to any writes to the same variable that execute too late to affect the value returned by that read. 
  2. `po` = "program order", linking statements within a given process in the order that they appear in the instruction stream. 
  3. `rf` = "reads from", linking a given write to any reads that load the value stored by that write. 

The `fr` relation can be somewhat counter-intuitive, so please look [here](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/fr.html>) for additional explanation. The `herd` tool provides many additional relations, which are tabulated [here](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/herd.html#built-in relations>). 

It is important to note that not all cycles are prohibited. To see this, consider the following: 

> [Litmus Test #2](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/C-MP+o-o+o-o.litmus>)

```
>       1 C C-MP+o-o+o-o
>       2
>       3 {
>       4 }
>       5
>       6 P0(int *x, int *y)
>       7 {
>       8   WRITE_ONCE(*y, 1);
>       9   WRITE_ONCE(*x, 1);
>      10 }
>      11
>      12 P1(int *x, int *y)
>      13 {
>      14   int r1;
>      15   int r2;
>      16
>      17   r1 = READ_ONCE(*x);
>      18   r2 = READ_ONCE(*y);
>      19 }
>      20
>      21 exists
>      22 (1:r1=1 /\ 1:r2=0)
>
```

This is exactly the same as the previous litmus test except that the `smp_mb()` calls have been removed. Despite the fact that the outcome `r1 == 1 && r2 == 0` exhibits the same cycle as above, it can in fact occur on weakly ordered systems where, for example, P0's writes and P1's reads can be reordered by the hardware. On such systems, the `smp_mb()` statements are necessary to ensure that the order of execution of the writes and reads is the same as their order in the source code. This can be confirmed by running the tool in the same way as before, but on the new litmus test: 

> 

```
>     herd7 -conf strong.cfg C-MP+o-o+o-o.litmus
>
```

The output will be as follows: 

> Outcome for [Litmus Test #2 (strong model)](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/C-MP+o-o+o-o.litmus>)

```
>      1 Test C-MP+o-o+o-o Allowed
>      2 States 4
>      3 1:r1=0; 1:r2=0;
>      4 1:r1=0; 1:r2=1;
>      5 1:r1=1; 1:r2=0;
>      6 1:r1=1; 1:r2=1;
>      7 Ok
>      8 Witnesses
>      9 Positive: 1 Negative: 3
>     10 Condition exists (1:r1=1 /\ 1:r2=0)
>     11 Observation C-MP+o-o+o-o Sometimes 1 3
>     12 Hash=c3bdaae6256fa364ad31fb3c1e07c0f5
>
```

Note that all four possible states are present, and note also the "`Sometimes 1 3`" near the end of the output. 

**Quick Quiz 1** : Can't the compiler also reorder these accesses?   
Answer

On sufficiently weakly ordered systems, the cyclic outcome in Litmus Test #2 could occur even without instruction reordering, because the writes might not propagate from P0 to P1 in the order they were executed. And even on more strongly ordered systems, it would be sufficient to reorder either the reads or the writes; it is not necessary to reorder both. For example, if P1's accesses were reordered then we could have the following sequence of events: 

  1. P1 reads from `y`, 
  2. P0 writes to `y`, 
  3. P0 writes to `x`, and 
  4. P1 reads from `x`. 
This sequence says that P1 reads from `y` before reading from `x`, i.e., the reads are reordered. If this were to happen, there would no longer be a cycle, as indicated in the following diagram (the dotted arrow to the right indicates P1's reordering): 

> ![cyclenot.png](https://static.lwn.net/images/2017/mm-model/cyclenot-new.png)

This illustrates an important point: cycles in time of instruction execution are impossible, because time is linearly ordered (in our universe, even if [not in all solutions to Einstein's field equations](<https://en.wikipedia.org/wiki/G%C3%B6del_metric>)). Part of a memory model's job is to provide the conditions under which one instruction must execute before another and to check for any resulting cycles. On the other hand, if there is no such cycle then it is possible to find an order of execution for all the instructions which is compatible with the memory model's ordering requirements (for example, by doing a topological sort). If this potential execution order did not violate any of the memory model's other requirements, it would demonstrate that the litmus test's assertion could hold. 

Okay, we admit the preceding paragraph is an oversimplification. Modern CPUs do not execute instructions at precise moments in time; instead they run instructions through complicated multi-stage pipelines and engage in multiple issue (running more than one instruction through the same pipeline stages in parallel). Furthermore, other ordering requirements come into play along with time of execution, such as cache coherence (see below). Nevertheless, the basic idea is valid. 

It is worth pointing out that computer hardware almost always has additional restrictions beyond what the memory models describe; CPU designers generally do not implement all of the behaviors allowed by the instruction set architecture. The fact that a memory model says a particular litmus test's assertion might hold does not mean it can actually happen on any given computer. As a simple example, the finite write buffers found in real hardware prevent that hardware from actually doing all the reorderings of writes that memory models typically allow. It also goes the other way--sometimes CPU designers mistakenly implement a behavior that is prohibited by the instruction set architecture (otherwise known as a "silicon bug" or "CPU erratum"). 

###  Specifying a memory model in terms of prohibited cycles

As we have just seen, there is a close relationship between orderings and the existence of cycles: If some events are constrained to be ordered in a certain way then that ordering cannot contain a cycle. Conversely, if a given relation among various events does not contain any cycles then it is possible to order those events consistently with the relation. Thus, if we can precisely specify which instructions must execute before others in a given piece of Linux kernel code, we will be well on our way to constructing a formal model that defines the kernel's execution-ordering guarantees in terms of cycles among instructions. Even better, this model can then be used to construct a tool that analyzes litmus tests for execution-ordering problems. (And of course, the same technique can be used for describing a memory model's other ordering requirements.) 

The `herd` tool implements a language, called `cat`, designed to represent memory models, which it does by specifying what cycles are prohibited. This specification is defined in terms of sets and relations involving memory-access events, barriers, and threads. (For our purposes, each processor in a litmus test corresponds to a distinct thread.) `herd` is discussed in more detail [here](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/herd.html>); in this section we will see how to write some simple memory models in the `cat` language. 

But first, what cycles should the Linux kernel memory model prohibit? Here is a partial list: 

  * Placing a full memory barrier (`smp_mb()`) between each pair of memory accesses in each process will prohibit all cycles. In other words, `smp_mb()` can be said to _restore sequential consistency_ , that is, to ensure that all processors agree on a global order of all memory accesses by all processors. The next section shows example `herd` code that accomplishes this. 
  * So-called _[out-of-thin-air](<http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4375.html>)_ computations must be ruled out. These are cycles where each CPU writes a value depending on the value written by the previous CPU in the cycle, and they are handled by the [Happens-Before](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/StrongModel.html#Cat File: Happens-Before>) section of the model's [cat](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/StrongModel.html#Strong-Model Cat File>) file. The underlying reason why the model does not produce out-of-thin-air values is that any `WRITE_ONCE()` that depends on a `READ_ONCE()` is ordered after it, regardless of the type of dependency. 
  * All CPUs should agree on the order of accesses to any single memory location. Making this happen is described below. 
  * A chain of release-acquire pairs, where each load-acquire returns the value stored by the preceding store-release, should never form a cycle. The strong model's bell and cat files prohibit such cycles. 
  * Cycles violating [RCU's guarantees](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/RCUguarantees.html>) must be prohibited. This is handled by the RCU-specific portions of the [bell](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/StrongModel.html#Bell File: RCU Read-Side Critical Sections>) and [cat](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/StrongModel.html#Cat File: RCU>) files. 

There are quite a few additional nuances of Linux-kernel use cases and peculiarities of specific hardware, but this list provides a good starting point. The following sections present trivial "toy" memory models that prohibit the first two types of cycles. 

####  Relaxed memory order: toy specification

The following shows a simple `herd` program that represents a fragment of the Linux kernel memory model involving simple memory accesses (`READ_ONCE()` and `WRITE_ONCE()`) and strong memory barriers (`smp_mb()`): 

> [toy-RMO.cat](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/toy-RMO.cat>)

```
>       1 "Toy RMO"
>       2
>       3 include "cos.cat"
>       4
>       5 let rfe = rf & ext
>       6 let fence = fencerel(F)
>       7
>       8 let rmo-order = fence | rfe | co | fr
>       9 acyclic rmo-order
>
```

Line 1 provides a name for the model, and line 3 pulls in some definitions that can be thought of as the `herd` equivalent to the C-language: 

> 

```
>     #include <stdio.h>
>
```

However, instead of defining I/O primitives, "`cos.cat`" defines some basic relations, including the `fr` relation mentioned earlier. 

For Litmus Test #2 and Litmus Test #1 above (assuming the cyclic execution), the built-in `rf` ("reads-from") relation contains the following links: 

  * `WRITE_ONCE(*x, 1)` ⟶ `r1 = READ_ONCE(*x)`, 
  * `INIT(*y, 0)` ⟶ `r2 = READ_ONCE(*y)`. 
(where `INIT(*y, 0)` is the "write" that initializes `y`), and `fr` ("from-read") contains: 

  * `r1 = READ_ONCE(*y)` ⟶ `WRITE_ONCE(*y, 1)` 
`cos.cat` also defines the `co` ("coherence order") relation, which links each write to all later writes to the same variable (just the writes, not the reads). Initialization counts as a write; it is always the first write in the coherence order for each variable. Thus the `co` relation for these litmus tests looks like this: 

  * `INIT(*x, 0)` ⟶ `WRITE_ONCE(*x, 1)`
  * `INIT(*y, 0)` ⟶ `WRITE_ONCE(*y, 1)` 

Line 5 computes `rfe` ("reads-from external"), which is a restricted version of the `rf` relation that covers only write-read pairs where the write and the read are executed by different threads. It does this by intersecting (the `&` operator) the `rf` relation with the predefined `ext` relation, which links all pairs of instructions belonging to different threads. For the two litmus tests above, the `rfe` relation turns out to be exactly the same as the `rf` relation. 

Line 6 uses the standard `fencerel()` function and `F` event set to define a relation that links any two instructions separated by a memory barrier. For Litmus Test #2, which contains no instances of `smp_mb()`, this relation is empty. For Litmus Test #1, it contains the following links: 

  * `WRITE_ONCE(*y, 1)` ⟶ `WRITE_ONCE(*x, 1)`
  * `r1 = READ_ONCE(*x)` ⟶ `r2 = READ_ONCE(*y)` 

Line 8 defines the `rmo-order` relation as the union (the `|` operator) of the `fence`, `rfe`, `co`, and `fr` relations. `rmo-order` includes all pairs of instructions for which this toy model of relaxed memory order (RMO) requires the first to execute before the second. Line 9 expresses this requirement by stating that the `rmo-order` relation is acyclic (contains no cycles). 

For Litmus Test #2, `rmo-order` does not contain a cycle, as shown below: 

> ![rmo-acyclic.png](https://static.lwn.net/images/2017/mm-model/rmo-acyclic.png)

(The dotted "po" edges are for illustration only; they are not present in the `rmo-order` relation and do not contribute to any cycles.) 

On the other hand, for Litmus Test #1, the additional links added by the `fence` relation do create a cycle: 

> ![rmo-cyclic.png](https://static.lwn.net/images/2017/mm-model/rmo-cyclic.png)

Thus this model correctly distinguishes the "message-passing" examples with and without memory barriers, as can be seen by downloading [toy-RMO.cat](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/toy-RMO.cat>) and passing it via the `-cat` command-line argument for Litmus Test #2 as follows: 

> 

```
>     herd7 -conf strong.cfg -cat toy-RMO.cat C-MP+o-o+o-o.litmus
>
```

This produces the following output: 

> Outcome for [Litmus Test #2 (toy-RMO model)](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/C-MP+o-o+o-o.litmus>)

```
>      1 Test C-MP+o-o+o-o Allowed
>      2 States 4
>      3 1:r1=0; 1:r2=0;
>      4 1:r1=0; 1:r2=1;
>      5 1:r1=1; 1:r2=0;
>      6 1:r1=1; 1:r2=1;
>      7 Ok
>      8 Witnesses
>      9 Positive: 1 Negative: 3
>     10 Condition exists (1:r1=1 /\ 1:r2=0)
>     11 Observation C-MP+o-o+o-o Sometimes 1 3
>     12 Hash=c3bdaae6256fa364ad31fb3c1e07c0f5
>
```

Given the lack of a cycle in the `rmo-order` relationship, the counter-intuitive cyclic execution is permitted, as indicated by "`Sometimes 1 3`" in the output. In contrast, for Litmus Test #1, with memory barriers, the command line: 

> 

```
>     herd7 -conf strong.cfg -cat toy-RMO.cat C-MP+o-mb-o+o-mb-o.litmus
>
```

produces the following output: 

> Outcome for [Litmus Test #1 (toy-RMO model)](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/C-MP+o-mb-o+o-mb-o.litmus>)

```
>      1 Test C-MP+o-mb-o+o-mb-o Allowed
>      2 States 3
>      3 1:r1=0; 1:r2=0;
>      4 1:r1=0; 1:r2=1;
>      5 1:r1=1; 1:r2=1;
>      6 No
>      7 Witnesses
>      8 Positive: 0 Negative: 3
>      9 Condition exists (1:r1=1 /\ 1:r2=0)
>     10 Observation C-MP+o-mb-o+o-mb-o Never 0 3
>     11 Hash=3240a31645e46554cb09739d726087ad
>
```

As expected, the memory barriers exclude the counter-intuitive outcome where `r1 == 1 && r2 == 0`. 

####  Relaxed memory order: coherence included

Consider this ridiculous single-thread litmus test: 

> [Litmus Test #3](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/C-CO+o-o.litmus>)

```
>       1 C C-CO+o-o.litmus
>       2
>       3 {
>       4 }
>       5
>       6 P0(int *x)
>       7 {
>       8   *x = 3;
>       9   *x = 4;
>      10 }
>      11
>      12 exists
>      13 (x=3)
>
```

On the face of it, this test can never succeed. If we set `x` to 3 and then overwrite it with the value 4, how can `x` possibly end up containing 3? Nevertheless, running the [Toy RMO](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/toy-RMO.cat>) model shows that this outcome is permitted: 

> Outcome for [Litmus Test #3 (toy-RMO model)](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/C-CO+o-o.litmus>)

```
>      1 Test C-CO+o-o Allowed
>      2 States 2
>      3 x=3;
>      4 x=4;
>      5 Ok
>      6 Witnesses
>      7 Positive: 1 Negative: 1
>      8 Condition exists (x=3)
>      9 Observation C-CO+o-o Sometimes 1 1
>     10 Hash=b9e4f0d747854e10ad7310b4381f3652
>
```

This is because the model does not forbid it, and everything that is not explicitly forbidden is permitted. The model does not account for cache coherence, a feature supported by most modern microprocessors--and demanded by the vast majority of sane kernel hackers. That's one reason why this model should be considered to be a toy. 

_Cache coherence_ (sometimes referred to as "per-location sequential consistency") requires that the writes to any one location in memory occur in a single total order (the coherence order), which all the processors must agree on. It also says that within each thread, the coherence order must be consistent with the program order, as described by the following four coherence rules: 

  * _Write-write coherence:_ If two writes in the same thread access the same location, the write that comes first in program order must come first in the coherence order for that location. 
  * _Write-read coherence:_ If a write W precedes (in program order) a read R of the same location, then R must read from W or from a write that occurs after W in the location's coherence order. 
  * _Read-write coherence:_ If a read R precedes (in program order) a write W of the same location, then R must read from a write that occurs before W in the location's coherence order. 
  * _Read-read coherence:_ If two reads R and R' in the same thread access the same location, where R comes before R' in program order, either they must read from the same write or else the write read by R must occur before the write read by R' in the location's coherence order. 

In Litmus Test #3 above, there are three writes to the location where `x` is stored: the initializing write of 0 (implicit in lines 3-4), and the writes of 3 and 4 (lines 8-9). The initializing write always comes first in the coherence order, and the value tested in the "exists" clause is always the value stored by the write that comes last in the coherence order (called the _final write_). Thus for the test to succeed, the coherence order for `x` would have to be: `x=0`, `x=4`, `x=3`. But this would violate the write-write coherence rule, because the write that sets `x` to 3 comes before (in program order) the write that sets it to 4. 

(Note: The C11 standard recognizes the notion of _sequenced-before_ rather than that of program order. For the most part the two are the same, referring to the order in which loads and stores occur in the source code, but there are a few differences. For example, the compiler is not required to evaluate the arguments to a function call in any particular order. Thus, even though the statement: 

```
printf("%d %d", WRITE_ONCE(x, 3), WRITE_ONCE(x, 4));
```

will always print out "3 4", after it executes `x` may be equal either to 3 or 4. We will not worry such subtleties for now. But we will point out that in Litmus Test #3, the "`*x = 3`" write _is_ sequenced before the "`*x = 4`" write, and the compiler is not permitted to reorder them. That is why we have omitted the `WRITE_ONCE()` calls and reverted to plain ordinary assignment. It's okay in this case, because `x` isn't shared between processors and we're only trying to make a simple point. But note that even with this two-line test program, the compiler is permitted to eliminate the "`*x = 3`" write entirely.) 

Our Toy RMO memory model can be strengthened to take cache coherence into account. Here is the result: 

> [coherent-RMO.cat](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/coherent-RMO.cat>)

```
>       1 "Coherent RMO"
>       2
>       3 include "cos.cat"
>       4
>       5 let rfe = rf & ext
>       6 let fence = fencerel(F)
>       7
>       8 let rmo-order = fence | rfe | co | fr
>       9 acyclic rmo-order
>      10
>      11 let com = rf | co | fr
>      12 let coherence-order = po-loc | com
>      13 acyclic coherence-order
>
```

**Quick Quiz 2** : The `rf`, `co`, and `fr` terms in the definition of `com` describe write-read, write-write, and read-write links respectively, corresponding to three of the four coherence rules. Why is there no term corresponding to the read-read rule?   
Answer

Aside from the name change on line 1, the only difference is the addition of lines 10-13. Line 11 defines the `com` relation as the union of the `rf`, `co`, and `fr` relations. If you imagine inserting reads into the coherence order for a variable, by placing each read between the write that it reads from and the following write, you'll see that in each case `com` links a memory access to one that comes later in the coherence order. (`com`'s name arises from the fact that it describes the ways different processors can _com_ municate by writing to and reading from shared variables in memory.) 

`po-loc` in line 12 is another standard relation; it is the intersection of `po` and `loc`, where the `loc` relation links all pairs of memory accesses that refer to the same location in memory. Thus, `po-loc` links each memory access to all those that occur after it in program order and access the same variable. Lines 12-13 go on to define `coherence-order` as the union of `po-loc` and `com` and to require that `coherence-order` not have any cycles. 

Since Litmus Test #3 contains no reads, its `rf` and `fr` relations are empty and therefore `com` ends up being the same as `co`. In the non-intuitive execution accepted by the Toy RMO model (where `x=3` comes last in the coherence order), `com` contains the following links: 

  * `INIT(*x, 0)` ⟶ `*x = 3`, 
  * `INIT(*x, 0)` ⟶ `*x = 4`, and 
  * `*x = 4` ⟶ `*x = 3`, 
while `po-loc` contains only: 

  * `*x = 3` ⟶ `*x = 4`. 
Putting these together yields an obvious cycle in `coherence-order`, which causes the Coherent RMO model to forbid the counter-intuitive outcome in Litmus Test #3: 

> Outcome for [ Litmus Test #3 (coherent-RMO model)](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/C-CO+o-o.litmus>)

```
>      1 Test C-CO+o-o Allowed
>      2 States 1
>      3 x=4;
>      4 No
>      5 Witnesses
>      6 Positive: 0 Negative: 1
>      7 Condition exists (x=3)
>      8 Observation C-CO+o-o Never 0 1
>      9 Hash=b9e4f0d747854e10ad7310b4381f3652
>
```

Here's a slightly more sophisticated test that probes the read-read coherence rule: 

> [Litmus Test #4](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/C-CO+o-o+o-o.litmus>)

```
>       1 C C-CO+o-o+o-o.litmus
>       2
>       3 {
>       4 }
>       5
>       6 P0(int *x)
>       7 {
>       8   WRITE_ONCE(*x, 3);
>       9   WRITE_ONCE(*x, 4);
>      10 }
>      11
>      12 P1(int *x)
>      13 {
>      14   int r1;
>      15   int r2;
>      16
>      17   r1 = READ_ONCE(*x);
>      18   r2 = READ_ONCE(*x);
>      19 }
>      20
>      21 exists
>      22 (1:r1=4 /\ 1:r2=3)
>
```

Because of the write-write coherence rule, we know that the coherence order for `x` must be: `x=0`, `x=3`, `x=4`. If `r1` and `r2` were to end up equal to 4 and 3 respectively, it would mean the later read (in program order) had read from the earlier write (in `x`'s coherence order), thereby violating read-read coherence. 

To see why the Coherent RMO model forbids this result, consider how the various relations would turn out. Because `x=4` must come last in the coherence order for `x`, the `co` relation contains these links: 

  * `INIT(*x, 0)` ⟶ `WRITE_ONCE(*x, 3)`, 
  * `INIT(*x, 0)` ⟶ `WRITE_ONCE(*x, 4)`, and 
  * `WRITE_ONCE(*x, 3)` ⟶ `WRITE_ONCE(*x, 4)`. 
Since there are some read instructions in this test, the `rf` and `fr` relations are non-empty. The links in `rf` are: 

  * `WRITE_ONCE(*x, 4)` ⟶ `r1 = READ_ONCE(*x)` and 
  * `WRITE_ONCE(*x, 3)` ⟶ `r2 = READ_ONCE(*x)`, 
while `fr` contains only: 

  * `r2 = READ_ONCE(*x)` ⟶ `WRITE_ONCE(*x, 4)`. 
Finally, `po-loc` contains: 

  * `WRITE_ONCE(*x, 3)` ⟶ `WRITE_ONCE(*x, 4)` and 
  * `r1 = READ_ONCE(*x)` ⟶ `r2 = READ_ONCE(*x)`. 

Putting these together shows that `coherence-order` contains the following length-3 cycle: 

  1. `r2 = READ_ONCE(*x)`
  2. `WRITE_ONCE(*x, 4)`
  3. `r1 = READ_ONCE(*x)` 
The links in this cycle are `fr` followed by `rf` followed by `po-loc`, as shown in this figure: 

> ![](https://static.lwn.net/images/2017/mm-model/read-read-coherence.png)

As can be seen in the following `herd` output, this cycle is prohibited: 

> Outcome for [Litmus Test #4 (strong model)](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/C-CO+o-o+o-o.litmus>)

```
>      1 Test C-CO+o-o+o-o Allowed
>      2 States 6
>      3 1:r1=0; 1:r2=0;
>      4 1:r1=0; 1:r2=3;
>      5 1:r1=0; 1:r2=4;
>      6 1:r1=3; 1:r2=3;
>      7 1:r1=3; 1:r2=4;
>      8 1:r1=4; 1:r2=4;
>      9 No
>     10 Witnesses
>     11 Positive: 0 Negative: 6
>     12 Condition exists (1:r1=4 /\ 1:r2=3)
>     13 Observation C-CO+o-o+o-o Never 0 6
>     14 Hash=e28b27408fda33a59c7f2cd8a5ff7615
>
```

**Quick Quiz 3** : But don't Itanium and SPARC RMO allow read-read reordering of acccesses to a single variable by a single CPU? How does the model handle these CPUs?   
Answer

**Quick Quiz 4** : Whatever happened to memory-barrier pairing???   
Answer

**Exercise:** Assuming only that the `co` relation gives a total ordering of all writes to a particular memory location, prove that any cache-coherent execution of any program (i.e., an execution that obeys the four coherence rules) results in a `coherence-order` relation without cycles. And conversely, prove that if an execution does violate any of the coherence rules then its `coherence-order` relation does contain a cycle. 

This background will help you to understand the strong memory model itself. 

### Conclusions

We have presented a Linux-kernel memory model that we hope will be useful for education, concurrent design, and for inclusion in other tooling. As far as we know, this is the first realistic formal memory model that includes RCU ordering properties. In addition, we believe this to be the first realistic formal memory model of the Linux kernel. 

This model is not set in stone, but subject to change with the evolution of hardware and of Linux-kernel use cases. We expect the change rate to be roughly similar to the historical change rate of `Documentation/memory-barriers.txt`, however, we believe that the [guiding principles](<https://lwn.net/Articles/718628/#Guiding Principles>) underlying this memory model will be more durable. 

The strong model accepts significant complexity to attain greater strength. In contrast, the [weak model](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/WeakModel.html>) accepts some weakenings in order to achieve some degree of simplicity. Candidate weakenings include: 

  1. Omitting C11 release sequences. 
  2. Simplifying the preserved-program-order relations by omitting the `po-loc`, `rd-rdw`, `detour`, and `rdw` relations from them. 
  3. Omitting B-cumulativity, which has the effect of allowing some cycles as exemplified by [this litmus test](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/C-wmb-is-B-cumulative.litmus>). 
  4. Retaining a less-ornate version of the `obs` (observation) relation, which is called `short-obs`. 
  5. Retaining a less-ornate version of the `cpord` (coherence-point ordering) relation. This allows write-only cycles as exemplified by [this litmus test](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/C-2+2W+o-wmb-o+o-wmb-o.litmus>). 

Although we expect that this memory model will prove quite valuable, it does have a few limitations in addition to those called out earlier [here](<https://lwn.net/Articles/718628/#Support Existing Non-Buggy Linux-Kernel Code>) and [here](<https://lwn.net/Articles/718628/#Be Compatible with Hardware Supported by the Linux Kernel>): 

  1. These memory models do not constitute official statements by the various CPU vendors on their respective architectures. For example, any of these vendors might report bugs at any time against any version of this memory model. This memory model is therefore not a substitute for a carefully designed and vigorously executed validation regime. In addition, this memory model is under active development and might change at any time. 
  2. It is quite possible that this memory model will disagree with CPU architectures or with real hardware. For example, the model might well choose to allow behavior that all CPUs forbid if forbidding that behavior would render the model excessively complex. On the other hand, any situation where the model forbids behavior that some CPU allows constitutes a bug, either in the model or in the CPU. 
  3. This tool is exponential in nature. Litmus tests that seem quite small compared to the entire Linux kernel might well take geologic time for the `herd` tool to analyze. That said, this tool can be extremely effective in exhaustively analyzing the code at the core of a synchronization primitive. 
  4. The `herd` tool can only detect problems for which you have coded an assertion. This weakness is common to all formal methods, and is one reason that we expect testing to continue to be important. In [the immortal words of Donald Knuth](<https://cs.stanford.edu/~uno/faq.html>): "Beware of bugs in the above code; I have only proved it correct, not tried it." 

On the other hand, one advantage of formal memory models is that tools based on them can detect any problem that might occur, even if the probability of occurrance is vanishingly small, in fact, even if current hardware is incapable of making that problem happen. Use of tools based on this memory model is therefore an excellent way to future-proof your code. 

### Acknowledgments

We owe thanks to H. Peter Anvin, Will Deacon, Andy Glew, Derek Williams, Leonid Yegoshin, and Peter Zijlstra for their patient explanations of their respective systems' memory models. We are indebted to Peter Sewell, Susmit Sarkar, and their groups for their seminal work formalizing many of these same memory models. We all owe thanks to Dmitry Vyukov, Boqun Feng, and Peter Zijlstra for their help making this human-readable. We are also grateful to Michelle Rankin and Jim Wasko for their support of this effort. 

###  Answers to quick quizzes

**Quick Quiz 1** : Can't the compiler also reorder these accesses? 

**Answer** : Given the current Linux-kernel definitions of `READ_ONCE()` and `WRITE_ONCE()`, no. These two macros map to volatile accesses, which the compiler is not allowed to reorder with respect to each other. 

However, if these macros instead mapped to non-volatile C11 `memory_order_relaxed` loads and stores, then the compiler _would_ be permitted to reorder them. And, as a general rule, compilers are much more aggressive about reordering accesses than even the most weakly ordered hardware. In both cases, those who don't like such code rearrangement call it "weak ordering" while those who do call it "optimization". 

**Back to Quick Quiz 1**.

**Quick Quiz 2** : The `rf`, `co`, and `fr` terms in the definition of `com` describe write-read, write-write, and read-write links respectively, corresponding to three of the four coherence rules. Why is there no term corresponding to the read-read rule? 

**Answer** : It's not needed. As we will see in the discussion of [Litmus Test #4](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/C-CO+o-o+o-o.litmus>), a violation of the read-read coherence rule involves a write being "interposed" between two reads in the coherence order. It therefore can be described as a length-3 cycle in `coherence-order`, involving an `fr` link followed by an `rf` link followed by a `po-loc` link. 

**Back to Quick Quiz 2**.

**Quick Quiz 3** : But don't Itanium and SPARC RMO allow read-read reordering of acccesses to a single variable by a single CPU? How does the model handle these CPUs? 

**Answer** : In the case of Itanium, `gcc` compiles volatile reads (as in `READ_ONCE()`) as `ld,acq`, which enforces read-read ordering. And the Linux kernel runs SPARC in TSO mode, which prohibits read-read reorderings in general, including to a single variable. 

**Back to Quick Quiz 3**.

**Quick Quiz 4** : Whatever happened to memory-barrier pairing??? 

**Answer** : Memory-barrier pairing can be thought of as a special case of cycles, but it was designed for a simpler time when people used much less aggressive lockless designs. Here is an example that breaks memory-barrier pairing: 

> [Litmus Test #5](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/C-R+o-wmb-o+o+mb+o.litmus>)

```
>       1 C C-R+o-wmb-o+o+mb+o.litmus
>       2
>       3 {
>       4 }
>       5
>       6 P0(int *a, int *b)
>       7 {
>       8   WRITE_ONCE(*a, 1);
>       9   smp_wmb();
>      10   WRITE_ONCE(*b, 1);
>      11 }
>      12
>      13 P1(int *a, int *b)
>      14 {
>      15   int r1;
>      16
>      17   WRITE_ONCE(*b, 2);
>      18   smp_mb();
>      19   r1 = READ_ONCE(*a);
>      20 }
>      21
>      22 exists
>      23 (b=2 /\ 1:r1=0)
>
```

Because the `smp_wmb()` orders writes and because the `smp_mb()` orders everything, straightforward application of memory-barrier pairing would lead you to believe that this cycle would be forbidden. This belief would be incorrect, as can be seen from running the litmus test against the strong model: 

> Outcome for [Litmus Test #5 (strong model)](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/C-R+o-wmb-o+o+mb+o.litmus>)

```
>      1 Test C-R+o-wmb-o+o+mb+o Allowed
>      2 States 4
>      3 1:r1=0; b=1;
>      4 1:r1=0; b=2;
>      5 1:r1=1; b=1;
>      6 1:r1=1; b=2;
>      7 Ok
>      8 Witnesses
>      9 Positive: 1 Negative: 3
>     10 Condition exists (b=2 /\ 1:r1=0)
>     11 Observation C-R+o-wmb-o+o+mb+o Sometimes 1 3
>     12 Hash=0a4dd1e17f6132a7145a13b711ccd167
>
```

The problem is that the `co` relationship between `P0()`'s and `P1()`'s stores does not imply any sort of causal or temporal relationship between the two stores. Real hardware can and does chose the coherence ordering after the fact, and so real hardware can and does allow the cycle. 

In short, memory-barrier pairing was useful in its day, but its day is rapidly drawing to a close. More sophisticated use of lockless algorithms requires more sophisticated mental models of memory barriers. 

**Back to Quick Quiz 4**.
