---
title: Meltdown and Spectre mitigations — a February update
url: https://lwn.net/Articles/746551/
date: "February 5, 2018"
category: "Security-Meltdown and Spectre"
author: "By Jonathan Corbet February 5, 2018"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
February 5, 2018

The initial panic over the Meltdown and Spectre processor vulnerabilities has faded, and work on mitigations in the kernel has slowed since [our mid-January report](<https://lwn.net/Articles/744287/>). That work has not stopped, though. Fully equipping the kernel to protect systems from these vulnerabilities is a task that may well require years. Read on for an update on the current status of that work. 

#### Variant 1

Perhaps the biggest piece of unfinished business in January was a proper response to Spectre variant 1 — the bounds-check bypass vulnerability. Variant 1 is likely to be difficult to fix at the hardware level, and so may be with us for a long time. Unfortunately, it is also difficult to address at the software level. 

The seemingly final form of [the patches for variant 1](<https://lwn.net/Articles/746576/>) has changed the interface yet again. Consider a simple code fragment that might be vulnerable to speculation that bypasses a bounds check: 

```
if (index < ARRAY_SIZE)
    	return array[index];
```

The way to protect this kind of reference to ensure that no out-of-bounds references to `array` occur would be: 

```
if (index >= ARRAY_SIZE)
        	return 0;  /* Or whatever error value makes sense */
        else {
        	index = array_index_nospec(index, ARRAY_SIZE);
    	return array[index];
        }
```

The protective macro `array_index_nospec()` no longer actually accesses the array; instead, it just manipulates the index value in a way that blocks speculation. It uses the masking technique described in the mid-January article, avoiding the rather more expensive barrier operations entirely. For cases where the operation that needs to be protected is more complex than a simple array access, there is another macro called `barrier_nospec()` that does use a barrier to block all speculative activity. It is rather more expensive to use than `array_index_nospec()`, on the x86 architecture at least, but sometimes there is no alternative. 

Actual uses of these new macros are relatively scarce at the moment. The `get_user()` function in the kernel is one area of concern, since it can be used to attempt an access to an arbitrary address in the kernel; since `get_user()` has the necessary bounds check to ensure that the given address points into user space, adding a call to `array_index_nospec()` (more correctly, an optimized assembly version of it for x86) is enough to prevent problems. The `__get_user()` variant, though, lacks those checks and is invoked in well over 1,000 call sites in the kernel. Protecting `__get_user()` requires tossing in a `barrier_nospec()` invocation. 

Another area of concern is the system-call table, which is indexed using an integer value (the system-call number) from user space. A call to `array_index_nospec()` is used to prevent out-of-bounds access to that table. Protections have also been added for file-descriptor lookup, in the KVM code, and in the low-level wireless networking code. Nobody believes that all of the potentially exploitable places have been found, though. 

Meanwhile, there is [an arm64 patch set](<https://lwn.net/Articles/746550/>) in circulation with mitigations similar to the x86 patches. ~~It has fewer mitigations currently~~ It doesn't repeat the non-architecture-specific mitigations found in the x86 tree, but does add protections to the `[futex()](<http://man7.org/linux/man-pages/man2/futex.2.html>)` system call that are not currently present (and maybe not needed) for x86. 

Finding the remaining locations where variant-1 protection is needed is likely to require fairly advanced static-analysis tools. The work done so far has relied on the proprietary Coverity tool, and has had to contend with a high false-positive rate. Everybody involved would like to see a free tool that could do this work, but nobody is apparently working on such a thing. That is certain to slow the rate at which vulnerable code is found and increase the rate at which new vulnerabilities are introduced. 

Arjan van de Ven has [suggested](<https://lwn.net/Articles/746578/>) that what really needs to happen is a centralization of many of the security checks that are currently dispersed throughout the kernel. He recommends the creation of a utility function described as: 

```
copy_from_user_struct(*dst, *src, type, validation_function);
```

Where the `validation_function()` would be automatically generated from the UAPI headers that describe the interface to the kernel. Widespread use of a function like this would free most developers from the need to worry about Spectre variant-1 vulnerabilities; it might also improve the (not always great) state of argument validation in general. 

#### Variant 2

Spectre variant 2 (branch prediction buffer poisoning) has been mostly mitigated by way of the "retpoline" mechanism that was merged for the 4.15 kernel release. With the [GCC 7.3 release](<https://lwn.net/Articles/745385/>), a retpoline-capable compiler is finally available. There are, however, a number of loose ends that are slowly being dealt with. 

There is still a fair amount of uncertainty around the question of when retpolines provide sufficient protection. The "indirect branch prediction barrier" (IBPB) operation provided by Intel in recent microcode updates will protect against poisoning, but its use is expensive, so there is a desire to avoid it whenever possible. There are cases, though, such as switching into and out of a virtualized guest, where IBPB is needed. 

There is also the inconvenient fact that Intel released a number of microcode versions with implementations of IBPB that, to put it politely, did not function as well as users would have liked. Dealing with that last problem requires avoiding IBPB entirely on the affected microcode versions. There was some discussion over whether the kernel should blacklist known-bad versions or use a whitelist of known-good alternatives; the latter approach was somewhat driven by worries that Intel was never going to get things right. In the end, though, the blacklist approach won out, on the theory that the problems have, in the end, been fixed. 

Similar concerns relate to the "indirect branch restricted speculation" (IBRS) barrier-like operation that, by some accounts, is needed to get full protection on Intel Skylake-generation processors. That, too, has had issues with some microcode versions. Those too, with luck, have been fixed; if not, David Woodhouse [warned](<https://lwn.net/Articles/746582/>): ""then I think Mr Shouty is going to come for another visit."" 

There is still some resistance to using IBRS at all, though. It also is an expensive operation, and nobody has demonstrated an exploit on Skylake processors when it is not used. Meanwhile, Ingo Molnar has [proposed](<https://lwn.net/Articles/746583/>) a different approach: use the ftrace machinery to keep track of the number of entries in the return-stack buffer (RSB) and force the use of a retpoline when it gets too deep. It is not yet clear that this idea can be implemented in a practical way; Thomas Gleixner has [played with it](<https://lwn.net/Articles/746585/>) but he ran into some complications and set it aside for now. 

One concern about variant 2 is that it might lend itself to attacks by one user-space process (or thread) against another. JavaScript code running in a browser appears to be the most likely vector for such an attack, but it's not the only one. [This patch](<https://lwn.net/Articles/746586/>), for example, is an attempt to protect high-value processes by issuing an IBPB barrier prior to switching into a process that has marked itself as being non-dumpable. The idea is to provide some protection for programs like GnuPG while avoiding the overhead of IBPB on every context switch. 

#### Other odds and ends

Protection against Meltdown ("variant 3") was mostly in place when the embargo fell in January; its basic form has not changed much since then. There are numerous bugs to fix, of course, and that work has been ongoing. The arm64 architecture gained kernel page-table isolation during the 4.16 merge window. There has also been some work on the x86 side to avoid using kernel page-table isolation on systems that do not need it — AMD processors and ancient Intel processors, for example. The whitelist of safe processors is slowly growing. 

Systems with Meltdown and Spectre mitigations also have a new sysfs directory (`/sys/devices/system/cpu/vulnerabilities`) listing known CPU vulnerabilities and their mitigation status. On your editor's laptop, they currently read: 

```
meltdown:	Mitigation: PTI
        spectre_v1:	Vulnerable
        spectre_v2:	Mitigation: Full generic retpoline
```

There have been some [concerns](<https://lwn.net/Articles/746587/>) that these files, which are world-readable, provide useful information to attackers and should be restricted. On the other hand, Alan Cox [responded](<https://lwn.net/Articles/746588/>), that this information is already readily available and it can be useful to utilities like just-in-time compilers, which might change their output when certain vulnerabilities are present. As of this writing, no patches changing the protections on those files have been merged. 

Other than that, though, everything described here has been merged for 4.16 and is quickly headed toward the stable kernel updates as well. There are a number of smaller issues not described here that were also addressed for 4.16; see [this pull request](<https://lwn.net/Articles/746589/>) for the full list. Clearly, even if things have slowed a bit to allow the developers involved to get some sleep, a lot is still happening to deal with the fallout from Meltdown and Spectre.
