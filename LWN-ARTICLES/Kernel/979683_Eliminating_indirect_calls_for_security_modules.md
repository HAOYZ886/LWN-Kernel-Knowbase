---
title: Eliminating indirect calls for security modules
url: https://lwn.net/Articles/979683/
date: "July 2, 2024"
category: "Releases-6.12; Security-Security modules"
author: "By Jonathan Corbet July 2, 2024"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
July 2, 2024

Like many kernel subsystems, the Linux security module (LSM) subsystem makes extensive use of indirect function calls. Those calls, however, are increasingly problematic, and the pressure to remove them has been growing. The good news is that there is [a patch series](<https://lwn.net/ml/all/20240629084331.3807368-1-kpsingh@kernel.org>) from KP Singh that accomplishes that goal. Its progress into the mainline has been slow — this change was first [proposed](<https://lwn.net/ml/linux-security-module/20200820164753.3256899-1-jackmanb@chromium.org/>) by Brendan Jackman and Paul Renauld in 2020 — and this work has been caught up in some wider controversies along the way, but it should be close to being ready. 

A security module provides a set of hooks, one for each operation within the kernel that it wants to control. Whenever that operation (opening a file, for example, or creating a new process) is invoked by user space, the security module's hook function will be called with information about the requested action. The hook then has the opportunity to see whether an action is allowed by the policy it is meant to enforce and, if not, block that action. The kernel can have more than one security module active at a time, each of which provides its own hook functions. Those functions are stored in a linked list; traversing that list and calling all of the hook functions is where the indirect calls come in. 

Calling functions through pointers is, of course, a common C-language technique that is used heavily throughout the kernel. These indirect function calls have increasingly come under scrutiny in recent years, mostly as a result of the threat posed by the Spectre class of hardware vulnerabilities. Indirect function calls can be points where a number of speculative-execution vulnerabilities can be exploited. The gyrations required to thwart such exploits — notably [retpolines](<https://lwn.net/Articles/774743/>) — come with a heavy run-time cost. 

That added cost is especially painful when it comes to the indirect function calls used by security modules. Almost anything that user space can do, if it involves the kernel, will be mediated by at least one security-module hook; if those hooks are made more expensive, the pain is felt throughout the system. The added performance hit is prohibitive on systems that are already running at full capacity, with the result that the use of security modules is not possible for some workloads. It is thus not surprising that there is interest in getting that lost performance back. 

The attention to security modules increased in April with the [disclosure](<https://lwn.net/Articles/969210/>) of the branch history injection hardware vulnerability, which is, once again, exploitable in code using indirect function calls. The LSM subsystem is, arguably, an especially appealing target for such exploits because it makes so many indirect calls, its hooks are attached to almost every system call, and the LSM call is often one of the first things done on entry into the kernel. 

This vulnerability forced the use of retpolines on CPUs that had, previously, been able to get away with less-expensive mitigations provided by the hardware. That provoked [a touchy conversation](<https://lwn.net/Articles/970070/>) with Linus Torvalds, who questioned the direction that the LSM subsystem had been taking for the last ten years. In the middle of that, though, he also [said](<https://lwn.net/ml/linux-security-module/CAHk-=wh+_xXpnHfUv=FwGWcce4XwqwKvtq7LcxK6WKmbf4eGGA@mail.gmail.com/>) that the work to switch the LSM subsystem to using static calls ""needs to be turned to 11""; that was the one part of Torvalds's message that nobody disagreed with strongly. 

#### Static calls

Static calls are widely used within the kernel in situations where an indirect call is necessary, but the target for that call is set only once (or at most rarely) in the life of the system. Their purpose is to provide the flexibility of indirect calls (albeit with an increased cost for changing the target of the call) with the improved performance and security of direct calls. The static-call infrastructure was first [added for the 5.9 kernel release](<https://git.kernel.org/linus/115284d89a43>) in 2020; it is conspicuously absent from the kernel's documentation, but there is an overview of the API in [`include/linux/static_call.h`](<https://elixir.bootlin.com/linux/v6.10-rc5/source/include/linux/static_call.h>). 

In the simplest case, kernel code will set up a static call with: 

```
DEFINE_STATIC_CALL(name, func);
```

Where `name` is the name to be associated with the static call, and `func()` is the function to be invoked. The `static_call()` macro can then be used to call the function: 

```
static_call(name)(args...);
```

This call will work like a normal function call; `func()` will be called with the given `args`, and its return value will be passed back to the caller. It is, however, a _direct_ call, much as if `func()` had been called directly in the code. As a result, this call is faster than an indirect call and lacks the associated speculative-execution problems. 

The value of indirect calls, of course, is that the target can be changed at run time; that is not normally the case with direct calls. Static calls use some architecture-specific trickery to get around this problem; if the target of a static call needs to be changed, that can be done with a call to: 

```
static_call_update(name, new_func);
```

After this call is made, a `static_call(name)()` invocation will make a (direct) call to `new_func()` rather than `func()`. 

How this mechanism is implemented depends on the architecture. Some architectures (arm and x86) are able to patch the call instructions directly in line, meaning that static calls are indeed just like normal direct calls and have no additional overhead. That said, the cost of patching the code in a running kernel is high, making suitable for use only in situations where the function to be called will be changed infrequently, if ever. Other architectures need to use a special trampoline for the static call; for architectures with no support at all, ordinary indirect function calls are used. There is more complexity to the API than described here; see the above-linked header file for details. 

#### Bringing static calls to LSMs

While the LSM subsystem is, as its name would suggest, modular, it is not set up for the arbitrary loading and unloading of modules. Instead, the set of available security modules is established (through the kernel configuration) at build time, and those modules are built directly into the kernel image. The set of _active_ security modules is then defined at boot time and never changes during the operation of the system. So the set of hook functions to be called can be worked out at boot time, and need never be altered thereafter. This seems like a situation that is well suited to static calls; that is, indeed, the approach taken by Singh's patch set. 

In current kernels, as mentioned above, a linked list of hook functions is maintained for each LSM hook; the kernel iterates through that list to invoke each hook function with an indirect call. With this patch series applied, that linked list is replaced with an array of static calls; the LSM subsystem now just has to step through the array, calling each hook in turn. In theory, the conversion should be straightforward. In practice, of course, there turns out to be a few little details that get in the way. 

One of those details relates to the fact that an LSM need not supply functions for every hook. In the old implementation, a missing hook would be absent from the linked list and would never be invoked, but an array works differently. It [turns out](<https://lwn.net/ml/linux-security-module/20220609234601.2026362-1-kpsingh@kernel.org/>) that providing a hook that returns the default value can have unwanted side effects; it is _not_ the same as leaving out the hook entirely. So each entry in the arrays of hook function must be protected by a [static key](<https://docs.kernel.org/staging/static-keys.html>) to avoid calls when a hook function is absent. 

There are other troublesome details as well. The set of possible security modules is defined in the kernel configuration and is known at boot time. A command-line parameter is then used to control both which modules are enabled and the order in which they are invoked. The kernel must then, at boot time, set up the requisite static calls in the correct order; the number of these calls, and the order in which they must be made, cannot be known ahead of time. There is some trickiness and [ugly macro code](<https://lwn.net/ml/all/20240516003524.143243-3-kpsingh@kernel.org>) involved, but the result is an end to indirect calls for LSM hooks. 

The result of this work is a performance improvement that averages about 3% and a system that, without all those indirect calls, is more secure overall. 

This patch set has been through 13 revisions since Singh picked it up at the beginning of 2023; it appears to have satisfied most reviewers. Kees Cook [asked](<https://lwn.net/ml/all/202406061049.F2FD4F8A@keescook>) for it to be merged soon, lest Torvalds return and ""make unilateral changes to the LSM"". But LSM subsystem maintainer Paul Moore [pushed back](<https://lwn.net/ml/all/CAHC9VhQyy6u9O=KPE4vNR3FO+5ofGky8hw53jadxW2n0RyLd=g@mail.gmail.com>), saying that he simply has not had the time to review the current version of the patches. More than two months after the last discussion, it seems that this is still a bit of a touchy subject. 

Nearly three weeks later, nothing appears to have changed, so whether this work will be applied in time for 6.11 is unclear. If that doesn't happen, though, a 6.12 merge seems almost certain (unless some sort of new problem turns up). Either way, the days of indirect calls in the LSM subsystem would appear to be numbered.
