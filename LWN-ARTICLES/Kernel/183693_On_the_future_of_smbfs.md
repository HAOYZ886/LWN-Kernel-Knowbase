---
title: On the future of smbfs
url: https://lwn.net/Articles/183693/
date: "May 15, 2006"
category: CIFS; smbfs
author: 
---

The venerable smbfs code allows Linux systems to mount filesystems exported via the SMB protocol. It thus can be used for accessing files exported from a Windows system. This filesystem has seen a lot of use over the years, but has, in recent times, been overtaken by the newer CIFS filesystem. At this point, CIFS receives almost all of the developer attention, and most users have (or, at least, should have) moved over. 

As an example of the difference in how smbfs and CIFS are maintained, consider the [2.6.16.11](<http://lwn.net/Articles/181190/>) stable kernel update, which contained a fix for a security problem in the CIFS code. Though CIFS has its roots in smbfs, nobody was paying enough attention to realize that smbfs might suffer from the same vulnerability. Thus, while 2.6.16.11 fixed the CIFS problem on April 24, the matching smbfs fix (which forced [2.6.16.14](<http://lwn.net/Articles/182631/>)), did not appear until May 4, eleven days later. In the mean time, smbfs was vulnerable to a known bug, for anybody who thought to look for it. 

The 2.6.17-rc4-mm1 kernel recognizes the unmaintained nature of smbfs with a patch marking it as being deprecated and slated for eventual removal. All remaining users are encouraged to move over to the CIFS implementation instead. For some users, the end has come sooner - the Fedora Core 5 kernel already [does not support smbfs](<https://lwn.net/Articles/183704/>). Since there is an alternative in the kernel and ready to go, this migration should not be a big problem. 

It is a nice scenario, but there is one little problem: the CIFS code cannot work with Windows 95 and Windows 98 systems. Without smbfs, Linux users will not be able to mount shares exported from hosts running those old versions of Windows. Some observers have commented that those versions of Windows are too old to support, but Linus [isn't buying it](<https://lwn.net/Articles/183706/>): 

But we do _not_ drop features just because they are deemed "unnecessary". As long as somebody actually _uses_ smbfs, and as long as those users are willing to test and perhaps send in patches for when/if it breaks, we should not drop it. 

The word from Andrew Morton is that Windows 9x support for CIFS is in the works, and should, with luck, by ready in time to go into 2.6.18. If things happen that way, then the 2.6.18 kernel might just include a deprecation notice for smbfs, and smbfs could be marked "broken" by the end of the year. Anybody still using smbfs should consider themselves warned.
