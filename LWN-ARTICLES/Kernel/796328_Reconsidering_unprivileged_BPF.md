---
title: Reconsidering unprivileged BPF
url: https://lwn.net/Articles/796328/
date: "August 16, 2019"
category: "BPF-Unprivileged"
author: "By Jonathan Corbet August 16, 2019"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
August 16, 2019

The BPF virtual machine within the kernel has seen a great deal of work over the last few years; as that has happened, its use has expanded to many different kernel subsystems. One of the objectives of that work in the past has been to make it safe to allow unprivileged users to load at least some types of BPF programs into the kernel. A recent discussion has made it clear, though, that the goal of opening up BPF to unprivileged users has been abandoned as unachievable, and that further work in that direction will not be accepted by the BPF maintainer. 

The BPF verifier goes to great lengths to ensure that any BPF program presented to the kernel is safe to run. Memory accesses are checked, execution is simulated to ensure that the program will terminate in a bounded period of time, and so on. Many of these checks are useful to ensure that all programs are safe and free of certain types of bugs, but others are aimed specifically at containing a potentially hostile program — an obvious necessity if the kernel is to accept BPF programs from unprivileged users. 

Much of this work was [done in 2015](<https://lwn.net/Articles/660331/>) for the 4.4 kernel; in particular, a great deal of effort went into preventing BPF programs from leaking kernel pointer values to user space. Those pointers could be highly useful to an attacker who is trying to figure out where specific data structures or code are to be found on a target system, so making them easily available to unprivileged processes is clearly a bad idea. "[Constant blinding](<https://git.kernel.org/linus/4f3446bb809f>)" was added for 4.7. In essence, this mechanism will exclusive-OR constant values in programs with a random number (repeating the operation at run time when the values are actually used), preventing an attacker from sneaking in unverified BPF code disguised as constants. Other patches have been aimed at preventing speculative-execution attacks by BPF programs. 

After all that work, though, there is still only one place where unprivileged users can, administrator willing, load BPF programs: as filters on open sockets. In 2015, BPF maintainer Alexei Starovoitov [declared](<https://lwn.net/Articles/660080/>) that ""I think it is time to liberate eBPF from CAP_SYS_ADMIN"". Nearly four years later, that has not happened, and the work that has been done more recently has been focused instead on giving administrators more control over who can load BPF programs; see the (unmerged) [`/dev/bpf` effort](<https://lwn.net/Articles/792124/>) for one example. 

While Starovoitov has stopped working on unprivileged BPF, others have still been putting some thought in that direction. Andy Lutomirski recently [posted a set of patches](<https://lwn.net/ml/linux-kernel/cover.1565040372.git.luto@kernel.org/>) intended to make BPF a bit more suitable for this use case. It implements access permissions for BPF maps, adds a way to mark specific BPF functions as requiring privilege, and allows the loading of all types of programs by unprivileged users. ""This doesn't let you *run* the programs except in test mode, so it should be safe. Famous last words."" These patches have received no comments. 

In the ongoing discussion about the `/dev/bpf` work, though, Starovoitov made the perhaps surprising [statement](<https://lwn.net/ml/netdev/20190806011134.p5baub5l3t5fkmou@ast-mbp/>) that ""unprivileged bpf is actually something that can be deprecated"". Lutomirski, unsurprisingly, [didn't like that idea](<https://lwn.net/ml/netdev/CALCETrXEHL3+NAY6P6vUj7Pvd9ZpZsYC6VCLXOaNxb90a_POGw@mail.gmail.com/>): 

I hope not. There are a couple setsockopt uses right now, and and seccomp will surely want it someday. And the bpf-inside-container use case really is unprivileged bpf -- containers are, in many (most?) cases, explicitly not trusted by the host. 

Starovoitov [responded](<https://lwn.net/ml/netdev/20190813215823.3sfbakzzjjykyng2@ast-mbp/>) that ""Linux has become a single-user system"" where anybody who can run any code at all can break out of containment and obtain root privileges. The whole idea of unprivileged BPF, he said, has been a mistake: 

When we say 'unprivileged bpf' we really mean arbitrary malicious bpf program. It's been a constant source of pain. The constant blinding, randomization, verifier speculative analysis, all spectre v1, v2, v4 mitigations are simply not worth it. It's a lot of complex kernel code without users. There is not a single use case to allow arbitrary malicious bpf program to be loaded and executed. 

Lutomirski [responded](<https://lwn.net/ml/netdev/CALCETrUkqUprujww26VxHwkdXQ3DWJH8nnL2VBYpK2EU0oX_YA@mail.gmail.com/>) (more than once) that some use cases do exist. He mentioned `seccomp()`, which still uses the old "classic BPF" language rather than the "extended BPF" that has been the target of development work in recent years; there are developers now who would like to have extended BPF features available in `seccomp()` filters. Per-user systemd instances are another example; systemd makes use of BPF now and could benefit from making that functionality available to unprivileged users as well. There might well be others if the kernel were able to support them, he said: ""There aren't major unprivileged eBPF users because the kernel support isn't there"". 

Starovoitov [made it clear](<https://lwn.net/ml/netdev/20190814220545.co5pucyo5jk3weiv@ast-mbp.dhcp.thefacebook.com/>) that he was not impressed, though: ""I'm afraid these proposals won't go anywhere"". He reiterated his claim that there are no known use cases for unprivileged BPF. What he would like to see, instead, is "less privileged BPF" where, for example, a process could be given a new `CAP_BPF` capability (or access to a `/dev/bpf` file) that would allow the loading of BPF programs without opening the door to other privileged operations. That, he said, would improve the safety of applications that actually exist without the need to expend effort supporting the unprivileged use case which, he claims, does not exist. 

And that is the impasse at which the conversation stands now. At its core, it's a fundamental difference of opinion over whether a Linux system can ever be truly hardened against an unprivileged user. If the answer is "no", then there is little point in maintaining a lot of complex code in the BPF subsystem to try to effect that hardening. Accepting that answer, though, is tantamount to saying that the Linux privilege model just doesn't work in the end: the combination of software bugs and hardware vulnerabilities will always undermine it, so we might as well just give up. That would be a discouraging conclusion to say the least.
