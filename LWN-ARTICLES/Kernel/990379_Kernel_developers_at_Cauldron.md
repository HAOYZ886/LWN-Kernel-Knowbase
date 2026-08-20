---
title: Kernel developers at Cauldron
url: https://lwn.net/Articles/990379/
date: "September 18, 2024"
category: "Development tools-Compiler toolchain; Development tools-SFrame"
author: "By Jonathan Corbet September 18, 2024 Cauldron"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
September 18, 2024

* * *

[Cauldron](<https://lwn.net/Archives/ConferenceByYear/#2024-GNU_Tools_Cauldron>)

A Linux system is made up of a large number of interdependent components, all of which must support each other well. It can thus be surprising that, it seems, the developers working on those components do not often speak with each other. In the hope of improving that situation, efforts have been made in recent years to attract toolchain developers to the kernel-heavy Linux Plumbers Conference. This year, though, the opposite happened as well: the [2024 GNU Tools Cauldron](<https://gcc.gnu.org/wiki/cauldron2024>) hosted a discussion where kernel developers were invited to discuss their needs. 

David Malcolm started the discussion by asking whether there is interest in performing more static analysis on the kernel. Steve Rostedt pointed out some of the tools that are used for that purpose now, noting that [sparse](<https://docs.kernel.org/dev-tools/sparse.html>) is useful for checking pointer annotations in the kernel. It can, for example, find code that does not treat user-space pointers with appropriate caution or follow the read-copy-update locking rules. David Faust said that there had been a proposal to incorporate the sparse annotations into BPF type format (BTF) tags, which might be possible with the help of [C2x](<https://en.wikipedia.org/wiki/C23_%28C_standard_revision%29>) attributes. 

Rostedt suggested that this kind of annotation could have helped to find a [recent BPF bug](<https://lwn.net/ml/bpf/ZrCZS6nisraEqehw@jlelli-thinkpadt14gen4.remote.csb/>). The BPF verifier was unaware of the fact that some tracepoints could fire with a null pointer value passed in and, as a result, did not require BPF programs attached to those tracepoints to check for null. That meant that some BPF programs were able to crash the kernel, which is not supposed to be possible. A "could be null" annotation could help the verifier in such situations. 

Malcolm pointed out that GCC has supported a `nonnull` attribute for a long time, but that is the opposite of the needed "might be null and must be checked" meaning. It is an optimization-related feature, and not really suited for this purpose. As the discussion moved on, Malcolm said that he had created [a tracker bug](<https://gcc.gnu.org/bugzilla/show_bug.cgi?id=106358>) on using GCC to run static analysis on the kernel. 

José Marchesi brought up the topic of the struct layout randomization GCC plugin that can be used to build the kernel. There is a problem, in that the randomization happens after the creation of debug data; if a structure's layout is reordered, the debug information will be incorrect. Clang, instead, orders the work correctly and does not have this problem. Segher Boessenkool asserted that the plugin is simply broken, and Sam James said that the plugin situation in general is "a mess". There is evidently a fix in circulation that depends on a new plugin hook for GCC. But I pointed out that the kernel project would like to move away from GCC plugins entirely in favor of having the necessary features supported directly by the compiler. Fixing the plugin would be welcome, but replacing it with a proper implementation would be better. 

There was some discussion about the value of struct layout randomization in general, with some calling it "security through obscurity". Rostedt, though, defended the technique as a useful way to limit the effectiveness of exploits. Bradley Kuhn said that there are some "serious licensing issues" around some of the GCC plugins used by the kernel, with some users getting "aggressive emails" from the original author of some of that work. It would be far better, he said, to rewrite that functionality from scratch, built into the compiler. 

It was also mentioned that layout reordering could also be used to optimize structures, eliminating internal holes. Boessenkool pointed out that this would violate the standard, which requires the address of a structure to be equal to the address of its first element. That seems like a price that some users would be willing to pay for a useful (and optional) feature, though. This part of the discussion concluded that struct layout reordering would be useful to have in GCC, but nobody said that they would work to actually implement it. 

Next on the list was the unwinding of user-space stacks in the kernel, possibly by making use of [SFrame](<https://lwn.net/Articles/940686/>) data. There is interest in possibly moving the kernel over to SFrame from the [ORC](<https://lwn.net/Articles/728339/>) format used by the kernel now, Rostedt said. User-space unwinding would be useful for the generation of profiles that include both the user and kernel sides of the equation. It would be implemented by setting a task flag in the kernel saying that a user-space stack trace is needed; that trace would be generated just prior to returning to user space from the kernel. 

Creating that trace would require access to user-space SFrame data from within the kernel, though. That, in turn, could require the addition of a system call for user space to provide that data. To complicate things, the SFrame situation could change rapidly over time, especially in processes running a just-in-time compiler. So perhaps the SFrame data would be stored in a memory region that is shared between the kernel and user space, eliminating the need to make a system call every time something changes. 

The final topic that was discussed was a desire to obtain hints from the compiler about functions that do not return or which contain jump tables. As Josh Poimboeuf explained, this information is needed to make the kernel's `objtool` utility work properly on 64-bit Arm systems; that, in turn, is needed to support live patching. Indu Bhagat said that this information could be useful for some control-flow-integrity applications as well. The discussion wandered inconclusively for a while after that, with no clear solution identified. 

The session did not manage to address half of the potential subjects that had been listed at the beginning. It did show, though, that there is value in getting groups of developers to talk with each other about their needs and wishes. The discussion will continue at the [Linux Plumbers Conference](<https://lpc.events/event/18/sessions/180/#20240918>) in Vienna and, presumably, at future events as well. 

[ Thanks to the Linux Foundation, LWN's travel sponsor, for supporting our travel to this event. ]
