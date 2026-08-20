---
title: Introducing lazytime
url: https://lwn.net/Articles/621046/
date: "November 19, 2014"
category: "Filesystems-Access-time tracking"
author: "By Jonathan Corbet November 19, 2014"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
November 19, 2014

POSIX-compliant filesystems maintain three timestamps for every file, corresponding to the times of the last change in the file's metadata or contents (known as its "ctime"), modification of the file's contents ("mtime"), and access of the file ("atime"). The first two timestamps are generally considered to be useful, but "atime" has long been seen as being too expensive for the benefits it provides. In current systems, there is a mount option (called "relatime") that mitigates the worst problems caused by atime, but it has a few issues of its own. Now a new "lazytime" option might replace relatime and provide the best of all worlds. 

The problem with atime is that it is supposed to be updated every time the associated file is accessed. Updating atime requires writing the file's inode back to disk, so atime tracking essentially turns every read operation into a write. For many workloads, the effect on performance can be severe. On top of that, there are few programs out there that make use of atime or depend on it being updated. So, ten years ago, it was common to mount filesystems with the "noatime" option, which disabled the tracking of access times entirely. 

The problem, of course, is that "few programs" is not the same as "no programs"; it turns out that there are indeed utilities that break in the absence of atime tracking. A classic example is mail clients that use atime to determine whether a mailbox has been read since mail was last delivered to it. After some discussion, the kernel community added the "relatime" mount option in the 2.6.20 development cycle. Relatime will cause most atime updates to be suppressed, but it will allow an atime update if the current recorded atime is prior to the current ctime or mtime. Later on, relatime was tweaked to update atime once every 24 hours regardless (but only if the file is accessed, of course). 

Relatime works well enough for most systems, but there are still those who would like better atime tracking without paying the performance penalty for it. Some users also dislike the fact that relatime, for all its value, causes the system to not be fully compliant with the POSIX specification. For the most part, people have put up with the minimal deficiencies in relatime (or put up with the cost of atime updates), but there is now an alternative on the horizon. 

That alternative takes the form of the [lazytime mount option](<https://lwn.net/Articles/620086/>), posted as an ext4-specific patch by Ted Ts'o. With lazytime enabled, a filesystem will keep atime current in a file's in-memory inode. But that inode will not be written to disk until there is some other reason to do so, or until the inode itself is being pushed out of memory. The effect will to have an atime that is always correct from the point of view of any program running on the system. The version of atime stored on disk may well lag significantly behind reality, though, and the current atime could be lost if the system were to crash. 

Dave Chinner was quick to [point out](<https://lwn.net/Articles/621067/>) that, while the option looked like it could be useful, the ext4 filesystem might not be the best place to implement it. If lazytime were to be implemented in the virtual filesystem (VFS) layer, then it would be available for all filesystems, not just ext4 and, perhaps most importantly, it would work the same way on all of them. Ted [agreed](<https://lwn.net/Articles/621068/>) that a VFS implementation might make sense; the next version of this patch seems almost certain to be implemented at that level. 

Dave also [suggested](<https://lwn.net/Articles/621071/>) that delaying the writing of atime updates indefinitely might not be advisable: 

However, we'd be fools to ignore the development of relatime, which in it's original form never updated the atime until m/ctime updated. 3 years after it was introduced, relatime was changed to limit the update delay to 24 hours (before it was made the default) so that applications that required regular updates to timestamps didn't silently stop working. 

Once again, Ted was amenable to this idea, so the next version will probably write out updated atime values a minimum of once every 24 hours. Without that change, atime updates could be held in memory for months at a time on a system like a database server (which keeps its files open indefinitely). 

Finally, there is the question of whether lazytime should become the default mount option. It satisfies POSIX (or, at least, will after another fix or two) without incurring the cost of normal atime updates, so it does seem like a better option than relatime, which is the current default. Ted, seemingly, would like to change the default in the near future, while Dave is a bit more concerned about regressions and would like to wait a couple of years to see how things work out. That led to a question of whether the feature will see enough testing in the meantime, but, as Dave noted, there will probably be enough interest in the feature to ensure that people will try it out. 

Whether that is true remains to be seen; relatime works well enough for most users, so there isn't necessarily a crowd of people looking to try a new filesystem mount option. But eventually some of the more adventurous distributions are likely to pick it up; at that point, any latent problems should probably come out before too long. So, when lazytime becomes the default in 2016 or so, it should indeed be well tested and shown to work without problems.
