---
title: The LoadPin security module
url: https://lwn.net/Articles/682302/
date: "April 6, 2016"
category: "Modules-Security modules; Security-Security modules"
author: "By Jonathan Corbet April 6, 2016"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
April 6, 2016

The merging of [security-module stacking](<https://lwn.net/Articles/635771/>) was supposed to jump-start the creation of a whole set of small, special-purpose security modules. By making it possible for security modules to play well together, stacking enables modules that solve one little problem rather than implementing an entire security policy. Thus far, the flood of new modules has yet to begin, but the [LoadPin module](<https://lwn.net/Articles/681613/>) from Kees Cook demonstrates the sort of problem that little security modules might be written to solve. 

Security-conscious developers have long worried about code that is loaded into the kernel at run time; such code could clearly, if malicious, subvert the security of the entire system. Mechanisms like [the signing of kernel modules](<https://lwn.net/Articles/525592/>) are designed to prevent the loading of such code. In the end, though, a signature on a loadable module really only guarantees one thing: where that module came from. If the provenance of kernel modules (or other files loaded into the kernel) could be ascertained by other means, it would not be necessary to sign them. 

ChromeOS keeps its security-sensitive files, including kernel modules, in read-only storage; that storage is verified as a whole before the system boots. In such a setting, the loading of modules from the read-only partition is safe regardless of whether they have been signed; they are known to come from Google and cannot have been modified since. So a reasonable security policy for ChromeOS systems might well be to load modules (without signature checking) from the secure partition, while rejecting the loading of modules from anywhere else. 

That is the policy that LoadPin was created to implement. It takes advantage of the relatively new [kernel file-loading mechanism](<https://lwn.net/Articles/676101/>) to intercept all attempts to load a file into the kernel; these include loading kernel modules, reading firmware, loading a security policy, or loading an image for `kexec()`. In each case, the filesystem containing the file to be loaded is compared against the filesystem used in the first load operation after boot; if the two don't match, the attempt is rejected. All that is required is to build the security module into the kernel; there is no other configuration needed or possible. 

The resulting mechanism is, obviously, quite simple in nature. Administrators who want to enable file loading from multiple filesystems, or who want to change the allowed filesystem during the lifetime of the system will be out of luck. But, for the simple case where the system boots from a trusted, read-only image, LoadPin does all that is needed. 

In truth, the interface isn't quite that simple. If (and only if) the first load operation is satisfied by a writable filesystem, the LoadPin module will log a warning and establish a sysctl knob (`kernel/load_pinning`) that can be used to turn the feature on or off. This, of course, is a debugging mechanism; normally one would not want to be able to write to the filesystem containing the trusted modules. After all, the immutability of the trusted filesystem is the foundation on which the trust in its contents is being built. On a production system, the ability to disable pinning would also be undesirable. 

It would not be surprising if other developers eventually showed up wanting to add more complex configuration options to this module. One could imagine wanting to apply different policies to different types of files (firmware or `kexec()` images), for example. The current module is also likely to run into trouble on systems that boot with an initramfs image; the first modules will almost certainly be loaded from that image (that's why it exists, usually), causing loads to be pinned to a temporary filesystem that will go away at the end of the bootstrap process. In the current patch, if the filesystem to which loading is pinned disappears, loading of files will be disabled entirely — behavior that makes sense, but which may not lead to the desired results in an initramfs setting. 

But that is work for later, should somebody decide that it is needed. For now, the simple security module would appear to be enough for Google's needs. Indeed, the copyright dates in the code suggest that it has been in use there since 2011. 

As Casey Schaufler suggested in [his talk](<https://lwn.net/Articles/674949/>) on writing one's own security module, the stacking mechanism makes it possible to implement a wide variety of possible policies. LoadPin, perhaps, will be the start of a series of modules from developers who have extended the kernel's security mechanisms in novel ways. The potential is there for some interesting ideas to be contributed back to the kernel. Given that relatively few people seem to think that we have solved all of the security problems with what we have now, that seems like it should be a step in the right direction.
