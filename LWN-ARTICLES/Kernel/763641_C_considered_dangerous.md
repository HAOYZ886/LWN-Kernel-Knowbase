---
title: C considered dangerous
url: https://lwn.net/Articles/763641/
date: "August 29, 2018"
category: "Security-Kernel hardening"
author: "By Jake Edge August 29, 2018 LSS NA"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
August 29, 2018

* * *

[LSS NA](<https://lwn.net/Archives/ConferenceByYear/#2018-Linux_Security_Summit_NA>)

At the North America edition of the [2018 Linux Security Summit](<https://events.linuxfoundation.org/events/linux-security-summit-north-america-2018/>) (LSS NA), which was held in late August in Vancouver, Canada, Kees Cook gave a presentation on some of the dangers that come with programs written in C. In particular, of course, the Linux kernel is mostly written in C, which means that the security of our systems rests on a somewhat dangerous foundation. But there are things that can be done to help firm things up by ""Making C Less Dangerous"" as the title of his talk suggested. 

He began with a brief summary of the work that he and others are doing as part of the [Kernel Self Protection Project](<https://kernsec.org/wiki/index.php/Kernel_Self_Protection_Project>) (KSPP). The goal of the project is to get kernel protections merged into the mainline. These protections are not targeted at protecting user-space processes from other (possibly rogue) processes, but are, instead, focused on protecting the kernel from user-space code. There are around 12 organizations and ten individuals working on roughly 20 different technologies as part of the KSPP, he said. The progress has been "slow and steady", he said, which is how he thinks it should go. 

[ ![\[Kees Cook\]](https://static.lwn.net/images/2018/lssna-cook-sm.jpg) ](<https://lwn.net/Articles/763644/>)

One of the main problems is that C is treated mostly like a fancy assembler. The kernel developers do this because they want the kernel to be as fast and as small as possible. There are other reasons, too, such as the need to do architecture-specific tasks that lack a C API (e.g. setting up page tables, switching to 64-bit mode). 

But there is lots of undefined behavior in C. This "operational baggage" can lead to various problems. In addition, C has a weak standard library with multiple utility functions that have various pitfalls. In C, the content of uninitialized automatic variables is undefined, but in the machine code that it gets translated to, the value is whatever happened to be in that memory location before. In C, a function pointer can be called even if the type of the pointer does not match the type of the function being called—assembly doesn't care, it just jumps to a location, he said. 

The APIs in the standard library are also bad in many cases. He asked: why is there no argument to `memcpy()` to specify the maximum destination length? He noted a recent [blog post](<https://raphlinus.github.io/programming/rust/2018/08/17/undefined-behavior.html>) from Raph Levien entitled "With Undefined Behavior, Anything is Possible". That obviously resonated with Cook, as he pointed out his T-shirt—with the title and artwork from the post. 

#### Less danger

He then moved on to some things that kernel developers can do (and are doing) to get away from some of the dangers of C. He began with variable-length arrays (VLAs), which can be used to overflow the stack to access data outside of its region. Even if the stack has a guard page, VLAs can be used to jump past it to write into other memory, which can then be used by some other kind of attack. The C language is "perfectly fine with this". It is easy to find uses of VLAs with the `-Wvla` flag, however. 

But it turns out that VLAs are [not just bad from a security perspective](<https://lwn.net/Articles/749064/>), they are also slow. In a micro-benchmark associated with a [patch removing a VLA](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=02361bc77888>), a 13% performance boost came from using a fixed-size array. He dug in a bit further and found that much more code is being generated to handle a VLA, which explains the speed increase. Since Linus Torvalds has [declared that VLAs should be removed](<https://lore.kernel.org/lkml/CA+55aFzCG-zNmZwX4A2FQpadafLfEzK6CC=qPXydAacU1RqZWA@mail.gmail.com/T/#u>) from the kernel because they cause security problems and also slow the kernel down; Cook said "don't use VLAs". 

Another problem area is `switch` statements, in particular where there is no `break` for a `case`. That could mean that the programmer expects and wants to fall through to the next case or it could be that the `break` was simply forgotten. There is a way to get a warning from the compiler for fall-throughs, but there needs to be a way to mark those that are truly meant to be that way. A special fall-through "statement" in the form of a comment is what has been agreed on within the static-analysis community. He and others have been going through each of the places where there is no `break` to add these comments (or a `break`); they have "found a lot of bugs this way", he said. 

Uninitialized local variables will generate a warning, but not if the variable is passed in by reference. There are some GCC plugins that will automatically initialize these variables, but there are also patches for both GCC and Clang to provide a compiler option to do so. Neither of those is upstream yet, but Torvalds has praised the effort so the kernel would likely use the option. An interesting side effect that came about while investigating this was a warning he got about unreachable code when he enabled the auto-initialization. There were two variables declared just after a `switch` (and outside of any `case`), where they would never be reached. 

Arithmetic overflow is another undefined behavior in C that can cause various problems. GCC can check for signed overflow, which performs well (the overhead is in the noise, he said), but adding warning messages for it does grow the kernel by 6%; making the overflow abort, instead, only adds 0.1%. Clang can check for both signed and unsigned overflow; signed overflow is undefined, while unsigned overflow is defined, but often unexpected. Marking places where unsigned overflow is expected is needed; it would be nice to get those annotations put into the kernel, Cook said. 

Explicit bounds checking is expensive. Doing it for `copy_{to,from}_user()` is a less than 1% performance hit, but adding it to the `strcpy()` and `memcpy()` families are around a 2% hit. Pre-Meltdown that would have been a totally impossible performance regression for security, he said; post-Meltdown, since it is less than 5%, maybe there is a chance to add this checking. 

Better APIs would help as well. He pointed to the evolution of `strcpy()`, through `str**n** cpy()` and `str**l** cpy()` (each with their own bounds flaws) to `str**s** cpy()`, which seems to be "OK so far". He also mentioned `memcpy()` again as a poor API with respect to bounds checking. 

Hardware support for bounds checking is available in the application data integrity (ADI) feature for SPARC and is coming for Arm; it may also be available for Intel processors at some point. These all use a form of "memory tagging", where allocations get a tag that is stored in the high-order byte of the address. An offset from the address can be checked by the hardware to see if it still falls within the allocated region based on the tag. 

Control-flow integrity (CFI) has become more of an issue lately because much of what attackers had used in the past has been marked as "no execute" so they are turning to using existing code "gadgets" already present in the kernel by hijacking existing indirect function calls. In C, you can just call pointers without regard to the type as it just treats them as an address to jump to. Clang has a CFI-sanitize feature that enforces the function prototype to restrict the calls that can be made. It is done at runtime and is not perfect, in part because there are lots of functions in the kernel that take one unsigned long parameter and return an unsigned long. 

Attacks on CFI have both a "forward edge", which is what CFI sanitize tries to handle, and a "backward edge" that comes from manipulating the stack values, the return address in particular. Clang has two methods available to prevent the stack manipulation. The first is the "safe stack", which puts various important items (e.g. "safe" variables, register spills, and the return address) on a separate stack. Alternatively, the "shadow stack" feature creates a separate stack just for return addresses. 

One problem with these other stacks is that they are still writable, so if an attacker can find them in memory, they can still perform their attacks. Hardware-based protections, like Intel's Control-Flow Enforcement Technology (CET), [provides a read-only shadow call stack](<https://lwn.net/Articles/758245/>) for return addresses. Another hardware protection is [pointer authentication](<https://lwn.net/Articles/718888/>) for Arm, which adds a kind of encrypted tag to the return address that can be verified before it is used. 

#### Status and challenges

Cook then went through the current status of handling these different problems in the kernel. VLAs are almost completely gone, he said, just a few remain in the crypto subsystem; he hopes those VLAs will be gone by 4.20 (or whatever the number of the next kernel release turns out to be). Once that happens, he plans to turn on `-Wvla` for the kernel build so that none creep back in. 

There has been steady progress made on marking fall-through cases in `switch` statements. Only 745 remain to be handled of the 2311 that existed when this work started; each one requires scrutiny to determine what the author's intent is. Auto-initialized local variables can be done using compiler plugins, but that is "not quite what we want", he said. More compiler support would be helpful there. For arithmetic overflow, it would be nice to see GCC get support for the unsigned case, but memory allocations are now doing explicit overflow checking at this point. 

Bounds checking has seen some "crying about performance hits", so we are waiting impatiently for hardware support, he said. CFI forward-edge protection needs [link-time optimization](<https://lwn.net/Articles/744507/>) (LTO) support for Clang in the kernel, but it is currently working on Android. For backward-edge mitigation, the Clang shadow call stack is working on Android, but we are impatiently waiting for hardware support for that too. 

There are a number of challenges in doing security development for the kernel, Cook said. There are cultural boundaries due to conservatism within the kernel community; that requires patiently working and reworking features in order to get them upstream. There are, of course, technical challenges because of the complexity of security changes; those kinds of problems can be solved. There are also resource limitations in terms of developers, testers, reviewers, and so on. KSPP and the other kernel security developers are still making that "slow but steady" progress. 

Cook's [slides [PDF]](<https://outflux.net/slides/2018/lss/danger.pdf>) are available for interested readers; before long, there should be a video available of the talk as well. 

[I would like to thank LWN's travel sponsor, the Linux Foundation, for travel assistance to attend the Linux Security Summit in Vancouver.]
