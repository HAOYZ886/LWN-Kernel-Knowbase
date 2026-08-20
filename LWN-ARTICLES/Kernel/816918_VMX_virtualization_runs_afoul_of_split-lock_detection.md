---
title: "VMX virtualization runs afoul of split-lock detection"
url: https://lwn.net/Articles/816918/
date: "April 7, 2020"
category: "Architectures-x86; Virtualization"
author: "By Jonathan Corbet April 7, 2020"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 7, 2020

One of the many features merged for the 5.7 kernel is [split-lock detection](<https://lwn.net/Articles/790464/>) for the x86 architecture. This feature has encountered [a fair amount of controversy](<https://lwn.net/Articles/806466/>) over the course of its development, with the result that the time between its initial posting and appearance in a released kernel will end up being over two years. As it happens, there is another hurdle for split-lock detection even after its merging into the mainline; this feature threatens to create problems for a number of virtualization solutions, and it's not clear what the solution would be. 

To review quickly: a "split lock" occurs when a processor instruction locks a range of memory that crosses a cache-line boundary. Implementing such locks requires locking the entire memory bus, with unpleasant effects on the performance of the system as a whole. Most architectures do not allow split locks at all, but x86 does; only recently have some x86 processors gained the ability to generate a trap when a split lock is requested. 

Kernel developers are interested in enabling split-lock detection as a way of eliminating a possible denial-of-service attack vector as well as just getting rid of a performance problem that could be especially problematic for latency-sensitive workloads. In short, there is a desire for x86 to be like other architectures in this regard. The implementation of this change has evolved considerably over time; in [the patch that was merged](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6650cdd9a8cc>), there is a new boot-time parameter (`split_lock_detect=`) that can have one of three values. Setting it to `off` disables this feature, `warn` causes a warning to be issued when user-space code executes a split lock, and `fatal` causes a `SIGBUS` signal to be sent. The default value is `warn`. 

The various discussions around split-lock detection included virtualization, which has always raised some interesting questions. A system that runs virtualized guests is a logical place to enable split-lock detection, since a guest can disrupt others with hostile locking behavior. But a host that turns on split-lock detection risks breaking guests that are unprepared for it; this problem extends to the guest operating system, which will be directly exposed to the alignment-check traps caused by split-lock detection. It may not be possible for the administrator of the host to even know whether the guest workloads are ready or not. So various kernel developers wondered what the best policy regarding virtualization should be. 

It seems that some of that discussion fell by the wayside as the final patch was being prepared, leading to an unpleasant surprise. Kenneth Crudup first reported that split-lock detection caused VMware guests to crash, but the problem turns out to be a bit more widespread than that. 

Intel's "virtual machine extensions" (VMX, also referred to as "VT-x") implements hardware-supported virtualization on x86 processors. A `VMLAUNCH` instruction places the processor in the virtualized mode, where the client's system software can (mostly) behave like it is running on bare hardware while being contained within its sandbox. It turns out that, if split-lock detection is enabled and code running within a virtual machine attempts a split lock, the processor will happily deliver an alignment-check trap to a thread running in the VMX mode; what happens next depends on the hypervisor. And most hypervisors are not prepared for this to happen; they will often just forward the trap into the virtual machine, which, not being prepared for it, will likely crash. Any hypervisor using VMX is affected by this issue. 

Thomas Gleixner responded to the problem with [a short patch series](<https://lwn.net/ml/linux-kernel/20200402123258.895628824@linutronix.de/>) trying to cause the right things to happen. One of the affected hypervisors is KVM; since it is a part of the kernel, the right solution is to just make KVM handle the trap properly. Gleixner included [a patch](<https://lwn.net/ml/linux-kernel/20200402124205.334622628@linutronix.de/>) causing KVM to check to see whether the machine was configured to receive an alignment-check trap and only deliver it if so. That patch is likely to be superseded by [a different series](<https://lwn.net/ml/linux-kernel/20200402155554.27705-1-sean.j.christopherson@intel.com/>) written by Xiaoyao Li, but the core idea (make KVM handle the trap correctly) is uncontroversial. 

The real question is what should be done the rest of the time. All of the other VMX-using hypervisors are out-of-tree, so they cannot be fixed directly. Gleixner's original patch was arguably uncharacteristic of his usual approach to such things: it disabled split-lock detection globally if a hypervisor module was loaded into the kernel. But, since modules don't come with a little label saying "this is a hypervisor", Gleixner's patch would, instead, read through each module's executable code at load time in search of a `VMLAUNCH` instruction. Should such an instruction exist, the module is deemed to be a hypervisor. Unless a special flag ("`sld_safe`") is set in the module info area, the hypervisor will be assumed to be unready for split-lock detection and the feature will be turned off. 

It is not at all clear that this approach will be adopted. Among other things, it turns out that not all VMX hypervisors include `VMLAUNCH` instructions in their code. As Gleixner [noted](<https://lwn.net/ml/linux-kernel/87o8s8l7t3.fsf@nanos.tec.linutronix.de/>) later in the discussion, VirtualBox doesn't directly contain _any_ of the VMX instructions; those are loaded separately by the VirtualBox module, outside of the kernel's module-loading mechanism. ""This 'design' probably comes from the original virtualbox implementation which circumvented GPL that way"", Gleixner observed. Other modules use `VMXON` rather than `VMLAUNCH`. 

Eventually these sorts of problems could be worked around, but there is another concern with this approach that was [expressed](<https://lwn.net/ml/linux-kernel/20200406122343.GA10683@infradead.org/>), in typical style, by Christoph Hellwig: 

This is just crazy. We have never cared about any out of tree module, why would we care here where it creates a real complexity. Just fix KVM and ignore anything else. 

There is a fair amount of sympathy for this approach in kernel-development circles, but there is still a reluctance to ship something that is certain to create unexpected failures for end users even if it is not seen as a regression in the usual sense. So a couple of other ideas for how to respond to this problem have been circulating. 

One of those is to continue scanning module code for instructions that indicate hypervisor functionality. But, rather than disabling split-lock detection on the system as a whole, the kernel would simply refuse to load the module. There are concerns about the run-time cost of scanning through module code, but developers like Peter Zijlstra also [see an opportunity](<https://lwn.net/ml/linux-kernel/20200406144020.GP20730@hirez.programming.kicks-ass.net/>) to prevent the loading of modules that engage in other sorts of unwelcome behavior, such as directly manipulating the CPU's control registers. [A patch](<https://lwn.net/ml/linux-kernel/20200407110236.930134290@infradead.org/>) implementing such checks has subsequently been posted. 

An alternative, [suggested](<https://lwn.net/ml/linux-kernel/20200406125010.GA29306@infradead.org/>) by Hellwig, is to find some other way to break the modules in question and prevent them from being loaded. Removing some exported symbols would be one way to do that. Zijlstra posted [one attempt](<https://lwn.net/ml/linux-kernel/20200406140403.GL20730@hirez.programming.kicks-ass.net/>) at "fixing" the problem that way; Hellwig has [a complementary approach](<http://git.infradead.org/users/hch/misc.git/shortlog/refs/heads/sanitize-vmalloc-api>) as well. 

As of this writing, it's not clear which approach will be taken; the final 5.7 kernel could be released with both of them, or some yet unseen third technique. Then, just maybe, the long story of x86 split-lock detection will come to some sort of conclusion.
