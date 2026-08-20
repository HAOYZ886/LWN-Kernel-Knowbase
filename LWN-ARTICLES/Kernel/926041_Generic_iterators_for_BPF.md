---
title: Generic iterators for BPF
url: https://lwn.net/Articles/926041/
date: "March 17, 2023"
category: "BPF-Loops; Releases-6.4"
author: "By Jonathan Corbet March 17, 2023"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
March 17, 2023

BPF programs destined to be loaded into the kernel are generally written in C but, increasingly, the environment in which those programs run differs significantly from the C environment. The BPF virtual machine and associated verifier make a growing set of checks in an attempt to make BPF code safe to run. The proposed addition of an iterator mechanism to BPF highlights the kind of features that are being added — as well as the constraints placed on programmers by BPF. 

One of the many checks performed by the BPF verifier at program-load time is to convince itself that the program will terminate within a reasonable period of time, a process that involves simulating the program's execution. This constraint has made supporting loops in BPF programs challenging since the beginning; it has only been possible to use loops [since the 5.3 release](<https://lwn.net/Articles/794934/>). Even with that addition, convincing the verifier that a loop will terminate can be a challenge; this annoyance has led to, among other things, the addition of features like [`bpf_loop()`](<https://lwn.net/Articles/877062/>), which puts the looping logic for some simple cases into the kernel's C code. 

Not all problems are readily addressable by a simple function like `bpf_loop()`, though. Many loops in BPF programs are simply iterating through a set of objects, and BPF developers would like easier ways to do that. While numerous languages have some sort of built-in notion of iteration over a set, C does not. As noted above, though, BPF is not really C; [this patch set](<https://lwn.net/ml/bpf/20230307232913.576893-1-andrii@kernel.org/>) from Andrii Nakryiko reiterates (so to speak) that point by adding an iteration mechanism to the BPF virtual machine. 

In languages that support the concept of iteration with a specific type, there is usually a set of methods to implement for a new iterator type; they can be thought of as "start iteration", "next item", and "finish iteration". The proposed BPF mechanism follows that same pattern. Code to support iteration must be written (in real C) in the kernel, and it must supply four things, starting with a structure type to represent the iterator itself; the size of this structure must be a multiple of eight bytes. The iterator structure will have a name like `bpf_iter_foo`, and will contain whatever data the iterator needs to maintain its state. 

The "new" function (or "constructor") must be called `bpf_iter_foo_new()`. Its first parameter will be a structure of the iterator type (which must be declared and instantiated in the BPF program); it can take an arbitrary number of other parameters. This function should initialize the iterator and return either zero or a negative error code; if initialization fails, the iterator must still be set up properly so that subsequent calls do the right thing. 

The "next item" function is `bpf_iter_foo_next()`; it accepts the iterator as its only argument and returns a pointer to the next element (in whatever type the iterator supports). Even an iterator that just returns an integer must return a pointer to that integer. Returning a null pointer indicates that iteration is complete — or that some sort of error has occurred. 

The `bpf_iter_foo_destroy()` function (the "destructor") takes a pointer to the iterator structure as its only argument and returns `void`; it completes iteration and performs any needed cleanup. 

All of these functions must be declared as [kfuncs](<https://lwn.net/Articles/856005/>) with some flags indicating their special roles. The constructor must be marked as `KF_ITER_NEW`, the next function as `KF_ITER_NEXT|KF_ITER_NULL`, and the destructor as `KF_ITER_DESTROY`. 

With this infrastructure in place, the verifier can perform a number of checks on an iterator, starting with the requirement that the constructor must be called before any other operations. Calls to the next function will be checked to ensure that the program is looking for the null return that indicates the end of iteration. The verifier ensures that the destructor is called at the end, and that the iterator is not accessed thereafter. It also uses the type information to ensure that a given iterator type is only passed to a set of functions that is declared to deal with that type. 

The BPF subsystem also has some requirements on the C code implementing iterators, including the rule that the next function must return null after a reasonable number of calls. Since the verifier cannot know how many times an iterator-driven loop might run, its ability to enforce limits on the number of instructions executed by a BPF program is reduced; iterators have to help by not letting a program run indefinitely. 

The patch series adds a mechanism enforcing the naming of the iterator type (it must start with `bpf_iter_`) and of the associated functions, which must be constructed by appending `_new()`, `_next()`, or `_destroy()` to the iterator type name. The arguments and return type of each function are also checked; if a check fails, the registration of the functions will fail. 

One nice feature of this implementation is that iterators are, as far as the verifier is concerned, completely self-describing. Specifically, that means that there is no need to change the verifier itself to add new iterator types in the future, as long as they conform to this pattern. 

As an example of how all this works, the series includes [a sample "numbers" iterator](<https://lwn.net/ml/bpf/20230307232913.576893-5-andrii@kernel.org/>) that simply steps through a series of integers. The usage example on the BPF side looks like: 

```
struct bpf_iter_num it;
        int *v;
    
        bpf_iter_num_new(&it, 2, 5);
        while ((v = bpf_iter_num_next(&it))) {
            bpf_printk("X = %d", *v);
        }
        bpf_iter_num_destroy(&it);
```

This code will execute the body of the loop with `*v` holding values from two to four, inclusive. 

Iterating through a count in this way is not hugely exciting, of course; that can already be done with `bpf_loop()` or, in the case like the above with constant bounds, by just coding a `for` loop. One expects that there are rather more advanced use cases in mind for this functionality, in the [extensible scheduler class](<https://lwn.net/Articles/922405/>) perhaps, but that has not been spelled out in the patch posting. Those can be expected to appear after the series is merged; given the apparent lack of opposition at this point, that seems likely to happen soon.
