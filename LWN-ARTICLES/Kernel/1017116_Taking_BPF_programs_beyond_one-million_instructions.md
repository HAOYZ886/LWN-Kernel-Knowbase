---
title: "Taking BPF programs beyond one-million instructions"
url: https://lwn.net/Articles/1017116/
date: "April 16, 2025"
category: "BPF-Verifier"
author: "By Daroc Alden April 16, 2025 LSFMM+BPF"
---

By **Daroc Alden**  
April 16, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

The BPF verifier is not magic; it cannot solve the [ halting problem](<https://en.wikipedia.org/wiki/Halting_problem>). Therefore, it has to err on the side of assuming that a program will run too long if it cannot prove that the program will not. The ultimate check on the size of a BPF program is the one-million-instruction limit — the verifier will refuse to process more than one-million instructions, no matter what a BPF program does. Alexei Starovoitov gave a talk at the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit about that limit, why correctly written BPF programs shouldn't hit it, and how to make the user experience of large BPF programs better in the future. 

One million was ""just a number that felt big in 2019"", Starovoitov said. Now, in 2025, programs often hit this limit. This prompts occasional requests for the BPF developers to raise the limit, but most programs won't hit the limit if they're written to take advantage of modern BPF features. Starovoitov's most important advice to users struggling with the limit was to remove unnecessary inlining and loop unrolling. 

[ ![\[Alexei Starovoitov\]](https://static.lwn.net/images/2025/alexei-starovoitov-small.png) ](<https://lwn.net/Articles/1017199/>)

Guidance is still circulating on the internet to annotate BPF functions with `always_inline`. That hasn't been necessary — or helpful — since 2017. The same is true of using `#pragma unroll` on loops; the BPF verifier has supported bounded loops since 2019. Removing those two things is the most common way that Starovoitov has been able to help users shrink the sizes of their BPF programs. It may or may not always help, but it definitely lets the verifier process the program more quickly, he said. 

If those don't help, there are a few other changes that users can make with more care. One thing to consider is the use of global functions in place of static functions — a distinction introduced to BPF in kernel version 5.6 in order to reduce time spent doing verification and allow for more flexible usage of BPF programs. Static functions are functions marked with the `static` keyword in C, whereas global functions are all other functions. Global functions are verified independent of the context in which they are called, whereas static functions take that context into account. When the verifier is considering static functions, it handles every call site separately. This is good and bad — it means that complex type and lifetime requirements are easily verified, but it also means that every call to a static function counts the entire body of the function against the one million instruction limit. If a static function is called in a loop, this can quickly eat up a program's instruction budget. 

```
// Verified up to 100 times.
        for (int i = 0; i < 100; i ++) static_function(i);
```

One member of the audience asked whether Starovoitov's example would really result in the static function being verified multiple times, or whether the path pruner would prune later verifications as being redundant. Starovoitov explained the pruner would only kick in if nothing depending on the loop index were passed to the static function. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Global functions, on the other hand, are only verified once — meaning the cost of verifying the body is only paid once, and each further call only counts as one instruction as far as the one-million-instruction limit is concerned. Because global functions are verified without having any existing context on what registers contain what types of values, requirements for any arguments to the function need to be specified explicitly in the types, and that the verifier cannot take per-call-site information on bounds into account. On the other hand, if the argument to a function is something like a pointer to a BPF arena, the verifier doesn't need to do bounds checks on it anyway, so that's not as much of a problem. There are also various annotations, such as `__arg_nullable` or `__arg_ctx`, that the user can add to help the verifier understand the arguments to a global function. 

Another trick that is sometimes useful is changing how loops are written. Bounded loops are similar to static functions, in that they ""just work"", Starovoitov said, but they require the verifier to process the body multiple times. Loops with large numbers of iterations can easily run up against the limit. BPF does have several alternatives available, however. Depending on the situation, either iterators or run-time checks are an option: 

```
bpf_for (i, 0, 100) // open-coded iterator
        // See below for why this loop uses "zero" in place of "0"
        for (int i = zero; i < 100 && can_loop; i++) // Loop with run-time check
```

The primary difference between bounded loops and the other options is how the verifier reacts to determining that two iterations of the loop lead to the same symbolic state. In a bounded loop, if an iteration ends in the same verifier state that it started in (i.e., without any tracked variables getting a different value), the verifier rejects it as a potentially infinite loop. In a loop using an iterator or the [ `can_loop`](<https://elixir.bootlin.com/linux/v6.13.7/source/tools/testing/selftests/bpf/bpf_experimental.h#L360>) macro (the eventual result of the [BPF `may_goto` work](<https://lwn.net/Articles/964381/>)), the verifier knows the runtime will halt a loop that continues too long, so it doesn't need to reject the loop. 

Starovoitov then went into a long list of correct and incorrect ways to write BPF loops. The logic behind why each option was correct or incorrect required a certain amount of background on the BPF verifier to understand, and people did not find the consequences of each kind of loop obvious. 

```
// Ok, only processes loop body once:
        bpf_for (i, 0, 100) { function(arr[i]); }
    
        // Bad, verifier can't prove that the access is in bounds:
        int i = 0;
        bpf_repeat(100) { function(arr[i]); i++; }
```

The `bpf_repeat()` macro, an earlier predecessor of `bpf_for()`, doesn't give the verifier enough information to know exactly how many times the loop will be repeated. So the verifier ends up concluding that `i` could potentially be any number, and might therefore be out of bounds for `arr`. Adding a check that `i` is less than 100 helps a little bit, but the verifier still doesn't know how many loop iterations will happen, so it tries to check every value of `i` to make sure the bounds check is right: 

```
// Bad. Accesses are in-bounds, but it hits one million instructions
        // trying to be sure of that:
        int i = 0;
        bpf_repeat(100) {
            if (i < 100) function(arr[i]);
            i++;
        }
```

Adding an explicit `break` lets the verifier break out of the loop after a limited number of iterations, but it still has to process the body of the loop 100 times to reach that point. 

```
// Ok, but the loop body is processed many times:
        int i = 0;
        bpf_repeat(100) {
            if (i < 100) function(arr[i]); else break;
            i++;
        }
```

A workaround for that is to initialize the loop variable from a global variable, instead of from a constant. The trick is that because the verifier assumes a global variable can be changed by other parts of the code, initializing a loop variable from a global variable makes the verifier track the value of the loop variable as being an unknown value in a range, instead of a known constant. In turn, that means that after incrementing the loop variable, the verifier notices that its knowledge about the variable hasn't changed: an unknown number plus one is still an unknown number. Therefore, the verifier determines that this would be an infinite loop, if it were not for the use of the `bpf_repeat()` macro, which lets the BPF runtime terminate the loop if it continues too long. The end result is that the verifier only needs to examine the loop body once, instead of examining it with different known values of `i`: 

```
// Ok, loop body only processed once, because the global variable
        // changes how the verifier processes bounds in the loop:
        int zero = 0; // Global variable
    
        int i = zero;
        bpf_repeat(100) {
            if (i < 100) function(arr[i]); else break;
            i++;
        }
```

These behaviors are ""not obvious to anyone, even me"", Starovoitov reassured everyone. Eduard Zingerman said that Starovoitov's examples hadn't even discussed widening, to which Starovoitov replied: ""I will get into it. You're doing advanced math."" The pitfalls of `can_loop` were no less esoteric: 

```
// Ok, but processes loop body multiple times because the verifier
        // sees each loop as having a different state:
        for (int i = 0; i < 100 && can_loop; i++) {
            function(arr[i]);
        }
```

Even adopting the same global-variable-based workaround doesn't always help, because compilers have some optimizations that apply to basic C for loops that don't apply to the `bpf_for()` macro: 

```
// Bad. Could work or could fail to prove the bounds check depending
        // on how the compiler handles the loop:
        int zero = 0; // Global variable
    
        for (int i = zero; i < 100 && can_loop; i++) {
            function(arr[i]);
        }
```

The solution is to prevent the compiler from applying the problematic optimizations, at least for now. Eventually, Starovoitov hopes to expand the verifier's ability to understand transformed loop variables to render this unnecessary. 

```
// Ok, only processes loop body once:
        int zero = 0; // Global variable
    
        for (int i = zero; i < 100 && can_loop; i++) {
            // Tell the compiler not to assume anything about i, so the loop
            // is left in a format the verifier can handle.
            barrier_var(i);
            function(arr[i]);
        }
```

""I know this is overwhelming, and it's bad"", Starovoitov said. The point is that it's hard for experts too, he continued. The situation with arenas is a little better. Because the verifier doesn't need to do bounds checks on arena pointers for safety, its state-equivalence-checking logic can generalize more inside loops. Combining the global-variable trick, `can_loop`, and arena pointers results in the best loop that he shared, which can always be verified efficiently: 

```
// Verified in one pass, effectively lets the user write any loop:
        int __arena arr[100];
    
        for (int i = zero; i < 100 && can_loop; i++) {
            function(arr[i]);
        }
```

So users have ways to write loops without worrying about the one-million-instruction limit — but there are still clearly parts of the BPF user experience that can be improved. Ideally, users would not need to care about the precise details of how they write their loops. 

One approach that Starovoitov tried was to attempt to widen loop variables (that is, generalize the verifier's state from "i = 0" to "i = [some unknown number within a given range]") when checking loops with `can_loop`. That would have allowed the verifier to recognize when two passes through the loop are essentially the same (differing only by the loop variable's initial value), and remove the need for the global-variable trick. Unfortunately, implementing that idea turned out to be hard. ""I bashed my head against the wall"", Starovoitov said, before giving up on it. 

Another idea was to use bounds checks inside of loops to split the verifier state into two cases, which could potentially permit the same kind of analysis. That turns out not to work well because of a compiler optimization called [ scalar evolution](<https://www.npopov.com/2023/10/03/LLVM-Scalar-evolution.html>), which turns `i++` loops into `i += N` loops, which have discontiguous ranges of possible values for the loop variables. John Fastabend had a patch in 2017 that was supposed to cope with scalar evolution, Starovoitov said, so that idea is still viable. That's the path that he currently intends to work on. 

#### Making things better

There is already a lot of specialized code for handling loops in the verifier, Starovoitov said. For example, the code dealing with scalar precision: its job is to make states look as similar as possible, to allow pruning to happen more effectively. It turns out not to really work for loops, so there's special detection logic in the verifier to work around problems with calculating whether different registers are still alive in the presence of loops. Starovoitov called the resulting design a ""house of cards"" and said that it's time to step back and think about how to rewrite the verifier. 

Right now, the verifier has two ways of determining whether a register remains alive; the old one is path-sensitive (that is, it depends on the path the verifier took to reach a particular place in the code), the new one is not. The new one is a lot more similar to the kind of compiler algorithm you find in [ the Dragon Book](<https://suif.stanford.edu/dragonbook/>) or in the academic literature. But the new analysis only works for registers, not the stack, so the old one has to be kept around. If the new analysis could be made to work for the stack, the old analysis could be removed, Starovoitov said, simplifying the logic around loops and making it easier to fix some of the bad user experience. 

One way to do that might be to eliminate the stack in its current form (at least inside the verifier). The BPF stack has a fairly small maximum size; the verifier could potentially convert the entire stack into virtual registers. Or the BPF developers could change the ISA to tell compilers that there are now 74 registers and no stack. Theoretically, doing things that way would both dramatically simplify the verifier code and improve the just-in-time compilation of BPF to machine code by making use of more registers, when available on an architecture, he explained. 

Zingerman immediately pointed out the problem with that idea, though: access to the contents of the stack based on a variable, instead of a fixed stack offset. Starovoitov agreed that was a problem. There are also some things, like iterators, that have to live on the stack that can't really be converted into registers, he said. 

The overarching point of all of this is that the verifier needs to be simplified, Starovoitov concluded. [ `verifier.c`](<https://elixir.bootlin.com/linux/v6.13.7/source/kernel/bpf/verifier.c>) is the biggest file in the kernel and ""should be shot"". That simplification is the best path forward for enabling BPF to continue to grow and develop.
