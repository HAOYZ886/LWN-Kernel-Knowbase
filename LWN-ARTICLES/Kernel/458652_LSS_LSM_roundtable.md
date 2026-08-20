---
title: "LSS: LSM roundtable"
url: https://lwn.net/Articles/458652/
date: "September 14, 2011"
category: "Security-Security modules"
author: "By Jake Edge September 14, 2011"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
September 14, 2011

A longstanding "flaw" (depending on who you talk to) in the Linux security module (LSM) subsystem is the inability to stack LSMs. That particular problem came up for discussion at the Linux Security Summit (LSS), which was held on September 8 in conjunction with the Linux Plumbers Conference. The participants were mostly in favor of stacking LSMs, the question was how to get there. 

Allowing administrators to enable more than one LSM at a time has been a [recurring problem](<https://lwn.net/Articles/368730/>). Some would like to be able to mix and match the protections offered by the different security solutions, but that is not currently possible. In addition, some specialized security functionality has been proposed at various times, but typically shunted toward an LSM-based solution. Unfortunately, in most distributions, the single LSM slot is already occupied by SELinux, AppArmor, or some other LSM, so separate LSMs with extra protections are of no use to many administrators. Thus the interest in stacking (or chaining) LSMs. 

Ubuntu security team member Kees Cook and Smack developer Casey Schaufler led the discussion, with Schaufler presenting a list of directions for LSM that he jokingly described as ""suggestions that we've come up with and think you should come up with too"". The list covered five items, but all were targeted at problems that stem from having multiple concurrent (i.e. stacked) LSMs. 

#### Stacking

David Howells [proposed](<https://lwn.net/Articles/426921/>) a possible solution for stacking LSMs back in February, but it won't allow two mandatory access control (MAC) LSMs (e.g. Smack and SELinux) to coexist. Schaufler has promised another, more general solution (which he calls "Glass"), but it still doesn't work for all four LSMs (SELinux, Smack, TOMOYO, and AppArmor) at once, though ""it's really close"". 

There is a question of why users would want to combine the existing LSMs, but Schaufler said that there are several combinations that people want to try. The two that are hardest to get working together (SELinux and Smack) turn out to be the ""only uninteresting combination"", he said. He has heard of users wanting to use Smack and AppArmor (or TOMOYO) at the same time, as well as folks that want one thing that SELinux can do along with something else that TOMOYO can do. 

As Cook pointed out, though, another likely scenario is that administrators will want to augment the distribution-provided security framework with additional restrictions that could come from a specialized LSM. Cook's [Yama](<https://lwn.net/Articles/393008/>) is one such solution. It restricts `ptrace()` and [symbolic links in "sticky" directories](<https://lwn.net/Articles/390323/>) in ways that many are in favor of, though it has not yet [made it into the mainline](<https://lwn.net/Articles/398607/>). For that use-case, the idea would be to not have to lose the distribution's LSM to add others like Yama. 

Howells's patches could support fairly simple scenarios like SELinux (or AppArmor) plus Yama, but Schaufler isn't convinced that restricting the combinations is the right way forward. There are, however, a bunch of technical problems that will need to be solved in order to do arbitrary stacking. Some are fairly straightforward to handle, like how to share the security context "blob" between multiple LSMs, but others are more difficult. 

If there is a stack of LSMs, what happens when one LSM chooses to deny access? If the other LSMs in the stack are bypassed because of that denial decision, they may get an incomplete picture of the accesses being requested. Howells's patch does short-circuit other LSMs that way, but at the time it was proposed Schaufler was concerned about LSMs that collect statistical information on accesses that would factor into subsequent access decisions. 

#### Security IDs

But the biggest problem area is with Security IDs (`secid`s). These are 32-bit identifiers used by LSMs (currently only SELinux and Smack) to identify security contexts when callbacks are made from other subsystems (notably audit and networking) for access decisions. But, if both are active, the `secid` space needs to be shared somehow. There are two ways to do that, Schaufler said, ""one isn't good, the other is painful"". Essentially, you could either split the `secid` into two 16-bit pieces (the not good choice), one for each LSM (which, of course, opens the question of what to do for three or more `secid`-using LSMs), or you could set up some kind of mapping where each LSM had its own `secid` space and those get mapped to a value in a shared space (the painful choice). 

Alternatively, Schaufler advocates getting rid of the `secid`s entirely and using security blob pointers everywhere. There is at least one major stumbling block to that plan, however, as getting a blob pointer into the networking code will be somewhere between difficult and impossible. Networking maintainer David Miller is adamantly opposed to putting such a pointer into the `sk_buff` structure, and he NAKed that when it was proposed earlier. There are some lifecycle management and performance issues that Miller is concerned about, according to Paul Moore. In fact, Moore is pretty confident that Miller hasn't changed his mind, as he offered a ""case of your favorite scotch"" to anyone who could convince Miller to add the pointer to `sk_buff`s. 

According to Schaufler, LSM stacking is clearly needed, particularly in the embedded space. In addition, without the ability to stack LSMs, people are becoming discouraged from writing new, more specialized LSMs. While Schaufler believes there are various access restrictions that can't be done using the existing LSMs, the SELinux folks (Stephen Smalley in particular) are not so sure. That said, though, Smalley is not opposed to something that would allow stacking Yama with SELinux (for example). Rather than trying to get a fully general stacking method into the mainline, Cook suggested that a ""trimmed-down approach"", along the lines of what Howells proposed, be tried instead. 

#### Other multi-LSM wrinkles

There are other things that need to be worked out in any multi-LSM scenario, including what to do about `/proc/PID/attr/current`. According to Schaufler (with the agreement of AppArmor maintainer John Johansen), the LSMs that came after SELinux made a mistake by reusing the `current` file to contain information on the security context of the process. Because stacking wasn't allowed, there was no real reason not to reuse that file, but now it could cause problems. 

One possibility is to include the name of the LSM in the path somewhere (e.g. `/proc/PID/attr/smack-current`), but that isn't a complete solution because existing user-space programs expect to find `current`. Cook suggested that whichever LSM gets loaded first gets `current` (in addition to its LSM-specific file). Or ""out of respect for our elders"", all LSMs could defer to SELinux for `current`, Schaufler said. 

That leads to a related problem: determining which LSM is active (or LSMs are active in a multi-LSM world). Currently, each LSM has its own _ad hoc_ method for user space to figure out whether it is running. Adding a `/sys/kernel/security/lsm` file with the names of any active LSMs in it would help. Any LSM that is ""actively enforcing policy"" (e.g. not SELinux in permissive or disabled modes) would add itself in the order in which the LSM was loaded. 

Another related problem is the lack of consistency in `/sys/kernel/security` that led to the suggestion of an "LSMKit" (which was greeted with laughter when Cook jokingly suggested it). There are a number of tools that display security context information (e.g. `ls -Z`) that will be confused in a multi-LSM world. Creating a library that would enumerate the active LSMs and gather up the relevant context information would simplify those tools, as well as providing some consistency of the kind of information that gets presented. Those gathered at the summit seemed favorably disposed toward that idea, though it is unclear if anyone will actually have the time to work on it. 

Schaufler noted that the general agreement about the need for LSM stacking was new. It is ""the first time no one has stood up and said 'This is an abomination'"", he said. But, Smalley said that didn't mean that he thinks it's a good idea either. Basically, he said that ""arbitrary composition [of security modules] is known to be a bad thing"", but that he recognizes some will still want to be able to do it. As long as full-fledged security frameworks like SELinux and AppArmor can live in ""their own separate worlds"", he is not opposed to having some way to compose LSMs. 

But, Smalley still thinks that there could be a single LSM that is used by everyone. Getting there is a matter of understanding all of the requirements that are being solved by various LSMs and incorporating them all into one. Schaufler is skeptical of that approach, and believes that it is ""beyond us technologically"" to fully understand all of the requirements that are or will be needed. Good solutions tend to come along periodically, he said, and we should have ways to accommodate them. 

#### Wrapping up

Debian currently only compiles one LSM (SELinux) into its kernel due to the memory that gets wasted by the unused code for inactive LSMs. Cook brought this issue up because he would like to see Debian kernels build in more LSMs and allow users to choose which to activate at boot time. It is a ""tiny"" amount of memory, according to Cook, but Debian is unwilling to add any more LSMs until there is some way to recover the lost memory. 

At first, there was concern that the idea was to return to the days where LSMs were actually kernel modules that could loaded and unloaded (which caused innumerable problems when the active LSM was unloaded). But Cook said all that was really needed was a way to unload all but the active LSM. As long as this unloading mechanism didn't touch the active LSM, and that the feature itself was optional, no one seemed to object to it. So it is mostly just a matter of someone finding the time to write the code. 

The fate of Yama was the last thing discussed in the LSM roundtable. The protections that it offers are valuable; several people in the room said they would enable it if it were in the mainline (and the stacking problem were solved). But, seemingly, no matter how Cook structures the code (in the core or as an LSM), it gets NAKed, partly because it does not represent a coherent security framework as the existing LSMs do. 

Part of the concern is that LSMs would become a "dumping ground" for various security fixes/enhancements that are not deemed acceptable for core kernel code. Smalley wanted to ensure that there was not a proliferation of small, specialized LSMs and would instead like to see Yama become the LSM for discretionary access control (DAC) enhancements. Any other proposals for those kinds of changes could be pushed toward Yama, rather than creating a whole new LSM. 

That idea seemed to gain some traction but, unless Yama gets into the mainline, it's a moot point. There was some discussion of Christoph Hellwig's NAK that kept Yama out the last time it came up. Smalley and others are not really convinced that his NAK is valid unless Yama touches VFS internals (which it doesn't). Cook pointed out that the `ptrace()` restrictions can't be done with any of the existing LSMs and that the symlink restrictions are ""provably correct"", but there is no path into the kernel that he's found. At this point, the plan seems to be to propose Yama again, perhaps as the "enhanced DAC" LSM, and to try to overcome any NAKs by better explaining the benefits Yama provides. The clear sense was that a more concerted effort would be made to get Yama into the mainline in the near future. 

[ I would like to thank all LWN subscribers for travel assistance to attend the security summit. ]
