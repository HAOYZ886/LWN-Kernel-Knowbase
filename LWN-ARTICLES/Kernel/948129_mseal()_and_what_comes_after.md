---
title: mseal() and what comes after
url: https://lwn.net/Articles/948129/
date: "October 20, 2023"
category: Security; System calls
author: "By Jonathan Corbet October 20, 2023"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
October 20, 2023

Jeff Xu recently [proposed](<https://lwn.net/ml/linux-kernel/20231016143828.647848-1-jeffxu@chromium.org/>) the addition of a new system call, named `mseal()`, that would allow applications to prevent modifications to selected memory mappings. It would enable the hardening of user-space applications against certain types of attacks; some other operating systems have this type of feature already. There is support for adding this type of mechanism to the Linux kernel as well, but it has become clear that `mseal()` will not land in the mainline in anything resembling its current form. Instead, it has become an example of how _not_ to do kernel development at a number of levels. 

Xu described the new system call's purpose as: 

> Memory sealing additionally protects the mapping itself against modifications. This is useful to mitigate memory corruption issues where a corrupted pointer is passed to a memory management syscall. For example, such an attacker primitive can break control-flow integrity guarantees since read-only memory that is supposed to be trusted can become writable or .text pages can get remapped. 

The target user for this functionality is the Chrome browser which, among other things, includes a just-in-time (JIT) compilation engine for JavaScript code. Since it generates executable code on the fly, JIT compilation must be done with care, lest it create (and run) the wrong kind of code. As described in [this blog post](<https://v8.dev/blog/control-flow-integrity>) by Stephen Röttger, a lot of effort has gone into control-flow integrity and preventing the JIT system from becoming a tool for an attacker. If, however, an attacker is able to somehow force a memory-management system call that changes memory permissions, all bets are off. Thus, the Chrome developers would like to have a mechanism that puts those system calls off-limits for specific regions of memory, hardening the browser against that sort of attack. 

The cover letter notes that `mseal()` is similar to [`mimmutable()`](<https://lwn.net/Articles/915640/>), which was added recently to OpenBSD. The prototype for the proposed system call is quite different from `mimmutable()`, though: 

```
int mseal(void *addr, size_t len, unsigned int types, unsigned int flags);
```

The range of memory to be affected is indicated by `addr` and `len`. The `flags` must be passed as zero, and `types` controls which system calls will be blocked on that address range: 

  * `MM_SEAL_MPROTECT`: [`mprotect()` and `pkey_mprotect()`](<https://man7.org/linux/man-pages/man2/mprotect.2.html>)
  * `MM_SEAL_MMAP`: [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>)
  * `MM_SEAL_MUNMAP`: `munmap()`
  * `MM_SEAL_MREMAP`: [`mremap()`](<https://man7.org/linux/man-pages/man2/mremap.2.html>)
  * `MM_SEAL_MSEAL`: future `mseal()` calls 

Linus Torvalds was quick to [object](<https://lwn.net/ml/linux-kernel/CAHk-=whFZoap+DBTYvJx6ohqPwn11Puzh7q4huFWDX9vBwXHgg@mail.gmail.com/>) to the patch series, saying: ""I have no objections to adding some kind of 'lock down memory mappings' model, but this isn't it"". He had a number of complaints about the details of the implementation, but he later [made it clear](<https://lwn.net/ml/linux-kernel/CAHk-=wj87GMTH=5901ob=SjQqegAm2JYBE7E4J7skJzE64U-wQ@mail.gmail.com/>) the design of the system call was wrong. Blocking `munmap()`, for example, makes little sense if other operations that can result in the unmapping of addresses (`mmap()` and `mremap()`, for example), are still allowed. The effort that was put into only blocking operations from specific system calls, he [said](<https://lwn.net/ml/linux-kernel/CAHk-=wixGw88-OzcFbCLEuAzSe53oUUozdM-E_RJwvejgY6ySA@mail.gmail.com/>), was overtly wrong; if unmapping a range of memory (for example) is blocked, it must be blocked from all directions or the protection provided will be illusory. 

Matthew Wilcox [questioned](<https://lwn.net/ml/linux-kernel/ZS1URCBgwGGj9JtM@casper.infradead.org/>) the complexity of the interface, suggesting instead that a couple of flags added to `mprotect()` would suffice. A memory region, he said, should either be immutable (with the possible option of further reducing access) or not, without regard to which system call was used. He later added: 

> This is the seccomp disease, only worse because you're trying to deny individual syscalls instead of building up a list of permitted syscalls. If we introduce a new syscall tomorrow which can affect VMAs, we'll then make it the application's fault for not disabling that new syscall. That's terrible design! 

The conversation even brought about a rare appearance on linux-kernel by OpenBSD maintainer Theo de Raadt, who [agreed with Torvalds](<https://lwn.net/ml/linux-kernel/55960.1697566804@cvs.openbsd.org/>) and suggested that Linux should simply add `mimmutable()` rather than reinventing that functionality in a more complex form. Torvalds [was amenable](<https://lwn.net/ml/linux-kernel/CAHk-=wjS=xg12RVQdTNxEurjo21eXQBQO0D5My6Aox4LCfsO1A@mail.gmail.com/>) to this idea, though he suggested adding a `flags` argument for future changes — an idea that de Raadt [disliked](<https://lwn.net/ml/linux-kernel/19404.1697568901@cvs.openbsd.org/>). That reflects the fact that OpenBSD controls its user space, so it can add a `flags` parameter later if the need arises; Linux has no such luxury, so that parameter must be present from the beginning if it is to exist at all. 

Xu, instead, [resisted](<https://lwn.net/ml/linux-kernel/CALmYWFs81T=XnT=AXQTo0+9FXo=OBAV_4rrYPSn9-16O-gBTZg@mail.gmail.com/>) the idea, prompting a typical (if relatively mild) [de Raadt response](<https://lwn.net/ml/linux-kernel/95482.1697587015@cvs.openbsd.org/>). Indeed, Xu continued to cling to his proposed design despite the comments he had received, leading to a somewhat exasperated [post](<https://lwn.net/ml/linux-kernel/ZS%2F3GCKvNn5qzhC4@casper.infradead.org/>) from Wilcox, who tried to direct the conversation toward what the patch series is actually trying to accomplish: 

> Let's start with the purpose. The point of mimmutable/mseal/whatever is to fix the mapping of an address range to its underlying object, be it a particular file mapping or anonymous memory. After the call succeeds, it must not be possible to make any address in that virtual range point into any other object. 
> 
> The secondary purpose is to lock down permissions on that range. Possibly to fix them where they are, possibly to allow RW->RO transitions. 
> 
> With those purposes in mind, you should be able to deduce for any syscall or any madvise(), ... whether it should be allowed. 

Xu, Wilcox concluded, needed do a better job of listening to the developers who were trying to help him. 

At this point, it is clear that `mseal()` will not enter the kernel in anything like its current form. That leads to the question of what should be done instead. Röttger [jumped into the conversation](<https://lwn.net/ml/linux-kernel/CAEAAPHYgg49WtpE7Jdq6HDecFp5RHPhdxtPTUaNF12RONu5aDA@mail.gmail.com/>) to point out that a pure `mimmutable()` solution does not do everything that the Chrome developers would like to see; they have cases where they want to prevent unmapping, but still need to be able to change memory protections with `mprotect()`. De Raadt [described](<https://lwn.net/ml/linux-kernel/20251.1697817328@cvs.openbsd.org/>) that case as ""partial sealing"" that means that the memory in question is not actually protected. 

There will presumably be some sort of follow-up proposal that maintains that capability while removing the more complex options provided by `mseal()`. But whether that proposal will be `mimmutable()` or some variant thereof remains to be seen. 

One can point to a number of things that went wrong here. The original proposal was seen by many as an implementation of what the Chrome developers said they wanted without looking deeply at what the real requirements (for Chrome and any other potential users) are. Google has no shortage of experienced developers who could have reviewed this submission before it was posted publicly, but that does not appear to have happened, with the result that a relatively inexperienced developer was put into a difficult position. Feedback on the proposal was resisted rather than listened to. The result was an interaction that pleased nobody. 

Despite all of that, there is a use case here that everybody involved appears to see as legitimate. So it is just a matter of finding the right solution to the problem, and hopefully that problem is better understood now. If the next attempt looks a lot more like `mimmutable()` and reflects the feedback that has been given, the kernel may yet get the sealing capability that addresses the Chrome use case and provides for wider user-space hardening as well.
