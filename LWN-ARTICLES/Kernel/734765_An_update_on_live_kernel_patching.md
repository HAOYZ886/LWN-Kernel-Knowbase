---
title: An update on live kernel patching
url: https://lwn.net/Articles/734765/
date: "September 27, 2017"
category: "Linux Plumbers Conference-2017"
author: "By Jake Edge September 27, 2017 Linux Plumbers Conference"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
September 27, 2017

* * *

[Linux Plumbers Conference](<https://lwn.net/Archives/ConferenceByYear/#2017-Linux_Plumbers_Conference>)

In the refereed track at the 2017 Linux Plumbers Conference (LPC), Jiri Kosina gave an update on the status and plans for the live kernel patching feature. It is a feature that has a long history—pre-dating Linux itself—and has had a multi-year path into the kernel. Kosina reviewed that history, while also looking at some of the limitations and missing features for live patching. 

The first question that gets asked about patching a running kernel is "why?", he said. That question gets asked in the comments on [LWN articles](<https://lwn.net/Articles/706327/>) and elsewhere. The main driver of the feature is the high cost of downtime in data centers. That leads data center operators to plan outages many months in advance to reduce the cost; but in the case of a zero-day vulnerability, that time is not available. Live kernel patching is targeted at making small security fixes as a stopgap measure until the kernel can be updated during a less-hurried, planned outage. It is not meant for replacing the kernel bit by bit over time, but as an emergency measure when the kernel is vulnerable. 

#### History

The history of the idea behind live patching goes back at least as far as the 1940s, he said. He referenced the classic Richard Feynman book [_Surely You're Joking, Mr. Feynman!_](<https://en.wikipedia.org/wiki/Surely_You%27re_Joking,_Mr._Feynman!>), where Feynman described a system he used to change the program being run by early proto-computers. He color-coded certain groups of punch cards in the program. That way, he could replace a small subset of the program in a non-destructive way. That was the beginning of live patching, Kosina said. 

The first implementation of live patching for Linux that he is aware of is ksplice, which was [announced in 2008](<https://lwn.net/Articles/280058/>). It was originally a research project for a PhD. thesis and the code was released as open-source software. The mechanism used `stop_machine()` to stop the kernel, then inspected the stack to see if the patch would interfere with any task currently running. If the function being patched was found on the stack, ksplice refused to patch it and retried later. 

One of the major contributions that ksplice made was in its automatic patch generation by comparing binary kernels, he said. The original kernel binary and the patched kernel binary were compared. Function inlining and other optimizations make it hard to know what actually will change even from a simple source code change. The ksplice project was acquired by Oracle in 2011 and the source code was closed; it is still used by the Oracle Linux distribution today. 

Based on requests from SUSE customers, Kosina had been working on an alternative approach, kGraft, which was [released in 2014](<https://lwn.net/Articles/596854/>). Around the same time, Red Hat [released kpatch](<https://lwn.net/Articles/597407/>), which it had been working on; both were aimed at live kernel patching, but had different ways to to achieve convergence to a fully patched state. Kpatch was similar to ksplice, in that it stopped the system and inspected its state, while kGraft used a lazy migration technique to slowly migrate all processes to use the new code. That lazy migration normally takes just milliseconds to complete. 

The kGraft patches are commercially supported by SUSE, which violates the company's "upstream first" principle, he said. Patches are created manually, with the help of the toolchain, which has some advantages over automatic binary comparisons. Even with automatic generation, there is a need to look at the patch generated (and to possibly adjust it); for example, if a structure needs to change, existing versions of the structure need to be modified in place. There is still a need for more tooling to assist with the manual patch generation, he said. 

As a side note, Kosina pointed to the [checkpoint/restore in user space](<https://lwn.net/Articles/525675/>) (CRIU) project as another potential way to do a kind of live patching. For some use cases, it might make sense to checkpoint all of the user-space processes, `kexec()` to the new kernel, then restore all of user space. That would allow changing to a completely new kernel, but it would not be immediate (or live). It also would reinitialize the hardware, which may not be desirable. 

He went into a bit more detail on the lazy migration scheme. After the patch is made, a process that enters or leaves the kernel gets marked as now living in the "new universe", so it will always get the patched function from that point on. Anything that is running in the kernel at the time of the patch will end up running the old version of the code; a trampoline function is used to decide which of the two versions of the function to call. Kernel threads have been marked with "safe points" where the switch can be made, which turned out to not be that difficult, surprisingly. In addition, long-sleeping processes (e.g. blocking in `get_tty()`) are identified and sent a fake signal that simply has the effect of setting the new-universe flag and putting them back to sleep. 

#### A meeting of the minds

There were competing solutions, so a [meeting was held](<http://linuxplumbersconf.org/2014/wp-content/uploads/2014/10/LPC2014_LivePatching.txt>) at the 2014 LPC in Düsseldorf to discuss the matter. Each solution was presented and the developers came up with a plan to try to merge one unified scheme. It would start with a minimal base on top of Ftrace, with a simple API. Live patches could be registered with a list of functions to be replaced, and it only supported a limited set of patch types that could be applied. That was [merged into the mainline](<https://lwn.net/Articles/634649/>) in February 2015. 

Since then, ideas have been cherry-picked from kpatch and kGraft to be added to the kernel under the `CONFIG_LIVEPATCH` option. There is now a combined, hybrid consistency model that uses lazy migration by default, but falls back to stack examination for long-sleeping processes and kthreads. Originally, the feature was x86-only, but it has been added to s390 and PowerPC-64, with ARM64 in the works. 

The stack examination is a crucial piece of the feature; without reliable stack unwinding, it is impossible to provide consistency. Josh Poimboeuf created the [ORC unwinder](<https://lwn.net/Articles/728339/>) to provide a reliable way to get a stack trace. In addition, [`objtool` (formerly `stacktool`) has been added](<https://lwn.net/Articles/658333/>) to ensure that assembly language pieces of the kernel will also produce a valid stack trace. 

Earlier efforts at getting reliable stack traces either used frame pointers, which had a severe performance penalty, or DWARF debugging records, which turned out to be unreliable and slow. ORC is effectively a stripped-down version of DWARF that has nothing more than is needed for reliable stack unwinding. The ORC unwinder was merged into 4.14 and will also be used for oops and panic output. So far, it is only available for x86_64, but is in progress for other architectures; the main work is on `objtool`, Kosina said, as the ORC unwinder is straightforward to port. 

Patches are currently hand-written, though tools are coming. The source for a patch is a single C file, which makes it easy to review and to store in Git. It creates new functions and declares them as replacements for existing kernel functions; that gets compiled into a loadable kernel module that has an initialization function to register the replacements and then to enable those changes. 

#### More to do

There are some limitations of the feature, currently. For one, there is no way to deal with data structure changes or changes to the semantics of existing elements. There may be a straightforward solution for simply adding a new field to an existing structure using shadow variables. A "lazy state transformation", analogous to lazy migration, may be another way to deal with changing data structures; new functions that can work with both the old and new structures could be created. 

There are still some problems with those approaches, however. Many kernel data structures are protected by exclusive access mechanisms, such as spinlocks and mutexes, which will be problematic to handle. If the locking rules need to change as part of the patch, it will be difficult to avoid deadlocks. There is also an effort to provide ways to fix things up during the patching process using patch callbacks, though that functionality will need to be used with some care. 

There are lots of traps in verifying that the patches created will still be within the consistency model; certain things just may not fit. That is currently verified through inspection and reasoning; a [guide for patch authors](<https://github.com/dynup/kpatch/blob/master/doc/patch-author-guide.md>) has been started to help with that as well. There is a lot of work being done on tooling to help tame the combinatorial explosion that comes from different optimizations that GCC will perform. For example, GCC can change the ABI for functions if it knows about all of the callers, so patches to those functions cannot be handled (or the GCC `-fipr-ra` option must not be used). Many of those kinds of problems could be detected automatically, Kosina said. 

Kprobes are another tricky area. It is difficult to switch an existing kprobe to a new function, which may cause some surprises. There is also an inability to patch hand-written assembly code; Ftrace is not able to work with that code. User-space live patching is something that could perhaps be done, but is much more difficult. For one thing, user-space applications are often built with tools other than GCC, which expands the problem. In addition, it is harder to define a checkpoint where the consistency can be assured. 

Kosina answered a few questions after the talk. The kernel address-space layout randomization (KASLR) feature has no impact on live patching. Loadable modules, on the other hand, are not easily handled. Patching the on-disk version of the module and causing a reload may be the best approach. Module signing also came up; live patches are modules, so if signed modules are required, the patch itself will need to be signed before it can be loaded. 

[I would like to thank LWN's travel sponsor, The Linux Foundation, for assistance in traveling to Los Angeles for LPC.]
