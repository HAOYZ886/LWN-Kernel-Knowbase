---
title: Using drgn on production kernels
url: https://lwn.net/Articles/952942/
date: "November 28, 2023"
category: "Debugging; Development tools-Kernel debugging"
author: "By Jake Edge November 28, 2023 LPC"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
November 28, 2023

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2023-Linux_Plumbers_Conference>)

The [drgn](<https://github.com/osandov/drgn>) Python-based kernel debugger was developed by Omar Sandoval for use in his job on the kernel team at Meta. He now spends most of his time working on drgn, both in developing new features for the tool and in using it to debug production problems at Meta, which gives him a view of both ends of that feedback loop. At the [2023 Linux Plumbers Conference](<https://lpc.events/event/17/page/198-lpc-2023-overview>) (LPC), he led a session on drgn in the [kernel debugging microconference](<https://lpc.events/event/17/sessions/167/#20231114>), where he wanted to brainstorm on how to add some new features to the debugger and, in particular, how to allow them to work on production kernels. 

#### Quick intro

Sandoval had a [presentation on drgn](<https://lwn.net/Articles/789641/>) (which is pronounced "dragon") in 2019 that covered some of the basics of the tool, which has presumably evolved since then. He has given other in-depth talks on drgn, he said, but he would just be doing a quick introduction to the tool at the LPC session. After that, he wanted to focus on the two features, writing to memory and setting breakpoints, and to justify why his team wants to be able to do those things on kernels running in production ("as crazy as it sounds"). He hoped that the brainstorming could come up with both a mechanism for supporting the features and an API that is "friendly enough, but also not so dangerous in the sense that you won't accidentally do something that you didn't mean to do". 

[ ![\[Omar Sandoval\]](https://static.lwn.net/images/2023/lpc-sandoval-sm.png) ](<https://lwn.net/Articles/953121/>)

Drgn is a "programmable debugger"; rather than having built-in commands, it provides building blocks, representations of kernel objects, types, stack traces, and more, that can be used to create the tool needed for the job at hand. There are, for example, many kernel-specific helper functions that provide access to various internal data structures, such as to find task structures or to walk various slab caches. Those can be used in an interactive session and then turned into scripts that can be saved (or shared with others) for the next time a similar problem arises, he said. 

He did a brief demo of drgn on a virtual machine on his laptop; the [YouTube video](<https://youtu.be/68EBBgEltXA?t=18670>) of the presentation from the conference livestream is available for the curious. In the Python read-eval-print loop (REPL), he had a handful of import statements pre-typed and then proceeded to demonstrate some of the capabilities, such as looking up the idle task using its variable name (`init_task`) with one of the kernel helpers. 

He also showed a loop using the `for_each_task()` kernel helper that found the task structure for a `cat` he had running in a shell; he could then print the stack trace for that task, which had all of the symbol information, filenames, and line numbers. He used the stack-frame number to index into an array to further investigate a particular stack frame, including things like its local variables and their values. There are also a large number of contributed scripts that consist of "people's debugging sessions" that can be examined in addition to all of the helpers that come with drgn. 

All of what he had shown is read-only, however; you can read any memory in the live kernel or in a kernel dump. But users have been asking for read-write features for the live kernel for some time; that would allow overwriting memory and setting breakpoints in the running kernel. That makes sense for development workflows, Sandoval said; drgn could attach to the kernel running in QEMU using its GDB stub. That functionality is something that developers are used to when debugging, so he would like to support it in drgn. 

He has a proposed memory-writing API that, at its most basic level, just takes a target address and buffer of bytes to write there, which makes the user responsible for figuring out the right address and how to place the values into the buffer correctly based on the kernel type. On top of that would be a more user-friendly interface that would mirror the read side to a certain extent; objects can be looked up, then their fields can be used as Python attributes, with drgn ensuring that the write is done correctly. It could potentially also take a Python dictionary with structure fields as keys to write a structure with those values. The API is still up for debate as he has not implemented anything yet. 

"Breakpoints are a little more complicated, but not too much." There are a few different ways a user might want to set a breakpoint: by address, function name, function name and offset, or filename and line number. Then handling any breakpoints might be done with a synchronous event loop, where the events indicate which thread hit the breakpoint, allow access to things like the stack trace and local variables for the stack frames, and provide a way to resume the thread after the processing is done. 

Once again, Sandoval said that he was interested in hearing about simpler alternatives or use cases that still needed to be covered. Chris Mason said that he wanted to be able to see when a frequently called function is being called from some other specific function; Sandoval said that could be done with his API just by looking at the stack frame in the breakpoint and resuming unless it is being called by the function of interest. Another attendee suggested watchpoints for memory, which Sandoval thought could be added to the API in a way similar to the `set_breakpoint()` call he was proposing. 

Because drgn is programmable, many of the different use cases can be handled with programs of different sorts, he said. If some of the use cases need a performance boost, perhaps BPF could be used to do things like pre-filter breakpoints. Another attendee suggested using drgn for doing error injection in testing, which Sandoval thought could fit right in, though there may be a need for a way to overwrite registers as part of the API. 

#### Production

Those features are obviously useful in development, but his team at Meta has run into a few scenarios where it would be helpful to be able use them on production systems. For example, there have been cases where being able to overwrite some part of memory in the kernel would be enough to work around an emergency that has gotten people out of bed. It could be used to fix reference counts that are not getting decremented correctly, reset overflows or underflows of accounting information, change invalid states, and more. 

A more concrete example is "an embarrassing bug in Btrfs" (fixed by [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=981a37bab5e5>)) where enabling asynchronous discard was not handled quite correctly. It manifested in reports of disks starting to run slowly, which was eventually tracked down to discards (i.e. telling the device that certain blocks are no longer used so that the it can do garbage collection on them) not being issued at all. After some "heroic debugging", the problem was tracked down and promptly fixed in the tree, but it would have been convenient to be able to run a drgn script on affected machines to change the single bit that would actually enable discard for the Btrfs mounts. 

There are "a lot of caveats about doing this in production", though. You have to be careful about what you are overwriting—and when—"race conditions are definitely a thing". It is not meant as a replacement for a live patch or an updated kernel, but is instead a test of the fix and a stopgap measure. He hoped that explained the why, so he wanted to turn to "all the crazy ways we might be able to do this". 

For development, the solution is easy, Sandoval said, simply use the GDB stub that is provided by QEMU. He listed some possibilities for the production use case, starting with bringing back `/dev/kmem`, which is "almost a joke" of a suggestion. He mentioned an [LWN article](<https://lwn.net/Articles/851531/>) that celebrated its removal and he noted that the `/dev/kmem` interface to read and write the kernel's memory was a "beautiful thing for rootkits". Drgn is not a rootkit, but debuggers do share some elements with rootkits, so `/dev/kmem` would be the "most straightforward way to support this, but I don't think anyone is going to accept that patch". 

An alternative might be a custom kernel module that is effectively `/dev/kmem`, which is not that much better, he said. But, in order to enable the feature, there will need to be some way to write values to an arbitrary address, so the key will be in getting the access controls right. BPF could perhaps be used, but that "kind of goes against everything that BPF believes in, which is that your program should be safe". 

Another possible approach would be to interface to [kgdb](<https://www.kernel.org/doc/html/latest/dev-tools/kgdb.html>), though Meta does not enable it in production kernels "but it's not the worst thing we could do". Kgdb already supports both memory writes and breakpoints, but, as far as he can tell, it was never intended to be used on a live, running system. For example, hitting a breakpoint stops every CPU, so drgn cannot still be running; perhaps it could be modified to only stop certain CPUs and leave one running for drgn. 

An attendee asked why users would want other CPUs running when kgdb hits a breakpoint. The whole idea is that other CPUs cannot interfere with the state of the kernel at the time of the breakpoint. Sandoval said that works fine when there is another system that is driving the debugging, but that he wants to be able to log into a broken machine and run drgn there. Another audience member said that if there is a breakpoint set, drgn could cause it to be hit, leading to a deadlock. 

Sandoval acknowledged that problem as a difficult one. His idea is to have a watchdog that would raise an non-maskable interrupt (NMI) to cancel the breakpoint in the deadlock situation. [Kprobes](<https://www.kernel.org/doc/html/latest/trace/kprobes.html>) were identified as another way to do breakpoints, which Sandoval thought might be workable. There would still need to be a kernel module that alerted drgn that the kprobe/breakpoint had been hit, as well as a watchdog for deadlock prevention, he said. 

The [kernel lockdown mode](<https://man7.org/linux/man-pages/man7/kernel_lockdown.7.html>) was brought up as a potential problem area by a participant; it is meant to restrict any mechanisms that might alter the running kernel, and may well be enabled on many production kernels. So had Sandoval thought about how drgn might work with—or around—lockdown? It probably makes sense to just disable drgn support on locked-down kernels, Sandoval said. 

When considering access control, the features that he wants to add to drgn are things that already could be done from a custom kernel module, thus `CAP_SYS_MODULE` and `CAP_SYS_ADMIN` could perhaps control access to whatever underlying mechanism is decided upon. There is the caveat that some organizations require signed kernel modules beyond just having the capabilities. That might mean that the drgn mechanism needs to validate the user based on keys on the kernel keyring in some fashion. 

Stephen Brennan pointed out that Python itself loads lots of code from various locations on the system that needs to be somehow protected so that running drgn does not become a compromise vector. Sandoval said that he "kind of copped out and made it a per-user authentication thing", so that the user has to be careful about those kinds of things, but that type of access control has not worked out so well over the years, he said, pointing to setuid binaries in particular. 

Instead of having full breakpoints in drgn, Mason said, there could be a limited set of things that can be done when the code is reached. That could then be turned into BPF or a kprobe, which would then need to be inserted into the kernel; it would not change the security picture at all, but would simplify the problem of stopping all the CPUs and prevent the deadlocks. Sandoval said that one of the things in that defined set would need to be writing memory, however, so some solution for that part of the problem would still be required. 

As time ran out, he wrapped up by saying that he still had "more questions than answers", but encouraged attendees to find him later to discuss "more bad ideas"—or so that he could show them "cool drgn stuff", he said with a chuckle. 

[I would like to thank LWN's travel sponsor, the Linux Foundation, for assistance with my travel costs to Richmond for LPC.]
