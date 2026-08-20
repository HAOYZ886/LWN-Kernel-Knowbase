---
title: Once upon atime
url: https://lwn.net/Articles/244829/
date: "August 8, 2007"
category: "Filesystems-Access-time tracking"
author: "By Jonathan Corbet August 8, 2007"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
August 8, 2007

Among the metadata maintained by most filesystems is the last file access time, or "atime." This time can be a useful value to have - it lets an administrator (or a program) know when a file was last used. There is a strong downside to this feature, though: it forces a write to the disk every time a file is accessed. So read-only operations, which might have been satisfied entirely from cache, turn into filesystem writes to keep the atime value up to date. 

A recent discussion on write throttling turned to atime after Ingo Molnar pointed out that atime was probably a bigger performance problem than just about everything else. He [went on to say](<https://lwn.net/Articles/244830/>): 

Atime updates are by far the biggest IO performance deficiency that Linux has today. Getting rid of atime updates would give us more everyday Linux performance than all the pagecache speedups of the past 10 years, _combined_. 

He also claimed that it was ""perhaps the most stupid Unix design idea of all times"". 

Such discussion leads quickly to the question of what should be done about this old situation. One step that any Linux user can take now is to mount filesystems with the `noatime` option, which turns off the tracking of access times. For filesystem-intensive tasks, the performance reward can be immediately apparent. Unfortunately, turning off atime unconditionally will occasionally break software. Some mail tools will compare modification and access times to determine whether there is unread mail or not. The `tmpwatch` utility and some backup tools also use atime and can misbehave if atime is not correct. For this reason, distributors tend not to make `noatime` the default on installed systems. 

Another approach was added in 2.6.20: the `relatime` mount option. If this flag is set, access times are only updated if they are (before the update) earlier than the modification time. This change allows utilities to see if the current version of a file has been read, but still cuts down significantly on atime updates. This option is not heavily used, perhaps because few people have heard of it and many distributions lack a version of `mount` which is new enough to know about it. Using `relatime` can still confuse tools which want to ask questions like "has this file been accessed in the last week?" 

To fix that problem, Linus [suggested](<https://lwn.net/Articles/244831/>) a tweak to how `relatime` works: update it if the current value is more than a certain time in the past - one day, for example. Ingo responded with [a patch](<http://lwn.net/Articles/244384/>) implementing that behavior and adding a couple of new boot options: `relatime_interval`, which specifies the update interval in seconds, and `default_relatime`, which turns on the `relatime` option in all filesystems by default. 

Something resembling this version of the patch might go into 2.6.24. It was suggested that, whenever a file's inode is to be written to disk anyway, the kernel might as well update atime as well. Alan Cox objected that this change might make the overall behavior less predictable, which might not be desirable. No new version of the patch with this feature has been posted, so chances are it will not be in the version which gets merged \- if and when that happens.
