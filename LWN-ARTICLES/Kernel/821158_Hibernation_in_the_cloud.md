---
title: Hibernation in the cloud
url: https://lwn.net/Articles/821158/
date: "May 25, 2020"
category: Software suspend
author: "By Jonathan Corbet May 25, 2020 OSPM"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
May 25, 2020

* * *

[OSPM](<https://lwn.net/Articles/820337/>)

Hibernation is normally thought of as a laptop feature — and an old and obsolete laptop feature at that. One does not normally consider it to be relevant in cloud settings. But, at the 2020 [Power Management and Scheduling in the Linux Kernel summit](<http://retis.sssup.it/ospm-summit/>) (OSPM), Andrea Righi argued that there may actually be a place for hibernation on cloud-based systems if it can be made to work reliably. 

The core idea behind hibernation is powering the system down entirely, but restoring it to its previous state, including any running processes, when the system is powered up. To do that, the contents of memory, which will not survive loss of power, must be written to persistent storage before turning things off. The advantage of hibernation is that the system can retain its state indefinitely without power; the cost is a great deal of I/O at both hibernation and resume times. 

Hibernation was a hot topic back in 2004, when it was usually known as "software suspend"; see [the LWN kernel index entry for software suspend](<https://lwn.net/Kernel/Index/#Software_suspend>) to understand just how hot. Work in this area slowed around 2008, though, when suspend-to-RAM functionality (often just called "suspend") became widely available. Support for hibernation was dropped entirely in Ubuntu 12.04. The Fedora 29 release included an experiment with suspend-then-hibernate functionality, but that "didn't go well" and was dropped. Hibernation mostly seems like a dead topic, he said. 

So it is interesting that Amazon [added support for hibernating EC2 instances](<https://aws.amazon.com/blogs/aws/new-hibernate-your-ec2-instances/>) at the end of 2018. Hibernation has suddenly arrived in the cloud, which is a rather different use case than has been seen before. The value there is to be able to pause a workload to save money. For example, Amazon's "spot instances" run at low priority when there are spare resources available; they can be shut down with ten minutes notice at any time. That is "not nice", but "you get what you pay for". This is a setting where hibernation can help; rather than just losing state when the instance is shut down, it can hibernate and resume working when resources are again available. 

#### How it works

Hibernation works by writing memory contents to a "hibernation image" on disk; that image is somewhat smaller than the RAM in the system. Data can be compressed on its way to the image, and recoverable pages (clean, file-backed data in the page cache primarily) can simply be dropped. Rafael Wysocki added that hibernation was designed with the assumption that the bulk of user data will be swapped out at any given time; the amount that is left will be less than 50% of RAM. On the next boot, Righi continued, the kernel will look at the specified resume image; if a valid signature is found, the image will be restored back into memory. Then some tricky, architecture-specific code jumps back into the old kernel state and the system resumes where it left off. 

The biggest issue with hibernation, he said, is whether it is reliable. That can't always be counted on; any device in the system can prevent hibernation if something isn't in the right state. That is not a huge problem for hibernation itself, since no data is lost, though it can be an issue if you are relying on it working. But any device can mess up the resume process as well, and that is a much bigger problem. It is also possible for the kernel to run out of memory while hibernating or resuming, which will kill the whole thing. 

Beyond that, there are still bugs present in this code, despite its long history; Righi mention one that [was fixed](<https://lore.kernel.org/linux-pm/2167643.HFCj9E3NaD@kreacher/t/>) in late 2019. There are security implications, since the hibernation image holds sensitive data in persistent storage. Memory and disk speed can be a problem; he dealt with one customer who was reporting that hibernation was timing out; it turns out that they were running on an [![\[Andrea Righi\]](https://static.lwn.net/images/conf/2020/ospm/AndreaRighi-sm.png)](<https://lwn.net/Articles/821160/>) instance with slower storage, and that the timeout period had not been wisely chosen. It is also possible that the memory needing to be saved won't fit into the hibernation image. 

Debugging hibernation problems is a special challenge in any setting, he said, and it can be worse in cloud settings, especially if you do not have access to the hypervisor. 

Improving the reliability of hibernation depends a lot on better hardware support. Here cloud settings may have an advantage, because the "hardware" tends to be uniform regardless of where an instance is running. It can be helpful to reduce memory usage when the image is being stored, which argues against the use of stacked block devices; kernel code should avoid large allocations in the hibernation and resume paths. 

Performance (as measured in hibernation time) can be improved by decreasing the size of the hibernation image; that can be done by tweaking `/sys/power/image_size`. A smaller size will cause more recoverable memory to be dropped, cutting down on the amount of I/O required at the cost of colder caches on resume. A larger image size has the opposite effect; hibernation takes longer, but the system will run faster after it resumes. 

Then, there is the trick of running `swapoff` after resuming the system as a way of forcing all data from the swap area back into RAM. It can reduce the time required for the system to stop paging and get back to full speed. But using `swapoff` turns out to be slow because the swap code does not properly use readahead when bringing the data back into RAM. There is [a fix for this problem](<https://lwn.net/ml/linux-kernel/20200418084705.GA147642@xps-13/>) in linux-next now. Wysocki said that the kernel could just do this swapping-in at resume time automatically, which would be a nicer solution to the problem. 

For the future, Righi said, the kernel could perform opportunistic swapping during idle times; that would put more data into persistent storage and speed up hibernation. He has tried some hacks in this area; they work, but he would like a better solution. In conclusion, he said, hibernation can bring some real benefits for cloud-based systems, but the reliability issues need to be addressed first. 

#### Rafael responds

Once Righi finished, Wysocki essentially took over with a talk of his own, saying that he wanted to respond to a few of Righi's points. He agreed that hibernation is not used much currently, but he uses it himself for desktop systems that don't have a battery. He has not seen a single failure since 2016. That said, the whole system was designed around the assumption that hibernation and resume would happen on the same machine; it's surprising that it works for cloud instances at all. 

He acknowledged that there were plenty of nasty bugs in x86 hibernation support, but most of those were fixed in 2016. Support in the architecture code is solid, but there are still problems with some drivers. Most drivers support suspend these days, though, and the hibernation support generally derives from that, so device-level support for hibernation is broad. Most laptops he has tried work out of the box without problems, though he admitted he hasn't done huge amounts of testing. 

He repeated that there could be problems where the resume happens on a different machine from where hibernation took place. Given the hardware emulation provided by the hypervisor, the system should be essentially the same, but he stressed that there are "no warranties". 

The real problem with hibernation support is that it places huge stresses on the memory-management subsystem. It forces data out to swap by allocating all of the memory in the system, with the out-of-memory killer disabled. Strange things can happen when you do that but, from the point of view of the memory-management developers, it's an obscure corner case and they never find the time to improve it. 

With regard to security, he said, if a cloud provider makes hibernation available, it's up to them to take care of encrypting the hibernation image and such. Attempts to add encryption support to the kernel have run afoul of "security people", who didn't like the duplication of functionality. Somehow, the key used to encrypt the image has to be passed to the resume kernel, which is not easy. So there are some challenges to face there. 

On that note, the session wound down, and the 2020 edition of OSPM came to a close.
