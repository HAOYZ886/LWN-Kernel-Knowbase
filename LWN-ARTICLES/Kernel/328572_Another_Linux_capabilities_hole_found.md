---
title: Another Linux capabilities hole found
url: https://lwn.net/Articles/328572/
date: "April 15, 2009"
category: "Security-Vulnerabilities"
author: "By Jake Edge April 15, 2009"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
April 15, 2009

A recent [patch](<https://lwn.net/Articles/328590/>) posted to the linux-kernel mailing list fixes a long-standing flaw in the Linux capabilities implementation. The problem has existed since capabilities were added to the kernel during the 2.1 development series--more than ten years ago. One of the obvious questions is how a bug of that sort could have escaped notice for so long. 

The problem was [reported](<https://lwn.net/Articles/328594/>) in March by Igor Zhbanov, who provided an excellent analysis of the flaw and how it can be exploited. The basic problem lives in the VFS and NFS code which tries to drop privileges, by way of capabilities, before performing operations. The mask of capabilities bits that was used for that purpose does not include `CAP_MKNOD` (the ability to make a device node entry) or `CAP_LINUX_IMMUTABLE` (which allows changing the `S_APPEND` and `S_IMMUTABLE` file attributes). That means that those capabilities bits are not removed before the file operation is performed. 

Zhabanov shows that on a compromised client machine, the root user could give another user `CAP_MKNOD`, which would allow that user to run the `mknod` command and create a device entry owned by them. If this was done on an NFS-mounted filesystem, that entry would be created on the server still owned by the user. This works even if the `root_squash` option--essentially mapping root users on client machines to "nobody" on the server machine--was used on the export. 

If the user on the compromised machine can execute code on the server or any other client, they can directly access the device that underlies the device node entry. They will not require any special permissions on the other machines because the device node is owned by them. For example, creating the equivalent of `/dev/hda` on the server's filesystem might allow direct access to the hard disk block device on any system that had the NFS filesystem mounted. Uglier exploits can certainly be imagined. 

This is clearly a nasty problem. Linus Torvalds merged the fix for the recently released 2.6.30-rc2 kernel. One would guess the -stable tree folks won't be too far behind. Serge Hallyn also provided patches for 2.4 and 2.2 kernels, though the latter has become completely unsupported. 

The patch was greeted with a [question](<https://lwn.net/Articles/328604/>) from Valdis Kletnieks: ""Wow. How did this manage to stay un-noticed for this long?"" Torvalds had a characteristically blunt [answer](<https://lwn.net/Articles/328605/>): ""Because nobody uses capabilities?"" While that might explain how the bug went undetected for so long, it doesn't help alleviate the problem. Whether folks are using capabilities or not is irrelevant, the kernel itself certainly _is_. 

This is not the first time capabilities have been the source of a nasty, exploitable hole. The unfortunately-named "[sendmail-capabilities bug](<http://userweb.kernel.org/~morgan/sendmail-capabilities-war-story.html>)" provided a way to gain root privileges by exploiting the way `sendmail` dropped its privileges. The solution, when this bug was found in 2000, was to "cripple" capabilities in the kernel by disabling capability inheritance. That functionality was not restored until relatively recently. 

If distributions and other users were doing more with capabilities, it does seem likely that this particular problem would have been seen sometime in the last decade. But, by and large, Torvalds is right. For one thing, capabilities are a Linux-specific feature, so anyone writing portable code is likely to avoid using them. In addition, they are fairly difficult to wrap your head around; that complexity tends to lead folks to ignore capabilities. 

There have been some [efforts](<https://lwn.net/Articles/313838/>) at using capabilities in distributions more, but one has to wonder how many more exploits still lurk in that code. It is hard to imagine removing capabilities at this late date--it is a user-space interface from the kernel after all--but some must be wondering if the feature is worth all the trouble it has caused.
