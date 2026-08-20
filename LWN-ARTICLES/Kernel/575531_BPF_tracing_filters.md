---
title: BPF tracing filters
url: https://lwn.net/Articles/575531/
date: "December 4, 2013"
category: "BPF-Tracing; Tracing-with BPF"
author: "By Jonathan Corbet December 4, 2013"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
December 4, 2013

The kernel's tracing functionality allows for the extraction of vast amounts of information about how the system is operating. In fact, so much information can be produced that one quickly starts to look for ways to cut down the flood. For this reason, filters can be attached to tracepoints, limiting the conditions under which the tracepoints will fire. But the current filter implementation is relatively inflexible and slow; it would be nice to be able to do better. A new patch set appears to do exactly that by introducing yet another virtual machine into the kernel. 

In truth, the virtual machine added by [Alexei Starovoitov's patch set](<https://lwn.net/Articles/575444/>) is not entirely new; it is a version of the Berkeley packet filter (BPF) machine which is [used in the networking stack](<https://lwn.net/Articles/437981/>). The secure computing (seccomp) functionality [also uses BPF](<https://lwn.net/Articles/475043/>) to regulate access to system calls. Alexei's idea is to apply BPF to the question of deciding which tracepoints should fire, but he has taken the idea rather further than his predecessors. 

To begin with, the "extended BPF" implemented in his patch set rather expands the capabilities of the BPF virtual machine. That machine was designed to be unable to damage the kernel; it only allows forward jumps to guarantee that programs will not loop, has no pointer types, etc. The extended BPF machine operates rather differently. The two registers available in BPF have been expanded to ten. Backward jumps are allowed (for reasons that will be mentioned below). Extended BPF programs can manipulate pointers and call kernel functions. In other words, there is quite a bit more power available here than in previous versions of the BPF machine. 

These capabilities notwithstanding, Alexei claims that extended BPF programs are entirely safe to load into the kernel; he has gone as far as to [suggest](<https://lwn.net/Articles/575542/>) that unprivileged users could eventually be allowed to insert extended BPF programs into the kernel. To ensure this safety, the kernel performs a range of checks on every program before accepting it. Every jump is mapped and, while backward jumps are allowed, jumps to previously executed parts of the program are not, so loops should not be possible. Execution of the program is simulated with an in-kernel static analysis tool that tracks the contents of every register; pointer operations are only allowed if it is known that the pointer destination is meant to be accessible. Kernel functions can be called, but only those that have been explicitly made available to BPF programs running in that particular context. The total length of the program is limited, as are various resources used or declared by the program. And so on. 

The BPF machine implements a simple sort of assembly language, which, while adequate for the creation of the sort of simple program it is intended for, is not necessarily convenient for users to write in. Users will not need to worry about such problems with Alexei's mechanism, since there are backends for both GCC and LLVM that allow filter code to be written in a restricted form of C. The GCC backend is available from [a GitHub repository](<https://github.com/iovisor/bpf_gcc/commit/9e7223f8f09c822ecc6e18309e89a574a23dbf63>), while the LLVM version is in the LLVM tree itself. This feature, incidentally, is why extended BPF allows backward jumps: the compilers will emit them as a result of their optimization work. 

The extended BPF machinery is not specific to any particular use within the kernel. Instead, it is meant to be invoked from a specific kernel subsystem with a context describing the set of available functions and any use-specific data. So, for packet filtering, that context might include the packet under consideration. In the case of tracing, the context is a subset of the processor's register contents when the tracepoint is hit. So filters must have a knowledge of which data structures will be in which registers — information which may not be readily available, especially for users who don't want to dig through the source code. This aspect has been acknowledged as one of the weakest points of the current implementation; it will likely be improved before this functionality is considered for merging. 

A simple example provided with the patch set looks like this: 

```
/*
         * tracing filter example
         * if attached to /sys/kernel/debug/tracing/events/net/netif_receive_skb
         * it will print events for loobpack device only
         */
        #include <linux/skbuff.h>
        #include <linux/netdevice.h>
        #include <linux/bpf.h>
        #include <trace/bpf_trace.h>
        
        void filter(struct bpf_context *ctx)
        {
        	char devname[4] = "lo";
        	struct net_device *dev;
        	struct sk_buff *skb = 0;
        
        	skb = (struct sk_buff *)ctx->regs.si;
        	dev = bpf_load_pointer(&skb->dev);
        	if (bpf_memcmp(dev->name, devname, 2) == 0) {
        	    char fmt[] = "skb %p dev %p \n";
        	    bpf_trace_printk(fmt, sizeof(fmt), (long)skb, (long)dev, 0);
        	}
        }
```

This filter code derives the address of the `sk_buff` from the passed-in context (it's in the "rsi" register), uses that to load the pointer to the associated `device` structure, then compares the device name stored therein against the loopback device name, finally outputting a trace message if the comparison succeeds. 

On supported architectures, the BPF code is compiled to native machine code once it is accepted into the kernel. So one might expect it to be fast. Alexei ran a test on a networking tracepoint that would be hit one million times; the filter program was designed to reject all tracepoint hits, on the theory that filters will usually filter things out most of the time. The BPF filter was notably faster than the kernel's current filter mechanism, working through one million calls in about 2/3 of the time. Interestingly, is was also quite a bit faster than tracing with no filtering at all; the cost of running the filter was quite a bit less than the cost of generating the trace output. 

Ingo Molnar [looked at the patch set](<https://lwn.net/Articles/575547/>) and came to a simple conclusion: ""Seems like a massive win-win scenario to me."" He did have one concern, though: he wants the ability to extract BPF programs from the kernel and turn them back into some sort of useful source form. This would, he said, make the licensing of BPF programs clear: 

I think it would be fundamentally important to make sure that this is all within the kernel's license domain, to make it very clear there can be no 'binary only' BPF scripts. 

By up-loading BPF into a kernel the person loading it agrees to make that code available to all users of that system who can access it, under the same license as the kernel's code (or under a more permissive license). 

Others expressed concerns about the security of the system; Andi Kleen [pointed out](<https://lwn.net/Articles/575548/>) that "safe" virtual-machine systems have proved to have holes in the past, and that this one probably does as well. 

Beyond security, there are a number of questions to be answered before this patch set is likely to make it into the kernel. The register-oriented interface for access to relevant data seems awkward at best. It's not clear whether BPF filters should replace normal tracepoint output, or just decide whether that output should happen. There is also the question of how this functionality relates to the [Ktap mechanism](<https://lwn.net/Articles/551314/>); the addition of two independent virtual machines for tracing seems like an unlikely prospect. But this work has clearly generated a lot of interest, so answers to these questions may well be forthcoming.
