---
title: "LSS: The kernel hardening roundtable"
url: https://lwn.net/Articles/458805/
date: "September 15, 2011"
category: "Security-Kernel hardening"
author: "By Jake Edge September 15, 2011"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
September 15, 2011

Hardening the kernel to make attackers' jobs harder was the topic of a wide-ranging discussion at the Linux Security Summit (LSS) held on September 8, 2011. Reducing the attack surface of the kernel, protecting it from user-space attacks, and finding ways to mitigate entire classes of exploitable bugs were all on the table. As might be expected, the biggest barrier to getting these hardening patches accepted into the mainline is often performance concerns. While no firm conclusions were drawn, many ideas were discussed, some of which may eventually find their way into the mainline. 

#### Attack surface

The discussion began with an effort to quantify the ""exposed surface"" of the kernel as roundtable leader Will Drewry of Google's Chrome OS team put it. He and the other roundtable leader, Kees Cook of the Ubuntu security team, put together their own list, but also asked those present to add to it. Obvious attack surfaces like the system call interface, `/proc` and sysfs files, the networking stack, and device drivers were mentioned, but also less obvious things like filesystem parsing, auto-loaded kernel modules, device scanning, CPU or other hardware bugs, side-channel timing attacks, and so on. 

Enumerating the attack surface ""helps define what to pay attention to"", Cook said. The intent of many of the kernel hardening patches is to try ""to kill off a whole class of problems, rather than shooting individual bugs"", he said. The latter is where most of the current kernel security effort goes, he said. Drewry added that the intent is to figure out what can be done now to reduce those attack surfaces. Many of the attack surfaces are still present even in a system that runs a mandatory access control (MAC) system like SELinux, Cook said, because the system call interface is still available to be used (and abused). That is one of the problems with looking to the LSM interface to provide confinement, he added. 

Casey Schaufler also pointed out that there is often special-purpose hardware in Linux systems--in years past it was graphics hardware, but these days tends to be video hardware--that is allowed to be directly accessed from user space. That opens up a number of potential security problems, he said, but that won't stop it from happening. The capabilities provided by allowing direct access to these devices are ""so compelling that security concerns are secondary"". 

But there are kernel installations that are more security-sensitive, Cook said, that could benefit from restricting some features even at the cost of performance. If a particular hardening feature has no real cost, it could be put into the kernel without providing a configuration option to disable it. Others, that do have a cost, could be optional and distributions or users could enable them based on their needs. 

#### API/ABI restrictions

The ""biggest single exposure"" in Linux systems is applications that run as root, Schaufler said, like the X server. Because the kernel is one ""gigantic privileged application"" it can't be protected against other privileged applications like X, Cook said. But, applications could have the ABI available to them reduced, Drewry said, which would reduce the damage they could do if they are compromised. 

