---
title: Tracing unsigned modules
url: https://lwn.net/Articles/588799/
date: "March 5, 2014"
category: "Development model-Loadable modules; Modules-Signed"
author: "By Jake Edge March 5, 2014"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
March 5, 2014

The reuse of one of the "tainted kernel" flags by the signed-module loading code has led to a problem using tracepoints in unsigned kernel modules. The problem is fairly easily fixed, but there was opposition to doing so, at least until a "valid" use case could be found. Kernel hackers are not particularly interested in helping out-of-tree modules, and fixing the problem was seen that way—at first, anyway. 

Loadable kernel modules have been part of the kernel landscape for nearly 20 years (kernel 1.2 in 1995), but have only recently gained the ability to be verified by a cryptographic signature, so that only "approved" modules can be loaded. Red Hat kernels have had the feature for some time, though it was implemented differently than what eventually [ended up in the kernel](<https://lwn.net/Articles/515007/>). Basically, the kernel builder can specify a key to be used to sign modules; the private key gets stored (or discarded after signing), while the public key is built into the kernel. 

There are several kernel configuration parameters that govern module signing: `CONFIG_MODULE_SIG` controls whether the code to do signature checking is enabled at all, while `CONFIG_MODULE_SIG_FORCE` determines whether _all_ modules must be signed. If `CONFIG_MODULE_SIG_FORCE` is not turned on (and the corresponding kernel boot parameter `module.sig_enforce` is not present), then modules without signatures or those using keys not available to the kernel will still be loaded. In that case, though, the kernel will be marked as tainted. 

The taint flag used, though, is the same as that used when the user does a `modprobe --force` to force the loading of a module built for a different kernel version: `TAINT_FORCED_MODULE`. Force-loading a module is fairly dangerous and can lead to kernel crashes due to incompatibilities between the module's view of memory layout and the kernel's. That can lead to crashes that the kernel developers are not interested in spending time on. So, force-loading a module taints the kernel to allow those bug reports to be quickly skipped over. 

But loading an unsigned module is not likely to lead to a kernel crash (or at least, not because it is unsigned), so using the `TAINT_FORCED_MODULE` flag in that case is not particularly fair. The tracepoint code discriminates against force-loaded modules because enabling tracepoints in mismatched modules could easily lead to a system crash. The tracepoint code does allow `TAINT_CRAP` (modules built from the staging tree) and `TAINT_OOT_MODULE` (out-of-tree modules) specifically, but tracepoints in the modules get silently disabled if there is any other taint flag. 

Mathieu Desnoyers [posted an RFC patch](<https://lwn.net/Articles/588804/>) to change that situation. It added a new `TAINT_UNSIGNED_MODULE` flag that got set when those modules were loaded. It also changed the test in the tracepoint code to allow tracing for the new taint type. It drew an [immediate NAK](<https://lwn.net/Articles/588905/>) from Ingo Molnar, who did not find Desnoyers's use case at all compelling: ""External modules should strive to get out of the 'crap' and 'felony law breaker' categories and we should not make it easier for them to linger in a broken state."" 

But the situation is not as simple as Molnar seems to think. There are distribution kernels that turn on signature checking, but allow users to decide whether to require signatures by using `module.sig_enforce`. Since it is the distribution's key that is stored in the kernel image, strict enforcement would mean that only modules built by the distribution could be loaded. That leaves out a wide variety of modules that a user might want to load: modules under development, back-ported modules from later kernels, existing modules being debugged, and so on. 

Module maintainer Rusty Russell was [fairly unimpressed](<https://lwn.net/Articles/588911/>) with the arguments given in favor of the change, at least at first, noting that the kernel configuration made it clear that users needed to arrange to sign their own modules: ""Then you didn't do that. You broke it, you get to keep both pieces."" That's not exactly the case, though, since `CONFIG_MODULE_SIG_FORCE` was _not_ set (nor was `module.sig_enforce` passed on the kernel command line), so users aren't actually required to arrange for signing. But Russell was looking for ""an actual valid use case"". 

The problem essentially boils down to the fact that the kernel is lying when it uses `TAINT_FORCED_MODULE` for a module that, in fact, hasn't been forced. Steven Rostedt [tried to make that clear](<https://lwn.net/Articles/588923/>): ""Why the hell are we setting a FORCED_MODULE flag when no module was forced????"". He also [noted](<https://lwn.net/Articles/588924/>) that he is often the one to get bug reports from folks whose tracepoints aren't showing up because they didn't sign their module. As he pointed out, it is a silent failure and the linkage between signed modules and tracepoints is not particularly obvious. 

Johannes Berg was eventually [able to supply](<https://lwn.net/Articles/588926/>) the kind of use case Russell was looking for, though. In his message, he summarized the case for unsigned modules nicely: 

The mere existence of a configuration to allow unsigned modules would indicate that there are valid use cases for that (and rebuilding a module or such for development would seem to be one of them), so why would tracing be impacted, particularly for development. 

Berg also provided another reason for loading unsigned modules: backported kernel modules from [the kernel.org wiki](<https://backports.wiki.kernel.org/>) to support hardware (presumably, in his case, wireless network hardware) features not present in the distribution-supplied drivers. He was quite unhappy to hear those kinds of drivers, which ""typically only diverge from upstream by a few patches"", characterized as crap or law-breaking. 

Berg's use case was enough for Russell to agree to the change and to add it to his pending tree. We should see Desnoyers's [final patch](<https://lwn.net/Articles/588803/>), which has some cosmetic changes from the RFC, in 3.15. At that point, the kernel will be able to distinguish between these two different kinds of taint and users will be able to trace modules they have loaded, signed or unsigned.
