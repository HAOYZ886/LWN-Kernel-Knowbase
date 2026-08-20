---
title: "BPF-based error injection for the kernel"
url: https://lwn.net/Articles/740146/
date: "November 29, 2017"
category: "Development tools-Kernel debugging; Fault injection"
author: "By Jonathan Corbet November 29, 2017"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
November 29, 2017

Diligent developers do their best to anticipate things that can go wrong and write appropriate error-handling code. Unfortunately, error-handling code is especially hard to test and, as a result, often goes untested; the code meant to deal with errors, in other words, is likely to contain errors itself. One way of finding those bugs is to inject errors into a running system and watching how it responds; the kernel may soon have a new mechanism for doing this sort of injection. 

As an example of error handling in the kernel, consider memory allocations. There are few tasks that can be performed in kernel space without allocating memory to work with. Memory allocation operations can fail ([in theory, at least](<https://lwn.net/Articles/627419/>)), so any code that contains a call to a function like `kmalloc()` must check the returned pointer and do the right thing if the requested memory was not actually allocated. But `kmalloc()` almost never fails in a running kernel, so testing the failure-handling paths is hard. It is probably fair to say that a large percentage of allocation-failure paths in the kernel have never been executed; some of those are certainly wrong. 

The kernel gained [a fault-injection framework](<https://lwn.net/Articles/209257/>) back in 2006; it can be used to test error-handling paths by causing memory allocation requests to fail. Just making `kmalloc()` fail universally is unlikely to be helpful, though; execution will almost certainly never make it to the code that the developer actually wants to test. The fault-injection framework has some parameters to control which allocation attempts should fail, but the mechanism is somewhat awkward to use and is not as flexible as one might like. So the number of developers actually using this framework is small. 

Fully generalizing fault injection would be a lot of work. A developer may want to see what happens when a specific `kmalloc()` call fails, but perhaps only when it is invoked from a specific call path or when some other condition is true. It has not been possible in the past to describe these conditions to the framework but, in recent years, a new technology has come along that can provide the required flexibility: the BPF virtual machine. 

It is already possible to attach a BPF program to an arbitrary function using the kprobe mechanism. Such programs are useful for information gathering, but they cannot be used to affect the execution of the function they are attached to. Thus, they are not usable for error injection. That situation changes, though, with [this patch set](<https://lwn.net/Articles/739442/>) from Josef Bacik, which is intended to turn BPF into a generalized mechanism for the injection of errors into a running kernel. 

The core of the new mechanism is a BPF-callable function called `bpf_override_return()`. If a BPF program attached to a kprobe calls this function, the execution of the function the program is attached to will be shorted out and its return value will be replaced with a value supplied by that BPF program. The patch set contains [an example](<https://lwn.net/Articles/740148/>) in the form of a test program: 

```
SEC("kprobe/open_ctree")
        int bpf_prog1(struct pt_regs *ctx)
        {
    	unsigned long rc = -12;
    
    	bpf_override_return(ctx, rc);
    	return 0;
        }
```

This function can be compiled to BPF using the LLVM compiler. The `SEC()` directive at the top specifies that this function should be attached to a kprobe placed at the beginning of [`open_ctree()`](<http://elixir.free-electrons.com/linux/v4.14.2/source/fs/btrfs/disk-io.c#L2555>), a function in the Btrfs filesystem implementation. After the placement of this probe and the attachment of the BPF function, a call to `open_ctree()` will be overridden and the value -12 (`-ENOMEM`) will be returned. This is a relatively simplistic example, of course; it is expected that many uses will require more sophisticated BPF programs to narrow down the set of situations where the injection will occur. 

This patch set had been through several revisions and appeared ready for inclusion into the mainline; it had even been [applied](<https://lwn.net/Articles/740149/>) to the networking tree for the 4.15 merge window. Things came to a halt, though, when Ingo Molnar [blocked](<https://lwn.net/Articles/740151/>) the progress of this patch set out of worries that it violated one of the basic promises behind the BPF virtual machine and could destabilize the kernel: 

One of the major advantages of having an in-kernel BPF sandbox is to never crash the kernel - and allowing BPF programs to just randomly modify the return value of kernel functions sounds immensely broken to me. 

After some discussion, a solution was agreed to: BPF programs would retain the ability to override kernel functions, but only for functions that have been specifically marked to allow this to happen. A new macro called `BPF_ALLOW_ERROR_INJECTION()` was introduced; it can be used to add the required annotation to a function. See, for example, [this patch](<https://lwn.net/Articles/740152/>) adding the marking for `open_ctree()`. Molnar [suggested](<https://lwn.net/Articles/740155/>) some additional conditions — only functions whose return value cannot crash the kernel should be annotated, and the override function should only change integer error values — but nothing enforces those rules in the current patch set. 

Bacik's patch set only marks that one function; it is not clear whether those markings will be added in any quantity to the mainline kernel, or whether they will, instead, be maintained as private patches by the developers who use them. One can imagine that there could be some resistance to marking up the mainline in this way. But, on the other hand, there would be value in marking functions like `kmalloc()` to enable the development of generic tools that can be used to test specific allocation-error handling paths. 

That question is only likely to be resolved once the mechanism is in place and patches marking functions for error injection start to appear. Meanwhile, the objections to the core mechanism have been addressed, and its path into the mainline appears to be clear. It has missed the 4.15 merge window, though, so it will almost certainly have to wait until 4.16.
