---
title: A capability set for user namespaces
url: https://lwn.net/Articles/978846/
date: "June 20, 2024"
category: "Capabilities; Namespaces-User namespaces"
author: "By Jonathan Corbet June 20, 2024"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
June 20, 2024

[User namespaces](<https://lwn.net/Articles/532593/>) in Linux create an environment in which all privileges are granted, but their effect is contained within the namespace; they have become an important tool for the implementation of containers. They have also become a significant source of worries for people who do not like the increased attack surface they create for the kernel. Various attempts have been made to restrict that attack surface over the years; the latest is [user namespace capabilities](<https://lwn.net/ml/all/20240609104355.442002-1-jcalmels@3xx0.net>), posted by Jonathan Calmels. 

The core idea behind user namespaces is that a user runs as root within them, while the namespace as a whole is still unprivileged in the system that hosts it. A root process within the namespace has access to many root-only operations that can be used to configure and run the environment within the namespace. By design, that access cannot harm the system outside of the namespace, but there is a catch: the root user within the namespace can make many system calls that would be unavailable to that user outside of the namespace. That exposes much more of the kernel API to unprivileged users, increasing the severity of any security-relevant bugs in that API. [A number of exploitable vulnerabilities](<https://lwn.net/ml/all/xv52m5xu5tgwpckkcvyjvefbvockmb7g7fvhlky5yjs2i2jhsp@dcuovgkys4eh>) have predictably emerged from that exposure. 

Fear of new vulnerabilities has caused some distributors to disable user namespaces entirely in the past. A [security-module hook](<https://lwn.net/Articles/903580/>) was added in 6.1 to control the ability to create user namespaces, despite objections from the user-namespace maintainer. Out-of-tree patches to control user namespaces also apparently exist. In a world where the kernel was bug-free, user namespaces would not be a problem; in the world we actually inhabit, they continue to worry security-oriented developers. 

#### Capabilities

While Linux appears to follow the traditional model where the root account has all privileges and non-root accounts have none, internally the implementation is rather more complicated. Privileges are represented by [capabilities](<https://man7.org/linux/man-pages/man7/capabilities.7.html>), a set of bits describing the various operations that a task is allowed to perform. For example, `CAP_CHOWN` allows a process to change the ownership of any file in the system, `CAP_BPF` gives access to the BPF virtual machine, and `CAP_SYS_ADMIN` covers a horrifyingly long list of privileged operations. 

In the traditional model, a process running as root has all capabilities available to it; in a Linux system, it is possible to restrict capabilities to a smaller set. Of course, the world is complex; rather than having one set of capabilities, a thread in Linux has five of them. As if that were not enough, those sets interact with three other sets that can be associated with executable files. The thread capability sets are: 

  * The **effective** set, which describes the capabilities that the thread can actually exercise at the moment. 
  * The **permitted** set, containing the capabilities that the thread is empowered to exercise. A thread can add a new capability to its effective set with the [`capset()`](<https://man7.org/linux/man-pages/man2/capset.2.html>) system call, but only if that capability exists in the permitted set. 
  * The **bounding** set, which contains the list of capabilities a thread can obtain by any means. If a capability is missing from the bounding set, the thread cannot obtain that capability even if it runs a privileged program that would otherwise enable that capability. 
  * The **ambient** set contains a set of capabilities that will be retained if the thread calls [`execve()`](<https://man7.org/linux/man-pages/man2/execve.2.html>) to run an unprivileged program. Capabilities are normally cleared by `execve()`; the ambient set allows a task to pass a subset of its capabilities through that call. 
  * The **inheritable** set defines the capabilities that can be passed through `execve()` to an executable file that has its own inheritable set. A capability must appear in _both_ sets to be permitted after `execve()`. 

A look at the unprivileged editor process in which this article is being written (as seen in `/proc/_pid_ /status`) shows: 

```
CapInh:	0000000000000000
        CapPrm:	0000000000000000
        CapEff:	0000000000000000
        CapBnd:	000001ffffffffff
        CapAmb:	0000000000000000
```

All of the sets are zero (indicating no capabilities set) with the exception of the bounding set, where all capabilities are allowed. 

An executable file can have its own permitted and inheritable sets that cause it to run with additional privilege (like a restricted form of setuid program), along with a single "effective" bit that causes the permitted set to also be established as the effective set. As described above, capabilities in the file's inheritable set are only enabled if they also appear in the inheritable set of the thread executing the file with `execve()`. In general, the interactions between the sets can be complex; see the above-linked capabilities man page for all the details. 

#### Yet another capability set

At its core, Calmels's patch set works by adding another capability set — the userns capability set — to the above pile. During a thread's normal operation, this capability set is not consulted by the kernel. The thread can change the capabilities in that set with a new [`prctl()`](<https://man7.org/linux/man-pages/man2/prctl.2.html>) call, but setting new capabilities there will only succeed if either those capabilities already exist in the thread's permitted set or the thread holds the `CAP_SETPCAP` capability. Additionally, the operation will also only succeed if the requested capabilities appear in the thread's bounding set. 

The new capability set comes into play, though, when a thread creates a new user namespace. At that point, the effective, permitted, bounding, and userns capability sets within the namespace will all be set to the creator's userns capability set. If the creator's set reflects a reduced set of capabilities, then root within the namespace will no longer be all-powerful there. Any system calls that need the missing capabilities will become off-limits, thus reducing (or so it is hoped) the attack surface that the kernel presents within the namespace. 

By default, the userns capability set contains the full list of capabilities, so no restrictions will be applied within user namespaces. This default preserves the existing behavior of user namespaces. 

The patch series also creates a new sysctl knob (`kernel.cap_userns_mask`) that is applied to all userns capability sets. By default this mask contains all capabilities; if the system administrator removes some capabilities from it, then no user namespace created within the system can have that capability internally. Finally, and somewhat controversially, there is a set of changes to allow BPF Linux security modules (LSMs) to adjust all of the capability sets (including the userns set) for a thread. 

#### Mixed reception

While there was no opposition to the idea of reducing capabilities within a user namespace, the mechanism implemented in this patch has not been universally popular. Casey Schaufler [called the first version of the series](<https://lwn.net/ml/all/2804dd75-50fd-481c-8867-bc6cea7ab986@schaufler-ca.com>) ""a bad idea"", [adding](<https://lwn.net/ml/all/df3c9e5c-b0e7-4502-8c36-c5cb775152c0@schaufler-ca.com>) that the interaction between the various capability sets was too complex for user-space developers now. He [suggested](<https://lwn.net/ml/all/799f3963-1f24-47a1-9e19-8d0ad3a49e45@schaufler-ca.com>) a mechanism built into user namespaces directly, or perhaps a [`clone()`](<https://man7.org/linux/man-pages/man2/clone.2.html>) flag, instead. John Johansen [suggested](<https://lwn.net/ml/all/641a34bd-e702-4f02-968e-4f71e0957af1@canonical.com>) that perhaps restricting capabilities within user namespaces should be implemented within the security-module mechanism; this idea may have led to the LSM hook added in the second version. 

That hook, though, [did not gain favor](<https://lwn.net/ml/all/CAHC9VhT5XWbhoY2Nw5jQz4GxpDriUdHw=1YsQ4xLVUtSnFxciA@mail.gmail.com>) from LSM maintainer Paul Moore, who worried about giving LSMs the ability to modify a thread's capabilities. He [pointed out](<https://lwn.net/ml/all/CAHC9VhQFNPJTOct5rUv3HT6Z2S20mYdW75seiG8no5=fZd7JjA@mail.gmail.com>) the potential for bad interactions between security modules, any of which might be using the capability sets to make access-control decisions. LSMs are currently restricted to modifying their own internal state, he said, and that situation should not change; modification of capability sets should only be done within the capability LSM. He [summarized](<https://lwn.net/ml/all/CAHC9VhTaD+LxEL8hMdbvFFJgU1xbEtmN7gSwSdKRS=hX_uVnVQ@mail.gmail.com>) that ""this patch is not acceptable at this point in time"". 

On the other hand, Serge Hallyn, the current maintainer of the kernel's capability subsystem, has been generally favorable to the idea, [saying](<https://lwn.net/ml/all/ZkidDlJwTrUXsYi9@serge-l-PF3DENS3>): ""I'm a container developer, and I'm excited about it"". He has provided Reviewed-by tags for most of the series, with the exception of the LSM hook; his [suggestion](<https://lwn.net/ml/all/20240615152010.GB44653@mail.hallyn.com>) is that the series should move forward with everything except that hook. 

That seems like the most likely outcome for this patch set. The capability-based solution did not find universal acclaim, but it does not appear that anybody is so opposed to it that they will fight its inclusion. While most users will never notice this relatively new feature, container developers may well take advantage of it to ratchet down the level of privilege (and vulnerability exposure) given to containers, and distributors may find that it helps them to get over their fear of user namespaces in general.