The only existing ""API management"" tool in the kernel (besides the LSM interface) is [seccomp](<https://lwn.net/Articles/332974/>), but it is too restrictive to be useful for many applications, Drewry said. Since seccomp only allows four system calls (`read()`, `write()`, `exit()`, and `sigreturn()`), it is too limited for many possible reduced-ABI applications. The Chrome/Chromium browser team would like to be able to reduce the system calls that its rendering processes can make. Seccomp is too limited for Chromium's needs, so they have implemented a more complicated [solution](<https://lwn.net/Articles/347547/>), with a "trusted" assembly language thread that mediates system calls. System call restrictions could also be enforced using `ptrace()`, Drewry said, but there is an ""intense amount of overhead"". 

What Drewry is looking for is some kind of [expanded seccomp](<https://lwn.net/Articles/441232/>) where a subset of system calls would be allowed. So far, his patches to implement that have been shot down from various directions, but there is hope that there may be some kind of resolution at the upcoming Kernel Summit. 

Some of the attendees were skeptical of an expanded seccomp approach. Schaufler pointed out that there is already a mechanism in the kernel (capabilities) for reducing the impact of vulnerabilities, but ""no one uses it"". Cook was not convinced that the granularity of capabilities was really all that useful because the number of [capability bits that are equivalent to root](<https://lwn.net/Articles/421671/>) is so large. 

As Drewry cast about for a way to limit system calls, there was discussion of possibly augmenting the LSM interface. As Cook pointed out, the current interface does not mediate all system calls, so it can't be used for Drewry's use case as it stands. James Morris noted that LSM is intended to be an access control framework and not anything more than that. In the end, Drewry doesn't particularly care how to get there, he is just looking for a way for ""reducing what I expose to untrusted applications"", he said. 

Schaufler also pointed out that reducing the ABI available to an application doesn't help ""if the ABI is completely well-defined and if it is consistent with the security policy"" of the system. ""That's a lot of 'if's"", Drewry responded, to general agreement, that neither of the two conditions are met on Linux systems. Because the system call interface is not well-defined, nor necessarily consistent with the system security policy, reducing the exposure of parts of that interface can help. Schaufler cautioned that the _ad hoc_ documentation makes it hard to decide where the bugs actually are: ""If the code is the documentation, it is impossible to have a bug"". 

There were questions about whether seccomp filtering (in whatever form) would actually be used by applications. Cook noted that, in addition to Chromium, several other projects popped up on linux-kernel to express interest in the feature, including QEMU, vsftpd, and others. One attendee also hypothesized a DNS server that was limited to `recvmsg()`, `sendmsg()`, and `write()` (to a log file) as another possible use-case. 

There were also concerns that seccomp filters would spread security policy throughout the system, but others saw that as a feature. Unlike MAC policy, which tends to be imposed from the outside, seccomp filter policy would embody ""the programmer's idea of what it should be able to do"", as Cook put it. While the system call granularity may not be exactly right, it is the place where user space enters the kernel, so mediating at that point makes some sense. 

Attendees theorized that if a flexible seccomp filter facility was available, multiple applications would take advantage of it. Smalley was a bit skeptical that it would be straightforward for most applications to use the facility because it might require a major rework of the program. He pointed to the privilege separation efforts that went on in OpenSSH as an example. That required ""significant refactoring"", he said. 

Drewry said that the Chromium team's plan is to move the browser to whatever solution becomes available to better contain the renderers. Right now, that is the "trusted thread" sandbox, but if there are other facilities available, Chromium will use them. That could be some kind of SELinux containment, seccomp filtering, or something else entirely. In the future, the team would also like to confine renderers based on where the data comes from, he said, so that all renderers running for a given site were protected from each other as well. 

#### PaX and grsecurity

The roundtable wrapped up with some discussion of bringing more of the [grsecurity and PaX](<http://grsecurity.net/>) hardening patches into the mainline. Those patches tend to be fairly intrusive and have performance implications that make them undesirable to many kernel hackers, but they do provide protections that some would find valuable. According to Cook, there are many pieces of grsecurity and PaX that could make their way into the mainline. 

Simple things, like `const`ifying function pointers, are essentially free and should be mainlined immediately: ""It's a shame that hasn't been done long ago"", one attendee said. Others that have more impact are trickier. Making them optional is one possibility, but even that has a cost that maintainers are likely to push back against. Adding another path through core kernel code can be a maintenance headache, and those may be difficult to get into the mainline. 

Andre Hedrick mentioned that he has been pulling apart the grsecurity/PaX patches to try to make them more palatable. For one thing, grsecurity depends on a role-based access control (RBAC) mechanism that isn't present in the mainline (and isn't implemented as an LSM, so it isn't likely to ever be, at least in that form). Hedrick is trying to remove that dependency from the grsecurity features of interest, like better address-space layout randomization (ASLR) and a fully relocatable kernel, both of which can thwart various kinds of attacks. 

One goal would be to find the grsecurity/PaX changes that have minimal impact and to get those into the mainline as non-optional protections. Turning RBAC into an LSM might be another useful exercise. grsecurity developer Brad Spengler provided a ""long list"" of features that could make their way into the kernel at last year's LSS, Cook said. That list would make a good starting point. 

Cook also noted several other efforts aimed at hardening the kernel. Those include the [work](<https://lwn.net/Articles/451405/>) that Openwall hacker Vasiliy Kulikov has been doing, much of which is being discussed on the [kernel-hardening mailing list](<http://www.openwall.com/lists/kernel-hardening/>). Also, the Ubuntu security team has been working on [a kernel hardening project](<https://wiki.ubuntu.com/SecurityTeam/Roadmap/KernelHardening>) of its own. There is no lack of ideas out there, and a clear need to make the kernel more resistant to attacks. Based on the discussion, and the various ongoing efforts, we are likely to see more and more hardening patches aimed at the mainline over the next few years. 

[ I'd like to thank LWN subscribers for supporting my travel to LSS. ]
