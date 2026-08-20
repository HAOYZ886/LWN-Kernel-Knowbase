---
title: A kernel change breaks GlusterFS
url: https://lwn.net/Articles/544298/
date: "March 27, 2013"
category: "Filesystems-ext4; User-space API"
author: "By Michael Kerrisk March 27, 2013"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Michael Kerrisk**  
March 27, 2013

Linus Torvalds has railed frequently and loudly against kernel developers breaking user space. But that rule is not ironclad; there are exceptions. As Linus once [noted](<http://thread.gmane.org/gmane.linux.kernel/1063039/focus=1065522>): 

But the "out" to that rule is that "if nobody notices, it's not broken" […] So breaking user space is a bit like trees falling in the forest. If there's nobody around to see it, did it really break? 

The story of how a kernel change caused a GlusterFS breakage shows that there are sometimes unfortunate twists to those exceptions. 

#### The kernel change and its consequences

[GlusterFS](<http://www.gluster.org/>) is a widely-used, free, [scale-out](<http://en.wikipedia.org/wiki/Scalability#scale_out>), distributed filesystem that is available on Linux and a number of other UNIX-like systems. GlusterFS was initially developed by Gluster, Inc., but since Red Hat acquired that company in 2011, it has mainly driven work on the filesystem. 

GlusterFS's problems sprang from an ext4 filesystem [patch](<http://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/commit/?id=d1f5273e9adb40724a85272f248f210dc4ce919a>) by Fan Yong that addressed a long-standing issue in ext4's support for the `readdir()` API by widening the "directory offset" values used by the API from 32 to 64 bits. That change was needed to reliably support `readdir()` traversals in large directories; we'll discuss those changes and the reasons for making them in [a companion article](<https://lwn.net/Articles/544520/>). One point from that discussion is worth making here: these "offset" values are in truth a kind of cookie, rather than a true offset within a directory. Thus, for the remainder of this article, we'll generally refer to them as "cookies". Fan's patch made its way into the mainline 3.4 kernel (released in May 2012), but appears also to have been ported into the 3.3.x kernel that was released with Fedora 17 (also released in May 2012). 

Fan's patch solved a problem for ext4, but inadvertently created one for GlusterFS servers that use ext4 as their underlying storage mechanism. However, nobody reported problems in time to cause the patch to be reconsidered. The symptom on affected systems, as noted in a July 2012 [Red Hat bug report](<https://bugzilla.redhat.com/show_bug.cgi?id=838784>), was that using `readdir()` to scan a directory on a GlusterFS system would end up in an infinite loop in some cases. 

The cause of the problem--as [detailed](<https://lwn.net/Articles/544254/>) by Anand Avati in a recent (March 2013) discussion on the ext4 mailing list--is that GlusterFS makes some assumptions about the "cookies" used by the `readdir()` API. In particular, although these values are 64 bits long, the GlusterFS developers noted that only the lower 32 bits were used, and so decided to encode some additional information--namely the index of the Gluster server holding the file--inside their own internal version of the cookie, according to this formula: 

```
final_d_off = (ext4_d_off * MAX_SERVERS) + server_idx
```

This GlusterFS internal cookie is exchanged in the 64-bit cookie that is passed in NFSv3 `readdir()` requests between GlusterFS clients and front-end servers. (An [ASCII art diagram](<https://lwn.net/Articles/544389/>) posted in the mailing list thread by J. Bruce Fields clarifies the relationship of the various GlusterFS components.) The GlusterFS internal cookie allows the server to easily encode the identify of the GlusterFS storage server that holds a particular directory. This scheme worked fine as long as only 32 bits were used in the ext4 `readdir()` cookies (`ext4_d_off`), but promptly blew up when the cookies switched to using 64 bits, since the multiplication caused some bits to be lost from the top end of `ext4_d_off`. 

An August 2012 gluster.org [blog post](<http://www.gluster.org/2012/08/glusterfs-bit-by-ext4-structure-change/>) by Joe Julian pointed out that the problem affected not only Fedora 17's 3.3 kernel, but also the kernel in Red Hat's Enterprise Linux distribution, because the kernel change had been backported into the much older 2.6.32 distribution kernel supplied in RHEL 6.3 and later. The recommended workaround was either to downgrade to an earlier kernel version that did not include the patch or to reformat the GlusterFS bricks (the fundamental storage unit on a GlusterFS node) to use XFS instead of ext4. (Using XFS rather than ext4 had already been recommended practice when using GlusterFS.) Needless to say, neither of these solutions was easily practicable for some GlusterFS users. 

#### Mitigating GlusterFS's problem

In his March 2013 mail, Anand bemoaned the fact that the manual pages gave no indication that the `readdir()` API "offsets" were cookies rather than something like a conventional file offset whose range might bounded. Indeed, the manual pages rather hinted towards the latter interpretation. (That, at least, is a problem that is now [addressed](<http://git.kernel.org/cgit/docs/man-pages/man-pages.git/commit/?id=73f4bf1eab4b4bd2adf9a37908b6e14a0a02fb6c>).) Anand went on to request a fix to the problem: 

You can always say "this is your fault" for interpreting the man pages differently and punish us by leaving things as they are (and unfortunately a big chunk of users who want both ext4 and gluster jeopardized). Or you can be kind, generous and be considerate to the legacy apps and users (of which gluster is only a subset) and only provide a mount option to control the large d_off behavior. 

But, as the ext4 maintainer, Ted Ts'o, [noted](<https://lwn.net/Articles/544257/>), Fan's patch addressed a real problem that affected well-behaved applications that did not make mistaken assumptions about the value returned by `telldir()`. Adding a mount option that nullified the effect of that patch would affect all programs using a filesystem and penalize those well-behaved applications by exposing them to the problem that the patch was designed to fix. 

Ted instead proposed another approach: a per-process setting that allowed an application to request the older `readdir()` cookie semantics. The advantage of that approach is that it provides a solution for applications that misuse the cookie without penalizing applications that do the right thing. This solution could, [he said](<https://lwn.net/Articles/544259/>), take the form of an ext4-specific `ioctl()` operation employed immediately after calling `opendir()`. Anand [thought](<https://lwn.net/Articles/544260/>) that should be a workable solution for GlusterFS. The requisite patch does not yet seem to have appeared, but one supposes that it will be written and submitted during the 3.10 merge window, and possibly backported into earlier stable kernels. 

So, a year after the ext4 kernel change broke GlusterFS, it seems that a (kernel) solution will be found to address GlusterFS's difficulties. In passing, it's probably fair to mention that one reason that the (proposed) fix took so long in coming was that the GlusterFS developers initially thought they might be able to work around the kernel change by making changes in GlusterFS. However, it ultimately [turned out](<https://bugzilla.redhat.com/show_bug.cgi?id=838784#c11>) to be impossible to exchange both a full 64-bit `readdir()` cookie and a GlusterFS storage server ID in the NFS `readdir()` requests exchanged between GlusterFS clients and front-end servers. 

#### Summary: the meta-problem

In the end, the GlusterFS breakage might have been avoided. Ted's proposed fix could have been rolled out at the same time as Fan's patch, so as to minimize any disruptions for GlusterFS users. Returning to Linus's quote at the beginning of this article puts us on the trail of a deeper problem. 

""If there's nobody around to see it, did it really break?"" was Linus's rhetorical question. The problem is that this is a test whose results can be rather arbitrary. Sometimes, [as was the case in the implementation of `EPOLLWAKEUP`](<https://lwn.net/Articles/520198/#EPOLLWAKEUP>), a kernel change that causes a minor breakage in a user-space application that is doing strange things will be reverted or modified because it is fortuitously spotted by someone close to the development scene--namely, a kernel developer who notices a misbehavior on their desktop system. 

However, other users may be so far from the scene of change that it can be a considerable time before they see a problem. By the time those users detect a user-space breakage, the corresponding stable kernel may already be several release cycles in the past. One can easily imagine that few kernel developers are running a GlusterFS node on their development systems. Conversely, one can imagine that most users of GlusterFS are running production environments where stability and uptime are critical, and testing an -rc kernel is neither practical nor a high priority. 

Thus, a rather important user-space breakage was missed--one that, if it had been detected, would almost certainly have triggered modification or reversion of the relevant patches, or stern words from Linus in the face of any resistance to making such changes. And, certainly, this is not a one-off case. Your editor did not need to look too far to find another example, where [a change in the way that POSIX message queue limits are enforced](<https://lwn.net/Articles/544264/>) in Linux 3.5 led to a [report](<https://bugs.launchpad.net/ubuntu/+source/manpages/+bug/1155695>) of breakage in a database engine nine months later. 

The "if there's nobody around to see it" metric requires that someone is looking. That is of course a strong argument that the developers of user-space applications such as GlusterFS who want to ensure that their applications keep working on newer kernels must vigilantly and thoroughly test -rc kernels. Clearly that did not happen. 

However, it seems a little unfair to place the blame solely on user space. The ext4 modifications that affected GlusterFS clearly represented a change to the kernel-user-space ABI (and for reasons that we describe in our follow-up article, that change was clearly necessary). In cases such as this (and the POSIX message queue change), perhaps even more caution was warranted when making the change. At the very least, a loud announcement in the commit message that the kernel changes represented a change to the ABI would have been helpful; that might have jogged some reviewers to think about the possible implications and resulted in the ext4 changes being made in a way that minimized problems for GlusterFS. A greater commitment on both sides to improving the documentation would also be helpful. It's notable that even after deficiencies in the documentation were mentioned as a contributing factor to GlusterFS problem, no-one sent a patch to improve said documentation. All in all, it seems that parties on both sides of the ABI could be doing a better job.
