---
title: Kernel address randomization
url: https://lwn.net/Articles/444503/
date: "May 24, 2011"
category: "Security-Kernel hardening"
author: "By Jonathan Corbet May 24, 2011"
---

By **Jonathan Corbet**  
May 24, 2011

Last week's Kernel Page included [a brief item](<https://lwn.net/Articles/443379/>) about the hiding of kernel addresses from user space. This hiding has come under fire from a number of developers who say that it breaks things (perf, for example) and that it does not provide any real additional security. That said, there does seem to be a consensus around the idea that it's better if attackers don't know where the kernel keeps its data structures. As it turns out, there might be a better way to do that than simply hiding pointer values. 

There is no doubt that having access to the layout of the kernel in memory is useful to attackers. As Dan Rosenberg [put it](<https://lwn.net/Articles/444511/>): 

I agree about the fact that kptr_restrict is an incomplete security feature. However, I disagree that it lacks usefulness entirely. Virtually every public kernel exploit in the past year leverages /proc/kallsyms or other kernel address leakage to target an attack. 

The hiding of kernel addresses is meant to deprive attackers of that extra information, making their task harder. One big problem with that approach is that most systems out there are running stock distribution kernels. Getting the needed address information from the distributor's kernel package is not a particularly challenging task. So, on these systems, there is no real mystery about the layout of the kernel, regardless of whether pointer values are allowed to leak to user space or not. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

While all of this was being discussed, another idea came out: why not randomize the location of the kernel in memory at boot time? Address space layout randomization has been used to resist canned attacks for a long time, but the kernel takes no such measure for itself. Given that the kernel image is relocatable, there is no real reason why it always needs to be loaded at the same address. If the kernel calculated a different offset for itself at every boot, it could subtract that offset from pointer values before passing them to user space. Those pointers could then be used by tools like perf, but they would no longer be helpful for anybody seeking to overwrite kernel data structures. 

Dan has been looking into kernel-space randomization with [some success](<https://lwn.net/Articles/444556/>); it turns out that simply relocating the kernel is not that hard. That said, he has run up against a few potential problems. The first of those is that there is very little entropy available at the beginning of the boot process, so the generation of a sufficiently random base address for the kernel is not entirely straightforward. It seems that enough bits of entropy can be derived from the real-time clock and time stamp counter to make it hard for an attacker to derive the base address later on, but a real random number would be better. 

Next, as [Linus pointed out](<https://lwn.net/Articles/444513/>), the kernel is not infinitely relocatable. There are a number of alignment requirements which constrain the kernel's placement, so, according to Linus, there is a maximum of 8-12 bits of randomization available. That means that an exploit attempt could find the right offset after a maximum of a few thousand tries. Given that computers can try things very quickly, that does not give a site administrator much time to respond. 

As others pointed out, though, that amount of randomness is probably enough. Failed exploit attempts have a high probability of generating a kernel oops; even if an administrator does not notice the oops immediately, it should come to their attention at some point. So the ability to stealthily take over a system is gone. Beyond that, failed exploits may well take down the system entirely (especially if, as is the case with many RHEL systems, the "panic on oops" flag is set) or leave it in a state where further exploit attempts cannot work. There is, it seems, a real advantage to forcing an attacker to guess. 

That advantage evaporates, though, if an attacker can somehow figure out what offset a given system used at boot time. Dan noticed one way that could happen: the unprivileged SIDT instruction can be used to locate the system's interrupt descriptor table. That location could, in turn, be used to calculate the kernel's base offset. Dynamic allocation of the table can solve that problem at the cost of messing with some tricky very-early-boot code. There could be other advantages to dynamically allocating the table, though; if the table were put into the per-CPU area, it might make the system a little more scalable. 

So this problem can be solved, but there will, beyond doubt, be other places where it will be possible for an attacker to obtain a real kernel-space address. There are simply too many ways in which that information might leak into user space. Plugging all of those leaks looks like one of those long-term tasks that is never really done. It may, however, be possible to get close enough to done that attackers will not be able to count on knowing the true location of the kernel in a running system. That may be a bit of security through obscurity that is worth having.
