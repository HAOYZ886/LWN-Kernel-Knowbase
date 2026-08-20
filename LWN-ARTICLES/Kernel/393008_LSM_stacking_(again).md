---
title: LSM stacking (again)
url: https://lwn.net/Articles/393008/
date: "June 23, 2010"
category: "Security-Security modules"
author: "By Jake Edge June 23, 2010"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
June 23, 2010

Kees Cook is back with another [proposal](<https://lwn.net/Articles/393010/>) for a kernel change that would, at least in his mind, provide more security, this time by restricting the `ptrace()` system call. But, like his earlier [symbolic link patch](<http://lwn.net/Articles/390323/>), this one is not being particularly well-received on linux-kernel. It has, however, sparked some discussion of a topic that seems to recur with some frequency in that venue: stacking Linux security modules (LSMs). 

Cook's patch is fairly straightforward; it creates a sysctl called `ptrace_scope` that defaults to zero, which chooses the existing behavior. If it is set to one, though, it only allows `ptrace()` to be called on descendants of the tracing process. The idea is to stop a vulnerability in one program (Pidgin, say) from being used to trace another program (like Firefox or GPG-agent), which would allow extracting credentials or other sensitive information. Like the previous symlink patch, it is based on a patch that has long been in the [grsecurity](<http://www.grsecurity.net/>) kernel. 

As with the previous proposal, Alan Cox was quick to [suggest](<https://lwn.net/Articles/393015/>) that it be put into an LSM: 

So NAK. If you want to use bits of grsecurity then please just write yourselves a grsecurity kernel module that uses the security hooks properly and stop messing up the core code. It's all really quite simple, the [infrastructure] is there, so use it. 

But, one problem with that plan is that LSMs do not stack. One can have SELinux, Smack, and TOMOYO enabled in a kernel, but only one--chosen at boot time--can be active. There have been [discussions](<http://lwn.net/Articles/368730/>) and [proposals](<http://lwn.net/Articles/114588/>) for LSM stacking (or chaining) along the way, but nothing has ever been merged. So, two "specialized" LSMs cannot do their separate jobs in the kernel and users will have to choose between them. 

For "full-featured" solutions, like SELinux, that isn't really a problem, as users can find or create policies to handle their security requirements. In addition, James Morris [points out](<https://lwn.net/Articles/393018/>) that SELinux has a boolean, `allow_ptrace`, to do what Cook is trying to do: ""You don't need to write any policy, just set it [allow_ptrace] to 0"". But, for those that don't want to use SELinux, that's no solution. As Ted Ts'o [puts it](<https://lwn.net/Articles/393130/>): 

i think we really need to have stacked LSM's, because there is a large set of people who will never use SELinux. Every few years, I take another look at SELinux, my head explodes with the (IMHO unneeded complexity), and I go away again... 

Yet I would really like a number of features such as this ptrace scope idea --- which I think is a useful feature, and it may be that stacking is the only way we can resolve this debate. The SELinux people will never believe that their system is too complicated, and I don't like using things that are impossible for me to understand or configure, and that doesn't seem likely to change anytime in the near future. 

Others were also favorably disposed toward combining LSMs, though the consensus seems to be for chaining LSMs in the security core rather than stacking, as was done with SELinux and Linux capabilities (i.e. `security/commoncap.c`). In the stacking model, each LSM is responsible for calling out to any other secondary LSMs for each security operation, whereas chaining is ""just a walk over a list of security_operations"" calling each LSM's version from the core, as Eric W. Biederman [described](<https://lwn.net/Articles/393133/>). But it's not as easy as it might seem at first glance, as Serge E. Hallyn, who proposed a stacking mechanism in 2004, [points out](<https://lwn.net/Articles/393147/>): 

The general answer tends to be "generic stacking doesn't work, LSMs need to know about each other." But even for that (as evidenced by the selinux+commoncap experience with stacking) is hairy, and more to the point it probably does not scale when we have 5-10 small LSMs. I.e. LSM 1 wants to prevent some action while LSM 2 requires that action to succeed so that it can properly prevent another action. Concrete examples are buried in the stacker discussions on the lsm list from 2004-2005. 

It seems that there may be some discussion of LSM stacking/chaining at the [Linux security summit](<https://security.wiki.kernel.org/index.php/LinuxSecuritySummit2010>), as part of Cook's presentation on "widely used, but out-of-tree" security solutions, but perhaps also in a "beer BOF" that Hallyn is proposing. 

The way forward for both of Cook's recent proposals looks to be as an LSM and, to that end, he has posted the [Yama LSM](<http://lwn.net/Articles/393012/>), which incorporates the symlink protections and `ptrace()` limitations that he previously posted. In addition, it adds the ability to restrict hard links such that they cannot be created for files that are either sensitive (e.g. setuid) or those that are not readable and writable by the link creator. Each of these measures can be enabled separately by sysctls in `/proc/sys/kernel/yama/`. 

While "Yama" might make one start looking for completions of an acronym ("Yet Another ..."), it is actually named for a [deity](<http://en.wikipedia.org/wiki/Yama>): ""Yama is roughly the 'lord of death/underworld' in Buddhist and Hindu tradition, kind of over-seeing the rehabilitation of impure entities"", Cook said. Given the number of NAKs that his recent patch proposals have received, calling Yama the ""NAKed Access Control system"", shows a bit of a sense of humor about the situation. DAC, MAC, RBAC, and others would now be joined by NAC if Yama gets merged. 

So far, discussion of Yama has been fairly light, and without any major complaints. While some are rather skeptical of the protections that Cook has been proposing, they are much less likely to care if they live in an LSM, rather than ""junk randomly spewed [all] over the tree"", as Cox [put it](<https://lwn.net/Articles/393156/>). 

Once these simpler security tasks are encapsulated into an LSM, Morris [said](<https://lwn.net/Articles/393159/>), the kernel hackers ""can evaluate the possible need for some form of stacking or a security library API"" to allow these measures to coexist with SELinux, AppArmor, and others. Given the fairly broad support for the LSM approach, it would seem that Yama, or some descendant, could make it into the mainline. Whether that translates to some kind of general mechanism for combining LSMs in interesting ways remains to be seen--it should be worth watching, stay tuned.
