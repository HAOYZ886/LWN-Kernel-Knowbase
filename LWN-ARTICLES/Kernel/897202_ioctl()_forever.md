---
title: "ioctl() forever?"
url: https://lwn.net/Articles/897202/
date: "June 8, 2022"
category: ioctl
author: "By Jake Edge June 8, 2022 LSFMM"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
June 8, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

In a combined storage and filesystem session at the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM), Luis Chamberlain and James Bottomley led a discussion about the use of [`ioctl()`](<https://man7.org/linux/man-pages/man2/ioctl.2.html>) as a mechanism for configuration. There are plenty of downsides to the use of `ioctl()` commands, and alternatives exist, but in general kernel developers have chosen to continue using this multiplexing system call. While there is interest in changing things, at least in some quarters, the discussion did not seem to indicate major changes on the horizon. 

#### Problems

Chamberlain began with a history lesson with "some rants" thrown in. `ioctl()` is still used a lot by filesystems and the block layer in Linux, but the wireless-networking subsystem, which he used to work on, successfully shifted away from `ioctl()`. The system call "wasn't really originally designed for what we think it was", he said; it is "essentially a hack". In [Douglas McIlroy's history of Unix](<https://www.cs.dartmouth.edu/~doug/reader.pdf>), it was called "a closet full of skeletons" that was mainly used to prevent the addition of too many new system calls. Those things should be kept in mind when thinking about `ioctl()`, he said. 

[ ![\[James Bottomley\]](https://static.lwn.net/images/2022/lsfmm-bottomley-sm.png) ](<https://lwn.net/Articles/897262/>)

The first version of Linux did not have `ioctl()`; it was added in Linux-0.96a in May 1992\. A small patch in 1993 changed a type from `unsigned int` to `unsigned long`, which eventually led to compatibility headaches for 32-bit `ioctl()` calls issued on 64-bit systems. The Unix idea that everything is a file is useful because it allows for flexibility, but it also allows for lazy API design, he said. Beyond that, `ioctl()` commands are not well documented and the interface does not allow for introspection 

Lack of introspection abilities is not a problem that Chamberlain has encountered directly, so he asked Bottomley to elaborate on that. The problem crops up in the container world, Bottomley said, and it is not just for `ioctl()` commands but also introspection for system calls. For example, securing a Docker container by limiting the system calls it can make does not really secure anything if there are opaque `ioctl()` commands that can be used to circumvent the restrictions. So there is a lot of concern about non-introspective interfaces because they "can't be policed properly by the tools we usually use for containers, like seccomp and even eBPF". 

The specific problem with `ioctl()` is that there is a "dense binary packet" that gets passed into the call, which makes it difficult for external tools to deduce what the packet contains. In theory, the kernel could switch to using XML or JSON, but that does not really change the underlying problem much, he said. The introspection problem remains "almost regardless of which interface we choose". 

There are other problems with `ioctl()`, Chamberlain said. For example, he asked Arnd Bergmann about `ioctl()` support for different architectures. He got back an itemized list of caveats. "The world is not peachy for architecture support as well". 

#### Greener grass

The Linux wireless-networking configuration underwent a shift from the `ioctl()`-based wireless extensions to the netlink-based nl80211 interface. Chamberlain invited attendees to compare [`include/uapi/linux/wireless.h`](<https://elixir.bootlin.com/linux/v5.18.2/source/include/uapi/linux/wireless.h>) with [`include/uapi/linux/nl80211.h`](<https://elixir.bootlin.com/linux/v5.18.2/source/include/uapi/linux/nl80211.h>) to see how much cleaner the new interface is. The netlink interface is not designed to be generic, so it may not be the right choice for filesystems and the block layer. But he is sure that that it is possible to find something better than the `ioctl()`-based interface we have now. 

Chamberlain handed the microphone back to Bottomley so that he could talk about [configfd](<https://lwn.net/Articles/809125/>) as a possibility. But Bottomley said that he was not going to promote configfd, though he did describe it a bit. It came out of his efforts on the [shiftfs filesystem](<https://lwn.net/Articles/718639/>), which was eventually supplanted by [ID-mapped mounts](<https://lwn.net/Articles/896255/>). Configfd was based on the `fsconfig()` system call, which allows setting a bunch of configuration information on a filesystem atomically, but configfd was bidirectional. David Howells, who developed `fsconfig()` and the related [new mounting API](<https://lwn.net/Articles/759499/>), interjected that `fsconfig()` was originally bidirectional as well, though Al Viro removed that piece before it was merged. 

Instead of defending configfd, Bottomley said, he wanted to talk about the necessity of `ioctl()`. When there is a need for "an exception to the normal semantic order of things", an `ioctl()` command can provide it. And there will always be a need for exceptions, no matter how tightly regulated that semantic order is. There will always be a requirement that two parties be able to communicate data that cannot be structured using the existing mechanisms—an exception. Whether that data is sent as JSON, XML, or binary data, it is, effectively, an `ioctl()`. 

The introspection problem is real, but is one that he thinks could be handled with documentation. Christian Brauner said that the problem goes beyond just `ioctl()`; there are a number of different problems with [`seccomp()`](<https://www.man7.org/linux/man-pages/man2/seccomp.2.html>) filtering because of the need to [inspect the system call arguments](<https://lwn.net/Articles/822256/>) to help make filtering decisions. Pointer arguments have been discouraged for new system calls because `seccomp()` cannot follow the pointers. But using pointers to structures is a technique for [creating extensible system calls](<https://lwn.net/Articles/830666/>), so `seccomp()` also needs to change. It is a problem "slightly to the side" of the `ioctl()` problem, but it needs to be solved as well, he said. 

Bottomley said that this shows that even if it were decreed that `ioctl()` commands should all move to new system calls, the problem with introspection would just move with it. Ted Ts'o said that kernel developers rightly keep a tight grip on new system calls and their interfaces. So adding a new system call involves an enormous bikeshedding exercise with lots of additional requirements, including documentation and working with features like `seccomp()` filtering. Often, the feature developer does not care about the container use case, even if they should, so they move it to an `ioctl()` command "so they can dodge the bikeshedding". 

The more perfect the kernel community tries to make the system call interface, the more incentive there is for developers to route around it, Ts'o said. He has heard people talk about adding a feature via a filesystem-specific `ioctl()` command as a way to avoid the "fsdevel bikeshed party". That is unfortunate, since there is plenty of useful architectural review that might come with trying to make the feature more widely usable, but it is understandable that people take the expedient approach. No one has infinite resources, Ts'o said. 

#### Alternatives

Josef Bacik asked what the alternative is. "You're not going to pry `ioctl()`s from my cold dead hands unless you give me something else." The Btrfs developers have "wasted a lot of time" in grand architectural discussions that ended up with the community saying that a feature should just be put into an `ioctl()` command—after a year of discussion. Bottomley said that he would argue `ioctl()` commands, used judiciously, are just fine. 

Kent Overstreet said that `ioctl()` commands are simply a driver-specific system call; there is a real need for that. It provides a mechanism to try out a feature in a more private way before it gets promoted to a system call, where it becomes permanent. Amir Goldstein agreed, noting that the "chattr" `ioctl()` command was implemented by two different filesystems before it was determined to be a generally useful feature and moved into the virtual filesystem (VFS) layer. 

There are multiple existing mechanisms for configuration in the kernel, Ts'o said. `ioctl()` commands are just system calls in disguise, both of which provide ways to do configuration, but procfs and sysfs files can also be used for that. Beyond those, the new mount API or configfd provide other configuration mechanisms. But which gets used depends in part on how much pain there is in trying to change the mechanism for a new task, he said. If the pain of adding `ioctl()` commands rises to the same level as for system calls, developers will simply find a "different escape hatch". 

But Chamberlain said that adding new wireless commands did not require additional system calls or `ioctl()` commands because it uses netlink. Those changes can be made in a domain-specific place without all of the problems that come from the other mechanisms. Brauner said that he had a hard time seeing what could replace the `ioctl()` interface, however; he wondered if Chamberlain was suggesting switching to something netlink-based. Chamberlain said that it was just one idea, but Howells noted that netlink could not be used because it depends on networking being configured into the kernel, which is not always the case. 

There was some discussion of alternatives, but it is clear that `ioctl()` itself is not going away and that it fills a need. Finding ways to make the `ioctl()` arguments more introspectable would be useful, as would better documentation. But if requiring those things causes the friction level for adding new commands to rise too much, it will have the opposite of its intended effect. No real solution seemed to be forthcoming from the discussion, though no one seems entirely satisfied with the status quo either.
