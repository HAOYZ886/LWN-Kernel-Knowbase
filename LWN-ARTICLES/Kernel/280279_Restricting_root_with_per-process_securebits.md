---
title: "Restricting root with per-process securebits"
url: https://lwn.net/Articles/280279/
date: "April 30, 2008"
category: Capabilities
author: "By Jake Edge April 30, 2008"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
April 30, 2008

Linux capabilities have had a long and somewhat tortuous journey as part of the Linux kernel. Slowly--and very carefully--functionality is being added to this security feature to get it to a point where it is a viable alternative to the all-or-nothing `setuid(0)` model. A recently merged [patch](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commit;h=3898b1b4ebff8dcfbcf1807e0661585e06c9a91c>) adds a per-process securebits feature that will allow capabilities-based daemons or subsystems to coexist with existing `setuid` utilities. 

Linux capabilities break up the privileged tasks normally associated with root (i.e. uid 0) into finer-grained abilities which can be individually granted or revoked for specific processes. The idea is to change the standard Unix model that root has all special privileges while all other users have none. The terminology is always a bit contentious, though, as Linux capabilities are derived from a POSIX proposal that was never adopted, but shares the name "capabilities" with an [entirely different approach](<http://en.wikipedia.org/wiki/Capability-based_security>); this article is only concerned with capabilities of the Linux variety. 

There has long been interest in creating a Linux system that did not rely upon a single root account. Capabilities are seen as the way to get there, but they have suffered from a bit of a chicken-and-egg problem. With the recent work to add [file-based capabilities](<http://lwn.net/Articles/211883/>) and [restore `CAP_SETPCAP` to its original meaning](<http://lwn.net/Articles/256519/>), a true capabilities-based system is becoming possible. In the patch, which has been merged for 2.6.26, Andrew Morgan describes the new functionality: 

The feature added by this patch can be leveraged to suppress the privilege associated with (set)uid-0. This suppression requires CAP_SETPCAP to initiate, and only immediately affects the 'current' process (it is inherited through fork()/exec()). This reimplementation differs significantly from the historical support for securebits which was system-wide, unwieldy and which has ultimately withered to a dead relic in the source of the modern kernel. 

The patch removes the global securebits variable, replacing it with an entry in `struct task_struct`, that can be manipulated by a process, but only for itself--and any children. Morgan envisions hybrid systems that have some utilities using capabilities to get their privileges along with some `setuid(0)` utilities. In that scenario, a capabilities-based utility or daemon may wish to limit what its children can do, even if they execute a `setuid(0)` binary. As part of the evolution, process trees can be created that cannot get root privileges. 

Processes which have the `CAP_SETPCAP` capability can change their securebits setting via the `prctl()` system call. There are three separate bits that govern the interaction of capabilities and `setuid`: 

  * `SECURE_NOROOT` - enabling this gives no special privileges to uid 0
  * `SECURE_NO_SETUID_FIXUP` - setting this bit disables capability fixes when transitioning from or to uid 0 via `setuid`. This might be done for compatibility with older programs that use `setuid` to reduce their privileges.
  * `SECURE_KEEP_CAPS` - when set, a process can retain its capabilities even when transitioning to a normal (not uid 0) user. This bit is cleared by `exec()`.

Each of these bits also has a companion `*_LOCKED` bit that, if set, will not allow any user program to alter the corresponding setting. As Morgan notes in the patch, a program that can set its capabilities (has `CAP_SETPCAP`) can drop all privileges for itself and any child process by doing: 

```
prctl(PR_SET_SECUREBITS, 0x2f);
```

This is the equivalent of setting `SECURE_NOROOT`, `SECURE_NO_ROOT_LOCKED`, `SECURE_NO_SETUID_FIXUP`, `SECURE_NO_SETUID_FIXUP_LOCKED`, and `SECURE_KEEP_CAPS_LOCKED`. 

The memory of the sendmail-capabilities bug from 2000 makes some a bit queasy--or worse--about any patches that involve capabilities and `setuid`. Andrew Morton [asks](<https://lwn.net/Articles/280386/>): ""what was the bug which caused us to cripple capability inheritance back in the days of yore? (Some sendmail thing?)"" That bug was caused because unprivileged users could take away the `CAP_SETUID` capability from `setuid` binaries like sendmail. When sendmail then used `setuid` to drop its privileges, it failed, but sendmail did not check, so it was still running with full privilege. This could be leveraged by a user to gain root privileges. It was a disconnect between capabilities and the longstanding behavior of Unix-like systems when dropping privileges. 

Morgan has written a [detailed description of the sendmail-capabilities bug](<http://userweb.kernel.org/~morgan/sendmail-capabilities-war-story.html>) in response to Morton's questions. He makes it clear that he wants to move toward full capability support without breaking existing code: 

I'm basically interested in evolving the capability implementation back to the POSIX.1e model and making it whole - but most certainly *without crippling legacy superuser support in the process* . 

As folk get more comfortable with this full capability model. I believe we can delete more cruft from the main kernel, but even that clean up will leave a fully functional legacy model in place. I feel it should be for something like init, or one of its children to be able to run subsystems in capability-only or legacy modes. 

Morton seemed satisfied that his concerns had been addressed, but still [wonders about the future for capabilities](<https://lwn.net/Articles/280388/>): ""So how do we ever get to the stage where we can recommend that distributors turn these things on, and have them agree with us?"" This was [echoed by Ismail Donmez](<https://lwn.net/Articles/280389/>), who was looking for concrete examples of how to use the per-process securebits feature. Morgan provides a pointer to some examples along with his belief that sometime soon the capabilities developers will become confident enough to recommend turning off the "experimental" flag for the `SECURITY_FILE_CAPABILITIES` kernel configuration. That flag governs both the file-based capabilities as well as the per-process securebits. In addition, Morgan [says](<https://lwn.net/Articles/280390/>): 

More importantly I'm hopeful that in that time we'll have accumulated enough documentation and user-space experience and examples to convince others that this is, indeed, a viable feature to support in mainstream distributions. 

A [developerWorks article on file-based capabilities](<http://www.ibm.com/developerworks/library/l-posixcap.html>) by Serge Hallyn and a [web page on POSIX capabilities](<http://www.friedhoff.org/posixfilecaps.html>) by Chris Friedhoff were both mentioned in the thread as good references for the work being done to actually _use_ capabilities in systems. Those pre-date the securebits work, so Donmez was looking for use-cases for the new feature. Morgan replied that containers were one, deferring to Hallyn who [has some ideas on using securebits](<https://lwn.net/Articles/280410/>): 

We tend to talk about 'system containers' versus 'application containers'. A system container would be like a vserver or openvz instance, something which looks like a separate machine. I was going to say I don't imagine per-process securebits being useful there, but actually since a system container doesn't need to do any hardware setup it actually might be a much easier start for a full SECURE_NOROOT distro than a real machine. Heck, on a real machine init and a few legacy [daemons] could run in the init namespace, while users log in and apache etc run in a SECURE_NOROOT container. 

But I especially like the thought of for instance postfix running in a carefully crafted application container (with its own virtual network card and limited file tree and no visibility of other processes) with SECURE_NOROOT on. 

Capabilities are an interesting, but complicated, security feature. For most of the ten years they have been part of the Linux kernel, they have either been broken, ignored, or both. With the latest work being done by Hallyn, Morgan, and others, capabilities are finally becoming a fully-working alternative to things like SELinux. It will be interesting to see if more user utilities will become capability-aware and whether distributions start using capabilities. Some day, root may just fade away.
