---
title: "A security-module hook for user-namespace creation"
url: https://lwn.net/Articles/903580/
date: "August 4, 2022"
category: "Namespaces-User namespaces; Releases-6.1; Security-Security modules"
author: "By Jonathan Corbet August 4, 2022"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
August 4, 2022

The Linux Security Module (LSM) subsystem works by way of an extensive set of hooks placed strategically throughout the kernel. Any specific security module can attach to the hooks for the behavior it intends to govern and be consulted whenever a decision needs to be made. The placement of LSM hooks often comes with a bit of controversy; developers have been known to object to the performance cost of hooks in hot code paths, and sometimes there are [misunderstandings](<https://lwn.net/Articles/902466/>) over how integration with LSMs should be handled. The disagreement over a security hook for the creation of user namespaces, though, is based on a different sort of concern. 

User namespaces, which can be created by unprivileged processes, give the creator complete control over user and group IDs. Within the namespace, the creator can run as root, but all interactions with the system are mapped back to the creator's user and group ID. They are a fundamental building block for unprivileged containers. In theory, user namespaces are entirely safe; in practice, they have long been accompanied by worries about the increased attack surface that comes from making formerly root-only actions available within the namespace. There have indeed been vulnerabilities resulting from interactions with user namespaces; see [this report](<https://lwn.net/ml/oss-security/nZdp4o4iHdicJfJwEJ-dtJrhs5aDa-cbvA3psbItS3dkwOwxmzwXanoaslI0T5nXjCNz0Cm5csVgCJxDWPWIaKDbF6mxaYch5xJo3QT-8_0%3D%40protonmail.com/>) for a recent example. Whether user namespaces are truly more prone to vulnerabilities than the rest of the kernel is not clear, though. 

See [this article](<https://lwn.net/Articles/532593/>) for more information on user namespaces. 

As Frederick Lawler notes in [this patch set](<https://lwn.net/ml/linux-kernel/20220801180146.1157914-1-fred@cloudflare.com/>), security modules currently have a degree of control over user-namespace creation, but that control relies on a hook that was not actually intended for access-control decisions. Among other things, that prevents any error code from being propagated back to the (presumably frustrated) user. The solution is to create a new hook (`security_create_user_ns()`) that is called prior to the creation of a new namespace, and which can cause the action to fail if it is not consistent with the current security policies. 

It is a relatively straightforward patch set, even after adding a self-test and a hook implementation for SELinux. Over four revisions, it has seen a number of tweaks and appears to be at a point where developers in the security community are happy with it. There is, of course, an exception; Eric Biederman [raised some objections](<https://lwn.net/ml/linux-kernel/877d45kri4.fsf@email.froward.int.ebiederm.org/>) in response to the posting of the third revision in late July. One was that blocking access to user namespaces as a way of reducing attack surface would only be effective if the bulk of the exploitable bugs in the kernel would be blocked; otherwise, he said, attackers would simply find a different bug to exploit. 

His larger complaint, though, was essentially an objection to applying any sort of access control to user namespaces at all. Over time, he said, numerous new kernel features have been restricted to the root account mostly because otherwise they could be used to confuse setuid programs. That has resulted in more code running as root, which is not good for security overall. User namespaces were meant to bring an end to that trend: 

> One of the goals of the user namespace is to avoid more and more code migrating to running as root. To achieve that goal ordinary application developers need to be able to assume that typically user namespaces will be available on linux. 
> 
> An assumption that ordinary applications like chromium make today. 
> 
> Your intentions seem to be to place a capability check so that only root can use user namespaces or something of the sort. Thus breaking the general availability of user namespaces for ordinary applications on your systems. 

Biederman concluded with a "Nacked-by" tag indicating rejection of the patch. 

He was alone in that view, though. SELinux maintainer Paul Moore [answered](<https://lwn.net/ml/linux-kernel/CAHC9VhQXSXWv=+WYwU=Qq0w3rd+zOFPHL5yut1JdV2K=DDRmmg@mail.gmail.com/>) that LSM hooks provide for more than just access control; auditing and observability are also important use cases for them. He asserted that ""integrating the LSM into the kernel's namespaces is a natural fit, and one that is long overdue"", and asked Biederman to suggest alternatives if the proposed hook is truly not acceptable. Ignat Korchagin (who, like Lawler, posted from a Cloudflare email address) [said](<https://lwn.net/ml/linux-kernel/CALrw=nGT0kcHh4wyBwUF-Q8+v8DgnyEJM55vfmABwfU67EQn=g@mail.gmail.com/>) that the real goal was to increase the use of user namespaces by providing more control over them: 

> So in a way, I think this hook allows better adoption of user namespaces in the first place and gives distros and other system maintainers a reasonable alternative than just providing a global "kill" sysctl (which is de-facto is used by many, thus actually limiting userspace applications accessing the user namespace functionality). 

Biederman did not respond further, and the conversation wound down. Moore [suggested](<https://lwn.net/ml/linux-kernel/CAHC9VhQw8LR9yJ9UkA-9aPNETQavt25G-GGSs-_ztg6ZpxNzxA@mail.gmail.com/>) that it was time to get the work merged: 

> There is the issue of Eric's NACK, but I believe the responses that followed his comment sufficiently addressed those concerns and it has now been a week with no further comment from Eric; we should continue to move forward with this. 

After the posting of version 4 on August 1, though, Biederman [made it clear](<https://lwn.net/ml/linux-kernel/87les7cq03.fsf@email.froward.int.ebiederm.org/>) that he did not believe his concerns had been dealt with. ""Nack Nack Nack"". 

What will come next is not entirely clear. Biederman still has not offered any sort of alternative approach to the problem, so developers are left with a somewhat unpleasant choice: either restart from the beginning in the hopes of finding a solution that Biederman will not try to block, or simply ignore Biederman and merge the hook anyway. Since Biederman is evidently opposed to any sort of access control for user-namespace creation, the space of mutually acceptable solutions may be small. Pushing code over a maintainer's objections is not done lightly in the kernel community, but it does occasionally happen. Moore has already [indicated](<https://lwn.net/ml/linux-kernel/CAHC9VhRpUxyxkPaTz1scGeRm+i4KviQQA7WismOX2q5agzC+DQ@mail.gmail.com/>) that this case may play out that way.
