---
title: "Disabling SELinux's runtime disable"
url: https://lwn.net/Articles/927463/
date: "April 20, 2023"
category: "Releases-6.4; SELinux; Security-Security modules"
author: "By Jonathan Corbet April 20, 2023"
---

By **Jonathan Corbet**  
April 20, 2023

Distributors have been enabling the SELinux security module for nearly 20 years now, and many administrators have been disabling it on their systems for almost as long. There are a few ways in which SELinux can be disabled on any given system, including command-line options, a run-time switch, or simply not loading a policy after boot. One of those ways, however, is about to be disabled itself. 

SELinux undoubtedly improves the security of a system; it can confine processes to the resources that they are intended to use. But SELinux can also get in the way, especially in situations where some program does not behave in the way that the policy authors expected. The tools for figuring out where a problem lies and amending SELinux policies have improved over the years but, for many, convincing SELinux to let some task proceed is simply not worth the trouble. These are the people who end up just turning it off altogether. 

The kernel provides a set of options for doing that, beyond building a kernel that does not include SELinux at all. The `selinux=0` command-line parameter will disable SELinux at boot. Another option is editing `/etc/selinux/config`, which can have the effect of preventing an SELinux policy from being loaded into the kernel. Without a policy, SELinux deems itself to be in an uninitialized state and will not enforce any restrictions. Finally, writing a zero to `/sys/fs/selinux/disable` will disable SELinux until the next boot, but only if no policy has yet been loaded. 

That last option, however, has been targeted for removal for some time. It was [deprecated](<https://git.kernel.org/linus/89b223bfb8a8>) for the 5.6 release in 2020. The 5.19 kernel saw [the addition of a five-second delay](<https://git.kernel.org/linus/43b666622c60>) whenever this option was used to disable SELinux; that delay was [increased to 15 seconds](<https://git.kernel.org/linus/e0d8259355cb>) in 6.2 for the benefit of anybody who hadn't gotten the hint so far. Now, [a patch disabling `/sys/fs/selinux/disable` entirely](<https://lwn.net/ml/linux-security-module/20230317195615.281810-1-paul@paul-moore.com/>) from Paul Moore has landed in linux-next and will almost certainly go upstream during the 6.4 merge window. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

One might well wonder why there is so much hostility toward a simple run-time system-configuration option. For developers who are working on the creation of highly secure systems, any sort of an "off" switch is a potential failure point. A system may be locked down with various security policies but, if an attacker can somehow get a zero byte written to `/sys/fs/selinux/disable` during the boot sequence, the system will run without SELinux enforcement and much of that work will have been for naught. Taking that option away adds one more obstacle to somebody who is trying to circumvent a system's security. 

Arguably, though, the more important concern is that, to support the ability to disable (or enable) SELinux at run time, the kernel must be able to write to the structures containing the hook functions used to call into the various security modules. Kernel developers have been working for years to eliminate this kind of writable function vector; each one of them is a tempting target for an attacker. In the case of security modules, protecting those vectors is doubly important; the security policy cannot be enforced without them. The security-module hooks are called from many of the most sensitive places in the kernel; if they can be changed, the result could be anything from the circumvention of the rules for exported kernel symbols to a complete compromise of the system. 

To avoid this kind of problem, security-oriented developers would like to store these hooks in [post-init read-only memory](<https://lwn.net/Articles/666550/>). Data that is marked with the special `__ro_after_init` attribute is writable when the system boots, but is changed to read-only at the end of the bootstrap process, before user space is allowed to run. This mechanism allows the kernel to initialize things — such as the active security module(s) — then to lock down the relevant data so that the configuration cannot be (easily) changed. 

Moore's patch removes the ability to disable SELinux at run time. The `/sys/fs/selinux/disable` file will continue to exist and accept writes, but the only effect it will have will be to generate a log message if an attempt to use it to disable SELinux is made. The hook vectors for all in-tree security modules are marked `__ro_after_init`, ending the ability to make changes after the system has booted. 

In a real sense, this is an API-breaking change. Any system that is using this feature, and which is counting on SELinux being disabled afterward, will not function properly with a 6.4 kernel. Chances are, though, that there will be few affected systems. Distributions that enable SELinux at boot, such as Fedora, [disabled this feature](<https://lwn.net/Articles/831748/>) in their kernel configurations years ago, so affected users should already have noticed the problem. That, Moore says, has not happened: 

> Finally, in the several years where we have been working on deprecating this functionality, there has only been one instance of someone mentioning any user visible breakage. In this particular case it was an individual's kernel test system, and the workaround documented in the deprecation notice ("selinux=0" on the kernel command line) resolved the issue without problem. 

For anybody out there who needs to turn off SELinux on a system where it is enabled by default at boot, the other two options remain open. The best solution is, as mentioned above, to put `selinux=0` on the kernel command line. It remains possible to edit `/etc/selinux/config` but, as Moore notes, doing so does not truly disable SELinux; it just prevents the loading of a policy, meaning that SELinux could be enabled later on by loading a policy. 

The result of this change is, hopefully, an inherently more secure kernel and a minimum of disruption for users who need to run without SELinux enabled for whatever reason. Such changes can be hard to make but, as this case shows, they can be possible with enough patience and a willingness to work at both the kernel and distribution levels.
