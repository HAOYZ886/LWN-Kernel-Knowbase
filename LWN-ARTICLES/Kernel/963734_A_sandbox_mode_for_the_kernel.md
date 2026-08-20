---
title: A sandbox mode for the kernel
url: https://lwn.net/Articles/963734/
date: "February 29, 2024"
category: "Security-Kernel hardening"
author: "By Jonathan Corbet February 29, 2024"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
February 29, 2024

The Linux kernel follows a monolithic design, and that brings a well-known problem: all code in the kernel has access to the entirety of the kernel's address space. As a result, a bug in (for example) an obscure driver may well be exploitable to wreak havoc on core-kernel data structures. Various attempts have been made over the years to increase the degree of isolation within the kernel. The latest of these, ["SandBox Mode"](<https://lwn.net/ml/linux-kernel/20240214113035.2117-1-petrtesarik@huaweicloud.com/>) proposed by Petr Tesařík, makes it possible for the kernel to run some limited code safely, but it has encountered a bit of a chilly reception. 

#### Sandbox mode

The intent behind this new mode is to allow the kernel to run a function in a way that it cannot affect the rest of the kernel. In its simplest form, sandbox mode is used by defining a function to be run in this isolated mode: 

```
#include <linux/sbm.h>
    
        static SBM_DEFINE_FUNC(untrusted_func, void *input_data, void *output_data);
```

That function would then be invoked with a sequence like: 

```
struct sbm sbm;
    
        sbm_init(&sbm);
        result = sbm_call(&sbm, untrusted_func, SBM_COPY_IN(&sbm, input_buffer, in_size),
    			                    SBM_COPY_OUT(&sbm, output_buffer, out_size));
```

This code will result in a call to `untrusted_func()`. The input and output buffers will be allocated, and the input data copied, before that function is called with pointers to the new buffers. On a successful return, output data will be copied back, and `sbm_call()` will return the value returned by `untrusted_func()`. 

