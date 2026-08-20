---
title: "Xen: finishing the job"
url: https://lwn.net/Articles/321696/
date: "March 4, 2009"
category: "Virtualization-Xen; Xen"
author: "By Jonathan Corbet March 4, 2009"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
March 4, 2009

Once upon a time, Xen was the hot virtualization story. The Xen developers had a working solution for Linux - using free software - well ahead of anybody else, and Xen looked like the future of virtualization on Linux. Much venture capital chased after that story, and distributors raced to be the first to offer Xen-based virtualization. But, along the way, Xen seemed to get lost. The XenSource developers often showed little interest in getting their code into the mainline, and attempts by others to get that job done ran into no end of obstacles. So Xen stayed out of the mainline for years; the first public Xen release happened in 2003, but the core Xen code was only merged for 2.6.23 in October, 2007. 

In the mean time, KVM showed up and grabbed much of the attention. Its path into the mainline was almost blindingly fast, and many kernel developers were less than shy about expressing their preference for the KVM approach. More recently, Red Hat has made things more formal with its [announcement](<http://www.redhat.com/about/news/prarchive/2009/agenda.html>) of a "virtualization agenda" based on KVM. Meanwhile, [lguest](<http://lwn.net/Articles/218766/>) showed up as an easy introduction for those who want to play with virtualization code. 

The Xen story is a classic example of the reasons behind the "upstream first" policy, which states that code should be merged into the mainline before being shipped to customers. Distributors rushed to ship Xen, then found themselves supporting out-of-tree code which, often, was not well supported by its creators. In particular, published releases of Xen often only supported relatively old kernels, creating lots of work for distributors wanting to ship something more current. Now at least some of those distributors are moving on to other solutions, and high-level kernel developers are questioning whether, at this point, it's worth merging the remaining Xen code at all. 

All told, Xen looks to be on its last legs. Or, perhaps, the rumors of Xen's demise have been slightly exaggerated. 

The code in the mainline implements the Xen "DomU" concept - an unprivileged domain with no access to the hardware. A full Xen implementation requires more than that, though; there is the user-space hypervisor (which is GPL-licensed) and the kernel-based "Dom0" code. Dom0 is the first domain started by the hypervisor; it is typically run with more privileges than any other Xen guest. The purpose of Dom0 is to carefully hand out privileges to other Xen domains, providing access to hardware, network interfaces, etc. as set by administrative policy. Actual implementations of Xen must include the Dom0 code - currently a large body of out-of-tree kernel code. 

Jeremy Fitzhardinge would like to change that situation. So he has posted [a core Xen Dom0 patch set](<http://lwn.net/Articles/321298/>) with the goal of getting it merged into the 2.6.30 release. Among the review comments was [this question](<https://lwn.net/Articles/321699/>) from Andrew Morton: 

I hate to be the one to say it, but we should sit down and work out whether it is justifiable to merge any of this into Linux. I think it's still the case that the Xen technology is the "old" way and that the world is moving off in the "new" direction, KVM? 

In three years time, will we regret having merged this? 

The questions asked by Andrew were, essentially, (1) what code (beyond the current posting) is required to finish the job, and (2) is there really any reason to do that? The [answer](<https://lwn.net/Articles/321701/>) to the first question was ""another 2-3 similarly sized series to get everything so that you can boot dom0 out of the box"". Then there are various other bits which may not ever make it into the mainline. But, says Jeremy, getting the core into the mainline would shrink the out-of-tree patches carried by distributors and generally make life easier for everybody. For the second question, Jeremy responds: 

Despite all the noise made about kvm in kernel circles, Xen has a large and growing installed base. At the moment its all running on massive out-of-tree patches, which doesn't make anyone happy. It's best that it be in the mainline kernel. You know, like we argue for everything else. 

Beyond that, Jeremy is arguing that Xen still has a reason to exist. Its design differs significantly from that of KVM in a number of ways; see [this message](<https://lwn.net/Articles/321702/>) for an excellent description of those differences. As a result, Xen is useful in different situations. 

Some of the advantages claimed by Jeremy include: 

  * Xen's approach to page tables eliminates the need for shadow page tables or page table nesting in the guests; that, in turn, allows for significantly better performance for many workloads. 

  * The Xen hypervisor is lightweight, and can be run standalone; the KVM hypervisor is, instead, the Linux kernel. It seems that some vendors (HP and Dell are named) are shipping a Xen hypervisor in the firmware of many of their systems; that's the code behind the "instant on" feature, among other things. 

  * Xen's paravirtualization support allows it to work with hardware which does not support full virtualization. KVM, instead, needs hardware support. 

  * The separation between the hypervisor, Dom0, and DomU makes security validation easier. The separation between domains also allows for wild configurations with each device being driven by a separate domain; one might think of this kind of thing as a sort of heavyweight microkernel architecture. 

KVM's advantages, instead, take the form of relative simplicity, ease of use, full access to contemporary kernel features, etc. By Jeremy's reasoning, there is a place for both systems in Linux. 

The relative silence at the end of the discussion suggests that Jeremy has made his case fairly well. Mistakes may have been made in Xen's history, but it is a project which remains alive, and which has clear reasons to exist. Your editor predicts that the Dom0 code will find little opposition at the opening of the 2.6.30 merge window.
