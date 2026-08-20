---
title: Validating Memory Barriers and Atomic Instructions
url: https://lwn.net/Articles/470681/
date: "December 6, 2011"
category: Development tools
author: "December 6, 2011 This article was contributed by Paul McKenney"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

December 6, 2011

This article was contributed by Paul McKenney

Some of the most obscure corners of the Linux kernel are those employing the dark arts of atomic instructions and memory barriers. Although things are better than they were (say) ten years ago, atomic instructions and memory barriers are still a not-infrequent source of confusion. 

One of the difficulties with atomic instructions and especially with memory barriers is that the documentation for the underlying machine instructions can be a bit difficult to understand. It is not hard to work around obtuse documentation for arithmetic instructions: Simply test them. However, simple testing does not work for memory barriers and atomic instructions because these instructions must handle non-deterministic situations. Furthermore, unlike arithmetic instructions, a given piece of hardware is free to be more strict that absolutely necessary in its interpretation of ordering requirements. Therefore, your test results might not apply to the next generation of that same hardware, which in turn might result in your code breaking horribly. 

**Quick Quiz 1** : Yeah, right!!! Since when has anything ever made the Linux kernel's memory barriers and atomic instructions easier??? Answer

What we need is a tool that evaluates a code sequence containing atomic operations and memory barriers, telling the developer whether or not a given assertion is guaranteed never to trigger. Given such a guarantee, the developer could then be reasonably confident that this code sequence would operate correctly on all current and future versions of the hardware in question. 