In the absence of architecture-specific support, that is about all that sandbox mode does; the [associated documentation](<https://lwn.net/ml/linux-kernel/20240214113035.2117-6-petrtesarik@huaweicloud.com/>) rightly describes this as ""weak isolation"". It might be enough to trap a simple overflow of the input or output buffers, but it still does not protect the kernel from any stray accesses that go further afield. 

In [a separate series](<https://lwn.net/ml/linux-kernel/20240214113035.2117-6-petrtesarik@huaweicloud.com/>), Tesařík provided a set of x86-64 architecture hooks that enhance the sandbox to provide stronger isolation. Specifically: 

  * The sandboxed function will be run with a separate set of page tables that limit its address space to the relevant code, the input buffer (mapped read-only), and the output buffer. As a result, the function will have no access to any other memory in the system. This change has some far-ranging implications; for example, it must be undone if an interrupt arrives so that the interrupt handler can run within the kernel's address space. 
  * The CPU is put into user mode, so that it cannot execute any privileged instructions; the function runs as if it were a user-space process. 
  * A separate kernel stack is allocated and the function is called on that stack so that it has no access to the normal kernel stack. There is also a separate exception stack that is used while sandbox mode is in effect. 
  * Any sort of CPU fault causes the immediate termination of the sandbox and an error return to the caller. 

At this point, according to the documentation, sandbox mode provides ""strong isolation"" that should suffice to prevent the sandboxed function from accessing the rest of the kernel. 

#### In search of users

But for what purpose has this mode been created? The documentation says that sandbox mode exists for ""parsing data from untrusted sources, especially if the parsing cannot be reasonably done by a user mode helper"", but there was no actual user included with the patch series, so there was no way to see what an intended user looks like. That, naturally, led to questions. Andrew Morton [remarked](<https://lwn.net/ml/linux-kernel/20240214053053.982b48d993ae99dad1d59020@linux-foundation.org/>) that the API seemed overly restrictive and wondered how it would be possible to get any real work done; he asked for an example to clarify the situation, a request that Greg Kroah-Hartman [echoed](<https://lwn.net/ml/linux-kernel/2024021425-audition-expand-2901@gregkh/>). 

Tesařík [answered](<https://lwn.net/ml/linux-kernel/20240214155524.719ffb15@meshulam.tesarici.cz/>) that the framework is ""quite limited"" in its current form, but that he intended to expand it over time. A bit later, he posted [a PGP-key parser](<https://lwn.net/ml/linux-kernel/20240216152435.1575-1-petrtesarik@huaweicloud.com/>) that would run within the sandbox mode as an example user, but that did little to increase acceptance of this work. As Dave Hansen [pointed out](<https://lwn.net/ml/linux-kernel/c65eb8f1-2903-4043-a3ab-945d880043b5@intel.com/>), the kernel does not currently contain a parser for PGP keys, so the new series just raised the question of why _that_ needs to be added too. Hansen said it would be far better to move some existing kernel functionality into the sandbox to show how it could be made safer. 

The response to that request was [yet another patch series](<https://lwn.net/ml/linux-kernel/20240222131230.635-1-petrtesarik@huaweicloud.com/>) moving the parsing of AppArmor profiles into a sandbox. Supporting this use case required making a number of changes to the sandbox mode itself, including a new "fixup" mechanism designed to make it possible to call specific kernel functions from within the sandbox. So, for example, if code within the sandbox needs to allocate memory, it can call `kmalloc()`. That call will result in a fault, which will result in the execution of a proxy version of `kmalloc()` that will restore the kernel's full address space for the duration of the call. 

Hansen [responded](<https://lwn.net/ml/linux-kernel/f6135f2c-BC8f-41c3-9c6a-8346d685e4dc@intel.com/>) that the "fixup" mechanism looked like a maintenance problem: ""Establishing and maintaining this proxy list will be painful. Folks will change the code to call something new and break this *constantly*."" He [concluded](<https://lwn.net/ml/linux-kernel/5de7d665-7047-497b-94fb-76ec2af3c9e2@intel.com/>) that sandbox mode did not seem like a good idea overall: ""I don't see any viable way forward for this approach"". He did not even comment on the need to add a special "`__nosbm`" marker to all functions that might land in the same page as one that has been marked for calls from within sandbox mode — an extra step that seems almost certain to go wrong at some point. 

The obvious conclusion is that sandbox mode is unlikely to make it into the mainline in anything resembling its current form. But there is clear value in isolating some kinds of kernel code, if there were only an acceptable way in which it could be done. One possibility is to use BPF, which is intended to provide isolation; non-trivial BPF programs can be tricky to get past the verifier, though, and the fact that they are loaded from user space may make some security-oriented people nervous. 

Another possibility might be the [user-mode blob feature](<https://lwn.net/Articles/755919/>) that was merged into the 4.18 kernel nearly six years ago. It was intended for a similar purpose — the parsing of firewall rules for the BPF-based "bpfilter" subsystem — but has never seen use in the mainline kernel. In response to [a query](<https://lwn.net/ml/linux-kernel/87y1bktjdk.fsf@meer.lwn.net/>) about using this feature instead of a new sandbox mode, Roberto Sassu [said](<https://lwn.net/ml/linux-kernel/33a1fae4-d713-4e93-89ff-ff9f377e8391@huaweicloud.com/>) that ""security people don't feel confident"" about using it. The main concern seems to be that, since user-mode blobs run in a separate, user-space process, they would be subject to manipulation by user space; sandbox mode, being fully contained within the kernel, should be better protected. 

If complete isolation from user space is also a requirement for this work, then it may be that there are no viable solutions for Linux at this time. Hardening the kernel is a worthy goal, but it is just one of many that have to be traded off in the creation of a kernel that is both useful and maintainable in the real world. In the absence of a better implementation, it would appear that sandbox mode does not offer enough to justify the tradeoffs it would require.
