---
title: Restricting the network
url: https://lwn.net/Articles/368730/
date: "January 6, 2010"
category: "Security-Security modules"
author: "By Jake Edge January 6, 2010"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
January 6, 2010

New security features can be affected by the "law of unintended consequences", because a seemingly simple restriction runs afoul of unanticipated interactions with other parts of the system--often other security mechanisms. These interactions can be difficult to spot immediately, which makes kernel hackers very careful about adding new security features. A recent proposal to provide a means for processes to restrict their network access--something that would be useful for a process sandbox for instance--ran into unintended consequences. But the somewhat _ad hoc_ nature of the feature, and its tuning for a fairly specific use case, also caused objections from some. 

The basic idea is fairly simple. Michael Stone would like to have a means for a process to reduce its privileges such that it can no longer make network connections. It would be a one way gate for a process (and any subsequent children) that would restrict network usage to previously opened connections. Because Stone's use case is for the desktop--specifically some parts of the OLPC Bitfrost security model--there would be an exception made for connecting to named `AF_UNIX` sockets, which would allow restricted processes to still be able to talk to the X server. 

When he initially [proposed](<http://article.gmane.org/gmane.linux.network/115744>) the idea in an RFC in January 2009, Stone took a straightforward approach using resource limits. He added a new boolean limit (`RLIMIT_NETWORK`) that could be set by a process to turn off further network activity. There was a problem in that scheme in that it didn't actually limit the process because it didn't stop it from using `ptrace()`. A subverted process could still do networking via another process by using `ptrace()` on it. 

In addition, James Morris [noted](<http://article.gmane.org/gmane.linux.kernel/777223>) that network namespaces might be a possible solution to the problem. After that round of comments, Stone came back with an updated [patchset](<http://lwn.net/Articles/366393/>) in December. He addressed the `ptrace()` issue by adding a test for the resource limit in `__ptrace_may_access()` that would prevent processes that are network-limited from using `ptrace()`. He also noted that using network namespaces didn't support one part of his use case: processes in a private namespace could no longer connect to the X server using `AF_UNIX` sockets. 

Using resource limits as the interface was not very well received by glibc maintainer Ulrich Drepper (""it's a pain to deal with rlimit extensions""), who [suggested](<https://lwn.net/Articles/368786/>) using `prctl()` instead. Stone quickly turned around [another version of the patch](<https://lwn.net/Articles/368787/>) that used `prctl()`, but a few problems cropped up along the way. 

At first blush, removing further network access seems like a harmless way for a process to voluntarily give up some portion of its privileges. But, when coupled with `setuid()` binaries that expect to be able to access the network, things get murkier. As Eric W. Biederman [put it](<https://lwn.net/Articles/368817/>): ""You can in theory confuse a suid root application and cause it to take action with it's elevated privileges that violate the security policy."" That is why privileges are required for entering a new network namespace (as well as for things like `chroot()`), because they can violate the assumptions made by `setuid()` programs. 

Stone is looking for a mechanism that doesn't require a privileged process, however, which is why he proposed resource limits or `prctl()` as the interface. But those don't alleviate the problem with suid programs. The so-called "sendmail capabilities bug" was brought up several times in the conversation about Stone's feature as a concrete example of how the interaction between security mechanisms can go awry. That bug was really in the kernel, but by manipulating the Linux capabilities of a process before spawning `sendmail` (which runs as `setuid(0)`), attackers could bypass the privilege separation that `sendmail` tries to enforce. Adding a new security mechanism (capabilities) suddenly--mistakenly--changed the behavior of a well-established security technique. 

Implementation bugs aside, there are concerns about sprinkling support for this feature in various places in the kernel: `ptrace()` and the networking stack, particularly since the changes have the `AF_UNIX` exception as a special case. Alan Cox [puts it](<https://lwn.net/Articles/368832/>) this way: 

This is a security model, it belongs as a security model using LSM. You can already do it with SELinux and the like as far as I can see but that's not to say you shouldn't submit it also as a small handy standalone security module for people who don't want to load the big security modules. 

Otherwise you end up putting crap in fast paths that nobody needs but everyone pays for and weird tests and hacks for address family and like into core network code. 

The fact the patches look utterly ugly should be telling you something - which is that you are using the wrong hammer 

Unfortunately, switching to an LSM-based solution opens the ""stacking-LSM can of worms again"", as Valdis Kletnieks [calls it](<https://lwn.net/Articles/368835/>). Currently, there is no general way to run multiple LSMs in a kernel. The idea has come up multiple times, but there are serious concerns about allowing it. Any new LSM is much less likely to be used, at least in distributions that already use one of the "monolithic" security modules like SELinux, TOMOYO, or the out-of-tree AppArmor. In another thread Stone [queried](<https://lwn.net/Articles/368836/>) linux-kernel on the use of LSM and expressed that concern: 

Unfortunately, I don't feel that I can make effective use of these hooks because they seem to be "occupied" by the large mandatory access control frameworks. 

Smack developer Casey Schaufler essentially [agreed](<https://lwn.net/Articles/368837/>), but encouraged Stone to go forward with an LSM-based solution: 

You're arguing for stacking a set of small security modules. This is a direction that has gotten slammed pretty hard in the past but that crops up every time someone like you comes along with a module that serves a specific purpose. Mostly the objections have come from people who will tell you that something else already does what you're trying to do, and that all you have to do is take on the entirety of their monolithic approach and you'll be happy. 

I'm behind you 100%. Use the LSM. Your module is exactly why we have the blessed thing. Once we get a collection of otherwise unrelated LSMs the need for a stacker will be sufficiently evident that we'll be able to get one done properly. 

There are good reasons to be concerned about stacking security modules, but they mostly stem from trying to combine things like SELinux and TOMOYO rather than small single-purpose modules. Serge E. Hallyn [warned](<https://lwn.net/Articles/368838/>) that ""the problem is that composing any two security policies can quickly have subtle, unforeseen, but dangerous effects."" But he also pointed out that there are ways to "hardcode" stacking with the assistance of the other LSM developers: 

So with your module, I'd recommend following the route of the capabilities LSM. You can provide an optional stand-alone LSM which only hooks your functions. Then smack, for instance, can call the functions in your LSM from within its own hooks, or it can simply explicitly assign its hooks to your functions in smack_ops. Selinux can do the same thing, although I suspect they would more likely implement their own functions for your newly hooked sites. 

While not opposed to that approach in principle, Stone [notes](<https://lwn.net/Articles/368839/>) that it requires others to change their code, something he has been trying to avoid: 

Doesn't it seem a bit strange to you to be recommending that everyone else using the five security hooks I want to use modify their code *in detail* to support my functionality given that my functionality is explicitly intended not to require any such work on their part? 

This seems frankly silly to me, not to mention expensive and error-prone. 

Another alternative would be to use SELinux to do the restriction as Kyle Moffett [suggested](<https://lwn.net/Articles/368841/>): ""If you aren't using SELinux at this time (and therefore have no existing policy), then it's actually pretty straightforward (relatively speaking) to set up for your particular goals."" He outlined an SELinux policy scheme to enforce the networking restrictions. Schaufler was [skeptical](<https://lwn.net/Articles/368842/>) of that approach--while noting his amusement that an SELinux advocate would call the default polices ""fantastically complicated"" as Moffett did. Schaufler expects the full policy to support Stone's use case to be rather complicated itself: 

I'm willing to bet all the beers you can drink in a sitting that the policy would be bigger than the proposed LSM. You can count that in either bytes or lines. 

Meanwhile, Stone proposed yet another [version](<http://lwn.net/Articles/368108/>) that uses the LSM hooks. The feature is still enabled through `prctl(PR_SET_NETWORK, PR_NETWORK_OFF)`, but the implementation is done via a `disablenetwork` LSM. But there is still the problem of removing the network for `setuid()` programs that are spawned from the restricted, unprivileged program. Some don't see that as a real problem, because the network could go away for other reasons (network cable pulled, open file limit set sufficiently low, and so forth), but others like Pavel Machek, who NAKed the patch, [disagree](<https://lwn.net/Articles/368856/>), envisioning plausible, if unlikely, scenarios where it could cause a problem. 

That led Biederman to [propose](<http://lwn.net/Articles/368263/>) a mechanism that would allow processes to call `prctl(PR_SET_NOSUID)` to permanently revoke their ability to execute `setuid()` programs (in much the same manner as the `MNT_NOSUID` mount option). Any process that did that would then be eligible to also revoke its network access. In addition, it would potentially allow entering private namespaces to become a non-privileged operation as namespaces suffer from the some of the same issues regarding `setuid()` programs. 

But, once again, Biederman's patch implements a security model of sorts, and belongs in an LSM, at least [according to Cox](<https://lwn.net/Articles/368859/>): ""Another fine example of why we have security hooks so that we don't get a kernel full of other 'random security idea of the day' hacks."" Which leads right back to the problem of stacking security modules. Like Schaufler, though, Cox seems to think LSM stacking will eventually come to pass: 

Yes it might mean the hooks need tweaking, yes it probably means the people who want these need to do some trivial stacking work, but if as many people are actually really interested as are having random 'lets add a button to disable reading serial ports on wednesday' ideas there should be no shortage of people to do the job right. 

Part of the problem is the whole raft of security mechanisms that Linux supports: `setuid()`, capabilities, LSMs, monolithic LSMs like SELinux, securebits (which was mentioned as a possible solution for `PR_SET_NOSUID`), seccomp, and more. As the sendmail capabilities bug showed, these can interact in unexpected ways. Adding a specific knob, whether it be disabling the network or `setuid()`, only addresses that particular problem, while potentially impacting the whole system in a negative way. 

It is rather counter-intuitive that allowing non-root programs to voluntarily drop some portion of their privileges should lead to other security problems. The root cause may really be `setuid()`, but that mechanism is so ingrained into Unix programming that there is little to be done but live with it--warts and all. But there will be more and more pressure to provide ways for processes to sandbox themselves (and others). The [seccomp changes](<http://lwn.net/Articles/332974/>) proposed by Google for its Chrome browser in May are another way of approaching the problem. 

Even with all of the competing--sometimes clashing--security mechanisms, one gets the sense that there is more infrastructural work to be done in Linux security. If the concern about generalized LSM stacking is only for the monolithic security models, one could imagine some kind of "LSM lite" that used the same hooks but had restrictions on behavior such that modules could stack. Perhaps some of these restrictions could be implemented as some kind of trusted user space daemon that changed the capabilities of running processes. So far, it's not clear where things are headed, but it does seem clear that sandboxing is something that folks want to be able to do, and that there are some approaches to that problem that Linux does not yet support.
