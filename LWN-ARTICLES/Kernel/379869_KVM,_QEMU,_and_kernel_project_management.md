---
title: "KVM, QEMU, and kernel project management"
url: https://lwn.net/Articles/379869/
date: "March 23, 2010"
category: "Development model; KVM; QEMU; Virtualization-KVM"
author: "By Jonathan Corbet March 23, 2010"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
March 23, 2010

The KVM virtualization subsystem is seen as one of the great success stories of contemporary kernel development. KVM came from nowhere into a situation with a number of established players - both free and proprietary \- and promptly found a home in the kernel and in the marketing plans of a number of Linux companies. Both the code and its development model are seen as conforming much more closely to the Linux way of doing things than the alternatives; KVM is expected to be _the_ long-term virtualization solution for Linux. So, one might well wonder, why has KVM been the topic of one of the more massive and less pleasant linux-kernel discussions in some time? 

Yanmin Zhang was probably not expecting to set off a flame war with the posting of [a patch](<http://lwn.net/Articles/379500/>) adding a set of KVM-related commands to the "perf" tool. The value of this patch seems obvious: beyond allowing a host to collect performance statistics on a running guest, it enables the profiling of the host/guest combination as a whole. One can imagine that there would be value to being able to see how the two systems interact. 

The problem, it seems, is that this feature requires that the host have access to specific information from the running KVM guest: at a minimum, it needs the guest kernel's symbol table. More involved profiling will require access to files in the guest's namespaces. To this end, Ingo Molnar [suggested](<https://lwn.net/Articles/379870/>) that life would be easier if the host could mount (read-only) all of the filesystems which were active in the guest. It would also be nice, he said elsewhere, if the host could easily enumerate running guests and assign names to them. 

The [response](<https://lwn.net/Articles/379871/>) he got was "no way." Various security issues were raised, despite the fact that the filesystems on the host would not be world-readable, and despite the fact that, in the end, the host has total control over the guest anyway. Certainly there are some interesting questions, especially when frameworks like SELinux are thrown into the mix. But Ingo took that answer as a statement of unwillingness to cooperate with other developers to improve the usability of KVM, especially on developers' desktop systems. What followed was a sometimes acrimonious and often repetitive discussion between Ingo and KVM developer Avi Kivity, with a small group of supporting actors on both sides. 

Ingo's position is that any development project, to be successful, must make life easy for users who contribute code. So, he says, the system should be most friendly toward developers who want to run KVM on their desktop. Beyond that, he [claims](<https://lwn.net/Articles/379872/>) that a stronger desktop orientation is crucial to our long-term success in general: 

I.e. the kernel can very much improve quality all across the board by providing a sane default (in the ext3 case) - or, as in the case of perf, by providing a sane 'baseline' tooling. It should do the same for KVM as well. 

If we don't do that, Linux will eventually stop mattering on the desktop - and some time after that, it will vanish from the server space as well. Then, may it be a decade down the line, you won't have a KVM hacking job left, and you won't know where all those forces eliminating your project came from. 

Avi, needless to say, [sees things differently](<https://lwn.net/Articles/379873/>): 

It's a fact that virtualization is happening in the data center, not on the desktop. You think a kvm GUI can become a killer application? fine, write one. You don't need any consent from me as kvm maintainer (if patches are needed to kvm that improve the desktop experience, I'll accept them, though they'll have to pass my unreasonable microkernelish filters). If you're right then the desktop kvm GUI will be a huge hit with zillions of developers and people will drop Windows and switch to Linux just to use it. 

But my opinion is that it will end up like virtualbox, a nice app that you can use to run Windows-on-Linux, but is not all that useful. 

Ingo's argument is not necessarily that users will flock to the platform, though; what seems to be important is attracting developers. A KVM which is easier to work with should inspire developers to work with it, improving its quality further. Anthony Liguori, though, [points out](<https://lwn.net/Articles/379876/>) that the much nicer desktop experience provided by VirtualBox has not yet brought in a flood of developers to fix its performance problems. 

Another thing that Ingo is unhappy with is the slow pace of improvement, especially with regard to the [QEMU](<http://wiki.qemu.org/Main_Page>) emulator used to provide a full system environment for guest systems. A big part of the problem, he says, is the separation between the KVM and QEMU, despite the fact that they are fairly tightly-coupled components. Ingo [claimed](<https://lwn.net/Articles/379884/>) that this separation is exactly the sort of problem which brought down Xen, and that the solution is to pull QEMU into the kernel source tree: 

If you want to jump to the next level of technological quality you need to fix this attitude and you need to go back to the design roots of KVM. Concentrate on Qemu (as that is the weakest link now), make it a first class member of the KVM repo and simplify your development model by having a single repo. 

From Ingo's point of view, such a move makes perfect sense. KVM is the biggest user of the QEMU project which, he says, was dying before KVM came along. Bundling the two components would allow ABI work to be done simultaneously on both sides of the interface, with simultaneous release dates. Kernel and user-space developers would be empowered to improve the code on both sides of the boundary. Bringing perf into the kernel tree, he [says](<https://lwn.net/Articles/379888/>), grew the external developer community from one to over 60 in less than one year. Indeed, integration into the kernel tree [is the reason](<https://lwn.net/Articles/379889/>) why perf has been successful: 

If you are interested in the first-hand experience of the people who are doing the perf work then here it is: by far the biggest reason for perf success and perf usability is the integration of the user-space tooling with the kernel-space bits, into a single repository and project. 

Clearly, Ingo believes that integrating QEMU into the kernel tree would have similar effects there. Just as clearly, the KVM and QEMU developers disagree. To them, this proposal looks like a plan to fork QEMU development - though, it should be said, KVM already uses a forked version of QEMU. This fork, [Avi says](<https://lwn.net/Articles/379891/>), is ""definitely hurting."" [According to Anthony](<https://lwn.net/Articles/379892/>), moving QEMU into the kernel tree would widen that fork: 

We lose a huge amount of users and contributors if we put QEMU in the Linux kernel. As I said earlier, a huge number of our contributions come from people not using KVM. 

The KVM/QEMU developers are unconvinced that they will get more developers by moving the code into the kernel tree, and they seem frankly amused by the notion that kernel developers might somehow produce a more desktop-oriented KVM. They see the separation of the projects as not being a problem, and wonder where the line would be drawn; Avi suggested that the list of projects which _don't_ belong in the kernel might be shorter in the end. In summary, they see a system which does not appear to be broken - QEMU is said to be improving quickly - and that "fixing" it by merging repositories is not warranted. 

Particular exception was taken to Ingo's assertion that a single repository allows for quicker and better development of the ABI between the components. Slower, [says Zachary Amsden](<https://lwn.net/Articles/379893/>), tends to be better in these situations: 

This is actually a Good Thing (tm). It means you have to get your feature and its interfaces well defined and able to version forwards and backwards independently from each other. And that introduces some complexity and time and testing, but in the end it's what you want. You don't introduce a requirement to have the feature, but take advantage of it if it is there. 

Ingo, though, [sees things differently](<https://lwn.net/Articles/379894/>) based on his experience over time: 

It didn't work, trust me - and i've been around long enough to have suffered through the whole 2.5.x misery. Some of our worst ABIs come from that cycle as well... And you can also see the countless examples of carefully drafted, well thought out, committee written computer standards that were honed for years, which are not worth the paper they are written on. 

'extra time' and 'extra bureaucratic overhead to think things through' is about the worst thing you can inject into a development process. 

As the discussion wound down, it seemed clear that neither side had made much progress in convincing the other of anything. That means that the status quo will prevail; if the KVM maintainers are not interested in making a change, the rest of the community will be hard-put to override them. Such things have happened - the x86 and x86-64 merger is a classic example - but to override a maintainer in that way requires a degree of consensus in the community which does not appear to be present here. Either that, or a decree from Linus - and he has been silent in this debate. 

So the end result [looks like this](<https://lwn.net/Articles/379895/>): 

Please consider 'perf kvm' scrapped indefinitely, due to lack of robust KVM instrumentation features: due to lack of robust+universal vcpu/guest enumeration and due to lack of robust+universal symbol access on the KVM side. It was a really promising feature IMO and i invested two days of arguments into it trying to find a workable solution, but it was not to be. 

Whether that's really the end for "perf kvm" remains to be seen; it's a clearly useful feature that may yet find a way to get into the kernel. But this disconnect between the KVM developers and the perf developers is a clear roadblock in the way of getting this sort of feature merged for now.
