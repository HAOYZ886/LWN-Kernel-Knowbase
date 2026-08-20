---
title: "CAP_SYS_ADMIN: the new root"
url: https://lwn.net/Articles/486306/
date: "March 14, 2012"
category: Capabilities
author: "March 14, 2012 This article was contributed by Michael Kerrisk ."
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

March 14, 2012

This article was contributed by [Michael Kerrisk](<http://man7.org/>).

Capabilities are--at least in theory--a nice idea: divide the privileges of root (user ID 0) into small pieces so that a process can be granted just enough power to perform specific privileged tasks. If the pieces are small enough, and well chosen, then, even if a privileged program is compromised (e.g., by a buffer overrun), the damage that can be done is limited by the set of capabilities that are available to the process. Good examples of the use of such fine-grained privileges are `CAP_KILL`, which permits sending signals to arbitrary processes, and `CAP_SYS_TIME`, which permits setting the system clock.

As of Linux 3.2, there are 36 capabilities. You can see a list of them, along with some of the main powers they each grant, in the [_capabilities(7)_](<http://man7.org/linux/man-pages/man7/capabilities.7.html>) manual page. Capabilities can (since Linux 2.6.24) be attached to an executable file, to create the capabilities equivalent of a set-user-ID-root program: when the executable is run, the resulting process starts with a limited set of capabilities (instead of the full power of root, as is the case for set-user-ID-root programs).

The key point from the beginning of this article is _small pieces_ , and it's here that the Linux capabilities implementation has gone astray.

When a kernel developer adds a new feature that should require privilege, what capability should they use, or should they perhaps even create a new capability? Although parceling root privileges into small pieces is useful from a security perspective, we don't want too many pieces, since then the task of administering capabilities would become unwieldy. Thus, it usually makes sense to employ an appropriate existing capability to control access to a new privileged kernel feature.

And this is where the problem begins. First, there is--unsurprisingly, given the Linux development model--no central authority determining how capabilities should be assigned to privileged operations. Second, there is very little guidance on what capability to choose. (Probably the best existing guide is to look at the _capabilities(7)_ man page. By comparing with existing uses in that page, we can get some guidance on choosing the capability that best matches a new use case.)

So in practice, what happens? A kernel developer looks at the list of available capabilities in the kernel [`include/linux/capability.h`](<http://lxr.linux.no/#linux/include/linux/capability.h>) header file, and is likely left bewildered wondering which capability to choose. (It appears that the original intent was that this header file would be updated with comments for all of the usages of each capability, so as to give an overview of capability usage, but in practice those comments have been updated only sporadically.) But the developer does know one thing: their feature will likely be administered by system administrators, and, helpfully, there is a capability called `CAP_SYS_ADMIN`. So, lacking sufficient information for a decision, the developer chooses `CAP_SYS_ADMIN` for their new feature.

Which brings us to where we are today: of the 1167 uses of capabilities in C files in the Linux 3.2 source code, 451 of those uses are `CAP_SYS_ADMIN`. That's rather more than a third of all capability checks. We might wonder if `CAP_SYS_ADMIN` is overrepresented because of duplications of similar operations in the kernel `arch/` trees, or because `CAP_SYS_ADMIN` is commonly assigned as the capability governing administrative functions on device drivers. However, even after eliminating `drivers/` and architectures other than x86, `CAP_SYS_ADMIN` still accounts for 167--about 30%--of the 552 uses of capabilities. (Fuller details about usage of capabilities in current and earlier kernels can be found [here](<http://blog.man7.org/2012/03/kernel-capability-usage-statistics.html>).)

So, on the one hand, the powers granted by `CAP_SYS_ADMIN` are so numerous and wide ranging that, armed with that capability, there are several avenues of attack by which a rogue process could gain all of the other capabilities. (As has been [summarized by Brad Spengler](<http://forums.grsecurity.net/viewtopic.php?f=7&t=2522>), the ability to be leveraged for full root privileges is a weakness of many existing capabilities; `CAP_SYS_ADMIN` is just the most egregious example.) On the other hand, so many privileged operations require `CAP_SYS_ADMIN` that it is the capability most likely to be assigned to a privileged program.

To summarize: `CAP_SYS_ADMIN` has become the new root. If the goal of capabilities is to limit the power of privileged programs to be less than root, then once we give a program `CAP_SYS_ADMIN` the game is more or less over. That is the manifest problem revealed from the above analysis. However, if we look further, there is evidence of an additional problem, one that lies in the Linux development model.

As noted above, if we eliminate `drivers/` and architectures other than x86, `CAP_SYS_ADMIN` accounts for 30% of the uses of capabilities. However, when capabilities were first introduced in Linux 2.2, the corresponding figures were 23 of 147 uses (16%). This supports a hypothesis that when random kernel developers are faced with the question "What capability should I use to govern access to the privileged feature that I'm adding to the kernel?", the answer often goes "I'm not sure… maybe `CAP_SYS_ADMIN`?". In other words, the Linux kernel development model (where, for example, there is no overall coordination of the use of capabilities) appears not to scale well when multiple developers face questions of this sort. (In retrospect, it also seems clear that the choice of the name `CAP_SYS_ADMIN` was rather unfortunate. The name conveys no real information about what operations the capability should govern, and it's an easy choice that looks safe to kernel developers who are uncertain of what capability to use.)

What could be done to improve matters? There's no quick and easy way out of the existing situation, but there are some steps that could be taken:  

  * Avoid new kinds of uses of `CAP_SYS_ADMIN`. (As this article was being written, Linux 3.3-rc is adding 13 new uses of capabilities. Most of them are `CAP_SYS_ADMIN`, and at least some of them may be new kinds of uses of that capability. One such use has [been averted](<https://lwn.net/Articles/486334/>), however.)

  * Rename `CAP_SYS_ADMIN` to `CAP_AS_GOOD_AS_ROOT`. Well, maybe not. But such a change would help get the point across to kernel developers looking to choose a capability for their new feature.

  * Publish better guidelines on the use of capabilities. Past attempts to do this (the _capabilities(7)_ man page and comments in `include/linux/capability.h`) have only had limited success (the guidelines are incomplete, and haven't done much to alleviate the problem). However, some more explicit guidelines, coupled with some measurements of the kernel source (see next point), might achieve better results.

  * Regularly publish statistics on the use of capabilities in the kernel source and monitor new uses of capabilities in each kernel release (e.g., employ some scripting to look at capability-related changes in the _diff_ for the current _-rc_ release).

  * Existing uses of `CAP_SYS_ADMIN` could be divided out into other existing capabilities, and possibly some new capabilities. Those capabilities could then be assigned to privileged programs instead of `CAP_SYS_ADMIN`. (For application backward-compatibility, the kernel capability checks wouldn't _remove_ `CAP_SYS_ADMIN`, but rather would check for `CAP_SYS_ADMIN` _or_ its replacement. This would allow old binaries that have the `CAP_SYS_ADMIN` capability to continue to work, while new binaries would be assigned the replacement capability.) One or two steps in this direction have already been made, for example, with [the addition of the `CAP_SYSLOG` capability](<https://lwn.net/Articles/417489/>) in Linux 2.6.37. An obvious first point of focus would be non-generic uses of `CAP_SYS_ADMIN` in areas other than drivers and the file-system trees. Next points of focus could be generic uses of `CAP_SYS_ADMIN` in the `drivers/` and `fs/` trees.

  * Do a similar analysis of other heavily used capabilities, especially `CAP_NET_ADMIN`, to see whether splitting would be useful for those capabilities. (`CAP_NET_ADMIN` has 395 uses in Linux 3.2. However, all of those uses are restricted to code in the `drivers/net/` and `net/` subdirectories. If we remove `CAP_NET_ADMIN` from the discussion, then there are more uses of `CAP_SYS_ADMIN` in the kernel source than all of the remaining capabilities combined.)

As well as the above, of course the problem outlined by Brad Spengler that many capabilities can be leveraged to gain full root access remains to be addressed. (Ongoing work on namespaces will help improve this situation for some capabilities when used in conjunction with containers.)

In summary, capabilities go some way toward improving application security, but there's still further work needed before they can deliver on their early promise of being a mechanism for providing discrete, non-elevatable privileges to applications. Furthermore, as the example of the ever-widening scope of `CAP_SYS_ADMIN` shows, some questions requiring coordinated answers are currently not well addressed by the distributed Linux development model.

[Acknowledgment: Thanks to [Serge Hallyn](<http://s3hh.wordpress.com/>) for comments on an early draft of this article.]
