---
title: Another LSM stacking approach
url: https://lwn.net/Articles/518345/
date: "October 3, 2012"
category: "Security-Security modules"
author: "By Jake Edge October 3, 2012"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
October 3, 2012

Anyone who follows Linux kernel security discussions has probably heard of the "LSM stacking issue". It is a perennial topic on the mailing lists and solutions have been proposed from time to time. The basic problem is that only one Linux Security Module (LSM) can be active in a running kernel, and that single slot is often occupied by one of the "monolithic" solutions (e.g. SELinux or AppArmor) supplied by distributions. That leaves some of the smaller or more special-purpose LSMs--or users who want to use multiple approaches--out in the cold. 

Back in February 2011, David Howells [proposed a stacking solution](<https://lwn.net/Articles/426921/>) for LSMs. At the time, Casey Schaufler mentioned a solution he had been working on that would be posted in a "day or two". That prediction turns out to have been overly optimistic, but his [solution](<https://lwn.net/Articles/516650/>) has surfaced--more than a year-and-a-half later. He also discussed the patches in a lightning talk at the recently held Linux Security Summit. 

There are three types of LSMs available in the kernel today and there are use cases for combining them in various ways. Administrators might want to add some AppArmor restrictions on top of the distribution-supplied SELinux configuration--or use SELinux-based sandboxes on a TOMOYO system. The two "labeled" LSMs, SELinux and Smack, require that files have extended attributes (xattrs) containing labels that are used for access decisions. The two "path-based" LSMs, AppArmor and TOMOYO, both base their access decisions on the paths used to access files in the system. The only other LSM currently available is Yama, which is something of a container for discretionary access control (DAC) enhancements. 

Yama is the LSM that is perhaps most likely to be stacked. It adds some restrictions to the `ptrace()` attach operation that Ubuntu and ChromeOS use, and other distributions are considering it as well. In fact, Yama developer Kees Cook has [proposed](<https://lwn.net/Articles/514975/>) making the LSM unconditionally stackable via the `CONFIG_SECURITY_YAMA_STACKED` kernel build option (which was merged for 3.7). Over the years, though, various other security ideas have been proposed and pointed in the direction of the LSM API, so other targeted LSMs may come about down the road. Making each separately stackable is less than ideal, so a more general solution is desirable. In addition, combining labeled and path-based solutions manually can't really be sanely done. 

When Howells posted his solution, he explicitly disallowed combining the two labeled LSMs because of implementation difficulties (mainly with respect to the LSM-specific `secid` which is used by SELinux and Smack, but none of the others). There was also a belief that mixing SELinux and Smack (or AppArmor and TOMOYO for that matter) is not a particularly sought-after feature. But Schaufler thought that was an unnecessary restriction, one that he was trying to address in his solution. 

As it turns out, Schaufler ended up at the same place. His proposal also defers stacking (or "composing") SELinux and Smack, noting that it ""has proven quite a challenge"". But he was able to get the other combinations working--at least to the extent that the kernel would boot without complaints in the logs. The Smack tests passed as well. Performance for Smack with AppArmor, TOMOYO, and Yama enabled is ""within the noise"", he said. 

Schaufler's version ensures that the hooks for _each_ enabled LSM are called, which is different than Howells's approach that short-circuited the other hooks if one denied the access. Instead, Schaufler patches call each LSM's hooks, remembering the last non-zero return (denial or error of some sort) as the return value for the hook. His argument is that an LSM could reasonably expect to see--and possibly record information about--each access decision, even if it has been denied by another LSM. 

Much of the "guts" of the changes are described in the [infrastructure patch](<https://lwn.net/Articles/518450/>), which is the largest of the five patches. The others make fairly modest (if pervasive) changes to SELinux, Smack, TOMOYO, and AppArmor to support stacking. As it turns out, Yama ""required no change and gets in free"". The changes to the individual LSMs are optional, as they can still be used (in a non-stackable way) without them. 

Stacking is governed by the `CONFIG_SECURITY_COMPOSER` option. If that is not chosen, all of the existing LSMs function as they do today. If stacking is built in, the `security=` boot parameter can then be used to control which LSMs are enabled. For example, `security=selinux,apparmor` will enable those two. If nothing is specified on the boot command line, all of the LSMs built into the kernel will be enabled. The `/proc/PID/attr/current` interface has also been changed to report information from any of the active LSMs (only SELinux, Smack, and AppArmor actually use that interface today). 

Existing kernels store pointers to the hooks implemented by an LSM in a `struct security_operations` called `security_ops`. Schaufler's patch replaces that with an array of `security_operations` pointers called `composer_ops`. That array is indexed based on the `order` that is assigned to each LSM as it is registered. The first entry (`composer_ops[0]`) is reserved for the Linux capabilities hooks. Those have been manually "stacked" into the LSMs for some time, so entries in `composer_ops[0]` get zeroed out if one of the other LSMs implements the hook (as the capabilities checks will be done there). If there is no entry in `composer_ops[0]`, each of the hooks in the other entries in that array are called, as described above. 

The security "blobs" (private storage for each LSM) are still managed by the LSMs, but because there are blob pointers sprinkled around various kernel data structures (e.g. inodes, files, sockets, keys, etc.), a "composer blob" is used. That blob contains pointers to each of the active LSM blobs, and new calls are used to get and set the blob pointers (e.g. `lsm_get_inode()` or `lsm_set_sock()`). Most of the changes for the individual LSMs are converting to use this new interface. 

So far, most of the comments have been about implementation details; Schaufler addressed those in the second version of the patch set. Notably missing, at least so far, were some of the concerns about strange interactions between stacked LSMs leading to vulnerabilities that have come up in earlier discussions. But, without any major complaints, one would guess some more testing will be done, including gathering some additional performance numbers, before the linux-kernel gauntlet will be run. The rest of the kernel developers have heard about the need for stacking LSMs enough times that it seems likely that Schaufler's patches (or something derived from them) will eventually pass muster.
