---
title: Snet and the LSM API
url: https://lwn.net/Articles/316940/
date: "January 28, 2009"
category: "Security-Security modules"
author: "By Jake Edge January 28, 2009"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
January 28, 2009

A new security module, called [snet](<http://www.synack.fr/project/snet/snet.html>) (which is short for "security for network syscalls") was recently [posted as an RFC](<https://lwn.net/Articles/316939/>) on the linux-security-module mailing list. Its purpose is rather simple--much simpler than the two current mainline users of the LSM interface--intercept system calls for networking and call out to user space to determine if they are to be allowed. The idea is to be able to create Linux versions of the "personal firewall" that is popular on Windows machines. Reaction to snet was mixed, partially because of a disdain for that type of security tool, but also because it is implemented using LSM. 

Snet, developed by Samir Bellabes, consists of a kernel piece which uses LSM to hook the "interesting" socket-related system calls (`socket()`, `bind()`, `connect()`, `listen()`, and `accept()`), as well as a user space library that can be used to accept or deny those calls. Communication between the kernel and user space is handled by a netlink socket using libnl. The decisions are then cached in the kernel to reduce the number of calls required to user space. That last part is important because personal firewalls typically pop up a request on the user's display asking them to decide whether to allow the system call. Timeouts can be established for the user-space calls, along with a default response if the timeout is reached. 

This "user request" feature of personal firewalls is one thing that many find objectionable. As Paul Moore [puts it](<https://lwn.net/Articles/317017/>): ""my opinion is that it is a poor option for security and typically only results in training the user to click the 'allow' button when the pfwall [dialog] box pops up on his/her screen"". Yet it is a "feature" of other operating systems and not completely unreasonable for Linux to support. From that perspective, snet seems like a reasonable starting point. 

There are a few other problems, though, stemming from the decision to use the LSM API. Peter Dolding [seems to think](<https://lwn.net/Articles/317018/>) this capability should be added to netfilter, rather than built as a standalone solution. Others pointed out that netfilter is sufficiently low-level that any context about users or processes that are performing these operations is not available. That could change, but it would take a concerted effort to change the netfilter code, which doesn't seem likely near-term, if ever. 

A larger problem comes from the inability to stack LSM modules. If a user is interested in the kinds of protection that snet can provide, they must forgo any other LSM-implemented security solution (i.e. SELinux, Smack, AppArmor, TOMOYO, etc.). A [parallel discussion](<https://lwn.net/Articles/317020/>) about LSM stacking is also occurring on linux-security-module, partially motivated by the needs of snet and other "smaller" security solutions. Those tools do not implement a full-scale security solution _a la_ SELinux or Smack, but instead try to handle a smaller subset of the problem. 

LSM stacking also came up at the [LCA security panel](<http://lwn.net/Articles/315974/>), so it is certainly on the minds of Linux security developers. Casey Schaufler [sums up](<https://lwn.net/Articles/316947/>) the current state of affairs along with a look to a possible future: 

Stacking of special purpose LSMs would be a great idea. One reason that we don't have special purpose LSMs is that you can't stack them, you have to provide the entire "solution" in the one LSM. Of course, complete solutions don't stack. 

I would be very interested to see an LSM that does nothing but multiplex other LSMs. That would make multiple unrelated LSMs feasible without trying to create something that could deal with SELinux's and Smack's different notions of network access control model. You could revive the notion of loadable modules while you're at it. The LSM Multiplexer LSM could put any restrictions on the LSMs it is willing to support. 

It seems likely that someone will try to build an LSM-multiplexer before too long. In addition to snet, the [TuxGuardian](<http://tuxguardian.sourceforge.net/>) project appears to be reawakening after a period of quiet. It is similar to snet, and also uses LSM to trap network accesses. Other projects are also mentioned in the threads on linux-security-module. In the end, it is just too limiting to require that all security modules implement a full-scale security solution, and since LSM is the only accepted way to implement some of these hooks, some middle ground will likely be found. 

In another related [thread](<https://lwn.net/Articles/317022/>), Schaufler [notes](<https://lwn.net/Articles/317023/>) that a lot of what is being described for personal firewalls could be implemented using SELinux--at least as a starting point. One sticking point to that particular solution is the user interaction required. It is hard to see how an SELinux-derived solution could interact with the user for some decisions, but not others. It also is clearly outside of the scope of what SELinux is intended for. 

While snet may implement "bad security" in some minds, the discussion about it, especially with regard to LSM stacking has been very valuable. It may turn out that there is no sane way to stack arbitrary security modules in a way that a) makes sense and b) doesn't drive all of the security developers insane. But there are some reasonable use cases for that capability so it would seem that an investigation of those possibilities is warranted. With luck we will soon see where it leads.
