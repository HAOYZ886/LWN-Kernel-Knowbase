---
title: "Rethinking bpfilter and user-mode helpers"
url: https://lwn.net/Articles/822744/
date: "June 12, 2020"
category: "BPF-bpfilter; Modules-ELF modules; Networking-Packet filtering"
author: "By Jonathan Corbet June 12, 2020"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
June 12, 2020

The [bpfilter](<https://lwn.net/Articles/755919/>) subsystem, along with its "user-mode blobs" infrastructure, attracted a lot of attention when it was merged for the 4.18 kernel in 2018. Since then, however, development in this effort has been, to put it charitably, subdued. Now, two years after its merging, bpfilter may be in danger of being removed from the kernel as a failed experiment. 

#### Some history

Bpfilter is yet another in-kernel packet-filtering system; like netfilter and nftables, it exists for the creation of firewalls and related infrastructure. Rather than try to provide all of the filtering primitives that a user might want, bpfilter simply allows the loading of BPF programs to inspect and accept (or reject) packets. In theory, the benefits of bpfilter could be huge. It replaces a lot of firewalling code in the kernel, has the potential to be significantly faster than either of the other two mechanisms, and should be more flexible as well. It is not unreasonable to expect that something like bpfilter could eventually displace the other firewall subsystems and become the standard solution on Linux systems. 

For that to happen, though, the bpfilter developers would still have to do a lot more work. Beyond filling out the filtering functionality itself, they would need to find a way to avoid breaking the untold millions of systems out there that are currently using netfilter. Administrators of those systems have a lot of time invested in the creation of their firewall configurations and would not take kindly to the idea that they have to learn BPF and start over. Without seamless compatibility, bpfilter cannot take netfilter's place. 

The solution that the bpfilter developers chose was to reimplement the netfilter user-space API, so that the existing tools and configurations would continue to work. Internally to the kernel, though, netfilter rules would be translated into BPF programs, which would then be attached in the appropriate places. That solution should provide the best of all worlds: a shiny new packet-filtering subsystem with no changes required of users. 

The only problem with this idea is that compiling netfilter rules to BPF is not a small job; it would be a large chunk of code running in the kernel that would have to be prepared for any kind of weirdness that user space might present to it. The memory footprint of that code would not be welcome, and making sure that it was secure would be a difficult task that would likely end up generating a fair number of CVE entries before it was done. 

To avoid this problem, the "user-mode blob" concept was invented. Developers also often call it a "user-mode helper" even though that term has been used for many years in the kernel for a program that is run in user space at the kernel's request; this article will mirror that usage. A user-mode helper in this context is an executable program that is built into the kernel; when needed, it is run as a separate, user-space process that communicates with the kernel via pipes. For the translation of netfilter rules, this helper could read the rules provided by user space, do the translation in a setting where memory can be paged out and most bugs would not be exploitable, then write the result back to the kernel. 

When they were merged, bpfilter and user-mode helpers seemed like features that would have far-ranging effects. Greg Kroah-Hartman [called](<https://lwn.net/Articles/749117/>) user-mode helpers ""a very generic, new, user/kernel api that a whole bunch of people are going to want to use"", and some speculated that the Linux kernel was finally heading in the direction of a microkernel architecture. 

#### ...and then?

After the code was merged, though, things got quiet. Bpfilter has not seen any development work of note, and nothing in the kernel is actually making use of user-mode helpers. When a recent bug report failed to get a response from any of the developers of this code, Eric Biederman [looked a little deeper](<https://lwn.net/ml/linux-fsdevel/875zc4c86z.fsf_-_@x220.int.ebiederm.org/>) and didn't like what he saw: 

There have been no replies from the people who I expect would be maintainers of the code. When I look at the history of the code all it appears to have received since it was merged was trivial maintenance updates. There has been no apparent work to finish fleshing out the code to do what it is was aimed to do. 

He pointed out a number of perceived bugs in the code, and suggested that the most straightforward way to fix those bugs would be to just remove bpfilter and user-mode helpers. 

Proposing to remove somebody's code from the kernel is usually an effective way of getting their attention; it worked this time. Alexei Starovoitov was quick to [turn up](<https://lwn.net/ml/linux-fsdevel/20200606201956.rvfanoqkevjcptfl@ast-mbp/>) to block the change. Linus Torvalds also [showed up](<https://lwn.net/ml/linux-fsdevel/CAHk-=wi=rpNZMeubhq2un3rCMAiOL8A+FZpdPnwFLEY09XGgAQ@mail.gmail.com/>), pointing out that the code is, in fact, not used, and questioning the idea of the user-mode helpers in the first place: 

If people really convert netfilter rules to bpf, they'll likely do so in user space. This bpfilter thing hasn't gone anywhere, and it _has_ caused problems. 

So Alexei, I think the burden of proof is not on Eric, but on you. 

Starovoitov [responded](<https://lwn.net/ml/linux-fsdevel/20200607014935.vhd3scr4qmawq7no@ast-mbp/>) that the bpfilter effort has taken longer than expected; it was hampered by limitations in BPF that restricted the size of the rule sets that could be implemented. Those limitations are just now being overcome; the "BPF iterator" functionality (described in [this article](<https://lwn.net/Articles/818714/>)) that was merged for 5.8 is an important part of that; ""soon bpf will be able to handle millions of rules"". He later [added](<https://lwn.net/ml/linux-fsdevel/20200608162027.iyaqtnhrjtp3vos5@ast-mbp.dhcp.thefacebook.com/>) that he was not opposed to the removal of the bpfilter functionality for now; it can always be restored if and when progress is made in that area — something he said was not likely to happen for at least six months. 

But, he continued, the user-mode helpers feature is important and needs to stay in the kernel. Beyond the bpfilter use case, he mentioned a couple of others. One of those was described as an extension of `/proc`; in essence, it is using the BPF iterator mechanism to create new, `/proc`-like files at run time. Another is a form of initramfs that was built into the kernel and would set up various subsystems early in the boot process. Kroah-Hartman [said](<https://lwn.net/ml/linux-fsdevel/20200608164045.GC425601@kroah.com/>) that using user-mode helpers to implement USB drivers in user space is on his ""TODO for this year"", but made no promises as to when he might actually do that work. 

Part of the problem with the user-mode helper concept is that there is no example of its use at the moment. Since nobody is using it, bugs are not found, the code does not improve, and other developers have no example to work from. If some interesting functionality were to be implemented using user-mode helpers, the situation might change. If no such functionality shows up, instead, it might be a strong indication that this feature is not as useful as it seems. 

The discussion came to no conclusion on the question of whether this code should be removed or not. Experience shows that it is rare for code to be removed when there is a developer actively arguing for its retention. Even that will not keep unused code around indefinitely, though; unless some actual work is done on this code, it will eventually go away as an experiment that did not lead anywhere useful.
