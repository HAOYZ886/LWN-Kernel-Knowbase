---
title: One more pidfdfs surprise
url: https://lwn.net/Articles/976125/
date: "May 31, 2024"
category: "Development model-User-space ABI; pidfd"
author: "By Jonathan Corbet May 31, 2024"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 31, 2024

The "pidfdfs" virtual filesystem was added to the 6.9 kernel release as a way to export better information about running processes to user space. It replaced a previous implementation in a way that was, on its surface, fully compatible while adding a number of new capabilities. This transition, which was intended to be entirely invisible to existing applications, already [ran into trouble](<https://lwn.net/Articles/963749/>) in March, when a misunderstanding with SELinux caused systems with pidfdfs to fail to boot properly. That problem was quickly fixed, but it turns out that there was one more surprise in store, showing just how hard ABI compatibility can be at times. 

A pidfd is a file descriptor that identifies a running process. Within the kernel, it must have all of the data structures that normally go along with file descriptors so that kernel subsystems know what to do with it. The kernel has, since the 2.6.22 release in 2007, had a small helper mechanism providing anonymous inodes to back up file descriptors on virtual filesystems that do not have a real file behind them. When the [pidfd abstraction](<https://lwn.net/Articles/773459/>) was added to the 5.1 kernel, it was naturally implemented using anonymous inodes, and all worked as intended. 

Eventually, though, the limitations of this implementation began to make themselves felt. In response, Christian Brauner reworked the implementation away from anonymous inodes, creating the separate pidfdfs filesystem. The new filesystem supported the use of [`statx()`](<https://man7.org/linux/man-pages/man2/statx.2.html>) on a pidfd; that, in turn, made real inode numbers available to user space that could be used to compare two pidfds for equality. Added functionality (such as killing a process when the last pidfd referring to it is closed) became possible. The new implementation also gave security modules a say in pidfd operations; this was the source of the first set of problems but, in the longer term, will help administrators with the control of their system. 

Once the security-module problem was worked out, it seemed like the pidfdfs problems had been taken care of. But then, Jiri Slaby [reported](<https://lwn.net/ml/linux-fsdevel/210098f9-1e71-48c9-be08-7e8074ec33c1@kernel.org>) that kernels with pidfdfs broke both the [util-linux](<https://github.com/util-linux/util-linux>) test suite and the [`lsof`](<https://man7.org/linux/man-pages/man8/lsof.8.html>) utility. Brauner [answered](<https://lwn.net/ml/linux-fsdevel/20240515-anklopfen-ausgleichen-0d7c220b16f4@brauner>) that the util-linux problem had been fixed upstream, but that `lsof` was a surprise. It turns out that there were two problems in need of solving, one rather more predictable than the other. 

In pre-6.9 systems, "`ls -l`" or [`lstat()`](<https://man7.org/linux/man-pages/man2/lstat.2.html>) would show a pidfd as a symbolic link to the string "`anon_inode:[pidfd]`". As of 6.9, instead, the result would be "`pidfd:[_inode_]`", showing the inode number assigned to the pidfd. Since `lsof` was looking for the pre-6.9 version of the string, it failed to recognize or do the right thing with pidfds. 

But, it turns out, there is more. When the anonymous-inode code was added, the author never bothered (or simply forgot) to set the file-type field in each inode as it was created. As a result, system calls like [`stat()`](<https://man7.org/linux/man-pages/man2/stat.2.html>) will report the file type as zero, which is not actually a defined file type. That would cause command-line tools like [`stat`](<https://man7.org/linux/man-pages/man1/stat.1.html>) to describe the result as type as ""weird file"", which was objectively true. This little quirk never created trouble for any tools that actually worked with files backed by anonymous inodes, so it never was fixed. 

It was noticed, though. Somebody working on `lsof` cleverly realized that a file type of zero was a convenient way to recognize anonymous-inode files. So `lsof` acquired a test for that condition, taking advantage of an ABI quirk that was never intended, much less documented. Once a `stat()` call on a pidfd started returning a proper file type, `lsof` no longer recognized the file and got hopelessly confused. Linus Torvalds [was unimpressed](<https://lwn.net/ml/linux-fsdevel/CAHk-=whMVsvYD4-OZx20ZR6zkOPoeMckxETxtqeJP2AAhd=Lcg@mail.gmail.com>) when all of this became clear: 

> What a crock. That's horrible, and we apparently never noticed how broken anon_inodes were because nobody really cared. But then lsof seems to have done the *opposite* and just said (for unfathomable reasons) "this can't be a normal regular file". 

That said, he also allowed that ""we probably just have to live in the bed we made""; breaking `lsof` was a user-space regression in need of fixing. 

Brauner put together a patch that was [merged](<https://git.kernel.org/linus/db3d841ac9ed>) prior to the 6.10-rc1 release; it has not yet found its way into the 6.9 stable updates. The patch restored the older output format for pidfds and caused the file-type field to be explicitly masked to zero, restoring the previous behavior. With that fix, `lsof` works again, and people are mostly happy. 

When he [sent the patch](<https://lwn.net/ml/linux-fsdevel/20240521-girlanden-zehnfach-1bff7eb9218c@brauner>), though, Brauner said that he ""would like to try to move away from the current insanity"" in the near future. He hopes that `lsof` will be fixed to be able to handle the newer output format, and that it will be possible to remove this compatibility hack. Torvalds [seems willing to try](<https://lwn.net/ml/linux-fsdevel/CAHk-=wgt2W6jmfCc9FPB+WC09Cqo4YTmwyAeCQq6Mxkx3EjACQ@mail.gmail.com>), but he pointed out that some users (and their distributions) can be quite slow to update their user-space tools, so it may be a long time before this change is no longer needed. 

In summary: [Hyrum's law](<https://www.hyrumslaw.com/>) has shown its applicability yet again. Leaving zero in the type field was never meant to be a part of the ABI for anonymous inodes; it is just a bug, an artifact of a job that was not completely done. But, since that behavior was visible, code came to depend on it, and the bug can no longer be fixed. This episode is another hint that kernel interfaces could benefit from a higher level of scrutiny than they typically get before showing up in a released kernel.
