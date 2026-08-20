---
title: A different approach to BPF loops
url: https://lwn.net/Articles/877062/
date: "November 29, 2021"
category: "BPF-Loops"
author: "By Jonathan Corbet November 29, 2021"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
November 29, 2021

One of the key features of the extended BPF virtual machine is the verifier built into the kernel that ensures that all BPF programs are safe to run. BPF developers often see the verifier as a bit of a mixed blessing, though; while it can catch a lot of problems before they happen, it can also be hard to please. Comparisons with a well-meaning but rule-bound and picky bureaucracy would not be entirely misplaced. The [`bpf_loop()` proposal](<https://lwn.net/ml/bpf/20211123183409.3599979-1-joannekoong@fb.com/>) from Joanne Koong is an attempt to make pleasing the BPF bureaucrats a bit easier for one type of loop construct. 

To do its job, the verifier must simulate the execution of each BPF program loaded into the kernel. It makes sure that the program does not reference memory that should not be available to it, that it doesn't leak kernel memory to user space, and many other things — including that the program will actually terminate and not lock the kernel into an infinite loop. Proving that a program will terminate is, as any survivor of an algorithms class can attest, a difficult problem; indeed, [it is impossible](<https://en.wikipedia.org/wiki/Halting_problem>) in the general case. So the BPF verifier has had to find ways to simplify the problem. 

Initially, "simplifying the problem" meant forbidding loops altogether; when a program can only execute in a straight-through manner, with no backward jumps, it's clear that the program must terminate in finite time. Needless to say, BPF developers found this rule to be a bit constraining. To an extent, loops can be simulated by manually unrolling them, but that is tiresome for short loops and impractical for longer ones. So work soon began on finding a way to allow BPF programs to contain loops. Various [approaches](<https://lwn.net/Articles/773605/>) to the loop problem were tried over the years; eventually [bounded loop support](<https://lwn.net/Articles/794934/>) was added to the 5.3 kernel in 2019. 

The problem is thus solved — to an extent. The verifier checks loops by simulating their execution for each combination of initial states and demonstrating that each loop terminates before executing the maximum number of allowed instructions. This verification can take some time and, for some programs, the verifier is simply unable to conclude that the loops will terminate, even though those programs may be correct and safe. There are simply too many possible states and iterations to work through. 

The difficulty of verifying loops is complicated by the fact that, by necessity, the verifier works with BPF code, which is a low-level instruction set. The semantics of a loop encoded in a higher-level language are gone by this time. The code may just iterate over the elements of a short array, for example, but the verifier has to piece that together from the BPF code. If there were a way to code a bounded loop in a way that the verifier could see, life would be a lot easier. 

That, in short, is the purpose of Koong's patch. It adds a new helper function that can be called from BPF code: 

```
long bpf_loop(u32 iterations, long (*loop_fn)(u32 index, void *ctx),
        		  void *ctx, u64 flags);
```

A call to `bpf_loop()` will result in `iterations` calls to `loop_fn()`, with the iteration number and the passed-in `ctx` as parameters. The `flags` value is currently unused and must be zero. The `loop_fn()` will normally return zero; a return value of one will end the iteration immediately. No other return values are allowed. 

Essentially, `bpf_loop()` takes the mechanics of the loop itself out of the BPF code and embeds it within the kernel's BPF implementation instead. It allows the verifier to know immediately that the loop will terminate, since that is outside the control of the BPF program itself. It is also easy to calculate how many instructions may be executed within the loop in the worst case; that and the limit on stack depth will prevent programs that run nearly forever as the result of nested loops. 

For BPF programmers, the benefit is that any loop that can be implemented using `bpf_loop()` becomes much easier to get past the verifier; whole layers of bureaucracy have been shorted out, as it were. Note that loops that, for example, follow a linked list are possible with `bpf_loop()`; the developer need only supply a maximum possible length as the number of iterations, then terminate early when the desired element has been found or the end of the list has been hit. The form of programs may shift a bit to fit the template, but it should be possible to make that change in many cases. 

Another significant advantage is that the time required to verify BPF programs is greatly reduced, since the verifier does not need to actually simulate the execution of all those loops. Some [benchmarks](<https://lwn.net/ml/bpf/20211123183409.3599979-4-joannekoong@fb.com/>) show what a difference that can make; one program that takes nearly 30 seconds to verify in current kernels can be verified in 0.15s instead. That significantly increases the practicality of many types of BPF program. 

There are many reasons why Fortran remained dominant in numerical applications for so long; one of those is that `do` loops, by their predictable structure, are relatively easy to vectorize. The purpose of `bpf_loop()` is different, but it works by the same mechanism: constraining what can be expressed in the language to make it easier for the computer to understand what is really being done. That, in turn, should make it easier for developers to convince the computer that it can safely run their programs.