Previous LWN articles ([here](<http://lwn.net/Articles/243851/>) and [here](<http://lwn.net/Articles/279077/>)), as well as Mathieu Desnoyers's [doctoral dissertation [PDF]](<http://www.lttng.org/pub/thesis/desnoyers-dissertation-2009-12.pdf>), describe how to use `Promela` and `spin` to test assertions on code fragments that execute concurrently. Because `Promela` and `spin` search the code fragments' full state space, the developer can be reasonably confident that the code fragments will operate correctly on real hardware. Unfortunately, most code fragments using atomic instructions and memory barriers require detailed knowledge of the semantics of a given CPU's memory barriers and atomic instructions. `Promela` and `spin` do not have such knowledge, so the developer must explicitly write `Promela`/`spin` code that captures this detailed knowledge. This is not helpful if the developer in question does not possess this detailed knowledge in the first place. 

It would clearly be much better to have a tool that already understands this detailed knowledge. I am happy to report that such tools are now becoming available, for example, from Peter Sewell and Susmit Sarkar at the University of Cambridge, Luc Maranget, Francesco Zappa Nardelli, and Pankaj Pawan at INRIA, and Jade Alglave at Oxford University. This group has done some excellent work over the past few years [ formalizing memory models for real computer systems](<http://www.cl.cam.ac.uk/~pes20/weakmemory/index.html>), including that of ARM, Power, some additional 64-bit members of the Power family, x86 (and yes, academics consider x86 to be a weak-memory system), and C/C++11's new memory model. 

**Quick Quiz 2** : But what about x86? Answer

A prototype interactive version of this tool is available for [ARM](<http://www.cl.cam.ac.uk/~pes20/ppcmem/index.html#ARM>) and [Power](<http://www.cl.cam.ac.uk/~pes20/ppcmem/>). 

This article describes this tool as follows: 

  1. Anatomy of a Litmus Test. 
  2. What Does This Litmus Test Mean?
  3. Running a Litmus Test. 
  4. Full State-Space Search. 
  5. Conclusions. 

Following this is the obligatory Answers to Quick Quizzes. 

####  Anatomy of a litmus test

The input to this tool is a "litmus test" that consists of assembly language, register initializations, and an assertion, for example, as follows for the [ Power interface to the model](<http://www.cl.cam.ac.uk/~pes20/ppcmem/>): 

> 

```
>       1 PPC SB+lwsync-RMW-lwsync+isync-simple
>       2 ""
>       3 {
>       4 0:r2=x; 0:r3=2; 0:r4=y;   0:r10=0 ; 0:r11=0; 0:r12=z ;
>       5 1:r2=y; 1:r4=x;
>       6 }
>       7  P0                 | P1           ;
>       8  li r1,1            | li r1,1      ;
>       9  stw r1,0(r2)       | stw r1,0(r2) ;
>      10  lwsync             | sync         ;
>      11                     | lwz r3,0(r4) ;
>      12  lwarx  r11,r10,r12 | ;
>      13  stwcx. r11,r10,r12 | ;
>      14  bne Fail1          | ;
>      15  isync              | ;
>      16  lwz r3,0(r4)       | ;
>      17  Fail1:             | ;
>      18 
>      19 exists
>      20 (0:r3=0 /\ 1:r3=0)
>
```

The ARM interface works exactly the same way, but with ARM instructions substituted for the Power instructions and with the initial "PPC" replaced by "ARM". You can select the ARM interface by clicking on "Change to ARM Model" at the web page called out above. 

In the example, line 1 identifies the type of system ("ARM" or "PPC") and contains the title for the model. Line 2 provides a place for an alternative name for the test, which you will usually want to leave blank as shown in the above example. Comments can be inserted between lines 2 and 3 using the OCaml (or Pascal) syntax of "(* *)". 

Lines 3-6 give initial values for all registers; each is of the form "P:R=V", where "P" is the process identifier, "R" is the register identifier, and "V" is the value. For example, process 0's register r3 initially contains the value 2\. If the value is a variable ("x", "y", or "z" in the example) then the register is initialized to the address of the variable. It is also possible to initialize the contents of variables, for example, "x=1" initializes the value of "x" to 1. Uninitialized variables default to the value zero, so that in the example, "x", "y", and "z" are all initially zero. 

Line 7 provides identifiers for the two processes, so that the "0:r3=2" on line 4 could instead have been written "P0:r3=2". Line 7 is required, and the identifiers must be of the form "Pn", where "n" is the column number, starting from zero for the left-most column. This may seem unnecessarily strict, but it does prevent considerable confusion in actual use. 

**Quick Quiz 3** : Why does line 8 initialize the registers? Why not instead initialize them on lines 4 and 5? Answer

Lines 8-17 are the lines of code for each process. A given process can have empty lines, as is the case for P0's line 11 and P1's lines 12-17. Labels and branches are permitted, as demonstrated by the branch on line 14 to the label on line 17\. That said, too-free use of branches will expand the state space. Use of loops is a particularly good way to explode your state space. 

Lines 19-20 show the assertion, which in this case indicates that we are interested in whether P0's and P1's r3 registers can both contain zero after both threads complete execution. This assertion is important because there are a number of use cases that would fail miserably if both P0 and P1 saw zero in their respective r3 registers. 

This should give you enough information to construct simple litmus tests. Some additional documentation is [available](<http://diy.inria.fr/doc/litmus.html>), though much of this additional documentation is intended for a different research tool that runs tests on actual hardware. Perhaps more importantly, a large number of pre-existing litmus tests are available with the [online tool](<http://www.cl.cam.ac.uk/~pes20/ppcmem/>) (available via the "Select ARM Test" and "Select POWER Test" buttons). It is quite likely that one of these pre-existing litmus tests will answer your Power or ARM memory-ordering question. 

####  What does this litmus test mean?

**Quick Quiz 4** : But whatever happened to line 17, the one that is the `Fail:` label? Answer

P0's lines 8 and 9 are equivalent to the C statement `x=1` because line 4 defines P0's register `r2` to be the address of `x`. P0's lines 12 and 13 are the mnemonics for load-linked ("load register exclusive" in ARM parlance and "load reserve" in Power parlance) and store-conditional ("store register exclusive" in ARM parlance), respectively. When these are used together, they form an atomic instruction sequence, roughly similar to the compare-and-swap sequences exemplified by the x86 `lock;cmpxchg` instruction. Moving to a higher level of abstraction, the sequence from lines 10-15 is equivalent to the Linux kernel's `atomic_add_return(&z, 0)`. Finally, line 16 is roughly equivalent to the C statement `r3=y`. 

P1's lines 8 and 9 are equivalent to the C statement `y=1`, line 10 is a memory barrier, equivalent to the Linux kernel statement `smp_mb()`, and line 11 is equivalent to the C statement `r3=x`. 

Putting all this together, the C-language equivalent to the entire litmus test is as follows: 

```
1 void P0(void)
      2 {
      3   x = 1; /* Lines 8 and 9 */
      4   atomic_add_return(&z, 0); /* Lines 10-15 */
      5   r3 = y; /* Line 16 */
      6 }
      7 
      8 void P1(void)
      9 {
     10   y = 1; /* Lines 8-9 */
     11   smp_mb(); /* Line 10 */
     12   r3 = x; /* Line 11 */
     13 }
```

**Quick Quiz 5** : Can both threads' `r3` registers be equal to zero after both threads complete? Answer

Now that we have a meaningful litmus test, it is time to run it. 

####  Running a litmus test

To run the litmus test, paste it into the textbox at [ http://www.cl.cam.ac.uk/~pes20/ppcmem/](<http://www.cl.cam.ac.uk/~pes20/ppcmem/>), but use [a version without line numbers](<https://lwn.net/Articles/470785/>). Press the "Interactive" button at the lower left-hand side of the page, and you should then see something like this: 

> [![\[Screen shot\]](https://static.lwn.net/images/2011/ppcmem/ppcmem_0-sm.png)](<https://lwn.net/Articles/470787/>)

(Click on the images for full-size versions) 

The upper portion of the screen beginning with "Storage subsystem state" tracks an abstraction of the state of the processor's write buffers, caches, and cache-coherence messages. The next two portions, beginning with "Thread 0 state" and "Thread 1 state", track the instruction and memory-reference progress of each of the two threads. For more details on the information presented by this tool, refer to the PLDI 2011 paper entitled "Understanding Power Multiprocessors", which may be found [here](<http://www.cl.cam.ac.uk/~pes20/ppc-supplemental/>). 

Because this model is intended for a super-scalar weak-memory system, a given thread can have several different operations that might happen at a given time. For example, for thread 0 there are two operations, indicated by the links ("Commit" and "Read reserve k:W z 0"), while thread 1 has only a single "Commit" link. Clicking on a given link allows you to manually advance the state of the system. 

In this particular case, it is most instructive to start by clicking the links for thread 0, ignoring thread 1 completely and also ignoring any links that appear in the "Storage subsystem state" section preceding thread 0\. There will come a time when the "stwcx" instruction gives you a choice between "Commit (fail)" and "Commit (succ)". Choosing "Commit (succ)" results in the following state: 

> [![\[Screen shot\]](https://static.lwn.net/images/2011/ppcmem/ppcmem_1-sm.png)](<https://lwn.net/Articles/470789/>)

At this point, it is time to click the links for thread 1, resulting in the following state: 

> [![\[Screen shot\]](https://static.lwn.net/images/2011/ppcmem/ppcmem_2-sm.png)](<https://lwn.net/Articles/470790/>)

Now, click the last link, which is in the "Storage subsystem state" section and is labeled "(0:) Write propagate to thread: f:W y 1 to Thread 0", and then click the "(0:) Barrier propagate to thread: g:Sync to Thread 0" that replaces it, and then the second link (labeled "Write reaching coherence point: f:W y 1"), and finally the "Acknowledge sync: Sync g:Sync". This results in the following state: 

> [![\[Screen shot\]](https://static.lwn.net/images/2011/ppcmem/ppcmem_3-sm.png)](<https://lwn.net/Articles/470792/>)

This will result in thread 1 gaining a "Read i:W x 0" link. Click this and the "Commit" link that replaces it. Then click all the remaining links in any order, resulting in the following final state: 

> [![\[Screen shot\]](https://static.lwn.net/images/2011/ppcmem/ppcmem_4-sm.png)](<https://lwn.net/Articles/470793/>)

**Quick Quiz 6** : But suppose that I didn't think to try this particular execution sequence. How could I have found this bug using this tool? Answer

**Quick Quiz 7** : Does the ARM Linux kernel have a similar bug? Answer

This shows that the values loaded into both threads' `r3` registers to be zero, in violation of the requirements set forth in `Documentation/atomic_ops.txt`. This bug is easily fixed by replacing the `isync` instruction with `sync`, and a [patch](<http://www.mail-archive.com/linuxppc-dev@lists.ozlabs.org/msg54753.html>) exists to carry out this fix. 

The fact that this tool is capable of finding this sort of bug is a testament to its usefulness. 

####  Full state-space search

The interactive web-based tool is fun to use and can be quite enlightening, but it is very difficult to make sure that you have checked for every possible error condition. For that reason, there is an executable tool that conducts a full state-space search, whose source code may be downloaded [here](<http://www.cl.cam.ac.uk/~pes20/ppcmem/help.html>), covered by the BSD license, with some components licensed under [LGPL with an exception for linking](<http://caml.inria.fr/ocaml/license.en.html>). The tool may be built by following the instructions in the README file, which requires a recent version of [OCaml](<http://caml.inria.fr/download.en.html>) (3.12.0 works for me, but 3.11.2 doesn't cut it). Please note that this tool is a research prototype that is completely unsupported. That said, I have found it to be quite useful. 

The tool is run from its build directory. For litmus tests containing only normal memory accesses and memory barriers, the following command suffices: 

```
./ppcmem filename.litmus
```

**Quick Quiz 8** : But when I built the tool, I didn't find an `armmem` program. How am I supposed to test ARM code fragments??? Answer

where "filename.litmus" is the path to the file containing your litmus test. 

The tail end of the output of this tool when presented the litmus test discussed earlier is as follows: 

> 

```
>     States 6
>     0:r3=0; 1:r3=0;
>     0:r3=0; 1:r3=1;
>     0:r3=1; 1:r3=0;
>     0:r3=1; 1:r3=1;
>     0:r3=2; 1:r3=0;
>     0:r3=2; 1:r3=1;
>     Ok
>     Condition exists (0:r3=0 /\ 1:r3=0)
>     Hash=e2240ce2072a2610c034ccd4fc964e77
>     Observation SB+lwsync-RMW-lwsync+isync-simple Sometimes 1 
>
```

The list of states includes "0:r3=0; 1:r3=0;", indicating once again that the old powerpc implementation of `atomic_add_return()` does not act as a full barrier. The "Sometimes" on the last line confirms this: the assertion triggers for some executions, but not all of the time. As before, the fix is to replace P0's `isync` with `sync`, which results in the following at the end of the tool's output: 

> 

```
>     States 5
>     0:r3=0; 1:r3=1;
>     0:r3=1; 1:r3=0;
>     0:r3=1; 1:r3=1;
>     0:r3=2; 1:r3=0;
>     0:r3=2; 1:r3=1;
>     No (allowed not found)
>     Condition exists (0:r3=0 /\ 1:r3=0)
>     Hash=77dd723cda9981248ea4459fcdf6097d
>     Observation SB+lwsync-RMW-lwsync+sync-simple Never 0 5
>
```

This output confirms the fix: "0:r3=0; 1:r3=0;" does not appear in the list of states, and the last line calls out "Never". Therefore, the model predicts that the offending execution sequence cannot happen. 

####  Conclusions

These tools promise to be of great help to people working on low-level parallel primitives that run on ARM and on Power. These tools do have some intrinsic limitations: 

  1. These tools do not constitute official statements by IBM or ARM on their respective CPU architectures. For example, both corporations reserve the right to report a bug at any time against any version of any of these tools. These tools are therefore not a substitute for careful stress testing on real hardware. Moreover, both the tools and the model that they are based on are under active development and might change at any time. On the other hand, this model was developed in consultation with the relevant hardware experts, so there is good reason to be confident that it is a robust representation of the architectures. 

  2. These tools currently handle a subset of the instruction set. This subset has been sufficient for my purposes, but your mileage may vary. In particular, the tool handles only word-sized accesses (32 bits), and the words accessed must be properly aligned. In addition, the tool does not handle some of the weaker variants of the ARM memory-barrier instructions. 

  3. The tools are restricted to small loop-free code fragments running on small numbers of threads. Larger examples result in state-space explosion, just as with similar tools such as `Promela` and `spin`. 

  4. The full state-space search does not give any indication of how each offending state was reached. That said, once you realize that the state is in fact reachable, it is usually not too hard to find that state using the interactive tool. 

  5. The tools will detect only those problems for which you code an assertion. This weakness is common to all formal methods, and is yet another reason why testing remains important. In the immortal words of Donald Knuth, "Beware of bugs in the above code; I have only proved it correct, not tried it." 

That said, one strength of these tools is that they are designed to model the full range of behaviors allowed by the architectures, including behaviors that are legal, but which current hardware implementations do not yet inflict on unwary software developers. Therefore, an algorithm that is vetted by these tools likely has some additional safety margin when running on real hardware. Furthermore, testing on real hardware can only find bugs; such testing is inherently incapable of proving a given usage correct. To appreciate this, consider the table in Section 8 on page 10 of [ Understanding POWER Multiprocessors [PDF]](<http://www.cl.cam.ac.uk/~pes20/ppc-supplemental/pldi105-sarkar.pdf>): the researchers routinely ran in excess of 100 billion test runs on real hardware to validate their model. In one case, behavior that is allowed by the architecture did not occur, despite 176 billion runs. In contrast, the full-state-space search allows the tool to prove code fragments correct. 

It is worth repeating that formal methods and tools are no substitute for testing. The fact is that producing large reliable concurrent software artifacts, the Linux kernel for example, is quite difficult. Developers must therefore be prepared to apply every tool at their disposal towards this goal. The tools presented in this paper are able to locate bugs that are quite difficult to produce (let alone track down) via testing. On the other hand, testing can be applied to far larger bodies of software than the tools presented in this paper are ever likely to handle. As always, use the right tools for the job! 

**Quick Quiz 9** : Where can I find out more about the formal model underlying this tool? Answer

Of course, it is always best to avoid the need to work at this level by designing your parallel code to be easily partitioned and then using higher-level primitives (such as locks, sequence counters, atomic operations, and RCU) to get your job done more straightforwardly. And even if you absolutely must use low-level memory barriers and read-modify-write instructions to get your job done, the more conservative your use of these sharp instruments, the easier your life is likely to be. 

####  Acknowledgments

I am grateful to Peter Sewell and Susmit Sarkar at the University of Cambridge, Luc Maranget, Francesco Zappa Nardelli, and Pankaj Pawan at INRIA, Jade Alglave at Oxford University, and a number of their colleagues for their efforts on this research topic. We all are indebted to Ben Herrenschmidt for fixing the bug noted in this article. I am thankful to great number of members of the C and C++ standards committees for many stimulating discussions on memory models and concurrency, including Hans Boehm, Lawrence Crowl, Peter Dimov, Clark Nelson, Raul Silvera, Mark Batty, N.M. Maclaren, Anthony Williams, Blaine Garst, Scott Owens, Tjark Weber, Michael Wong, Benjamin Kosnik, and Bjarne Stroustrup. I owe thanks to Derek Williams and Richard Grisenthwaite for their patient explanations of the Power and ARM memory models, respectively, and to Jim Wasko of IBM and Dave Rusling of Linaro for their support of this effort. 

####  Answers to quick quizzes

**Quick Quiz 1** : Yeah, right!!! Since when has anything ever made the Linux kernel's memory barriers and atomic instructions easier??? 

**Answer** : There really have been some improvements over the years, including updates to `Documentation/memory-barriers.txt` and `Documentation/atomic_ops.txt`. Perhaps more important, the `scripts/checkpatch.pl` script complains if memory barriers are not accompanied by a comment, which has made the code using memory barriers a bit less obscure. 

But yes, more help is needed, hence this article. 

**Back to Quick Quiz 1**.

* * *

**Quick Quiz 2** : But what about x86? 

**Answer** : The ppcmem tool could in fact be extended to handle x86 as well as ARM and Power, but there are no definite plans to carry out this work at the moment. 

**Back to Quick Quiz 2**.

* * *

**Quick Quiz 3** : Why does line 8 initialize the registers? Why not instead initialize them on lines 4 and 5? 

**Answer** : Either way works. However, in general, it is better to use initialization than explicit instructions. The explicit instructions are used in this example to demonstrate their use. In addition, many of the litmus tests available on the tool's [web site](<http://www.cl.cam.ac.uk/~pes20/ppcmem/>) were automatically generated, which generates explicit initialization instructions. 

**Back to Quick Quiz 3**.

* * *

**Quick Quiz 4** : But whatever happened to line 17, the one that is the `Fail:` label? 

**Answer** : The implementation of powerpc version of `atomic_add_return()` loops when the `stwcx` instruction fails, which it communicates by setting non-zero status in the condition-code register, which in turn is tested by the `bne` instruction. Because actually modeling the loop would result in state-space explosion, we instead branch to the `Fail:` label, terminating the model with the initial value of 2 in thread 1's `r3` register, which will not trigger the `exists` assertion. 

There is some debate about whether this trick is universally applicable, but I have not seen an example where it fails. (Famous last words!) 

**Back to Quick Quiz 4**.

* * *

**Quick Quiz 5** : Can both threads' `r3` registers be equal to zero after both threads complete? 

**Answer** : Not if `atomic_add_return()` acts as a full barrier, as is set out as required behavior in `Documentation/atomic_ops.txt`. 

**Back to Quick Quiz 5**.

* * *

**Quick Quiz 6** : But suppose that I didn't think to try this particular execution sequence. How could I have found this bug using this tool? 

**Answer** : You can easily miss problematic execution sequences when using the interactive tool. Which is why the next section covers the full state-space-search tool. 

**Back to Quick Quiz 6**.

* * *

**Quick Quiz 7** : Does the ARM Linux kernel have a similar bug? 

**Answer** : ARM does not have this particular bug, given that it places `smp_mb()` before and after the `atomic_add_return()` function's assembly-language implementation. Finding any other bugs that ARM might have is left as an exercise for the reader. 

**Back to Quick Quiz 7**.

* * *

**Quick Quiz 8** : But when I built the tool, I didn't find an `armmem` program. How am I supposed to test ARM code fragments??? 

**Answer** : Just replace the litmus file's initial "PPC" with "ARM" and put ARM assembly language into your litmus file and pass it to the `ppcmem` program. Yes, `ppcmem` is multilingual. Perhaps this is because the researchers who created it have a variety of native languages. Or maybe because none of them live or work in the USA. ;-) 

**Back to Quick Quiz 8**.

* * *

**Quick Quiz 9** : Where can I find out more about the formal model underlying this tool? 

**Answer** : Currently the best source of information is the PLDI 2011 paper entitled "Understanding Power Multiprocessors", which may be found [here](<http://www.cl.cam.ac.uk/~pes20/ppc-supplemental/>). This paper covers memory barriers, but not atomic instruction sequences. Additional papers that include atomic-instruction-sequence extensions to this model are likely to appear Real Soon Now. 

**Back to Quick Quiz 9**.

* * *
