---
title: "The kernel/user-space boundary"
url: https://lwn.net/Articles/571934/
date: "October 29, 2013"
category: "Development model-User-space ABI"
author: "By Jonathan Corbet October 29, 2013 2013 Kernel Summit"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
October 29, 2013

* * *

[2013 Kernel Summit](<https://lwn.net/Articles/KernelSummit2013/>)

H. Peter Anvin and Miklos Szeredi kicked off the invitation-only day of the 2013 Kernel Summit with a question: where, exactly, does the boundary between the kernel and user space lie? And, in particular, when is it possible to make an incompatible change to the kernel ABI with the understanding that the actual, supported ABI is provided by user-space code that is closely tied to the kernel? The answer they got was clear but, perhaps, not exactly what they wanted. 

Peter started by saying that we have a clear "don't break user space" policy. For the most part, living up to that policy is relatively straightforward; one avoids making incompatible changes to system calls and things continue to work. We are also getting better with other kernel interfaces like sysfs and `/proc`. But there was, he said, an [![\[Peter Anvin\]](https://static.lwn.net/images/conf/2013/lce-ks/PeterAnvin-sm.jpg)](<https://lwn.net/Articles/572004/>) interesting corner case last year: the GRUB2 bootloader was making a habit of looking at the kernel configuration files during installation for the setup of its menus. The restructuring of some internal kernel code broke GRUB2. At this point, Linus jumped in to claim that the kernel's configuration files do _not_ constitute a part of the kernel's ABI. When somebody does something that stupid, he said, one really cannot blame the kernel. 

Peter moved on to another problem, one he himself introduced some sixteen years ago. [The automounter ABI has issues](<https://lwn.net/Articles/494993/>) such that it failed to work with a 32-bit user space on a 64-bit kernel. A new type of pipe had to be introduced to fix this problem; it was, he said, an example of how far we are willing to go to avoid breaking applications. 

What about, he asked, cases where we need to shift to a new ABI altogether? Changes to the pseudo terminal (pty) interface are needed to get ptys to work from within containers; it's still not clear how to handle the master device in such situations. The control group interface is in flux, and there have been some disagreements with the systemd folks over who "owns" the container hierarchy as a whole. When it was suggested that systemd "wants to take over" control groups, Linus was quick to state that no such thing was going to happen. James Bottomley jumped in to note that the issue had been discussed and that a mutually acceptable solution was at hand. 

Another ABI issue is the current limitation, built into the Linux virtual filesystem layer, that no single I/O operation can transfer more than 2GB of data. As systems and memory sizes get larger, that limit may eventually hurt, he said, but Linus said that this limit would not be lifted. We are, he said, better than OS X, which causes overly large I/O requests to fail outright; Linux, instead, just transfers the maximum allowed amount of data. There are huge security implications to allowing larger I/O operations, to the point that there is no excuse for removing the limit. A whole lot of potential problems will simply be avoided if filesystem code just never sees 64-bit I/O request sizes. And, he said, if you try to do a 4GB write, "you're a moron." Such requests will not be any faster, there is just no reason to do it. 

In general, Linus said, he is fundamentally opposed to doing anything that might break user space; he was not sure why the topic was being discussed at all. The old issue of tracepoints came up, and Linus said that, if we break something with a tracepoint change, that is a problem and we will fix it. Greg Kroah-Hartman pointed out that some subsystem maintainers — himself and Al Viro, for example — are refusing to add tracepoints because they are afraid of being committed to supporting them forever. Others thought that this policy was excessively cautious, noting that actual problems with tracepoint ABI compatibility have been few and far between. No-tracepoints policies, Ingo Molnar said, are simply not justified. 

What about changes to `printk()` calls that break scripts that grep through the system logs? Linus answered that `printk()` is not special, and that problems there will be fixed as well. Masami Hiramatsu suggested that the sort of string-oriented data found in the logs is relatively easy to work with, and changes are easy to adapt to, but that hints that, perhaps, users are just coping with problems there rather than complaining about them. It would be interesting to see what would happen if a user were to actually complain about broken scripts resulting from a `printk()` change. Linus closed things off by complaining that the kernel developers have spent far more time worrying about this problem than they ever have dealing with actual issues. 

Miklos stepped up to ask more specifically: where is the boundary that sets the kernel ABI? Some parts of the operating system live in the kernel, [![\[Miklos Szeredi\]](https://static.lwn.net/images/conf/2013/lce-ks/MiklosSzeredi-sm.jpg)](<https://lwn.net/Articles/572009/>) while others can be found in libraries and the kernel. Sometimes things move; `rename()` was once in the C library, now it's a system call provided by the kernel. NFS servers have been found on both sides of the divide, graphics drivers have been moving into the kernel, sound drivers have moved in both directions, and filesystems can be found on both sides. 

Miklos may have been hoping for some sort of discussion of whether the interface between the kernel and some of these low-level components could be considered to be internal and not part of the kernel's ABI, but things didn't go in that direction. Instead, the discussion wandered a bit, covering whether parts of NetworkManager should be in the kernel (no, they would just have to call out to user space for authentication and such), drivers that require closed user-space components (still considered unwelcome), and the implementation of protocols like MTP, which, evidently, has more stuff in user space than should really be there. 

[Next: [Outreach Program for Women](<https://lwn.net/Articles/571935/>)]
