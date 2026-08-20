---
title: Stuffing the return stack buffer
url: https://lwn.net/Articles/901834/
date: "July 22, 2022"
category: "Security-Meltdown and Spectre"
author: "By Jonathan Corbet July 22, 2022"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
July 22, 2022

"[Retbleed](<https://comsec.ethz.ch/research/microarch/retbleed/>)" is the name given to a class of speculative-execution vulnerabilities involving return instructions. Mitigations for Retbleed have found their way into the mainline kernel but, as of this writing, some remaining problems have kept them from the stable update releases. Mitigating Retbleed can impede performance severely, especially on some Intel processors. Thomas Gleixner and Peter Zijlstra think they have found [a better way](<https://lwn.net/ml/linux-kernel/20220716230344.239749011@linutronix.de/>) that bypasses the existing mitigations and misleads the processor's speculative-execution mechanisms instead. 

If a CPU is to speculate past a return instruction, it must have some idea of where the code will return to. In recent Intel processors, there is a special hidden data structure called the "return stack buffer" (RSB) that caches return addresses for speculation. The RSB can hold 16 entries, so it must drop the oldest entries if a call chain goes deeper than that. As that deep call chain returns, the RSB can underflow. One might think that speculation would just stop at that point but, instead, the CPU resorts to other heuristics, including predicting from the branch history buffer. Alas, techniques for mistraining the branch history buffer are well understood at this point. 

As a result, long call chains in the kernel are susceptible to speculative-execution attacks. On Intel processors starting with the Skylake generation, the only way to prevent such attacks is to turn on the indirect branch restricted speculation (IBRS) CPU "feature", which was added by Intel early in the Spectre era. IBRS works, but it has the unwelcome side effect of reducing performance by as much as 30%. For some reason, users lack enthusiasm for this solution. 

#### Another way

Gleixner and Zijlstra decided to try a different approach. Speculative execution of return calls on these processors can only be abused if the RSB underflows. So, if RSB underflow can be prevented, this particular problem will go away. And that, it seems, can be achieved by "stuffing" the RSB whenever it is at risk of running out of entries. 

That immediately leads to two new challenges: knowing when the RSB is running low, and finding a way to fill it back up. The first piece is handled by tracking the current call-chain depth — in an approximate way. The build system is modified to create a couple of new sections in the executable kernel image to hold entry and exit thunks for kernel functions and to track them. When RSB stuffing is enabled, the entry thunk will be invoked on entry to each function, and the exit thunk will be run on the way out. 

The state of the RSB is tracked with a per-CPU, 64-bit value that is originally set to: 

```
0x8000 0000 0000 0000
```

The function entry thunk "increments" this counter by right-shifting it by five bits. The processor will sign-extend the value, so the counter will, after the first call, look like: 

```
0xfc00 0000 0000 0000
```

If twelve more calls happen in succession, the sign bit will have been extended all the way to the right and the counter will contain all ones, with bits beginning to fall off the right end; this counter thus cannot reliably count above twelve. In this way it mimics the RSB, which cannot hold more than 16 entries, with a safety margin of four calls; the use of shifts achieves that behavior without the need to introduce a branch. Whenever a return thunk is executed, the opposite happens: the counter is left-shifted by five bits. After twelve returns, the next shift will clear the remaining bits, and the counter will have a value of zero, which is the indication that something must be done to prevent the RSB from underflowing. 

That "something" is a quick series of function calls (coded in assembly and found at the end of [this patch](<https://lwn.net/ml/linux-kernel/20220716230954.334016834@linutronix.de/>)) that adds 16 entries to the call stack, and thus to the RSB as well. Each of those calls, if ever returned from, will immediately execute an `int3` instruction; that will stop speculation if those return calls are ever executed speculatively. The actual kernel does not want to execute those instructions (or all of those returns), of course, so the RSB-stuffing code increments the real stack pointer past the just-added call frames. 

The end result is an RSB that no longer matches the actual call stack, but which is full of entries that will do no harm if speculated into. At this point, the call-depth counter can be set to `-1` (all ones in the [two's complement](<https://en.wikipedia.org/wiki/Two%27s_complement>) representation) to reflect the fact that the RSB is full. The kernel is now safe against Retbleed exploitation — until and unless another chain of twelve returns happens, in which case the RSB will need to be stuffed again. 

#### Costs

Quite a bit of work has been put into minimizing the overhead of this solution, especially on systems where it is not needed. The kernel is built with direct calls to its functions as usual; at boot time, if the `retbleed=stuff` option is selected, all of those calls will be patched to go through the accounting thunks instead. The thunks themselves are placed in a huge-page mapping to minimize the translation lookaside buffer overhead. Even so, as the cover letter comments, there are costs: ""We both unsurprisingly hate the result with a passion"". 

Those costs come in a few forms. An ""impressive"" amount of memory is required to hold the thunks and associated housekeeping. The bloating of the kernel has a performance impact of its own, even on systems where RSB stuffing is not enabled. The extra instructions add to pressure on the instruction cache, slowing execution. That last problem could be mitigated somewhat, the cover letter says, by allocating the thunks at the beginning of each function rather than in a separate section. Gleixner has prepared a GCC patch to make that possible, and reports that some of the performance loss is gained back when it is used. 

The cover letter contains a long list of benchmark results comparing the performance of RSB stuffing against that of disabling mitigations entirely and of using IBRS. The numbers for RSB stuffing are eye-opening, including a 382% performance regression for one microbenchmark. In all cases, though, RSB stuffing performs better than IBRS. 

Better performance than IBRS is only interesting, though, if the primary goal of blocking Retbleed attacks has been achieved. The cover letter says this: 

> The assumption is that stuffing at the 12th return is sufficient to break the speculation before it hits the underflow and the fallback to the other predictors. Testing confirms that it works. Johannes [Wikner], one of the retbleed researchers, tried to attack this approach and confirmed that it brings the signal to noise ratio down to the crystal ball level. 
> 
> There is obviously no scientific proof that this will withstand future research progress, but all we can do right now is to speculate about that. 

So RSB stuffing seems to work — for now, at least. That should make it attractive in situations where defending against Retbleed attacks is considered to be necessary; hosting providers with untrusted users would be one obvious example. But nobody will be happy with the overhead, even if it is better than IBRS. For a lot of users, RSB stuffing will be seen as a clever hack that, happily, they do not need to actually use.
